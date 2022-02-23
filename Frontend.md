# Next page structure

- File structure

In Nextjs,the pages folder has to follow a particular structure which relates to the url. Consider the following file structure:

V Pages
    V [category]
        V [subcategory]
        [...productSlug].jsx
        index.jsx   <-- Page for subcategory
    index.jsx       <-- Page for category
index.jsx           <-- Page for home (/)
about.jsx
contact.jsx

When the user visits /contact or /about, the page for contact.jsx or about.jsx will; be loaded respectfully. If you name a file or folder with square brackets, it will treat it as a dynamic query string in the url. 
What this means is that if you visit /confectionary, the page will load the second index.jsx as shown above. 
If the user visits /confectionary/biscuits, the page will load the first index.jsx. 
Finally, if the user visits /confectionary/biscuits/twix, the page will load [...productSlug].jsx

The query string will then be available in the page for GET requests as shown below

- Pages

```js
import React, { useEffect } from "react";
import HomeContainer from "../containers/Home";
import client from "../client";
import { gql } from "@apollo/client";

import React from "react";
import ProductsContainer from "../../containers/Products";
import client from "../../client";
import { gql } from "@apollo/client";

const CategoryPage = (props) => {
  return <ProductsContainer {...props} />;
};

export async function getServerSideProps(context) {
  const { page, perPage, sort, category } = context.query;

  const {
    data: catData,
    loading: catLoading,
    error: catError,
  } = await client.query({
    query: gql`
      query Query($filter: FilterFindOneCategoryInput) {
        categoryOne(filter: $filter) {
          _id
        }
      }
    `,
    fetchPolicy: "network-only",
    variables: {
      filter: {
        slug: category,
      },
    },
  });

  return {
    props: {
      level: "category",
      catId: catData?.categoryOne?._id,
      page: page ?? 1,
      perPage: perPage ?? 12,
      sort: sort ?? "UPDATEDAT_DESC",
    },
  };
}

export default CategoryPage;
```
The page file uses a higher order component(HomePage) to return the container (HomeContainer). 
Everything that happens in this file happens client side which means that any console.logs or network errors will go to the terminal in VSCode rather than in the inspector in the browser.
Next comes with certain functions that you can use in the page to handle GET requests. One of them is getServerSideProps which you can use to get data, put them into the props and then pass those props to the container.
In the page, you can access the query string that you can use to make queries. In the example above, the category slug from the url is taken from the context and used as a variable in the graphql query.

# On scroll effects

```js
import React, { useEffect, useState } from "react";
import { Box } from "@chakra-ui/react";
import Header from "./Header";
import Footer from "./Footer";
import { throttle } from "lodash";
import Meta from "../Meta";

const PageContainer = ({ children, seoTitle, seoDescription, loading }) => {
  const [depth, setDepth] = useState("shallow");
  const checkPosition = () => {
    const verticalPos = window.scrollY;
    if (verticalPos < 100 && depth !== "shallow") {
      setDepth("shallow");
      return null;
    } else if (verticalPos >= 140 && verticalPos < 600 && depth !== "medium") {
      setDepth("medium");
      return null;
    } else if (verticalPos >= 640 && depth !== "deep") {
      setDepth("deep");
    }
    return null;
  };
  useEffect(() => {
    window.onscroll = throttle(checkPosition, 500);
  });
  return (
    <Box position="relative">
      <Meta title={seoTitle} description={seoDescription} />
      <Box position="sticky" top="0px" zIndex="6">
        <Header depth={depth} loading={loading} />
      </Box>
      <Box zIndex="3">{children}</Box>
      <Footer />
    </Box>
  );
};

export default PageContainer;
```

For this page there is a function (checkPosition) that checks the Y level for the site depending on where the scroll position is. In this example, it sets the state to either 'shallow', 'medium' or 'deep' which the site uses to animate components in and out of view in the header.

Whenever the user scrolls, the window.onscroll function will fire. This then invokes checkPosition via a throttle function from lodash. The throttle function will ensure the function only fires once per 500ms. This is to help performance as the function would be invoked too much and too fast without the throttle function as it will be invoked everytime the scroll poisition changes.

# useDidMountEffect

- function

```js
const useDidMountEffect = (func, deps) => {
  const didMount = useRef(false);
  useEffect(() => {
    if (didMount.current) func();
    else didMount.current = true;
  }, deps);
};
```
This is a custom hook that you can use inplace of a normal useEffect. Unlike a useEffect, this will not fire upon the first render of the component.

- usage
```js
  useDidMountEffect(() => {
    changeQuantity({ variables: { productId: item._id, quantity: quantity } });
  }, [quantity]);
```
Here, useDidMountEffect is used instead of a useEffect as the mutation would always fire when the component first loads for useEffect. You can still use the custom hook in exactly the same way as you'd use a normal useEffect

