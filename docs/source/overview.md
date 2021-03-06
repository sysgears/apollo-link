---
title: Overview
---

<h2 id="overview">Link Concepts Overview</h2>

Apollo Link is designed to be a powerful way to compose actions around data handling with GraphQL. Each link represents a subset of functionality that can be composed with other links to create complex control flows of data. At a basic level, a link is a function that takes an operation and returns an observable. Described with types, it looks like this:

```js
type Context = Object;

interface Operation {
  query: DocumentNode;
  variables: Object;
  operationName: string;
  extensions?: Object;
  getContext(): Context;
  setContext(newContext: Context | (prevContext: Context) => Context): void;
  toKey(): string;
}

// this is what makes up an Apollo Link
type RequestHandler = (operation: Operation) => Observable<ExecutionResult>
```

<h3 id="request">Request</h3>

At the core of a link is the `request` method. A link's request is called every time `execute` is run on that link chain (typically every operation). The request is where the operation is given to the link to return back data of some kind. Request must return an observable. Depending on where the link is in the stack, it will either use the second parameter to a link (the next link in the chain) or return back an `ExecutionResult` on its own.

The full description of a link's request looks like this:

```js
type NextLink = (operation: Operation) => Observable<ExecutionResult>

type RequestHandler = (operation: Operation, forward: NextLink) => Observable<ExecutionResult>
```

As you can see from these types, the next link is a way to continue the chain of events until data is fetched from some data source (typically a server).

<h3 id="terminating">Terminating Links</h3>

Since link chains have to fetch data at some point, they have the concept of a `terminating` link and `non-terminating` links. Simply enough, the `terminating` link is the one that doesn't use the `forward` argument, but instead turns the operation into the result directly. Typically this is done with a network request, but the possibilities are endless of how this can be done. The terminating link is the last link in the composed chain.

<h3 id="composition">Composition</h3>

Links are designed to be composed together to form control flow chains to manage a GraphQL operation request. They can be used as middleware to perform side effects, modify the operation, or even just provide developer tools like logging. They can be afterware which process the result of an operation, handle errors, or even save the data to multiple locations. Links can make network requests including HTTP, WebSockets, and even across the react-native bridge to the native thread for resolution of some kind.

When writing a `RequestHandler`, the second argument is the way to call the next link in the chain. We call it `forward` in the docs for a couple reasons. First, `observers` have a `next` function for sending new results to the subscriber. Second, if you think of composed links like a chain, the request goes `forward` until it gets data (for example from a server request), then it begins to go `back` up the chain to any subscriptions. The `forward` function allows the `RequestHandler` to continue the process to the next link in the chain.

Composition is done using helper functions exported from the `apollo-link` package, or conveniently located on the `ApolloLink` class itself. These helpers are explained more [here](./composition.html). We suggest using the helpers directly on the `ApolloLink` however to make it easier to read and reason about what is being done in your code.

<h3 id="context">Context</h3>

Since links are meant to be composed, they need an easy way to send metadata about the request down the chain of links. They also need a way for the operation to send specific information to a link no matter where it was added to the chain. To accomplish this, each `Operation` has a `context` object which can be set from the operation while being written and read by each link. The context is read by using `operation.getContext()` and written using `operation.setContext(newContext)` or `operation.setContext((prevContext) => newContext)`. The `context` is *not* sent to the server, but is used for link to link communication. For example:

```js
const timeStartLink = new ApolloLink((operation, forward) => {
  operation.setContext({ start: new Date() });
  return forward(operation);
});

const logTimeLink = new ApolloLink((operation, forward) => {
  return forward(operation).map((data) => {
    // data from a previous link
    const time = new Date() - operation.getContext().start;
    console.log(`operation ${operation.operationName} took ${time} to complete`);
    return data;
  })
});

const link = timeStartLink.concat(logTimeLink)
```

Each context can be set by the operation it was started on. For example with Apollo Client:

```js
const link = new ApolloLink((operation, forward) => {
  const { saveOffline } = operation.getContext();
  if (saveOffline) // do offline stuff
  return forward(operation);
})

const client = new ApolloClient({
  cache: new InMemoryCache()
  link,
});

// send context to the link
const query = client.query({ query: MY_GRAPHQL_QUERY, context: { saveOffline: true }});
```
