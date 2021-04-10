# Adding in Redux Retro

So far, we have described actions as representing something that happened in the app that requires the store to update the state. We have seen so far that actions are just plain objects that ultimately end up getting passed into the store's `dispatch` method:

```javascript
store.dispatch({ type: 'SEARCH_CHANGED', payload: searchText })
```

When we mapped our new ProductSearchPage stateless component in our ProductSearchContainer, the `SEARCH_CHANGED` action ultimately ended up being a replacement for the `onSearchTextChanged` handler method we had in our original ProductSearchPage class. To recap, that method looked like this:

```javascript
  private onSearchTextChanged(e) {
    this.setState({
      ...this.state,
      currentSearchText: e.target.value
    });
  }
```

In this case, the combination of a simple object-based action and the `productSearchReducer` are able to completely replace this event handler. For reference, here is that reducer again:

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

This is fairly similar to what we were doing in the `setState` call before, just in a different form. However, what about this `onSearch` event handler method from our original class?

```javascript
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
```

This method isn't simply updating one part of the state--there's more going on here. It updates the state to indicate that the search results are loading, then asynchronously waits for a service call to complete, and if it fails, it updates another part of the state with the error message. If the search results come back successfully, it ends up updating the state with the search results.

If we intend to implement the equivalent of these class methods in Redux, then dispatching a simple object with a `type` and a `payload` through the store is clearly not sufficient. The bad news is that Redux by itself doesn't have a solution for this problem because the Redux store's `dispatch` method is only capable of handling action objects. The good news is that Redux has add-on libraries like [Redux Thunk](https://github.com/reduxjs/redux-thunk) and [Redux Saga](https://github.com/redux-saga/redux-saga) that extend its capabilities.

Redux Retro was also developed to address this problem. In addition Redux Retro also provides simple TypeScript support that ensures actions and reducers are linked with type safety. Redux Retro also reduces the amount of boilerplate code required (e.g., no action strings or switch statements), has a way to access app state from actions to make decisions, and also has a simple and practical way to handle asynchronous logic with async/await (or promises). The main parts of the app that are different with Redux Retro are the actions and reducers. Let's see how our actions and reducers look in Redux Retro.

Let's start with the search text changed action:

```javascript
  import { Actions } from 'redux-retro';

  export default class ProductSearchActions extends Actions {
    public searchTextChanged(searchText: string) {
        return searchText;
    }
  }
```

In Redux Retro, we group all of our actions for a particular area of functionality into a class. When a method returns a value, an action gets dispatched through the store. Behind the scenes, it dispatches an action as an object just like we were doing before. For example, this `searchTextChanged` method dispatches whatever search text gets passed in through the store in an object like this: 

```javascript
{ type: 'SEARCH_TEXT_CHANGED', payload: 'Some search text' }
```

With Redux Retro, our reducer ends up looking like this:

```javascript
import { createReducer } from 'redux-retro';

const initialState: IProductSearchState = {
  searchResults: [];
  currentSearchText: '';
  searchResultsLoading: false;
  errorMessage: '';
};

const productSearchReducer = createReducer<IProductSearchState>(initialState)
  .bindAction(ProductSearchActions.prototype.searchTextChanged, (state, action) => {
    return {
      ...state,
      currentSearchText: action.payload
    };    
  });
```

This code is pretty similar to the reducer we created before, except TypeScript errors will occur if you specify an action that doesn't exist, return a new version of the state that doesn't match `IProductSearchState`, or pretend that `action.payload` is a different type than what the action actually returns. For example, in this case, `action.payload` would be a string, and so `action.payload.foo` would result in a TypeScript error.

Next, let's add in the remaining code to actually perform the search:

```javascript
  import productSearchApi from '../api/ProductSearchApi';

  import { Actions } from 'redux-retro';

  export default class ProductSearchActions extends Actions {
    public searchTextChanged(searchText: string) {
        return searchText;
    }

    public async searchForProduct() {
      const searchText = this.getState().productSearch.searchText;      

      try {
        this.productSearchPending();
        const searchResults = await productSearchApi.searchForProduct(searchText);
        this.productSearchSucceeded(searchText, searchResults);
      } catch (error) {
        this.productSearchFailed(error);
      }
    }

    public productSearchPending(): any {
      // This action is just here to control the loading state.
      // There's no payload, so return null to dispatch with a null payload

      return null;
    }

    public productSearchSucceeded(searchText: string, searchResults: IProduct[]) {
      return searchResults;
    }

    public productSearchFailed(error: Error) {
      return error.message;
    }        
  }
```

We added 4 more actions: `searchForProduct`, `searchResultsPending`, `productSearchSucceeded`, and `productSearchFailed`. Do you notice anything peculiar about the way the actions in this class are named? `searchTextChanged`, `productSearchPending`, `productSearchSucceeded`, and `productSearchFailed` are all named in the past-tense and describe something that has already happened, whereas `searchForProduct` is named in the present-tense and is named imperatively. This brings to light the two broad categories of actions:

1. **Declarative actions** - Describe events that have already happened and should result in a state change. In Redux Retro, this type of action method returns a value, which results in a store dispatch with the returned value as the payload. This type of action can be accomplished in Redux without any add-ons.

2. **Imperative actions** - These actions have logic in them, can be asynchronous, and typically end up calling declarative actions. This type of action is not possible in Redux alone and requires an add-on like Redux Retro.

 Let's look at the final `productSearchReducer` with these 4 actions added:

 ```javascript
const productSearchReducer = createReducer<IProductSearchState>(initialState)
  .bindAction(ProductSearchActions.prototype.searchTextChanged, (state, action) => {
    return {
      ...state,
      searchText: action.payload
    };    
  })
  .bindAction(ProductActions.prototype.productSearchPending, (state, action) => {
    return {
      ...state,
      loading: true
    };
  })
  .bindAction(ProductActions.prototype.productSearchSucceeded, (state, action) => {
      return {
        ...state,
        searchResults: action.payload,
        loading: false,
        errorMessage: ''
      };
  })
  .bindAction(ProductActions.prototype.productSearchFailed, (state, action) => {
    return {
      ...state,
      errorMessage: action.payload,
      loading: false
    };
  });  
```

Note that only the declarative actions are bound to the reducer.

## Redux Retro Containers

The container setup is slightly different in Redux Retro, mainly resulting from the fact that we have actions classes instead of just a dispatch method.

Here is what our new container looks like:

```javascript
// ProductSearchContainer.ts

import { connect } from 'react-redux-retro';

import ProductSearchPage from '../components/ProductSearchPage';

export const mapStateToProps = (state: IAppState) => {
  const productSearch = state.productSearch;

  return {
    searchResults: productSearch.searchResults,
    currentSearchText: productSearch.currentSearchText,
    searchResultsLoading: productSearch.searchResultsLoading,
    errorMessage: productSearch.errorMessage,
  };
};

export const mapActionsToProps = (actions: IAppActions) => {
  onSearch: actions.productSearch.searchForProduct,
  onSearchTextChanged: actions.productSearch.searchTextChanged
};

export default connect(mapStateToProps, mapDispatchToProps, ProductSearchPage);
```

Notice that we now have a `mapActionsToProps` method instead of a `mapDispatchToProps`. The `mapActionsToProps` function passes in our app's actions, and we pick out the productSearch actions and map the appropriate actions to the appropriate props.

The only difference in our root App component is that we now have to pass in our app's actions in addition to our app store so the `mapActionsToProps` function can have access to those actions:

```javascript
// App.tsx
import { render } from 'react-dom';
import { Provider } from 'react-redux-retro';

import store from '../store';
import actions from '../actions';

const App = () => (
  <Provider store={store} actions={actions}>
    <ProductSearchContainer />
  </Provider>
);

render(<App />, document.getElementById('app'));
```

The last step is declaring an interface for our app's actions and an object containing our app's actions that were required in the ProductSearchContainer and App component:

```javascript
// actions.ts

import store from './store';
import ProductSearchActions from './actions/ProductSearchActions';

export interface IAppActions {
  productSearch: ProductSearchActions;
}

export default {
  productSearch: new ProductSearchActions(store)
} as IAppActions;
```

## Up Next

Up until this point, we have been using `productSearchApi` to make our web API calls. We never defined the code for `productSearchApi`, and instead we just assumed it would fire off a HTTP request. The main enabler of Front-End First is the ability to quickly create mock services in the front-end code that can stand in for the real back-end services until the front-end is solidified. [Next](./product-search-mock-services.md), we will begin creating mock services with I/O Source.