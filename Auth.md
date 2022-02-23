# Log in mutation with graphQL compose

- the mutation

```js
    login: {
        description: "user login",
        type: UserTC,
        args: {email: "String!", password: "String!"},
        resolve: async (parent, {email, password}, context, info) => {
            // applies some basic validation to the inputted email and password
            validateLogin(email, password)
            //retrieves the user model using the email
            const user = await UserModel.findOne({email})
            if(!user) {
                throw new Error("User not found")
            }
            const isValid = await bcrypt.compare(password, user.password)
            if(!isValid) {
                throw new Error("Wrong password")
            }
            // create token and code store the email and id inside it
            user.token = jwt.sign({
                id: user.id,
                email: user.email
            },
            process.env.SECRET_KEY,
            { expiresIn: "10d" }
            )
            //return the user
            return user
        }
    }
```
Here, a normal mongoose resolver is used. The second argument to the resolver function are always the variables that are passed in to the mutation, in this case email and password. First, we get the user model using the email the user entered and throw and an error if it isn't found. Then we check the password using bcrypt.
bcrypt is used to encode the inputted password by the user and then compare that to the already encrypted password in the database.
Then we add a token property to the user model and use jwt to create it. You also need to use the secret key which should be in the env


- the model

```js
const UserSchema = new mongoose.Schema({
    firstname: {
        type: String,
        index: true
    },
    surname: {
        type: String,
        index: true
    },
    email: String,
    password: String,
    token: String,
    branch: {
        type: mongoose.Schema.Types.ObjectId,
        ref: "Branch"
    }
})
//create the user model with mongoose
export const UserModel = mongoose.model("user", UserSchema)

//create the user tc with graphql-compose-mongoose
export const UserTC = composeMongoose(UserModel, {})
```
The schema (UserSchema) is created with mongoose, the model (UserModel) is then made with mongoose and then finally the UserTC is created with graphql-compose-mongoose. You can then add mutations to the UserTC

- validate login util function

```js
import { UserInputError } from "apollo-server-express"

const validateLogin = (email, password) => {
    if(email.trim() === "") {
        throw new UserInputError("Your email cannot be blank")
    }
    if(password === "") {
        throw new UserInputError("Your password cannot be blank")
    }
}

export default validateLogin
```

# Auth provider

```js
import React, { useEffect, useContext } from "react";
import Context from "../context";
import { useRouter } from "next/router";
import PageWithLoader from "./PageWithLoader";

const AuthProvider = (WrappedComponent, isRequired) => {
  return function ComponentWithAuth(props) {
    const contextValue = useContext(Context);
    const { auth, authLoading } = contextValue;
    const router = useRouter();
    useEffect(() => {
      if (isRequired && !authLoading && !auth)
        router.push(`/login?prev=${router.asPath}`);
      if (router.pathname === "/login" && !!auth) {
        router.push(router.query.prev);
      }
    }, [auth]);

    return (
      <>
        {isRequired && !auth ? (
          <PageWithLoader />
        ) : (
          <WrappedComponent {...props} />
        )}
      </>
    );
  };
};

export default AuthProvider;
```

This is a higher order component which means it takes a component as an argument and returns a new component. The WrappedComponent is the original component that is passed in which with next, will be the entire container for the page. The isRequired param is a boolean which will be used to determine whether users will need to be logged in to access the page. If set to true, the user will need to be logged in to access it.

First, a new component, ComponentWithAuth, is returned from the AuthProvider. Then the token is obtained, from context or recoil. If the user is not logged in (there is no auth), the user is redirected to `/login?prev=${router.asPath}`. The prev query string enables the user to be sent back to the page they were on after they log in. Once the user has logged in, they will now have an auth which will trigger the useEffect. This time, the second if statement will fire and will redirect the user from the login page to router.query.prev which is the query string stored in the url from the redirect before.

Finally, the WrappedComponent is returned

PageWithLoader is just a blank container with a header, footer and spinner

# Loggin in (frontend graphql)

- Form in frontend

```js
import { Box, Flex, Text, Input, Button, useToast } from "@chakra-ui/react";
import { useForm } from "react-hook-form";
import Context from "./context";
import React, { useState, useEffect } from "react";
import Cookie from "js-cookie";
import { useMutation, useLazyQuery, gql } from "@apollo/client";
import jwtDecode from "jwt-decode";
import { useRouter } from "next/router";

const LoginForm = ({ setForgot }) => {
  const toast = useToast();
  const router = useRouter();
  const contextValue = React.useContext(Context);
  const { auth, setAuth, setBasket, basket, setAuthLoading } = contextValue;
  const { register, handleSubmit } = useForm();
  const [values, setValues] = useState({});

  const LOGIN = gql`
  mutation login($email: String!, $password: String!) {
    login(email: $email, password: $password) {
      email
      token
      accountStatus {
        name
      }
    }
  }
`;

  const [doLogin, { loading }] = useMutation(LOGIN);

  const doSubmit = (data) => {
    setValues(data);
    doLogin({
    variables: values,
    update(proxy, result) {
      const {
        data: { login: userData },
      } = result;
      setAuthLoading(false);
      const decoded = jwtDecode(userData.token);
      Cookie.set("token", result.data.login.token);
      setAuth(decoded);
      router.push("/");
    },
    onError(err) {
      console.log(err, "<-- err");
      toast({
        title: "Error",
        description: err.graphQLErrors?.[0]?.message,
        status: "error",
        duration: 5000,
        isClosable: false,
      });
    },
  });
  };
  return (
    <Box>
      <Text>
        Sign In
      </Text>
      <Box>
        <Text>
          Email
        </Text>
        <form onSubmit={handleSubmit(doSubmit)} id="loginForm">
          <Input
            id="loginEmailField"
            {...register("email")}
          />
          <Text>
            Password
          </Text>
          <Input
            id="loginPasswordField"
            {...register("password")}
            type="password"
          />
          <Button type="submit">
            <Text>SIGN IN</Text>
          </Button>
        </form>
        <Text
          _hover={{ cursor: "pointer" }}
          onClick={() => setForgot(true)}
        >
          Forgot Password?
        </Text>
      </Box>
    </Box>
  );
};
export default LoginForm;
```
This uses react hook forms. The mutation is done by first creating a function (doLogin) with useMutation from Apollo using a login query string (LOGIN). The mutation is then used in the doSubmit function with the variables passed in from the values hook. You can then use the update function to set the context with the token from the returned value of the mutation. There is also an onError function you can use to handle errors