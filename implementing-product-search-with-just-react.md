# Using Only React

Now that we've defined what our products look like from a data perspective and also have an idea of how the overall flow for how product data will be retrieved, let's try implementing it as simply as we can. In our first attempt, let's try implementing the search functionality using only React.

First, let's create a simple React component to show our search results. We're not concerned about styling in this tutorial, so let's just render the basic data for a product:

```javascript
// ProductSearchResults.tsx

import * as React from 'react';

interface ISearchResultsProps {
  searchResults: IProduct[];
}

export default (props: ISearchResultsProps) => (
  <div>
    {props.searchResults.map(product => (
      <div>
        <span>{product.image}</span>
        <span>{product.name}</span>
        <span>{product.price}</span>
        <span>{product.description}</span>          
      </div>
    ))}
  <div>
);
```

This React component is pretty simple: it takes in an array of products and renders some simple HTML for each product. Now let's create a product search page that implements the actual search functionality and leverages this component to show products. Since this React component is going to have state and other code that does more than just rendering markup, we will use a React class instead of a stateless functional component for our overall product search page. Let's start with the basics:

```javascript
// ProductSearchPage.tsx

import * as React from 'react';

import ProductSearchResults from './ProductSearchResults';

export interface IProductSearchPageState {
  searchResults: IProduct[];
};

export default class extends React.Component<any, IProductSearchState) {
  constructor(props: any, context: any) {
    super(props, context);

    this.state = {
      searchResults: []
    };
  }

  public render() {
    return (
      <div>
        <ProductSearchResults searchResults={this.state.searchResults} />
      </div>
    );
  }
};
```

This component has state that represents search results as a list of the products. In its render method, it uses the stateless functional component we defined earlier to render the list of products currently in the state. When this component gets initialized, the search results will be empty, and no products will be rendered.

Next, let's add a textbox to enter a search query:

```javascript
// ProductSearchPage.tsx

import * as React from 'react';

import ProductSearchResults from './ProductSearchResults';
import productSearchApi from '../api/ProductSearchApi';

export interface IProductSearchPageState {
  searchResults: IProduct[];
  currentSearchText: string;
};

export default class extends React.Component<any, IProductSearchState) {
  constructor(props: any, context: any) {
    super(props, context);

    this.state = {
      searchResults: [],
      currentSearchText: ''
    };
  }

  public render() {
    return (
      <div>
        <div>
          <input type="text" value={this.state.currentSearchText} onChange={this.onSearchTextChanged.bind(this)} />
        </div>
        <ProductSearchResults searchResults={this.state.searchResults} />
      </div>
    );
  }

  private onSearchTextChanged(e) {
    this.setState({
      ...this.state,
      currentSearchText: e.target.value
    });
  }
);
```

Notice that we also added a state parameter to store the current search text, along with an event handling method, `onSearchTextChanges`. As the user types characters into the textbox, the `onChange` event fires and calls the `onSearchTextChanged` method, which then updates the state with the latest search text. Keeping the component's state in sync the textbox allows us to easily access the search text any time we wish from `this.state.currentSearchText` rather than having to bypass React and dig into the DOM to read the value of the actual `input` element.

Now, we will add a button to trigger an actual search, along with the code to perform the search:

```javascript
// ProductSearchPage.tsx

import * as React from 'react';

import ProductSearchResults from './ProductSearchResults';
import productSearchApi from '../api/ProductSearchApi';

export interface IProductSearchPageState {
  searchResults: IProduct[];
  currentSearchText: string;
};

export default class extends React.Component<any, IProductSearchState) {
  constructor(props: any, context: any) {
    super(props, context);

    this.state = {
      searchResults: [],
      currentSearchText: ''
    };
  }

  public render() {
    return (
      <div>
        <div>
          <input type="text" value={this.state.currentSearchText} onChange={this.onSearchTextChanged.bind(this)} />
          <button onClick={this.onSearch.bind(this)}>Search</button>
        </div>
        <ProductSearchResults searchResults={this.state.searchResults} />
      </div>
    );
  }

  private onSearchTextChanged(e) {
    this.setState({
      ...this.state,
      currentSearchText: e.target.value
    });
  }

  private async onSearch() {
    const searchResults = await productSearchApi.searchForProduct(this.state.currentSearchText);

    this.setState({
      ...this.state,
      searchResults
    });
  }
);
```

As we determined in our technical design, the search text will get sent to a back-end web API, and that web API will then return the products matching the search query to the front-end, which the front-end will display. The "Search" button we added has an `onClick` handler bound to the `onSearch` method. We then read the `currentSearchText` from the state and pass it into `productSearchApi.searchForProduct`. We won't bother defining `productSearchApi`--just assume it fires off an asynchronous HTTP request. We will delve more deeply into services later in the tutorial.

Using [async/await](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Statements/async_function) syntax, we await the return of `searchForProducts` and capture the return value in `searchResults`. Once the search results are returned, we then update our component's state with the new search results, which will trigger a re-render and show those search results.

What if the API call doesn't succeed, though? Maybe we should handle errors and at least let users know that an error occurred instead of leaving them in limbo.

```javascript
// ProductSearchPage.tsx

import * as React from 'react';

import ProductSearchResults from './ProductSearchResults';
import productSearchApi from '../api/ProductSearchApi';

export interface IProductSearchPageState {
  searchResults: IProduct[];
  currentSearchText: string;
  errorMessage: string;
};

export default class extends React.Component<any, IProductSearchState) {
  constructor(props: any, context: any) {
    super(props, context);

    this.state = {
      searchResults: [],
      currentSearchText: '',
      errorMessage: ''
    };
  }

  public render() {
    return (
      <div>
        <div>
          <input type="text" value={this.state.currentSearchText} onChange={this.onSearchTextChanged.bind(this)} />
          <button onClick={this.onSearch.bind(this)}>Search</button>
        </div>
        {props.errorMessage ? <div>{props.errorMessage}</div> : null}
        <ProductSearchResults searchResults={this.state.searchResults} />
      </div>
    );
  }

  private onSearchTextChanged(e) {
    this.setState({
      ...this.state,
      currentSearchText: e.target.value
    });
  }

  private async onSearch() {
    try {
      this.setState({
        ...this.state,
        errorMessage: ''
      });

      const searchResults = await productSearchApi.searchForProduct(this.state.currentSearchText);

      this.setState({
        ...this.state,
        searchResults
      });
    } catch (error: Error) {
      this.onError(error);
    }
  }

  private onError(error: Error) {
    this.setState({
      ...this.state,
      errorMessage: error.message
    });
  }
);
```

Okay, so we added `errorMessage` to the state and now render an error message if if it exists in the state. We then wrap the API call in a `try/catch`, and update the `errorMessage` in the state if there is an error. When the user performs another search, we reset the error message.

Any other potential problems we should consider? Well, what if the search doesn't result in an error, but simply takes a while? It would be good to let the user know that we are waiting for the results of the web API call to return, so let's add that:

```javascript
// ProductSearchPage.tsx

import * as React from 'react';

import ProductSearchResults from './ProductSearchResults';
import productSearchApi from '../api/ProductSearchApi';

export interface IProductSearchPageState {
  searchResults: IProduct[];
  currentSearchText: string;
  searchResultsLoading: boolean;
  errorMessage: string;
  loading: boolean;
};

export default class extends React.Component<any, IProductSearchState) {
  constructor(props: any, context: any) {
    super(props, context);

    this.state = {
      searchResults: [],
      currentSearchText: '',
      searchResultsLoading: false,
      errorMessage: '',
      loading: false
    };
  }

  public render() {
    return (
      <div>
        <div>
          <input type="text" value={this.state.currentSearchText} onChange={this.onSearchTextChanged.bind(this)} />
          <button onClick={this.onSearch.bind(this)}>Search</button>
        </div>
        {props.searchResultsLoading : <div>Loading...</div> : null}
        {props.errorMessage ? <div>{props.errorMessage}</div> : null}
        <ProductSearchResults searchResults={this.state.searchResults} />
      </div>
    );
  }

  private onSearchTextChanged(e) {
    this.setState({
      ...this.state,
      currentSearchText: e.target.value
    });
  }

  private async onSearch() {
    try {
      this.setLoading(true);

      this.setState({
        ...this.state,
        errorMessage: ''
      });

      const searchResults = await productSearchApi.searchForProduct(this.state.currentSearchText);

      this.setState({
        ...this.state,
        searchResults
      });
    } catch (error: Error) {
      this.onError(error);
    } finally {
      this.setLoading(false);
    }
  }

  private onError(error: Error) {
    this.setState({
      ...this.state,
      errorMessage: error.message
    });
  }

  private setLoading(isLoading: boolean) {
      this.setState({
        ...this.state,
        searchResultsLoading: true
      });
  }  
);
```

We added a `loading` property in our state and render "Loading..." text when the property is set to true. Before we call out to the web API, we call `setLoading` to enable the loading indicator, and then we have a `finally` block that disables the loading indicator once the API call completes, regardless of whether it succeeded or not.

This single component implements the logic necessary for product search. Besides the fact that it needs more work to make it look good, what's wrong with it? Well, nothing if the app is never going do grow much larger than a few screens or have more than a couple of developers working on it. React by itself works perfectly well for building small, simple applications. However, as an application grows, along with the size of the team building the application, what usually ends up happening is the React components start to turn into huge files full of logic for calling APIs, business logic, navigation logic, and presentation logic all mixed together in one big incomprehensible pot of alphabet soup that nobody understands, is difficult to change, and is easy to break.

## Up Next

In the next section, we will [refactor our ProductSearchPage](./refactoring-product-search-to-use-redux.md) to use a more sustainable architecture that leverages Redux.