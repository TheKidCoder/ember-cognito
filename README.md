# ember-cognito
## An authenticator and library for using ember-simple-auth and Amazon Cognito

[![Build Status](https://travis-ci.org/paulcwatts/ember-cognito.svg?branch=master)](https://travis-ci.org/paulcwatts/ember-cognito)
[![Code Climate](https://codeclimate.com/github/paulcwatts/ember-cognito/badges/gpa.svg)](https://codeclimate.com/github/paulcwatts/ember-cognito)
[![Test Coverage](https://codeclimate.com/github/paulcwatts/ember-cognito/badges/coverage.svg)](https://codeclimate.com/github/paulcwatts/ember-cognito/coverage)
[![Greenkeeper badge](https://badges.greenkeeper.io/paulcwatts/ember-cognito.svg)](https://greenkeeper.io/)

Ember Cognito is an Ember Addon that integrates [ember-simple-auth](https://github.com/simplabs/ember-simple-auth/) 
with [Amazon Cognito](https://aws.amazon.com/cognito/) User Pools. 

ember-simple-auth is a lightweight library for implementing authentication/authorization in Ember.js. 
Amazon Cognito is a managed authentication system for mobile and web apps on Amazon Web Services.

ember-cognito implements an ember-simple-auth custom authenticator that can be used to authenticate with 
a Cognito User Pool. It also provides some helper classes that convert Cognito's callback-oriented API to
a promise-oriented API.

## Installation

Install as a standard Ember Addon:

* `ember install ember-cognito`

## Configure

In your `config/environment.js`  file:

```js
var ENV = {
  // ..
  cognito: {
    poolId: '<your Cognito User Pool ID>',
    clientId: '<your Cognito App Client ID>',
  }
};
```

Note that the Cognito JavaScript SDK requires that your App be created *without* a Client Secret.

## Usage

### Cognito Authenticator

The Cognito Authenticator authenticates an ember-simple-auth session with Amazon Cognito. It overwrites
the Amazon SDK's storage mechanism (which is hardcoded to use localStorage) to instead store the authentication
data in ember-simple-auth's session-store. This makes it easier to integrate Cognito in Ember unit tests.

You will call the authenticator just as you would a normal authenticator:

```js
import Ember from 'ember';

export default Ember.Controller.extend({
  session: Ember.inject.service('session'),

  actions: {
    authenticate() {
      let { username, password } = this.getProperties('username', 'password');
      this.get('session').authenticate('authenticator:cognito', username, password).then((cognitoUserSession) => {
        // Successfully authenticated!  
      }).catch((error) => {
        this.set('errorMessage', error.message || error);
      });
    }
  }
});
```

### Integrating with an Authorizer

The Cognito Authenticator will put the Cognito ID token in the `access_token` property in the session's
`authenticated` data. This means integrating with an Authorizer such as the OAuth2 Bearer requires no 
special configuration.

### Cognito SDK Vendor Shim

ember-cognito provides a vendor shim that allows you to import [Cognito Identity SDK's](https://github.com/aws/amazon-cognito-identity-js/) 
classes using ES modules:

```js
 import { CognitoUserPool, CognitoUserAttribute, CognitoUser } from 'amazon-cognito-identity-js';
```

### CognitoUser

[CognitoUser](https://github.com/paulcwatts/ember-cognito/blob/master/addon/utils/cognito-user.js) is a helper 
Ember object that provides promisified versions of the Cognito Identity SDK's callback methods.
For instance, you can take this code from the Cognito SDK:

```js
import { CognitoUser } from 'amazon-cognito-identity-js';

cognitoUser.getUserAttributes(function(err, result) {
  if (err) {
    alert(err);
    return;
  }
  for (let i = 0; i < result.length; i++) {
    console.log('attribute ' + result[i].getName() + ' has value ' + result[i].getValue());
  }
});
```

And rewrite it to use Promises and other ES2015 goodness:

```js
import { CognitoUser } from 'ember-cognito/utils/cognito-user';

cognitoUser.getUserAttributes().then((userAttributes) => {
  userAttributes.forEach((attr) => {
    console.log(`attribute ${attr.getName()} has value ${attr.getValue()}`);
  });
}).catch((err) => {
  alert(err);
});
```

### CognitoService

The Cognito service allows you to access the currently authenticated `CognitoUser` object. 
For instance, if you use a `current-user` service using the 
[ember-simple-auth guide](https://github.com/simplabs/ember-simple-auth/blob/master/guides/managing-current-user.md), 
you can use the Cognito Service to fetch user attributes:

```js
import Ember from 'ember';

export default Ember.Service.extend({
  session: Ember.inject.service(),
  cognito: Ember.inject.service(),
  cognitoUser: Ember.computed.readOnly('cognito.user'),
  username: Ember.computed.readOnly('cognitoUser.username'),

  load() {
    if (this.get('session.isAuthenticated')) {
      return this.get('cognitoUser').getUserAttributes().then((userAttributes) => {
        userAttributes.forEach((attr) => {
          this.set(attr.getName(), attr.getValue());
        });
      });
    } else {
      return Ember.RSVP.resolve();
    }
  }
});
```

See [the dummy app](https://github.com/paulcwatts/ember-cognito/blob/master/tests/dummy/app/services/current-user.js)
for an example of this in action.

## Support

ember-cognito is tested on Ember versions 2.4, 2.8, and 2.12+.
