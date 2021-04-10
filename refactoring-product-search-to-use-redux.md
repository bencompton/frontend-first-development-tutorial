# Refactoring Product Search to Use Redux

In the previous section, we implemented a fairly basic product search feature using only React. The purpose of using Redux, along with the other tools we will be using in this tutorial, is to extract all of the complex code out of your React components, like business logic, navigation, calling web APIs, etc., into what is collectively known as the "state management" code in your app. This leaves your React components with two basic tasks: show info to the user, and accept input from the user. In other words, your React components are responsible only for taking data that your state management logic provides and converting it into HTML, and then when a user interacts with that HTML (e.g., clicking a button), the React component passes that along to your state management logic.

Redux is a JavaScript library that implements the Flux architectural pattern, and is designed for creating well-organized single-page apps that can scale both in terms of the size of the app, as well as the size of the team working on the app. The way this is accomplished is by splitting up your state management logic into small, specialized pieces: a central app **store** that holds all of your app's state, **actions** and **reducers** to modify app state, and **selectors** to query your app's state.

While we are ultimately going to end up using [Redux Retro](https://github.com/bencompton/redux-retro/) in this tutorial, we are going to begin with plain Redux. Redux Retro builds upon Redux, so a full understanding of Redux is essential.

## Removing the State Management Logic

Since our goal is to refactor the state management logic out of our React component and move it into Redux, let's start by stripping the state management logic out of our ProductSearchPage:

```javascript
// ProductSearchPage.tsx

import * as React from 'react';

import ProductSearchResults from './ProductSearchResults';

export interface IProductSearchProps {
  searchResults: IProduct[];
  currentSearchText: string;
  searchResultsLoading: boolean;
  errorMessage: string;

  onSearch(): void;
  onSearchTextChanged(searchText: string): void;
};

export default () => (
  <div>
    <div>
      <input type="text" value={props.currentSearchText} onChange={props.onSearchTextChanged} />
      <button onClick={props.onSearch}>Search</button>
    </div>
    {props.searchResultsLoading : <div>Loading...</div> : null}
    {props.errorMessage ? <div>{props.errorMessage}</div> : null}
    <ProductSearchResults searchResults={props.searchResults} />
  </div>
);
```

Wow, our React component pretty much got eviscerated! It got converted from a hefty component class into a much leaner stateless functional component. Instead of managing its own state, it now has data to render passed into it as props (`searchResults`, along with `errorMessage`, `searchResultsLoading`, and `currentSearchText`). When it comes time to respond to something the user does, like typing into the search box or clicking the search button, that gets passed off to event handler functions that are passed in from elsewhere as props (`onSearch` and `onSearchTextChanged`) rather than being part of the component itself.

Now, let's start moving all of the logic that we just stripped out of the React component into Redux.

## App Store

At the center of Redux is your app's store. The store holds an app's current state, manages modifications of that state, and fires events when the state is updated. Instead of keeping your state spread out in various React components, your app's entire state goes into a single JavaScript object that your store manages. If we were to create a TypeScript interface for our shopping cart app's state that would be held in the store at this point, it would look like this:

```javascript
export interface IProductSearchState {
  searchResults: IProduct[];
  currentSearchText: string;
  searchResultsLoading: boolean;
  errorMessage: string;
}

export interface IAppState {
  productSearch: IProductSearchState;
}
```

As the app grows, so does its state: 

```javascript
export interface IAppState {
    productSearch: IProductSearchState,
    cart: ICartState,
    ...
    ...
}
```

## Actions

We will go over the specifics of how to create a store in a moment, but for now, let's assume we have a store. How do we modify the state held within the store? That is accomplished by dispatching an action through the store. An action is just a simple JavaScript object that describes something that happened in the app that should result in a state change. Actions have a `type` property that contains a name for the action, along with other properties containing data that does along with the action. A common convention is to add a `payload` property to the action object to hold the action's data. For example, say that the user changed the search text and we want to update the `currentSearchText` property in the app's state. To accomplish that, we would call the `dispatch` method of the store:

```javascript
store.dispatch({
  type: 'SEARCH_TEXT_CHANGED',
  payload: 'Mickey mouse watch'
});
```

We just dispatched an action of type `SEARCH_TEXT_CHANGED`, and we included a "payload", which is the actual search text in this case. The "payload" of an action could be anything: a string, a number, an array, an object, etc. How does the store actually update the state? For that, we have to define a reducer.

## Reducers

A reducer is a function that takes a piece of the app's current state and an action, and returns a new version of the piece of the state based on that action. For example, our Product Search Reducer would look like this:

```javascript
  const productSearchReducer = (state: IProductSearchState, action: any) => {
    switch (action.type) {
      case 'SEARCH_TEXT_CHANGED':
        return {
          ...state,
          currentSearchText: action.payload
        };
      default:
        return state;
    }
  };
```

The reducer above is specifically for the product search area of the state. Before we can move on and actually create our app's store, we need to have a top-level reducer that handles the entire app state. The top-level app reducer then calls other sub-reducers like the productSearchReducer we just created. Our main app reducer would look something like this:

```javascript
  const appReducer = (state: IAppState, action: any) => {
    return {
      productSearch: productSearchReducer(state.productSearch, action)
    };
  };
```

## Creating a Store

Now that we have our main app reducer, we can define our store:

```javascript 
  import { createStore } from 'redux';
  import { IProductSearchState } from './app/models/ProductSearch';

  export interface IAppState {
    productSearch: IProductSearchState;
  }

  const store = createStore<IAppState>(appReducer);
```

Pretty simple, right? Actually, we can make it even simpler and save ourselves the trouble of creating an overall app reducer by using the `combineReducers` function in Redux:

```javascript
  import { createStore, combineReducers } from 'redux';

  import { IProductSearchState } from './app/models/ProductSearch';

  export interface IAppState {
    productSearch: IProductSearchState;
  }

  const store = createStore(
    combineReducers<IAppState>({
      productSearch: productSearchReducer
    })
  );
```

The `combineReducers` function takes an object that maps reducers to state properties and automatically generates an overall app reducer that is similar to the one we manually defined. How nifty!

One last point to note is that when the store first initializes, it actually calls the reducers before any actions are dispatched in order to get an initial app state in place. When it does this, it doesn't pass in any state or action. Therefore, our reducer should include a default value for the state like so:

```javascript
  const initialState: IProductSearchState = {
    searchResults: [];
    currentSearchText: '';
    searchResultsLoading: false;
    errorMessage: '';
  };

  const productSearchReducer = (state: IProductSearchState = initialState, action: any) => {
    switch (action.type) {
      case 'SEARCH_TEXT_CHANGED':
        return {
          ...state,
          currentSearchText: action.payload
        };
      default:
        return state;
    }
  };
```

## Connecting to React

We have created a store, and have gone through the process of updating the state in that store via actions and reducers. Our store currently contains the state that we previously had in our `ProductSearchPage` React component, but unlike when we were calling `setState` in our Rect component, updating the store's state doesn't currently cause React to update the UI. How do we accomplish that?

Our store has a couple of very important methods that we have yet to cover: `subscribe` and `getState`. The `subscribe` method calls a callback function whenever an action is dispatched and the store has finished calling the reducers and updating the state. The `getState` method returns the full state object held within the store, or in our case an object of type `IAppState`.

When combining React with Redux, the entire app normally gets re-rendered whenever an action is dispatched, and then the new app state gets passed into the React components as props. Let's go through the basic boilerplate code for accomplishing that with our ProductSearch page.

To recap, we previously refactored the ProductSearch page to be a simple stateless functional component with no state management logic:

```javascript
// ProductSearchPage.tsx

import * as React from 'react';

import ProductSearchResults from './ProductSearchResults';

export interface IProductSearchProps {
  searchResults: IProduct[];
  currentSearchText: string;
  searchResultsLoading: boolean;
  errorMessage: string;

  onSearch(): void;
  onSearchTextChanged(searchText: string): void;
};

export default () => (
  <div>
    <div>
      <input type="text" value={this.props.currentSearchText} onChange={props.onSearchTextChanged} />
      <button onClick={props.onSearch}>Search</button>
    </div>
    {props.searchResultsLoading : <div>Loading...</div> : null}
    {props.errorMessage ? <div>{props.errorMessage}</div> : null}
    <ProductSearchResults searchResults={props.searchResults} />
  </div>
);
```

Let's go ahead and hook the ProductSearchPage component up to Redux:

```javascript
import { render } from 'react-dom';

store.subscribe(() => {
  const appState = store.getState();
  const productSearchState = appState.productSearch;

  render(
    <ProductSearchPage
      searchResults={productSearchState.searchResults}
      currentSearchText={productSearchState.currentSearchText};
      searchResultsLoading={productSearchState.searchResultsLoading};
      errorMessage={productSearchState.errorMessage};

      onSearch={() => store.dispatch({ type: 'SEARCH', payload: null })};
      onSearchTextChanged={searchText => store.dispatch({ type: 'SEARCH_CHANGED', payload: searchText })}
    />, 
    document.getElementById('app')
  );
});
```

We first subscribe to our store and create a callback function that will get called every time the store updates. Within that function, we create a variable with the current product search state, and then render our ProductSearchPage, passing in the state from the store as props. We also pass in functions that dispatch actions for the `onSearch`, and `onSearchTextChanged` event handler props in ProductSearchPage. When an action is dispatched, the `render` function will end up getting called again, the new state will get passed in as props, React will re-render the component in its virtual DOM and then update the real DOM with any required changes.

This approach certainly works, but in a large app, the file containing this code will eventually grow to a very large size and become difficult to maintain. To resolve this problem, we use the React Redux library to create what are called containers.

## Connecting to React with Containers

A "container" is an auto-generated React component that hooks up a top-level React component in an app, like our ProductSearchPage, to our state management code. We create a container by calling the `connect` function from React Redux:

```javascript
// ProductSearchContainer.ts
import { connect } from 'react-redux';

import ProductSearchPage from '../components/ProductSearchPage';

const mapStateToProps = (state: IAppState) => {
  const productSearch = state.productSearch;

  return {
    searchResults: productSearch.searchResults,
    currentSearchText: productSearch.currentSearchText,
    searchResultsLoading: productSearch.searchResultsLoading,
    errorMessage: productSearch.errorMessage,
  };
};

const mapDispatchToProps = (dispatch: any) => {
  onSearch: () => dispatch({ type: 'SEARCH', payload: null }),
  onSearchTextChanged: (searchText) => dispatch({ type: 'SEARCH_CHANGED', payload: searchText })
};

export default connect(mapStateToProps, mapDispatchToProps)(ProductSearchPage);
```

Here, we are calling the `connect` function, which takes a `mapStateToProps` function, and a `mapDispatchToProps` function, and also takes a React component. The `mapStateToProps` and `mapDispatchToProps` cleanly separate what we were doing previously in our store `subscribe` callback function to create the props for our ProductSearchPage. The return value of the `connect` function is a React component that will render our ProductSearchPage, and this new React component will have access to the store's state and dispatch method, and will automatically call our two mapping functions to derive the props for ProductSearchPage.

As our app grows, we will add more and more containers. For example, in a typical shopping cart app, we would likely end up having top-level pages for viewing the cart, viewing product details, completing the checkout process, etc. Each of these pages will end up having top-level React components and a container for each top-level React component.

The final step with React Redux is to give it access to our store and get it into our app's rendering pipeline. This is accomplished with React Redux's `Provider` component:

```javascript
// App.tsx
import { render } from 'react-dom';
import { Provider } from 'react-redux';

import store from '../store';

const App = () => (
  <Provider store={store}>
    <ProductSearchContainer />
  </Provider>
);

render(<App />, document.getElementById('app'));
```

React Redux does some voodoo behind the scenes using React's Context API, and voila, all containers that get rendered underneath this magical `Provider` component get access to our store. The `Provider` component also subscribes to our store and re-renders everything under it (our whole app) whenever the store updates.

We now have the code in place required to keep the store up to date as the user changes the search text. As the user types, our ProductSearchPage component calls its `onSearchTextChanged` prop, which ends up being a function passed into it through the container we just defined. That function dispatches the `SEARCH_TEXT_CHANGED` action through the store, and the store calls our `productSearchReducer`, which returns a new version of the state with the `currentSearchText` property updated. React Redux internally subscribes to our store and registers a callback function, similar to what we did when we were calling `store.subscribe` ourselves. When the store updates and calls that callback function, React Redux re-renders our ProductSearchContainer, which runs through the `mapStateToProps` function we defined and passes the latest search text into our ProductSearchPage component as a prop. Our ProductSearchPage then re-renders with the new prop value, and shows the latest search text in the textbox.

## Up Next

Next, we will create the actual search functionality with Redux. Unfortunately, Redux isn't quite up to the task by itself and will require an add-on library, as we will see in the [next section](./completing-product-search-with-redux-retro.md).