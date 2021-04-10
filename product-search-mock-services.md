# Product Search Mock Services

We are now going to look into replacing `productSearchApi` with a mock service implementation, and what is called a Source. Let's get started by creating the Source.

## Product Search Source

A Source is a class that handles data access for a particular area of an application like Product Search. Source classes leverage "proxies" from I/O Source. I/O Source provides proxies that handle service calls, as well as proxies for handling storage (i.e., LocalStorage). In this tutorial, we will only be dealing with service calls for simpicity.

I/O Source provides an `IServiceProxy` interface, along two concrete service proxy implementations: `MockServiceProxy` and `HttpServiceProxy`. When you define your Source class, you should always use the `IServiceProxy` interface instead of the concrete implementations and allow the concrete implementations to be constructor injected. We will be showing how the plumbing for this injection is accomplished later in this section.

 Lets add a Product Search Source with a method to call a service and search for products:

```javascript
// ProductSearchSource.ts

import { IServiceProxy } from 'io-source';

import { IProduct } from '../models/Product';

export default class {
  private serviceProxy: IServiceProxy;

  constructor(serviceProxy: IServiceProxy) {
    this.serviceProxy = serviceProxy;
  }
  
  public searchForProduct(searchText: string) {
    return this.serviceProxy.readViaService<IProduct[]>(`/products/search/${searchText}`);
  }
}
```

So, we have defined a class with a public method to search for products. The search text is passed as an argument to that method, and the method then calls into the service proxy's `readViaService` method, passing a URI to the product search service, concatenating the search text as a URI parameter. `readViaService` represents an HTTP GET request to the service URI, and when this class is instantiated with an HttpServiceProxy as the IServiceProxy implementation, an actual HTTP GET request will be sent.

## Mock Product Search Service

Next, we will add our mock Product Search service, which is the service implementation that will be called when the Source class gets instantiated with a `MockServiceProxy` as the `IServiceProxy` implementation. To accomplish this, we will need to define the mock service itself, some mock product data, and some code to perform the search within the mock data to simulate what the back-end would be doing.

First, let's define some mock product data:

```javascript
import { IProduct } from '../../../app/models/Product';

export default [{
  id: 1,
  name: 'Baseball glove',
  description: '...',
  price: 20,
  image: 'baseball-glove.jpg'
}, {
  id: 2,
  name: 'Soccer ball',
  description: '...',
  price: 15,
  image: 'soccer-ball.jpg'
},

...
...

] as IProduct[];
```

Let's define a function to add the mock service operation to our `MockServiceProxy`:

```javascript
import { MockServiceProxy } from 'io-source';

import { getProductSearchResults } from '../../test-data/products/ProductSearch';

export const configureProductSearchService = (serviceProxy: MockServiceProxy) => {
  serviceProxy.addReadOperation('/products/search/{searchText}', ({ searchText }) => {
    return getProductSearchResults(searchText);
  });  
};
```

In the code above, we add a read operation to our `MockServiceProxy` for the same URI that our ProductSearchSource class is using, along with a function that gets executed whenever that service operation gets called. We can define URI parameters in curly braces like `{searchText}`, and I/O Source will pass them into the first parameter of our service as an object, which can be conveniently destructured as shown in the code above.

Whenever this service gets called, our function will call `getProductSearchResults` and pass in the search text. Let's define  `getProductSearchResults` now:

```javascript
import Products from './Products';

export const getProductSearchResults = (searchText: string) => {
  return Products
    .filter(product => {
      return product.name.toLowerCase().indexOf(searchText.toLowerCase()) !== -1
    });
};
```

This is a really simple search algorithm that filters products based on whether or not the search text is contained within the product name. Mock services are usually simple aproximations of what happens on the back-end, and it's likely that the back-end service implementation would have a more sophisticated algorithm.

## Wiring up Sources with Actions

Let's revisit our previous ProductSearchActions class:

```javascript
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

    ...
    ...
```

The `searchForProduct` method references an imported object instance called `productSearchApi`. Let's replace that with our source. As we saw, the source gets its `IServiceProxy` constructor injected, and the ProductSearchSource itself will also need to be constructor injected:

```javascript
  export default class ProductSearchActions extends Actions {
    private sources: IAppSources;

    constructor(Store: Store<IAppState>, sources: IAppSources) {
      super(store);

      this.sources = sources;
    }

    public searchTextChanged(searchText: string) {
        return searchText;
    }

    public async searchForProduct() {
      const searchText = this.getState().productSearch.searchText;      

      try {
        this.productSearchPending();
        const searchResults = await this.sources.productSearch.searchForProduct(searchText);
        this.productSearchSucceeded(searchText, searchResults);
      } catch (error) {
        this.productSearchFailed(error);
      }
    }

    ...
    ...
```

For convenience, rather than injecting `ProductSearchSource` by itself, we are injecting an object of type `IAppSources`, which is an object that contains instances of sources for our entire app. Similarly to how we defined `IAppActions` earlier for the app's actions, let's define our sources:

```javascript
// sources.ts

import { IServiceProxy } from 'io-source';

import ProductSearchSource from './sources/ProductSearchSource';

export interface IAppSources {  
  productSearch: ProductSearchSource;
};

export const getSources = (serviceProxy: IServiceProxy) => {
  return {
    productSearch: new ProductSearchSource(serviceProxy)
  } as IAppSources;
};
```

Just like our app's `IAppState` will gain top-level properties as our app grows, so too will `IAppActions` and `IAppSources`. Our sources are getting created in a function so they can be created with different implementations of `IServiceProxy`. To continue with this pattern, we will have to modify our code that creates `IAppActions` to work similarly with their `IAppSources` dependency. To recap, our `IAppActions` are currently getting created like this:

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

This doesn't take into account that the sources need to be injected into the actions classes, so let's update it:

```javascript
import ProductSearchActions from './actions/ProductSearchActions';
import { IAppState, store } from './store';
import { IAppSources } from './sources';

export interface IAppActions {
  productSearch: ProductSearchActions;
}

export const getActions = (sources: IAppSources) => {
  return { 
    productSearch: new ProductSearchActions(store)
  } as IAppActions;  
};
```

## Composition Roots

The final step for wiring all of this up is to create what is called a composition root. It is here that we instantiate the appropriate type of service proxy and wire up our app. We can therefore have more than one composition root, and indeed it is useful to have one composition root for running the app with mock services, a second for running the app with real services, and a third for writing tests. Let's start by creating our mock service composition root:

```javascript
// app-with-mocks.ts

import { MockServiceProxy } from 'io-source';

const serviceProxy = new MockServiceProxy({ addRandomDelays: true });
```

The first step is to create our `MockServiceProxy`. A nice feature of `MockServiceProxy` is that it can be setup to add random delays to simulate the random delays that would occur in a real service when running the app. Next, let's add in the sources and actions:

```javascript
// app-with-mocks.ts

import { MockServiceProxy } from 'io-source';

import { runApp } from './app/components/App';
import { getSources } from './app/sources';
import { getActions } from './app/actions';

const serviceProxy = new MockServiceProxy({ addRandomDelays: true });
const sources = getSources(serviceProxy);
const actions = getActions(sources);
```

In order for mock services to work, we have to add mock service operations to our `MockServiceProxy`. Earlier, we created a `configureProductSearchService` function that takes a `MockServiceProxy` just for that purpose:

```javascript
import { MockServiceProxy } from 'io-source';

import { getProductSearchResults } from '../../test-data/products/ProductSearch';

export const configureProductSearchService = (serviceProxy: MockServiceProxy) => {
  serviceProxy.addReadOperation('/products/search/{searchText}', ({ searchText }) => {
    return getProductSearchResults(searchText);
  });  
};
```

We could certainly import this function and call it directly from our composition root, but to make it easy to add more services later, let's make a wrapper function called `configureAllServices` that we can use to setup this service and any mock services we may add later:

```javascript
// AllServices.ts

import { MockServiceProxy } from 'io-source';

import { configureProductSearchService } from './products/ProductSearchService';

export const configureAllServices = (serviceProxy: MockServiceProxy) => {
  configureProductSearchService(serviceProxy);
};
```

Okay, and now we can configure all of our mock services from the composition root:

```javascript
// app-with-mocks.ts

import { MockServiceProxy } from 'io-source';

import { runApp } from './app/components/App';
import { getSources } from './app/sources';
import { getActions } from './app/actions';
import { configureAllServices } from './test/mock-services/AllServices';

const serviceProxy = new MockServiceProxy({ addRandomDelays: true });
const sources = getSources(serviceProxy);
const actions = getActions(sources);

configureAllServices(serviceProxy);
```

The last step is to actually run our app by rendering our App's main React component. To refresh your memory, we previously defined it like this:

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

Since our app component depends on actions, and we're creating our sources and actions dynamically from our composition root, our main app component will need to work similarly:

```javascript
// App.tsx
import { Store } from 'redux';
import { render } from 'react-dom';
import { Provider } from 'react-redux-retro';

import { IAppActions } from '../actions';

export interface IAppProps {
  store: Store<IAppState>;
  actions: IAppActions;
}

export const App = (props: IAppProps) => (
  <Provider store={props.store} actions={props.actions}>
    <ProductSearchContainer />
  </Provider>
);

export const runApp = (store: Store<IAppState>, actions: IAppActions) => {
  render(<App />, document.getElementById('app'));
}
```

While we're at it, let's wrap the store initialization in a function so we can create a new store on demand for our app as well:

```javascript
  import { createStore, combineReducers } from 'redux';

  import { IProductSearchState } from './app/models/ProductSearch';

  export interface IAppState {
    productSearch: IProductSearchState;
  }

  export const getStore = () => {
    return createStore(
      combineReducers<IAppState>({
        productSearch: productSearchReducer
      })
    );
  };

```

Finally, we can run the app from our composition root:

```javascript
// app-with-mocks.ts

import { MockServiceProxy } from 'io-source';

import { runApp } from './app/components/App';
import { getStore } from './app/store';
import { getSources } from './app/sources';
import { getActions } from './app/actions';
import { configureAllServices } from './test/mock-services/AllServices';
import { runApp } from './app/components/App';

const serviceProxy = new MockServiceProxy({ addRandomDelays: true });
const sources = getSources(serviceProxy);
const actions = getActions(sources);
const store = getStore();

configureAllServices(serviceProxy);

runApp(store, actions);
```

Now that our app-with-mocks.ts file is complete, we can build the app with this file as the entry point for a build tool like Webpack or Parcel and build our app with all of the mock services included.

Creating a composition root to run the app without mocks is almost the same, except we substitute a `HttpServiceProxy` for the `MockServiceProxy`:

```javascript
// app-without-mocks.ts

import { HttpServiceProxy } from 'io-source';

import { runApp } from './app/components/App';
import { getStore } from './app/store';
import { getSources } from './app/sources';
import { getActions } from './app/actions';
import { configureAllServices } from './test/mock-services/AllServices';
import { runApp } from './app/components/App';

const serviceProxy = new HttpServiceProxy('./api/');
const sources = getSources(serviceProxy);
const actions = getActions(sources);
const store = getStore();

runApp(store, actions);
```

Unlike the `MockServiceProxy`, the `HttpServiceProxy` requires a relative path to the root of the web API. It is also no longer necessary to `configureAllServices` to setup the mock services. Now, we can point a build tool like Webpack or Parcel to this app-without-mocks.ts file, and the output will not include any of the mock services code--it will be a lean and mean build that includes only the essential code required to call the real service.

# Up Next

In this section we went through using I/O Source to run our app with mock data and mock services and have the ability to easily switch over and use real services just by building with a different root file. Now, let's look into how we can leverage the same mock services and mock data when [writing tests](./integration-tests.md).