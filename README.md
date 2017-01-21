# react-async-component 😴

Create Components that resolve asynchronously, with support for server side rendering and code splitting.

[![npm](https://img.shields.io/npm/v/react-async-component.svg?style=flat-square)](http://npm.im/react-async-component)
[![MIT License](https://img.shields.io/npm/l/react-async-component.svg?style=flat-square)](http://opensource.org/licenses/MIT)
[![Travis](https://img.shields.io/travis/ctrlplusb/react-async-component.svg?style=flat-square)](https://travis-ci.org/ctrlplusb/react-async-component)
[![Codecov](https://img.shields.io/codecov/c/github/ctrlplusb/react-async-component.svg?style=flat-square)](https://codecov.io/github/ctrlplusb/react-async-component)

```jsx
const AsyncProduct = createAsyncComponent({
  resolve: () => import('./components/Product'))
});

<AsyncProduct productId={1} /> // 🚀
```

## TOCs

  - [Introduction](#introduction)
  - [Usage](#usage)
  - [API](#api)
    - [createAsyncComponent(config)](#createasynccomponentconfig)
    - [withAsyncComponents(element)](#withasynccomponentselement)
  - [Examples](#examples)
    - [Creating Async Components](#creating-async-components)
      - [Simple](#simple)
      - [With Loading Component](#with-loading-component)
      - [Webpack 1/2 `require.ensure` Code Splitting](#webpack-12-require.ensure-code-splitting)
      - [Webpack 2 `import` / `System.import` Code Splitting](#webpack-2-import-system.import-code-splitting)
      - [Defer Loading to the Client/Browser](#defer-loading-to-the-clientbrowser)
    - [End to End Examples](#end-to-end-examples)
      - [Browser Only Application](#browser-only-application)
      - [Server Side Rendering Application](#server-side-rendering-application)
  - [Important Information for using in Server Side Rendering Applications](#important-information-for-using in-server-side-rendering-applications)
    - [SSR AsyncComponent Resolution Process](#ssr-asyncComponent-resolution-process)
    - [SSR Performance Optimisation](#ssr-performance-optimisation)
  - [Caveats](#caveats)
  - [FAQs](#faqs)

## Introduction

This library is an evolution of [`code-split-component`](). Unlike `code-split-component` this library does not require that you use either Webpack or Babel.  Instead it provides you a pure Javascript/React API which has been adapted in a manner to make it generically useful for lazy-loaded Components, with support for modern code splitting APIs (e.g `import()`, `System.import`, `require.ensure`).

## Usage

Firstly, you need to use our helper, which helps your application use asynchronous components in an efficient manner.

```jsx
import { withAsyncComponents } from 'react-async-component'; // 👈

const app = <MyApp />;

//                  👇
withAsyncComponents(app) // returns a Promise
  .then((result) => {
    ReactDOM.render(
      // We return a new version of your app that supports
      // asynchronous components.
      result.appWithAsyncComponents, // 👈
      document.getElementById('app')
    );
  });
```

Next up, let's make an asynchronous Component!  We provide another helper for this.

```jsx
import { createAsyncComponent } from 'react-async-component'; // 👈

const AsyncProduct = createAsyncComponent({
  // Provide a function that returns a Promise to the target Component.
  resolve: function resolveComponent() {
    return new Promise(function (resolve) {
      // The Promise the resolves with a simple require of the
      // `Product` Component.
      resolve(require('./components/Product'));
    });
  }
});

// You can now use the created Component as though it were a
// "normal" Component, providing it props that will be given
// to the resolved Component.
const x = <Product productId={10} />
```

The above may look a tad bit verbose.  If you are a fan of anonymous functions then we could provide a more terse implementation:

```jsx
const AsyncProduct = createAsyncComponent({
  resolve: () => new Promise(resolve =>
    resolve(require('./components/Product'))
  )
});
```

Okay, the above may not look terribly useful at first, but it opens up an easy point to integrating code splitting APIs supported by bundlers such as Webpack. We will provide examples of these as well as details on some other useful configuration options within the [`API`](#api) section. To wet your appetite though below is an example of using the `System.import` API provided by Webpack 2 to allow for code splitting on your async Component:

```jsx
const AsyncProduct = createAsyncComponent({
  resolve: () => System.import('./components/Product') // 🚀
});
```

## API

### `createAsyncComponent(config)`

Our async Component factory. Config goes in, an async Component comes out.

#### Arguments

  - `config` (_Object_) : The configuration object for the async Component. It has the following properties available:
    - `resolve` (_Function => Promise<Component>_) : A function that should return a `Promise` that will resolve the Component you wish to be async.
    - `ssrMode` (_Boolean_, Optional, default: 'render') : Only applies for server side rendering applications. We _highly recommend_ that you read the [SSR Performance Optimisation](#ssr-performance-optimisation) section for more on this configuration property.
    - `Loading` (_Component_, Optional, default: null) : A Component to be displayed whilst your async Component is being resolved. It will be provided the same props that will be provided to your resovled async Component.
    - `es6Aware` (_Boolean_, Optional, default: true) : If you are using ES2015 modules with default exports (i.e `export default MyComponent`) then you may need your Component resolver to do syntax such as `require('./MyComp').default`. Forgetting the `.default` can cause havoc with hard to debug error messages. To cover your back we will automatically try to resolve a `.default` on the result that is resolved by your Component. If the `.default` exists it will be used, else we will use the original result.
    - `name` (_String_, Optional, default: AsyncComponent) : Use this if you would like to name the created async Component, which helps when firing up the React Dev Tools for example.

#### Returns

A React Component.

### `withAsyncComponents(element)`

Decorates your application with the ability to use async Components in an efficient manner. It also manages state storing/rehydrating for server side rendering applications.

#### Arguments

 - `app` _React.Element_
   The react element representing your application.

#### Returns

A promise that resolves in a `result` object.  The `result` object will have the following properties available:

  - `appWithAsyncComponents` (_React.Element_) : Your application imbued with the ability to use async Components. ❗️Use this when rendering your app.
  - `state` (_Object_) : Only used on the "server" side of server side rendering applications. It represents the state of your async Components (i.e. which ones were rendered) so that the server can feed this information back to the client/browser.
  - `STATE_IDENTIFIER` (_String_) : Only used on the "server" side of server side rendering applications. The identifier of the property you should bind the `state` object to on the `window` object.

### Examples

#### Creating Async Components

##### Simple

```jsx
const AsyncProduct = createAsyncComponent({
  resolve: () => new Promise(resolve =>
    resolve(require('./components/Product'))
  )
});

<AsyncProduct productId={1} />
```

##### With Loading Component

```jsx
const AsyncProduct = createAsyncComponent({
  resolve: () => new Promise(resolve =>
    resolve(require('./components/Product'))
  ),
  Loading: ({ productId }) => <div>Loading product {productId}</div>
});

<AsyncProduct productId={1} />
```

##### Webpack 1/2 `require.ensure` Code Splitting

```jsx
const AsyncProduct = createAsyncComponent({
  resolve: () => new Promise(resolve =>
    require.ensure([], (require) => {
      resolve(require('./components/Product'));
    });
  )
});

<AsyncProduct productId={1} />
```

##### Webpack 2 `import` / `System.import` Code Splitting

Note: `System.import` is considered deprecated and will be replaced with `import`, but for now they can be used interchangeably (you may need a Babel plugin for the `import` syntax).

```jsx
const AsyncProduct = createAsyncComponent({
  resolve: () => System.import('./components/Product')
});

<AsyncProduct productId={1} />
```

##### Defer Loading to the Client/Browser

i.e. The component won't be resolved and rendered in a server side rendering execution.

```jsx
const AsyncProduct = createAsyncComponent({
  resolve: () => System.import('./components/Product'),
  defer: true
});
```

#### End to End Examples

The below examples show off a full workflow of using the `createAsyncComponent` and `withAsyncComponents` helpers.

In all of our examples below we are going to be making use of the `System.import` API supported by Webpack v2 to do asynchronous resolution of our Components and create code split points.

##### Browser Only Application

This is how you would use `react-async-component` in a "browser only" React application.

Let's imagine a Component that describes your application:

__MyApp.js__:
```jsx
//                     👇 create an async component
const AsyncHome = createAsyncComponent({
  resolve: () => System.import('./components/Home')
});

export default function MyApp() {
  return (
    <div>
      <h1>Hello world!</h1>
      <AsyncHome message="💖" />
    </div>
  );
}
```

And then a module that renders it:

__client.js__:
```jsx
import React from 'react';
import { render } from 'react-dom';
import { withAsyncComponents } from 'react-async-component'; // 👈
import MyApp from './components/MyApp';

const app = <MyApp />;

// 👇 run helper on your app.
withAsyncComponents(app)
  //        👇 and you get back a result object.
  .then((result) => {
    const {
      // ❗️ The result includes a decorated version of your app
      // that will allow your application to use async components
      // in an efficient manner.
      appWithAsyncComponents
    } = result;

    // Now you can render the app.
    render(appWithAsyncComponents, document.getElementById('app'));
  });
```

##### Server Side Rendering Application

This is how you would use `react-async-component` in a "server side rendering" React application.

Let's imagine a Component that describes your application:

__MyApp.js__:
```jsx
//                     👇 create an async component
const AsyncHome = createAsyncComponent({
  resolve: () => System.import('./components/Home'),
  ssrMode: 'boundary'
});

export default function MyApp() {
  return (
    <div>
      <h1>Hello world!</h1>
      <AsyncHome message="💖" />
    </div>
  );
}
```

And then a module that does the browser/client side rendering:

__client.js__:
```jsx
import React from 'react';
import { render } from 'react-dom';
import { withAsyncComponents } from 'react-async-component'; // 👈
import MyApp from './components/MyApp';

const app = <MyApp />;

// 👇 run helper on your app.
withAsyncComponents(app)
  //        👇 and you get back a result object.
  .then((result) => {
    const {
      // ❗️ The result will include a version of your app that is
      // rehydrated with the async component state returned by
      // the server.
      appWithAsyncComponents
    } = result;

    // Now you can render the app.
    render(appWithAsyncComponents, document.getElementById('app'));
  });
```

And then an express middleware (you could use any http server of your choice) to do the rendering:

__reactApplicationMiddlware.js__:
```jsx
import React from 'react';
import { withAsyncComponents } from 'react-async-component'; // 👈
import { renderToString } from 'react-dom/server';
import serialize from 'serialize-javascript';
import MyApp from './shared/components/MyApp';

export default function expressMiddleware(req, res, next) {
  const app = <MyApp />;

  // 👇 run helper on your app.
  withAsyncComponents(app)
    //        👇 and you get back a result object.
    .then((result) => {
      const {
        // ❗️ The result includes a decorated version of your app
        // that will have the async components initialised for
        // the renderToString call.
        appWithAsyncComponents,
        // This state object represents the async components that
        // were rendered by the server. We will need to send
        // this back to the client, attaching it to the window
        // object so that the client can rehydrate the application
        // to the expected state and avoid React checksum issues.
        state,
        // This is the identifier you should use when attaching
        // the state to the "window" object.
        STATE_IDENTIFIER
      } = result;

      const appString = renderToString(appWithAsyncComponents);
      const html = `
        <html>
          <head>
            <title>Example</title>
          </head>
          <body>
            <div id="app">${appString}</div>
            <script type="text/javascript">
              window.${STATE_IDENTIFIER} = ${serialize(state)}
            </script>
          </body>
        </html>`;
      res.send(html);
    });
}
```

### Important Information for using in Server Side Rendering Applications

This library is built to be used within server side rendering (SSR) applications, however, there are some important points/tips we would like to raise with you so that you can design your application components in an efficient and effective manner.

#### SSR AsyncComponent Resolution Process

It is worth us highlighting exactly how we go about resolving and rendering your `AsyncComponent` instances on the server. This knowledge will help you become aware of potential issues with your component implementations as well as how to effectively use our provided configuration properties to create more efficient implementations.

When running `react-async-component` on the server the helper has to walk through your react element tree (depth first i.e. top down) in order to discover all the AsyncComponent instances and resolve them in preparation for when you call the `ReactDOM.renderToString`. As it walks through the tree it has to call the `componentWillMount` method on your Components and then the `render` methods so that it can get back the child react elements for each Component and continue walking down the element tree. When it discovers an `AsyncComponent` instance it will first resolve the Component that it refers to and then it will continue walking down it's child elements (unless you set the configuration for your `AsyncComponent` to not allow this) in order to try and discover any nested `AsyncComponent` instances.  It continues doing this until it exhausts your element tree.

Although this operation isn't as expensive as an actual render as we don't generate the DOM it can still be quite wasteful if you have a deep tree.  Therefore we have provided a set of configuration values that allow you to massively optimise this process. See the next section below.

#### SSR Performance Optimisation

As discussed in the ["SSR AsyncComponent Resolution Process"](#ssr-asyncComponent-resolution-process) section above it is possible to have an inefficient implementation of your async Components.  Therefore we introduced a new configuration object property for the `createAsyncComponent` helper, called `ssrMode`, which provides you with a mechanism to optimise the configuration of your async Component instances.

The `ssrMode` configuration property supports the following values:

  - `'render'` (Default0) : In this mode a server side render will parse and resolve your `AsyncComponent` and then continue walking down through the child elements of your resolved `AsyncComponent`. This is the most expensive operation.
  - `'defer'` : In this mode your `AsyncComponent` will not be resolved during server side rendering and will defer to the browser/client to resolve and rendering it. This is the cheapest operation as the walking process stops immediately at that branch of your React element tree.
  - `'boundary'` : In this mode your `AsyncComponent` will be resolved and rendered on the server, however, the `AsyncComponent` resolution process will not try to walk down this components child elements in order to discover any nested AsyncComponent instances. If there are any nested instances their resolving and rendering will be deferred to the browser/client.

Understand your own applications needs and use the options appropriately . I personally recommend using mostly "defer" and a bit of "boundary". Try to see code splitting as allowing you to server side render an application shell to give the user perceived performance. Of course there will be requirements otherwise (SEO), but try to isolate these components and use a "boundary" as soon as you feel you can.

## Caveats

At the moment there is one known caveat in using this library: it doesn't support React Hot Loader (RHL). You can still use Webpack's standard Hot Module Replacement, however, RHL does not respond nicely to the architecture of `react-async-component`.

TODO: I'll post up some details why and perhaps we could work to find a solution.

## FAQs

> Let me know if you have any...
