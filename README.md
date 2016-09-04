# React Stores

## Objectives

1. Explain how to move state out of components and into stores
2. Describe how stores update based on actions
3. Describe how components subscribe to stores

## Overview

So far we have learned about a couple of ways to make our application and
component structures more modular: We abstracted out common event handlers into
reusable actions learned how to compose components together. In this lesson,
we're going to learn about a way to share state between components using a
concept called "store".

## What is a store?

A store is basically just a plain JavaScript object that allows components to
share state.

In a way, we can think of a store as a database. On the most fundamental
level, both constructs allow us to store data in some form or another.

In Flux-terms, a store typically has a couple of methods that we're going to go
into more detail as part of this lesson. Usually we won't create our own
stores from scratch, but instead simply customize one of the stores provided by
library that we're using (usually
[facebook/flux](https://www.npmjs.com/package/flux)).

## Stores — singletons by design

Singletons are application-level singletons. While there might be a variety of
different stores (such as a `UserStore`, `FeedStore`, `MessageStore` etc.),
there won't be multiple instances of the same store. Typically this means
exporting an individual store object and globally exposing it to all
component's that `require` it.

This is somewhat analogous to our database metaphora: You might store different
"kinds" of data that you store in different databases (e.g. you might 
unstructured data in MongoDB and relational data in something like PostgreSQL),
but all clients share the "same" database. Each application node has access to
the same data.

On the most fundamental level, a store encapsulates state. We can easily model
this using a ES6 class:

```js
class Store {
  constructor (initialState) {
    this.state = initialState;
  }

  setState (state) {
    this.state = state;
  }

  getState () {
    return this.state;
  }
}

module.exports = new Store({});
```

## Subscribing to stores

Of course having a store that simply wraps our state object isn't too useful
yet. React components are data-driven. If we update their state using
`setState`, React re-evaluates the component's `render` function and updates
the rendered DOM-structure.

Hence we need a way to wire up our components to our global store. In some
way, components need to be able to "listen" for state changes that occur in out
store:

![Flux Store](./assets/Flux - Store.png)

An arbitrary number of components can subscribe to state changes that occur in
the store. Component's can then react to the state change by updating their own
state and thus triggering a re-render.

But how can component's register themselves at the store?

Let's look at an example component for that!

## A `<Profile />` component

Let's assume for a moment that our store stores user data, e.g. the name and
profile descriptions of individual members (something like Facebook profile).

Each user can be represented by a flat object:

```js
{
  id: 0,
  firstName: 'Konrad',
  lastName: 'Zuse',
  bio: 'I like building stuff.'
}
```

Our store simply wraps an array of user records:

```js
class UserStore {
  constructor (initialState) {
    this.state = initialState;
  }

  setState (state) {
    this.state = state;
  }

  getState () {
    return this.state;
  }
}

module.exports = new UserStore([]);
```

Our profile component now renders the state of the `UserStore` component:

```js
const userStore = require('../stores/userStore');

class Profile extends React.Component {
  render () {
    const {userId} = this.props;
    const profile = userStore.find((user) => user.id === userId);

    if (!profile) {
      return (
        <div>Loading...</div>
      );
    }

    return (
      <dl>
        <dt>First Name</dt>
        <dd>{profile.firstName}</dd>
        <dt>Last Name</dt>
        <dd>{profile.lastName}</dd>
        <dt>Bio</dt>
        <dd>{profile.bio}</dd>
      </dl>
    );
  }
}
```

We're almost there now. Currently we assume that the user that should be
rendered is available in the store at the point when we render the
`<Profile />` component.

This might not always be the case. E.g. our users are most likely going to be
loaded from some sort of API. What if we render the component before 
respective HTTP request receives a response? In this case, our profile
component would simply display `Loading...` forever.

We therefore need to notify subscribed components about eventual state changes.
This sounds conceptually very similar to an event emitter and in fact that's
exactly what we're going to implement!

We can either inherit from some event emitter (e.g.
`require('events').EventEmitter`) or we can implement our own. To recap the
inner-workings of how an event emitter works, let's implement our own for now!

## Turning our store into an `EventEmitter`

Our store needs to expose an API for registering components that depend on its
state. Analogous to an `EventEmitter`, we call the respective method
`addListener` and `removeListener`.

Both methods accept a listener function that will be called with the updated
state object whenever the store's state changes.

To start off, we need to store an array of those listener functions on the
initiated store object:

```js
class UserStore {
  constructor (initialState) {
    this.state = initialState;

    // Our listener functions will be stored on UserStore#listeners.
    this.listeners = [];
  }

  // ...
}
```

Now components can register themselves using `UserStore.addListener`. The
`addListener` function is simply going to add the listener to the `listeners`
array.

```js
class UserStore {
  // ...
  addListener (listener) {
    this.listeners.push(listener);
  }
}
```

The `UserStore` iterates through all registered event listeners whenever a
state change occurs. All listeners will be called with the updated state as a
first argument:

```js
class UserStore {
  // ...
  setState (state) {
    this.state = state;
    for (let listener of this.listeners) {
      listener.call(null, state);
    }
  }
}
```

And the `<Profile />` component can now call this method in order to register
for subsequent state changes. In order to trigger a re-render whenever the
store gets updated, we copy the store's state onto the component's state:

```js
class Profile extends React.Component {
  componentDidMount () {
    userStore.addListener((state) => {
      this.setState(state)
    });
    this.setState(userStore.getState())
  }
  render () {
    const {userId} = this.props;

    // We're now accessing `this.state` instead of `userStore`.
    const profile = this.state.find((user) => user.id === userId);

    // ...
  }
}
```

And Voila! Our component is now wired up to our store.
There is just one small (but very important!) issue which we didn't address,
yet!

What happens when the component is being unmounted? In other words, what
happens when the user leaves the profile page and goes somewhere else?

Subsequent store updates would trigger a `setState` on an unmounted component!
Not only is this going to trigger a warning in React, it's also a very common
source for memory leaks in Flux architectures. Whenever we register components
on a store, we have to make sure we remove all listeners at the point where the
component is being unmounted.

`EventEmitter`s usually expose a `removeListener` method for cleaning up event
listeners, but it's quite common for Flux stores to simply return a function
from `addListener` that when called removes the listener from the store. Sounds
complicated? Let's look at the code!

```js
class UserStore {
  addListener (listener) {
    this.listeners.push(listener);
    const removeListener = () => {
      this.listeners = this.listeners.filter((l) => listener !== l);
    };
    return removeListener;
  }
}
```

Now we can update our `<Profile />` component's lifecycle methods to trigger
the `removeListener` method on "unmount":

```js
class Profile extends React.Component {
  // ...
  componentDidMount () {
    // We store a reference to the added event listener.
    this.removeListener = userStore.addListener((state) => {
      this.setState(state)
    });
    this.setState(userStore.getState())
  }
  componentWillUnmount () {
    // Destroy the listener when the component unmounts.
    this.removeListener();
  }
  // ...
}
```

And that's it! Our `<Profile />` component gets updated whenever we `UserStore`
gets updated.

## Store API

Let's take a moment and revisit the store methods we just added:

### `addListener(listener) #=> removeListener()`

Called by components to register a listener function. The listener function
will be called with the updated state object.

The returned `removeListener` function de-registers the previously registered
listener function.

## `getState() #=> state`

Used by components to get the initial state of the store. Returns the currently
encapsulated state object.

## `setState(state)`

Explicitly stores the passed in state on the store instance. In the next lesson
we're going to explore how to generalize data-flow in our application to allow
actions to mutate stores via a kind of central event bus ("Dispatcher").

## Resources

- [React: Flux Overview](https://facebook.github.io/flux/docs/overview.html)
- [Actions and the Dispatcher](https://facebook.github.io/flux/docs/actions-and-the-dispatcher.html#content)
