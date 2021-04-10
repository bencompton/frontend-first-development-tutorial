# Introduction

This tutorial demonstrates how to develop web applications with Front-End First Development. In this tutorial, we are going to demonstrate this technique by building a simple product search feature for a shopping cart web app with React and Redux. Although what we are going to build is simple and small, we will be using practices and libraries that are designed for building large, production-ready apps.

## Front-End First Development

Conventional wisdom suggests that the best place to start when develping an app is the back-end. You define the database schema, the web APIs, etc., and once you have back-end APIs ready for the front-end to call, you can then proceed with building the front-end. As many engineers have discovered, it is often the case that our initial assumptions about the front-end turn out to not be valid, and the front-end ultimately ends up working differently than we initially imagined. This results in having to re-work the back-end code. Quite often, it is more optimal to start with the front-end and work backwards. Once the front-end is perfected, the data contract between the front-end and the back-end is solidified, and the back-end code can be written with confidence.

Enabling Front-End First development was the main motivation for developing I/O Source. [I/O Source](https://github.com/bencompton/io-source) is an open source JavaScript library that allows building completely functional and self-contained single-page apps that use mock data and mock services, which can be easily switched over to call real back-end web APIs with no extra front-end code. With this technique, front-end engineers can perfect the user experience without getting bogged down in the back-end, and then add back-end services afterwards that fulfill the same data contract as the mock services.

This tutorial will also demonstrate how to create functional integration tests against the application's state management logic that re-use the same mock data and mock services that are used in the stand-alone single-page app. These integration tests are able to exercise the app in much the same way as the user would, yet not suffer from many of the same issues of slowness, brittleness, etc. that would result from automating the front-end with tools like [Selenium](https://www.seleniumhq.org). With the techniques and technology choices demonstrated in this tutorial, suites of hundreds of tests that cover most of what a user can do with an application can be created that can finish running within seconds.

## Tooling

The code in this tutorial will be written entirely in TypeScript. While ECMAScript has come a long way over the last few years, compile-time type checking is an extremely useful language feature. When using TypeScript, it sometimes takes a while to fix compile-time errors that occur. However, the vast majority of the time, the issues you will be fixing at compile time are issues that would otherwise occur at runtime--issues that can often be difficult to detect and reproduce. In short, TypeScript improves the quality of your code, and while it may sometimes take longer to write than ECMAScript, as the saying goes, quality is speed. In addition, compile-time type checking is extremely useful when refactoring, again providing immediate feedback about issues that would otherwise only surface at runtime.

In addition to TypeScript, React and Redux, here are some of the lesser-known front-end libraries that we will be using in this tutorial:

* [I/O Source](https://github.com/bencompton/io-source) - A library that provides a service implementation that can be easily swapped between mock services and real services at compile time.

* [Redux Retro](https://github.com/bencompton/redux-retro) - Extends Redux with a simple model for asynchronous actions, enhanced TypeScript support, less verbose syntax, and includes useful functionality from previous implementations of the Flux architectural pattern, like [Alt.js](http://alt.js.org).

* [Jest Cucumber](https://github.com/bencompton/jest-cucumber) - Cucumber's Gherkin is a great language for creating business-readable specifications for ATDT. While there is a Cucumber implementation for JavaScript, Jest and its ecosystem of tooling provide a great developer experience, and Jest Cucumber was created to capture the best of both worlds.

## Up Next

Now that we have established that we will be implementing product search with Front-End First and have gone through our basic technology choices, let's move on and start working on the [technical design for product search](./product-search-technical-design.md).