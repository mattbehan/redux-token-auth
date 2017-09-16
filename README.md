# Redux Token Auth

[![CircleCI](https://circleci.com/gh/kylecorbelli/redux-token-auth.svg?style=shield)](https://circleci.com/gh/kylecorbelli/redux-token-auth)

[![codecov](https://codecov.io/gh/kylecorbelli/redux-token-auth/branch/master/graph/badge.svg)](https://codecov.io/gh/kylecorbelli/redux-token-auth)

<img src='https://californiadevlife.files.wordpress.com/2017/01/beer-token.jpg' height='300'>

This is the raddest client-side solution for user authentication using React and Redux with a Rails backend that uses Devise Token Auth.

## TL;DR
Given a Rails backend using Devise Token Auth, this module provides several asynchronous Redux Thunk actions to:
- Register a user (`registerUser`).
- Sign in a user (`signInUser`).
- Sign out a user (`signOutUser`).
- Verify the current user’s auth token (`verifyToken`).

It also provides the corresponding Redux reducer to handle these actions.

Additionally, a helper function is provided to verify the current user's auth token upon initialization of your application (`verifyCredentials`).

## Installation
`npm install --save redux-token-auth`

Have you ever installed an NPM module any other way?

## Dependencies
Your project will need the popular <a href="https://github.com/gaearon/redux-thunk" target="_blank">Redux Thunk</a> middleware (written by none other than the man, the myth, the legend Dan Abramov himself) in order to function properly.

## Making it Work
There are three main things you need to do in order to get `redux-token-auth` rigged up:
1. Integrate `reduxTokenAuthReducer` into your Redux store.
2. Generate the Redux Thunk actions and credential verification helper function.
3. Call `verifyCredentials` in your `index.js` file.

`redux-token-auth` has two exports: a Redux reducer, and a function that generates a handful of asynchronous Redux Thunk actions, and a helper function that verifies the current user's credentials as stored in the browser's `localStorage`. React Native equivalents using `AsyncStorage` are roadmapped but not yet supported.
### 1. Redux Reducer
`redux-token-auth` ships with a reducer to integrate into your Redux store. Wherever you define your root reducer, simply import and include `reduxTokenAuthReducer` in your call to `combineReducers`:
```JavaScript
import { combineReducers } from 'redux'
import { reduxTokenAuthReducer } from 'redux-token-auth'

const rootReducer = combineReducers({
  reduxTokenAuth: reduxTokenAuthReducer,
})

export default rootReducer
```
We'll note here again that you need <a href="https://github.com/gaearon/redux-thunk" target="_blank">Redux Thunk</a> integrated into your store in order for `redux-token-auth` to work properly.

### 2. Generate Actions and Helper Function
`redux-token-auth` provides a function called `generateAuthActions` that takes a config object and returns the asynchronous Redux Thunks actions and the helper function to verify the user’s credentials upon initialization of your application. The following paragraphs explain the config object.
#### Auth URL

In order for `redux-token-auth` to communicate with your backend, you'll need to provide it with the base URL for your authentication endpoint. It's often a good idea to place this URL somewhere in a config file that sets it depending on the environment (development, test, or production). Here we'll assume you have a file called `constants.js` in the root directory of your project that does just that. The URL should be the full URL **not** end with `/`. For example `https://radapp.io/auth` or `http://localhost:300/auth`.

Now we'll be adding that URL to a config object that has three keys:
1. `authUrl`
2. `userAttributes`
3. `userRegistrationAttributes`

#### User and User Registration Attributes

The `userAttributes` and `userRegistrationAttributes` values are themselves objects that contain the shape of your `User` model. `redux-token-auth` will parse these objects in order to send the appropriate request bodies as well as to interpret response data to populate your client-side Redux state.

Regarding the difference between `userAttributes` and `userRegistrationAttributes`, while `userAttributes` contains all the attributes of your `User` model, `userRegistrationAttributes` contains only those attributes necessary for registration.

For each of these objects, the keys will be the names of the attributes as known to your frontend and the values will be strings that are the names of the attributes as known to your backend. An example would be:
```javascript
userAttributes: {
  firstName: 'first_name',
  lastName: 'last_name',
  imageUrl: 'image',
}
```

This is to account for the different case conventions between JavaScript and Ruby, and allows `redux-token-auth` to facilitate communication between the two. It can also bridge the gap between however you want to name your user attributes on the backend versus how you want to name them on the frontend. Maybe you want to call it `imageUrl` on the frontend and `image` on the backend. It’s your call. Go nuts.

It is important to note that `email` and `password` should **not** be included in these objects as they are already accounted for and adding them may result in unexpected behavior.

#### Bringing the Config Together

Create a file called something like `redux-token-auth-config.js` in the root directory of your project. Honestly, it doesn't need to be named that but that's what we'll call it here. Open that file and import `generateAuthActions` from the `redux-token-auth`. As noted above, this is a function that takes your config object as its only input. It returns an object containing several named Redux Thunk actions and the helper function to verify user credentials upon initialization of your app. Here's an example:

```JavaScript
// redux-token-auth-config.js
import { generateAuthActions } from 'redux-token-auth'
import { authUrl } from './constants'

const config = {
  authUrl,
  userAttributes: {
    firstName: 'first_name',
    imageUrl: 'image',
  },
  userRegistrationAttributes: {
    firstName: 'first_name',
  },
}

const {
  signInUser,
  signOutUser,
  registerUser,
  verifyCredentials,
} = generateAuthActions(config)

export {
  registerUser,
  signInUser,
  signOutUser,
  verifyCredentials,
}
```
Simply export these functions from your config file. Now they're available throughout your app by importing them from your config file.

`registerUser`, `signInUser`, and `signOutUser` are Redux Thunk actions and thus return Promises.

An example of using one of these functions would be in your sign in form:

```JavaScript
// components/SignInScreen.js
import React, { Component } from 'react'
import { connect } from 'react-redux'
import { signInUser } from '../redux-token-auth-config' // <-- note this is YOUR file, not the redux-token-auth NPM module

class SignInScreen extends Component {
  constructor (props) { ... }

  submitForm (e) {
    e.preventDefault()
    const { signInUser } = this.props
    const {
      email,
      firstName,
      password,
    } = this.state
    signInUser({ email, firstName, password }) // <-<-<-<-<- here's the important part <-<-<-<-<-
      .then(...)
      .catch(...)
  }

  render () {
    <div>
      <form onSubmit={this.submitForm}>...</form>
    </div>
  }
}

export default connect(
  null,
  { signInUser },
)(SignInScreen)
```

### 3. Verifying User Credentials on App Initialization
Upon initialization of your app, your user could potentially be logged in from their previous session and your application state should reflect that. `redux-token-auth` stores an authenticated user’s auth token in `localStorage`. In order to sync the stored token with both your backend and your Redux store, you’ll use the `verifyCredentials` function that was returned from `generateAuthActions`. In short, it checks for a token, sends a verification request to the backend, and upon receiving a successful response, updates your Redux store. Here's an example of how you wire it up in your `index.js` file, with a single line:

```JavaScript
import * as React from 'react';
import * as ReactDOM from 'react-dom';
import { Provider } from 'react-redux'
import { Store } from 'redux'
import App from './components/App'
import configureStore from './redux/configure-store'
import { verifyCredentials } from './redux-token-auth-config' // <-- note this is YOUR file, not the redux-token-auth NPM module

const store = configureStore()
verifyCredentials(store) // <-<-<-<-<- here's the important part <-<-<-<-<-

ReactDOM.render(
  <Provider store={store}>
    <App />
  </Provider>,
  document.getElementById('root'),
)
```

And that's really all there is to it!

## Roadmap
- React Native support
- Password reset actions
- Email verification actions

## Contributors
- Kyle Corbelli