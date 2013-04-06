ember-auth
==========

`ember-auth` provides token authentication support to
[ember.js](http://emberjs.com/).

Overview
========

`ember-auth` expects your server to implement token authentication.
It will then query for this token upon sign in; append this token for every
API request generated by the model query and persistence; and destroy this
token upon sign out.

**Important!** `ember-auth` is no replacement for secure server-side API code.
Read the [security page](https://github.com/heartsentwined/ember-auth/wiki/Security) for more information.

Installation
============

Read the [installation notes](https://github.com/heartsentwined/ember-auth/wiki/Install).

Getting started
===============

A [demo and tutorial](https://github.com/heartsentwined/ember-auth-rails-demo)
for rails + devise + ember-auth is available.

Pre-req
=======

`ember-auth` expects your server to provide an API interface with
token authentication support.

The token authentication API should expose two end points:
* a `POST` route for token creation
  * This should return the authentication token and the user model ID
    upon successful token creation in a JSON object.
* a `DELETE` route for token destruction

Your server should also use proper HTTP status codes, i.e. the 2xx series for
successful actions, and the 4xx series for errors. In particular, the following
will be relevant:
* `200 OK`:           general pages, or returning protected content upon
                      successful authentication
* `201 Created`:      suggested response code for successful token creation
* `400 Bad Request`:  suggested response code for wrong parameters when
                      consuming an API end point
* `401 Unauthorized`: trying to access protected content without proper
                      authentication
* `404 Not Found`:    ah, good old 404.

At present `ember-auth` only supports `DS.RESTAdapter`.

Usage
=====

Config
------

### Minimum requirement

Let's say your server exposes a token authentication interface as follows:
* `POST /users/sign_in` for token creation (sign in)
  * expects `email` and `password` as params
  * sample response: `{user_id: 1, auth_token: "jL3hbrhni82yxIHUD"}`
* `DELETE /users/sign_out` for token destruction (sign out)
  * expects `auth_token` as param
  * (no response requirement)

```coffeescript
Auth.Config.reopen
  tokenCreateUrl: '/users/sign_in'
  tokenDestroyUrl: '/users/sign_out'
  tokenKey: 'auth_token'
  idKey: 'user_id'
```

### Auto-load current user object

Let's say you have defined the corresponding user model as `App.User`.
You can have `ember-auth` auto-load the current user object with `userModel`:

```coffeescript
Auth.Config.reopen
  userModel: App.User
```

(Note that it is *not* a string `'App.User'`.)

`ember-auth` will call `App.User.find()` with `Auth.currentUserId`, which
would ultimately come from the user ID key in your server's JSON response.

### Different base URL

You can specify a different API host with `baseUrl`:

```coffeescript
Auth.Config.reopen
  baseUrl: 'https://api.example.com'
```

### Different token locations

`ember-auth` supports three different methods of sending the authentication
token on API requests:

* include it as one of the key-value pairs in the data/params hash (default)
* use the Authorization header ([RFC 1945](http://tools.ietf.org/html/rfc1945#section-10.2))
* send it in a custom header

#### Key-value pair in data/params hash

Send `auth_token = auth_token_value` alongside your other API request params:

```coffeescript
Auth.Config.reopen
  requestTokenLocation: 'param' # or omit this line - this is the default value
  tokenKey: 'auth_token'
```

#### Authorization header

Send a `Authorization: TOKEN auth_token_value` header:

```coffeescript
Auth.Config.reopen
  requestTokenLocation: 'authHeader'
  requestHeaderKey: 'TOKEN'
```

#### Custom header

Send a `X-API-TOKEN: auth_token_value` header:

```coffeescript
Auth.Config.reopen
  requestTokenLocation: 'customHeader'
  requestHeaderKey: 'X-API-TOKEN'
```

Persistence adapter
-------------------

Persistence adapter setup: you will use `Auth.RESTAdapter`; it is an
extension of `DS.RESTAdapter`.

```coffeescript
App.Store = DS.Store.extend
  revision: 11 # or whatever suitable
  adapter: Auth.RESTAdapter.create()
```

Sign in/out views and templates
-------------------------------

### 1. Widget style

"Widget" style sign in/out forms, for example at the end of a navigation bar.
The distinctive characteristic is that they do not have dedicated routes;
and that they are contained in a small view area within the app.

Make a `view` and a `template` for the authorization form area:

```coffeescript
App.AuthView = Em.View.extend
  templateName: 'auth'
```

```html
<script type="text/x-handlebars" data-template-name="auth">
  {{#if Auth.authToken}}
    {{view App.SignOutView}}
  {{else}}
    {{view App.SignInView}}
  {{/if}}
</script>
```

Note the use of `Auth.authToken` as the condition. `ember-auth` will store
the authentication token here when "signed in", and it will set it to `null`
when "signed out".

The sign in form:

```coffeescript
App.SignInView = Em.View.extend
  templateName: 'sign_in'

  email:    null
  password: null

  submit: (evt, view) ->
    evt.preventDefault()
    evt.stopPropagation()
    Auth.signIn
      email:    @get 'email'
      password: @get 'password'
```

```html
<script type="text/x-handlebars" data-template-name="sign_in">
  <form>
    <label>Email</label>
    {{view Ember.TextField valueBinding="view.email"}}
    <label>Password</label>
    {{view Ember.TextField type="password" valueBinding="view.password"}}
    <button>Sign In</button>
  </form>
</script>
```

Here we use the `Auth.signIn` helper. It accepts an object of params that will
be passed to the API call.

The sign out form:

```coffeescript
App.SignOutView = Em.View.extend
  templateName: 'sign_out'

  submit: (evt, view) ->
    evt.preventDefault()
    evt.stopPropagation()
    Auth.signOut()
```

```html
<script type="text/x-handlebars" data-template-name="sign_out">
  <form>
    <button>Sign Out</button>
  </form>
</script>
```

The `Auth.signOut` helper has the same signature as the `Auth.signIn` helper,
except that it will pass the authentication token as a parameter by default,
at the key specified in `Auth.Config.tokenKey`.
The [FAQ](https://github.com/heartsentwined/ember-auth/wiki/FAQ) has an
explanation of this default behavior.

### 2. Full page style

"Full page" style, e.g. a "sign_in" route where the sign in form itself
is the main content of the whole page.
The distinctive characteristic is that sign in / out pages have their own routes.

Make a `route`, a `controller` and a `template` for the sign in page:

```coffeescript
App.Router.map ->
  @route 'sign_in'
```

```coffeescript
App.SignInRoute = Ember.Route.extend()
```

```coffeescript
App.SignInController = Ember.ObjectController.extend
  email: null
  password: null

  signIn: ->
    Auth.signIn
      email:    @get 'email'
      password: @get 'password'
```

```html
<script type="text/x-handlebars" data-template-name="sign_in">
  <form>
    <label>Email</label>
    {{view Ember.TextField valueBinding="email"}}
    <label>Password</label>
    {{view Ember.TextField type="password" valueBinding="password"}}
    <button {{action "signIn"}}>Sign In</button>
  </form>
</script>
```

We register a `signIn` action to our Sign In button, and then use the
`Auth.signIn` helper to sign the user in. The `Auth.signIn` helper is
explained in the [Widget style section](#1-widget-style).

The sign out page:

```coffeescript
App.Router.map ->
  @route 'sign_out'
```

```coffeescript
App.SignOutRoute = Ember.Route.extend()
```

```coffeescript
App.SignOutController = Ember.ObjectController.extend
  signOut: ->
    Auth.signOut()
```

```html
<script type="text/x-handlebars" data-template-name="sign_out">
  <form>
    <button {{action "signOut"}}>Sign Out</button>
  </form>
</script>
```

Again, we register a `signOut` action on the button; the `Auth.signOut` helper
is explained in the [Widget style section](#1-widget-style).

Authenticated requests
----------------------

Using the Auth.RESTAdapter ensures all requests are authenticated by passing the
auth token as a parameter. If you need to make an authenticated request that
does not use the adapter you can call Auth.ajax directly.

For example, you can make the following call from inside any controller:

```coffeescript
Auth.ajax({ url: '/api/non_standard_route', type: 'POST', foo_key: 'bar_data'})
```

Authenticated-only routes
-------------------------

Authenticated-only routes setup: you will use `Auth.Route`; it is an
extension of `Ember.Route`.

The route `panel` - let's say, pointing to the user control panel - should
be a protected route:

```coffeescript
App.PanelRoute = Auth.Route.extend()
```

`Auth.Route` does nothing by default. It is there to provide a center place for
implement any `route`- and authentication-related logic:

```coffeescript
Auth.Route.reopen
  # do something
```

However, see Redirects section right below for built-in redirection support.

The Remember Me module also adds behavior to `Auth.Route`.

Redirects
---------

`ember-auth` provides five kinds of redirects to assist in building your UI.
All these require a `route` to redirect to, so they won't make sense if you
are using only the widget-style UI. ([Why?](https://github.com/heartsentwined/ember-auth/wiki/FAQ))

### Authenticated-only routes

You can have non-authenticated ("not signed in") users redirected to a
sign in route - let's say, named `sign_in` - when they visit an `Auth.Route`:

```coffeescript
Auth.Config.reopen
  signInRoute: 'sign_in'
  authRedirect: true
```

It is a good idea to make your sign out route authenticated-only with redirection:
non-authenticated users should not try to sign out; and in any case your
server API should reject sign out request from non-authenticated users anyway.

```coffeescript
App.SignOutRoute = Auth.Route.extend()
```

### Post- sign in redirect: fixed route

You can have the user redirected to a specified route - let's say, 'account' -
after signing in.

```coffeescript
Auth.Config.reopen
  signInRedirectFallbackRoute: 'account' # defaults to 'index'
```

You will need to modify your controller. Include the mixin
`Auth.SignInController`, and call its `registerRedirect` method from your
sign in action.

```coffeescript
# first line changed
App.SignInController = Ember.ObjectController.extend Auth.SignInController,
  email: null
  password: null

  signIn: ->
    @registerRedirect() # also changed here
    Auth.signIn
      email:    @get 'email'
      password: @get 'password'
```

### Post- sign in redirect: smart mode

"Smart" redirect. After sign in, the user is redirected to:
* one's previous route, unless one comes from the `signInRoute`
* the fallback route otherwise

Let's say your sign in route is called `sign_in`, and you want the fallback
route to be `account`

```coffeescript
Auth.Config.reopen
  signInRoute: 'sign_in'
  smartSignInRedirect: true
  signInRedirectFallbackRoute: 'account' # defaults to 'index'
```

Same modification to `controller` as the
[post- sign in fixed route redirect](#post--sign-in-redirect-fixed-route).

### Post- sign out redirect: fixed route

You can have the user redirected to a specified route - let's say, 'home' -
after signing out.

```coffeescript
Auth.Config.reopen
  signOutRedirectFallbackRoute: 'home' # defaults to 'index'
```

You will need to modify your controller. Include the mixin
`Auth.SignOutController`, and call its `registerRedirect` method from your
sign out action.

```coffeescript
# first line changed
App.SignOutController = Ember.ObjectController.extend Auth.SignOutController,
  signOut: ->
    @registerRedirect() # also changed here
    Auth.signOut()
```

### Post- sign out redirect: smart mode

This is rather awkward.
Do you really have a use case, where you will implement a logic that
auto-redirects the user to your sign out route in the first place,
such that "smart" redirecting the user back from one's previous route will
make sense? Anyway, here it is:

"Smart" redirect. After sign out, the user is redirected to:
* one's previous route, unless one comes from the `signOutRoute`
* the fallback route otherwise

Let's say your sign out route is called `sign_out`, and you want the fallback
route to be `home`

```coffeescript
Auth.Config.reopen
  signOutRoute: 'sign_out'
  smartSignOutRedirect: true
  signOutRedirectFallbackRoute: 'home' # defaults to 'index'
```

Same modification to `controller` as the
[post- sign out fixed route redirect](#post--sign-out-redirect-fixed-route).

Remember me
-----------

Your token creation API end point should accept polymorphic parameters:
either the regular set of sign in credentials, or a remember me token. e.g.,
`POST /api/token` expects either of these sets of params:
* `{email: foo@example.com, password: foag8wuef9aiwe}`, or
* `{remember_token: "fjlja8hfhf4"}`

The API end point should also return a remember me token, using the same key,
in its successful authentication response, e.g.
`{remember_token: "fjlja8hfhf4"}`

```coffeescript
Auth.Config.reopen
  rememberMe: true
  rememberTokenKey: 'remember_token'
  rememberPeriod: 14 # days
  rememberAutoRecall: true
```

`rememberTokenKey` is the key for the remember me token, in both the API
response and the expected param.
`rememberPeriod`, in days, is the valid period for the remember cookie.
Defaults to two weeks (14 days).
`rememberAutoRecall` controls whether Remember Me should attempt to
auto-sign in the user from local cookie. (see below) Defaults to true.

If you want to use `localStorage` instead of `cookie`:

```coffeescript
Auth.Config.reopen
  rememberStorage: 'localStorage' # defaults to 'cookie'
```

Remember Me will (attempt to) auto-sign in the user from the local cookie
when the user accesses an `Auth.Route` (only if one is not already signed in).
If you want to implement this behavior elsewhere, set `rememberAutoRecall` to
`false` in `Auth.Config`, and call `Auth.Module.RememberMe.recall()` elsewhere.

For example, if you want to delay until the application has finished loading
before attempting the recall (long network round-trip concern, perhaps?), then
you can implement this in the `didInsertElement` hook of your `ApplicationView`
if you want this behavior to be applied site-wide.

```coffeescript
App.ApplicationView = Em.View.extend
  didInsertElement: ->
    Auth.Module.RememberMe.recall()
```

`Auth.Module.RememberMe.recall()` accepts an object of options.
Currently, only `async` (=`true`/`false`) is supported. Normally you wouldn't
use this option; it is utilized internally to implement the recall behavior
before any redirection calculations:

```coffeescript
Auth.Module.RememberMe.recall { async: false }
```

The built-in behavior is to set the local remember me cookie on sign in
success, and destroy any local remember me cookie on sign out, and on any
sign in error. If you want to access these behaviors elsewhere, use the
following low-level methods:

* `Auth.Module.RememberMe.remember()`: set local remember me cookie
* `Auth.Module.RememberMe.forget()`: destroy local remember me cookie

`ember-auth` follows the common practice of "opt-in" remember me: if you have
turned on the remember me feature, but does not want to use it for a particular
sign in, simply *do not* return a remember token from the server response.
`ember-auth` will then simply skip setting the remember me cookie.

Bear in mind some [security caveats](https://github.com/heartsentwined/ember-auth/wiki/Security).

User-registration, forgot password, change password, etc
--------------------------------------------------------

These are operations on your user model, and they all follow the same pattern:
*do they require authentication?* If yes, put them under an `Auth.Route`.

Then just treat them as a normal ember model, with create, edit, etc actions.
You may want to set up some dedicated API end points at your server for
non-RESTful cases, e.g. the "forgot password" functionality.

Events
------

### Token authentication API events

The following events are emitted during token authentication API calls:

* `signInSuccess`
* `signInError`
* `signInComplete`
* `signOutSuccess`
* `signOutError`
* `signOutComplete`

All correspond to their `jQuery.ajax` (old-style API) namesakes, i.e.

```coffeescript
jQuery.ajax
  # ...
.done (json, status, jqxhr) =>
  # 'success' event triggered here
.fail (jqxhr) =>
  # 'error' event triggered here
.always (jqxhr) =>
  # 'complete' event triggered here
```

Subscribing to these events:

```coffeescript
Auth.on 'signInSuccess', ->
  # do something
```

You can access the token API response jqxhr object via `Auth.get('jqxhr')`.

Auto signing in through Remember Me's `recall()` will also trigger the
`signIn*` family of events.

### Auth.Route

`Auth.Route` emits the event

* `authAccess`

when an unauthenticated user accesses the `Auth.Route`. If you uses
redirection, the event is emitted before redirection.

Subscribing to this event:

```coffeescript
App.SecretsRoute = Auth.Route.extend
  init: ->
    @on 'authAccess', ->
      # display an overlay sign in form, for example

  model: ->
    App.Secret.find()
```

Customize ajax calls
--------------------

`Auth.ajax` is the central handler for all API calls, including those
generated by `ember-data`, `ember-auth`'s native authentication API calls,
and it is also available for any custom API calls. It ultimately delegates to
`jQuery.ajax(settings)`, and all preset options are overridable by supplying
your own options.

Function signature:

```coffeescript
  # @param {settings} jQuery.ajax options
  #   Defaults will be overrided by those set in this param
  ajax: (settings) ->
```

Example override:

```coffeescript
Auth.reopen
  ajax: (settings) ->
    settings.contentType = 'foo'
    @_super settings
```

All ajax calls will now be made with `contentType` set to `foo`, overriding
the default `application/json; charset=utf-8`.

Further use cases
-----------------

The source code at `src/auth.coffee` is a comprehensive list of public API
and helper methods; `src/config.coffee` contains an exhaustive list of
configurable options.

Contributing
============

You are welcome! As usual:

1. Fork
2. Branch
3. Hack
4. **Test**
5. Commit
6. Pull request

Tests
-----

`ember-auth` tests are written in [jasmine](http://pivotal.github.com/jasmine/),
run on a mini rails app.

1. Grab a copy of ruby. [RVM](http://rvm.io/) recommended.
2. `bundle install` to install dependencies.
3. `guard` to run tests.

`ember-auth` has been setup with [guard](https://github.com/guard/guard),
which will continuously monitor lib and spec files for changes and re-run
the tests automatically.

Building distribution js files
------------------------------

`rake dist`

License
=======

GPL 3.0
