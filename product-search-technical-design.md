# Product Search Technical Design

After getting a feel for the overall functional requirements of a feature, it is usually a good idea to start thinking through the technical design at a high level.

## Planning the Architecture

It's always a good idea to come up with several different ways a feature can be implemented, think through the pros and cons of each approach, and then select the design that makes the most sense for the goals of the project. While the product search feature we're implementing will be pretty simple, there are many possible ways to implement it. Here are a couple of possibilities that come to mind:

* Call a service to download all products and perform search queries entirely on the client
* Call a service with a search query and return a list of products with names containing the search query

With the first approach, the user will have to wait while the app downloads all of the products, but then searches will all be fast from that point. With the second approach the user would have to wait for the server to return the search results for each search. Given that the product library could be absolutely huge in a real e-commerce system, the amount of time required to download the entire product library would likely not justify the improved search performance. So, it seems that the most reasonable approach would be the second one.

## Data modeling

Now that we have an idea of how our service calls for this feature will look, let's work through our front-end data model. In Front-End First Development, this is a critical step. The goal is to think through what data will be displayed in the front-end, what data is required from the back-end, and what data can be derived on the front-end. To ensure the best performance, minimizing the amount of data being transferred between the front-end and the back-end should always be a consideration. For example, it is often acceptable to cache data on the client for a period of time, as we briefly considered in the first approach above. However, we need to carefully consider what data the user needs, how fresh that data needs to be, and what the typical use cases will be.

We previously decided that we will download the search results every time the user searches. Let's say that our product search results will show the product name, description, price, and a picture for each product. We will also need a product id of some sort. So, a TypeScript interface of our product data model could look something like this:

```javascript
export interface IProduct {
  id: number;
  name: string;
  description: string;
  price: number;
  image: string;
}
```

So, when we call the service, we could pass a string with the search query, and the service would then return an array of products (`IProduct[]`). Once we have downloaded the list of products from the server, the client-side would then render the products in React. Once we implement product search on the front-end and are satisfied with the results, it will then be a matter of implementing the service operation on the back-end that will accept a search query, perhaps use that search query as part of a database query, and then return the results in JSON that matches our the `IProduct[]` that the client-side is expecting. As we will see later in the tutorial when we start using I/O Source, we will be able to quickly implement a mock service with a set of mock product to data allow product searches to be performed entirely in the client-side, and then switch over to start calling the real back-end product search with no changes to our front-end logic.

# Up Next

We have considered some different designs for implemented product search and have thought through the front-end data model. We are now ready to begin implementing the product search functionality. To get a feel for how a client-rendered product search feature works with React with the simplest possible implementation, let's start by building the feature in [only React](./implementing-product-search-with-just-react.md) without using other libraries like Redux or I/O Source.