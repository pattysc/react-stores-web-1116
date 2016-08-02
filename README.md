# React Stores

## Objectives

1. Explain how to move state out of components and into stores
2. Describe how stores update based on actions
3. Describe how components subscribe to stores

## Overview

We're almost at full flux — we're just missing the dispatcher, which means that
stores and actions need to be aware of each other for the time being. Our
implementation might look like this:

```javascript
// stores/Store.js
module.exports = class Store {
  constructor(state) {
    this.listeners = []
    this.state = state
  }

  addListener(l) {
    this.listeners.push(l)

    return this.listeners.length - 1
  }

  removeListener(id) {
    this.listeners = [
      ...this.listeners.slice(0, id),
      ...this.listeners.slice(id + 1)
    ]
  }

  updateState(state) {
    this.state = state

    this.listeners.forEach(l => l())
  }
}

// stores/togglerStore.js
const Store = require('./Store')

const initialState = {
  on: false
}

module.exports = new Store(initialState)

// actions/togglerActions.js
const togglerStore = require('../stores/togglerStore')

function handleClick(e) {
  e.preventDefault()

  // we could update multiple stores here
  togglerStore.updateState({ on: !togglerStore.state.on })
}

// components/Toggler.js
const React = require('react')

const togglerActions = require('../actions/togglerActions')
const togglerStore = require('../stores/togglerStore')

module.exports = class Toggler extends React.Component {
  constructor(props) {
    super(props)

    this.handleClick = togglerActions.handleClick.bind(this)
    this.updateState = this.updateState.bind(this)

    this.state = togglerStore.state
  }

  componentDidMount() {
    this.listenerId = togglerStore.addListener(this.updateState)
  }

  componentWillUnmount() {
    togglerStore.removeListener(this.listenerId)
  }

  render() {
    return <a onClick={this.handleClick}>{this.state.on}</a>
  }

  updateState(state) {
    // we can also perform other transformations here
    this.setState(state)
  }
}
```

## Resources

- [React: Flux Overview](https://facebook.github.io/flux/docs/overview.html)
- [Actions and the Dispatcher](https://facebook.github.io/flux/docs/actions-and-the-dispatcher.html#content)
