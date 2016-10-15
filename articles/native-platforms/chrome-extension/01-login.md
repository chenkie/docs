---
title: Login
description: This tutorial demonstrates how to use the Auth0-Chrome SDK to add authentication and authorization to your Chrome extension
---

This quickstart guide will walk you through how to set up authentication in your Chrome extensions using Auth0. The tutorial is based on a sample application that uses Auth0's Lock widget within an extension popup. Once the user is authenticated, another popup displays their profile.

<%= include('../../_includes/_package2', {
  org: 'auth0-samples',
  repo: 'auth0-chrome-extension-sample',
  path: '00-Starter-Seed'
}) %>

## Add the Dependencies

To add authentication to your Chrome extension, two dependencies are required: Auth0's [Lock widget](https://github.com/auth0/lock) and the [auth0-chrome-extension](https://github.com/auth0/auth0-chrome-extension) SDK. Package these scripts locally in your project by copying the minified distribution files from Github, or get them through Bower or npm.

While optional, the tutorial makes use of [jwt-decode](https://github.com/auth0/jwt-decode). This package is useful for decoding JSON Web Tokens and reading their payloads, which is utilized here for checking token expiry times.

> Note: the Lock widget is not pre-bundled on npm so you will need to bundle it manually

**Bower**

```bash
bower install auth0-lock auth0-chrome-extension jwt-decode
```

**npm**

```bash
npm install auth0-lock auth0-chrome-extension jwt-decode
```

As a best practice, place the dependencies within a `vendor` directory in your project.

## Set Your Auth0 Credentials

The **client ID** and **domain** for your application are used to connect to Auth0 and these will be required when using Lock. Create an `env.js` file and populate it with your application's credentials.

```js
// src/env.js

window.env = {
  AUTH0_DOMAIN: '<%= account.clientId %>',
  AUTH0_CLIENT_ID: '<%= account.namespace %>',
};
```

These values will later be used when instantiating Lock.

## Set Permissions for the App

Chrome apps require explicit permissions to be defined when communicating with the outside world. Since the app will need to communicate with Auth0, several permissions need to be set in the `manifest.json` file.

```js
"content_security_policy": "script-src 'self' https://cdn.auth0.com blob: filesystem: chrome-extension-resource:",
"permissions": [
  "notifications",
  "webRequest",
  "webRequestBlocking",
  "identity",
  "https://*.chromiumapp.org/auth0*",
  "https://<%= account.namespace %>.auth0.com/*"
]
```

When Lock is instantiated, a call gets made to Auth0's CDN to check the application credentials that are passed to it. For this call to be made successfully, `https://cdn.auth0.com` needs to be included in `"content_security_policy"`.

For Auth0's callback to be successfully intercepted and for authentication to be completed, a callback URL is requird. Chrome extensions provide a URL for each application that can be accessed at `<app-id>.chromiumapp.org`, and this is the URL that is used for the callback. The domain for the application is also whitelisted as it will need to be accessed.

## Set the Callback URL

The callback URL for your application needs to be set in the Auth0 dashboard. Navigate to your [application's settings](${manage_url}/#/applications/${account.clientId}/settings) and place the URL specific to your Chrome extension in the **Allowed Callback URLs textarea**.

```bash
https://<app-id>.chromiumapp.org/auth0/callback
```

## Create the Main Popup

The UI for allowing users to log in, view their profile, and log out can be provided in a `browser_action.html` file.

```html
<!-- src/browser_action/browser_action.html -->
...
<body>
  <div id="mainPopup">
    <div class="profile hidden">
      <div class="picture"></div>
      <div class="floater-profile">
        <h5 class="nickname"></h5>
        <h4 class="name"></h4>
        <hr />
        <button class="btn btn-danger logout-button">Logout</button>
      </div>
    </div>
    <div class="loading hidden">
      <div class="loading-bar"></div>
      <div class="loading-bar"></div>
      <div class="loading-bar"></div>
      <div class="loading-bar"></div>
    </div>
    <div class="default">
      <div class="text-center">
        <h3>Chrome Demo</h1>
        <button class="btn btn-success login-button">Login With Auth0</button>
        <p class="caption">Press the login button above to login.</p>
      </div>
    </div>
  </div>
  <script src="../env.js"></script>
  <script src="../vendor/lock.min.js"></script>
  <script src="../vendor/Auth0Chrome.js"></script>
  <script src="../vendor/jwt-decode.min.js"></script>
  <script src="./lib/AuthService.js"></script>
  <script src="./main.js"></script>
</body>
```

This view also brings in the `vendor` scripts as well as a service called `AuthService` that will be created next. When the user clicks the extension in the Chrome menu, they will see a button and message prompting them to log in.

![login](/media/articles/native-platforms/chrome-extension/01-login-button.png)


## Create the AuthService

The best way to manage authentication related tasks is to provide a service that can be used across the whole application. The `AuthService` needs to have methods for logging the user in and out, retrieving their profile, and checking whether they are logged in.

```js
// src/browser_action/lib/AuthService.js

function AuthService (domain, clientId) {
  this.domain   = domain;
  this.clientId = clientId;
}

AuthService.prototype.getIdToken = function () {
  return localStorage.getItem('idToken');
}

AuthService.prototype.setIdToken = function (idToken) {
  localStorage.setItem('idToken', idToken)
}

AuthService.prototype.isLoggedIn = function () {
  const idToken = this.getIdToken();
  if(!idToken){
    return false;
  }
  const payload = jwt_decode(idToken);

  return (payload.exp > (Date.now() / 1000));
}


function checkIfSet(obj, key) {
  return !!(obj && obj[key] != null);
}


function isChromeExtension(){
  return !!(window.chrome && chrome.runtime && chrome.runtime.id)
}


AuthService.prototype.show = function (lockOptions) {
  const lock = new Auth0Lock(this.clientId, this.domain, lockOptions);
  lock.show();
}


AuthService.prototype.getProfile = function (cb) {
  const lock = new Auth0Lock(this.clientId, this.domain);
  const idToken = this.getIdToken();
  lock.getProfile(idToken, function (err, profile) {
    if(err){
      cb(err);
    }
    cb(null, profile);
  });
}

AuthService.prototype.logout = function(){
  localStorage.removeItem('idToken');
}
```

The `show` method is where Lock gets instantiated, taking the client ID and domain for your app, along with any Lock options that need to be set. This method also calls `lock.show` to display the widget.

> **Note:** for a full list of Lock's options, see the [customization guide](/libraries/lock/v10/customization).

This method can now be called from a main script that handles the popup.

## Add a `main.js` File

The HTML for the popup view is in place, but some JavaScript is needed to handle the view. Add a new file called `main.js` in `src/browser_actions`.

```js
// src/browser_actions/main.js

const authService = new AuthService(env.AUTH0_DOMAIN, env.AUTH0_CLIENT_ID);

// Minimal jQuery
const $  = document.querySelector.bind(document);

function renderProfileView() {
  $('.default').classList.add('hidden');
  $('.loading').classList.remove('hidden');

  $('.logout-button').addEventListener('click', function(){ 
    authService.logout();
    main();
  });

  authService.getProfile(function (err, profile) {
    if(err){
      document.body.innerHTML = 'There was an error fetching profile, ' + err.message + ' please reload the extension.';
      return;
    }

    ['picture', 'name', 'nickname'].forEach(function (key) {

      const element = $('.' +  key);
      if (element.nodeName === 'DIV') {
        element.style.backgroundImage = 'url(' + profile[key] + ')';
        return;
      }

      element.textContent = profile[key];
    });

    $('.loading').classList.add('hidden');
    $('.profile').classList.remove('hidden');
  });
}


function renderDefaultView() {
  $('.default').classList.remove('hidden');
  $('.profile').classList.add('hidden');
  $('.loading').classList.add('hidden');

  $('.login-button').addEventListener('click', function() {
    $('.default').classList.add('hidden');
    $('.loading').classList.remove('hidden');
    authService.show({
      theme: {
        logo: 'https://cdn1.tnwcdn.com/wp-content/blogs.dir/1/files/2014/08/canary-logo.png',
        primaryColor: '#FFC400'
      },
      auth: {
        responseType: 'token',
        scope: 'openid',
      },
      languageDictionary: {
        title: 'Chrome Sample',
      },
      container: 'mainPopup'
    });
  });
}

function main () {
  if (authService.isLoggedIn()) {
    renderProfileView();
  } else {
    renderDefaultView();
  }
}


document.addEventListener('DOMContentLoaded', main);
```

Remeber that the JavaScript library or framework you choose for your extension is at your discretion. This example shows usage with "minimal jQuery", which is effectively just using `querySelector` to bind the `document` to a `const` of `$`.

The `show` method from the `AuthService` is being called in `renderDefaultView` when the **Login** button gets clicked. Several options are being set, including the `theme` for the widget, along with the `responseType` and `scope`. For more information on each of the options, see the [Lock documentation](libraries/lock).

Now when the user clicks the **Login** button, Lock will be displayed.

![lock](/media/articles/native-platforms/chrome-extension/02-lock.png)

After a successful login, the user's `id_token` and `profile` object will be saved in local storage and their profile will be accessible from the Chrome menu popup.

![profile](/media/articles/native-platforms/chrome-extension/03-profile.png)



