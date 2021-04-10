# Integration Tests

With the way the app is now structured, most of the interesting logic ends up in the actions, reducers, sources, and mock services. The React components mostly exist at this point to translate data from the state into DOM elements, and to handle events from DOM elements (like typing in a textbox or clicking a button) and pipe those into the actions. Imagine if we defined a composition root that omitted the React components:

```javascript
// test-setup.ts

import { MockServiceProxy } from 'io-source';

import { runApp } from './app/components/App';
import { getStore } from './app/store';
import { getSources } from './app/sources';
import { getActions } from './app/actions';
import { configureAllServices } from './test/mock-services/AllServices';

const getTestSetup = () => {
  const serviceProxy = new MockServiceProxy({ addRandomDelays: false });
  const sources = getSources(serviceProxy);
  const actions = getActions(sources);
  const store = getStore();

  configureAllServices(serviceProxy);

  return {
    store,
    serviceProxy,
    sources,
    actions
  };
};
```

Calling `getTestSetup` would give us access to all of the pieces of our app, except for the React component. What good is an app without the React components, though? Well, since the containers are the bridge between the React components and the rest of the app, containers can be thought of as an API that exposes most of the app's client-side logic. Let's look at the product search container again:

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

If we could leverage `mapStateToProps` and `mapActionsToProps`, they would give us access to the props that would normally be getting passed into our `ProductSearchPage` React component. We could then simulate what would be happening inside of the `ProductSearchPage` component if a user were interacting with it like so:

```javascript
import { getTestSetup } from './test-setup';
import * as ProductSearchContainer from '../app/containers/ProductSearchContainer';

const search = async () => {
  const context = getTestSetup();

  ProductSearchContainer.mapActionsToProps(context.actions).onSearchTextChanged('Baseball glove');
  await ProductSearchContainer.mapActionsToProps(context.actions).onSearch();

  const searchResults = ProductSearchContainer.mapStateToProps(context.store.getState());

  console.log(searchResults.length);
};
```

The code above essentially has the same result as if a user had started up the app, entered "Baseball glove" into the search box, and then clicked the "Search" button. When we pull the search results from the `mapStateToProps` function, we're accessing the same data that the React component would have been rendering on the screen.

If you've ever used tools like Selenium to write automated tests, you are quite well aware that automating tests via the browser DOM normally results in tests that are slow to run, and that often have random timing issues that cause tests to fail inexplicably sometimes, but then pass when re-run. Indeed, while it is recommended to have end to end tests that automate the UI like that, the recommendation is usually to have very few of this type of test because of how slow they can be and how difficult they can be to maintain.

However, automating the app against the container APIs has many of the same benefits, yet side-steps the pitfalls. When automating against the container APIs, you're still exercising the app in much the same way it would be exercised via the React components. The difference is that tests that are automated against the container APIs have no I/O or DOM manipulation, and are therefore quite fast. In fact, hundreds of these tests can run in mere seconds, as opposed to minutes or even hours for a typical suite of Selenium tests. In addition, since we're automating against an API instead of against a live DOM in a browser, the tests can be much more reliable and simpler to write.

That's not to say that end to end tests using Selenium or similar tools are not needed. It just means that we can do most of our testing with fast integration tests that automate against the container APIs. If we wanted to get more code coverage, we could also consider unit testing the React components to ensure that they behave as expected with different props. Container API integration tests represent a sweet spot between small and isolated unit tests that perform extemely well and full end to end tests that test the entire app exactly as it is exercised in the real world.

## Writing tests with Jest Cucumber

[Jest Cucumber](github.com/bencompton/jest-cucumber/) is an add-in to the popular Jest test runner that enables running tests against Gherkin scenarios. For example, imagine that our team had come up with a scenario for product search that looks like this:

```gherkin
Feature: Product search

  Scenario: Search results returned
    Given a search query that matches a product
    When I search with that search query
    Then I should see that product in my search results
```

We can setup the code in Jest Cucumber to link to the feature file containing this scenario and allow us to write the step definition code for the scenario like so:

```javascript
// ProductSearch.steps.ts

import { defineFeature, loadFeature } from 'jest-cucumber';

const feature = loadFeature('src/test/integration-tests/features/ProductSearch.feature');

defineFeature(feature, (test) => {
  test('Search results returned', ({ given, when, then }) => {
    given('a search query that matches a product', () => {

    });

    when('I search with that search query', () => {
      
    });

    then('I should see that product in my search results', () => {
      
    });
  });
});
```

For the `given` step, let's look at the product data we previously created for our product search mock service:

```javascript
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

There we see that there is a "Baseball glove" product in the test data that the mock service is using. If we were to search for "Baseball glove", then we should get search results containing that one product. Let's add the code to do that:

```javascript
defineFeature(feature, (test) => {
  test('Search results returned', ({ given, when, then }) => {
    let searchQuery = '';

    given('a search query that matches a product', () => {
      searchQuery = 'Baseball glove';
    });

    when('I search with that search query', () => {

    });

    then('I should see that product in my search results', () => {
      
    });
  });
});
```

Next is the `when` step, in which we actually need to do the search:

```javascript
defineFeature(feature, (test) => {
  let context: any = null;

  beforeEach(() => {
    context = getTestSetup();
  });
  
  test('Search results returned', ({ given, when, then }) => {
    let searchQuery = '';

    given('a search query that matches a product', () => {
      searchQuery = 'Baseball glove';
    });

    when('I search with that search query', async () => {
      ProductSearchContainer.mapActionsToProps(context.actions).onSearchTextChanged(searchQuery);
      await ProductSearchContainer.mapActionsToProps(context.actions).onSearch();      
    });

    then('I should see that product in my search results', () => {
      
    });
  });
});
```

Finally, we need to implement the `then` step where we need to make sure that the search results came back as expected:

```javascript
defineFeature(feature, (test) => {
  let context: any = null;

  beforeEach(() => {
    context = getTestSetup();
  });
  
  test('Search results returned', ({ given, when, then }) => {
    let searchQuery = '';

    given('a search query that matches a product', () => {
      searchQuery = 'Baseball glove';
    });

    when('I search with that search query', async () => {
      ProductSearchContainer.mapActionsToProps(context.actions).onSearchTextChanged(searchQuery);
      await ProductSearchContainer.mapActionsToProps(context.actions).onSearch();      
    });

    then('I should see that product in my search results', () => {
      const searchResults = ProductSearchContainer.mapStateToProps(context.store.getState());      

      expect(searchResults.length).toBe(1);
      expect(searchResults[0].name).toBe(searchQuery);
    });
  });
});
```

## Up Next

We now have product search implemented with React, Redux Retro, and I/O Source, and have implemented an integration test that leverages the same mock data and mock service that is used when running the app in mock mode.

The next step is to look through the companion [starter template](github.cerner.com/ExtendedCareLTC/react-app-starter-template/) that goes along with this tutorial. The starter template demonstrates a more fully featured shopping cart app that can be cloned and used as a starting point for new apps. The starter template uses the Ant Design React UI library, has more functionality, more tests, and demonstrates how to actually build and run an app with [Parcel](https://github.com/parcel-bundler/parcel).
