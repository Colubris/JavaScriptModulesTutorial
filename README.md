# Overview

This sample Cloud Module is used in the [Integrating with Third Party Services Tutorial][tutorial] to show developers how to create a Cloud Module. The module showcases sending email using the Mailgun service, but it can be adapted to use another provider of your choice.

Mailgun is a set of powerful APIs that allow you to send, receive, track and store email effortlessly. You can check out their service at [www.mailgun.com][mailgun]. To use this module, you will need to head over to the Mailgun website and create an account.

# Installation

1. Clone this repository to get the Cloud Module.
  `git clone https://github.com/ParsePlatform/MyMailCloudModule.git`

2. Copy `myMailModule-1.0.0.js` over to your Cloud Code Directory, placing it in the `cloud` directory.

# Usage

To use the module in your Cloud Code functions, start by requiring the module and initializing it with your credentials:

```javascript
var client = require('cloud/myMailModule-1.0.0.js');
client.initialize('myDomainName', 'myAPIKey');
```

Then inside of your Cloud Code function, you can use the `sendEmail` function to fire off some emails:

```javascript
Parse.Cloud.define("sendEmailToUser", function(request, response) {
  client.sendEmail({
    to: "email@example.com",
    from: "MyMail@CloudCode.com",
    subject: "Hello from Cloud Code!",
    text: "Using Parse and My Mail Module is great!"
  }).then(function(httpResponse) {
    response.success("Email sent!");
  }, function(httpResponse) {
    console.error(httpResponse);
    response.error("Uh oh, something went wrong");
  });
});
```

This function takes two parameters. The first is a hash with the mail parameters you want to include in the request. The typical ones are from, to, subject and text, but you can find the full list on their documentation page. The second parameter to this function is an object with a success and an error field containing two callback functions.

Once you've deployed your Cloud Code function and this Cloud Module to Parse, you can test it out. The example below shows how to do this using cUrl:

```
curl -X POST \
  -H "X-Parse-Application-Id: YOUR_APPLICATION_ID" \
  -H "X-Parse-REST-API-Key: YOUR_REST_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{ }' \
  https://api.parse.com/1/functions/sendEmailToUser
```

# Tutorial

In this tutorial, we'll take a look at how you would create your own JavaScript module to integrate with a third-party service such as Mailgun. If you are not familiar with [Mailgun][mailgun], take a look at their website to learn more about what they offer. The Mailgun feature most relevant to this tutorial is the ability to send emails.

## 1. Creating Modules

JavaScript has long lacked the ability organize code into multiple files. To solve this problem, the [CommonJS][commonjs] modules standard was created. Along with [many][commonjs-implementations] JavaScript libraries and services, Cloud Code uses this standard to allow creating more modular JavaScript projects.

### 1.1. Loading Files with `require`

All of the JavaScript modules you want to use in Cloud Code must be placed in the `cloud/` directory. To include a module in your project you can use the `require("/path/to/module")` method.

```js
var myModule = require('cloud/myModule.js');
```

The `require()` method in Cloud Code requires an absolute path to the file that will be imported. If you run into issues loading your module, make sure your path starts with `cloud/`.

### 1.2. The `exports` Object

The `require` function returns an object automatically created in all modules called `exports`. When you create a module you use this `exports` object to define the set of functionality that will be available to users of your module. Take a look at this simple example.

```js
// myMathModule.js
exports.addTwoNumbers = function(x, y) {
    return x+y;
}

// cloud.js
var math = require('cloud/myMathModule.js');
console.log('3 + 4 is ' + math.addTwoNumbers(3,4));
```

You don't need to declare or create the `exports` object anywhere. It is always available when creating modules. All fields and functions added to the `exports` object are handed down by the `require` function. This means that anything not set on the `exports` object remains private to the module. In the following example, the value of PI is not directly accessible from `cloud.js`.

```js
// myMathModule.js
var PI = 3.14; // Private
exports.getPI() { // Public
	return PI;
}

// cloud.js
var math = require('cloud/myMathModule.js');
console.log(math.PI);      // Doesn't work!
console.log(math.getPI()); // Works!
```

## 2. Implementing the Mailgun Module
Now that we have a good understanding of modules, let's dive into the Mailgun JavaScript Module and take a look at how to implement something similar.

### 2.1. The initialize method

The first part of the module is the `initialize` method. Since most services will require users to provide credentials, creating an `initialize` method is a great way to centralize the location where these are set.

```js
var url = 'api.mailgun.net/v2';
var domain = '';
var key = '';

module.exports = {
  initialize: function(domainName, apiKey) {
    domain = domainName;
    key = apiKey;
    return this;
  },
  ...
}
```

The module starts by declaring three private properties. When the `initialize` method is called, the credentials are stored in the private variables. Along with the credentials, we've also stored the api's url in a private property. When the Mailgun API is updated, we will only need to change the url here instead of everywhere in the module.

It would be possible to pass the credentials with each function that uses the Mailgun API. But, by using an `initialize` method, the users of the module only need to set their credentials once like this:

```js
// cloud.js

// Initialize Your Module
var client = require('cloud/myMailModule-1.0.0.js');
client.initialize('myDomainName', 'myAPIKey');

// All my Cloud Code functions can use the Mailgun module
Parse.Cloud.beforeSave...
Parse.Cloud.define...
...
```

### 2.2. Sending an Email

The main use of our module is to send emails. The function used to do this is remarkably simple, but requires some understanding of the Mailgun API, Parse HTTP requests and general HTTP networking concepts. Let's start by looking at the code.

```js
module.exports = {
  ...
  sendEmail: function(params, options) {
    return Parse.Cloud.httpRequest({
      method: "POST",
      url: "https://api:" + key + "@" + url + "/" + domain + "/messages",
      body: params,
    }).then(function(httpResponse) {
      if (options && options.success) {
        options.success(httpResponse);
      }
    }, function(httpResponse) {
      if (options && options.error) {
        options.error(httpResponse);
      }
    });
  }
}
```

This function simply calls `Parse.Cloud.httpRequest` with the appropriate parameters. As discussed in the [Cloud Code docs][cloudcode-networking], the `httpRequest` function allows you to send a request to an external web server. This is the key to integrating with third-party services as it allows us to communicate with other APIs.

To determine the values for the `method`, `url` and `body` parameters, we need to take a look at Mailgun's [API reference documentation][mailgun-api-reference]. From this page we can find out that sending an email requires doing a POST request to the URL `https://api.mailgun.net/v2/{DomainName}/messages`. But how do we provide our Mailgun credential? Well, the majority of web APIs support basic HTTP authentication. This means we can add the credentials to the URL in the following format.

```
https://api:apikey@api.url.here
```

In the case of Mailgun, if we combine the URL and the credential we get:

```
https://api:{ApiKey}@api.mailgun.net/v2/{DomainName}/messages
```

So the value of `method` needs to be `POST` and the value of `url` should be the address shown above. As you can see from the function's code (repeated below for convenience), the value of `body` is simply set to the `params` variable which is supplied to the `sendEmail` function.

```js
module.exports = {
  ...
  sendEmail: function(params, options) {
    return Parse.Cloud.httpRequest({
      method: "POST",
      url: "https://api:" + key + "@" + url + "/" + domain + "/messages",
      body: params,
    }).then(function(httpResponse) {
      if (options && options.success) {
        options.success(httpResponse);
      }
    }, function(httpResponse) {
      if (options && options.error) {
        options.error(httpResponse);
      }
    });
  }
}
```

There are a variety of parameters that third party services will accept in the body of their requests, and these are subject to change at any time. Instead of hard coding a list of accepted parameters, our module lets the user provide a list of them and then relays them to Mailgun. This makes the module easier to update in the future and allows the module's API to be much more flexible. In the case of sending an email with Mailgun, the list of accepted body parameters can be found in their [API reference documentation][mailgun-api-reference].

The `success` and `error` parameters are two callback functions that are provided to the `sendEmail` function through the `options` parameters. The `httpRequest` method returns a [Promise][javascript-sdk-promises]. The promise's `then` method invokes the appropriate success or error callback based on whether the promise is resolved or rejected.

Let's see what a call to the `sendEmail` method looks like.

```js
// First we initialize our module
var client = require('cloud/myMailModule-1.0.0.js');
client.initialize('myDomainName', 'myAPIKey');

// Then we create a cloud function
Parse.Cloud.define("sendEmailToUser", function(request, response) {
  client.sendEmail({
    to: "email@example.com",
    from: "MyMail@CloudCode.com",
    subject: "Hello from Parse!",
    text: "Using Parse and My Mail Module is great!"
  }).then(function(httpResponse) {
    response.success("Email sent!");
  }, function(httpResponse) {
    console.error(httpResponse);
    response.error("Uh oh, something went wrong");
  });
});
```

In the example above, a Cloud Function named `sendEmailToUser` is defined that calls the `sendEmail` module method, passing in the parameters needed to send out an email. The success and error cases are echoed back to the user in the response.

### 2.3. Versioning

Versioning the module helps developers know which version they're currently running. While this is not a requirement, it's a good practice. We've named our module file `myMailModule-1.0.0.js` to indicate that this is version 1.0.0. Additionally, let's add a property to our module that provides versioning information.

```js
module.exports = {
  ...
  version: '1.0.0',
  ...
```

This makes the version information accessible via code. The example below shows a modified `sendEmailToUser` Cloud Function that includes the version in the text of the email sent out.

```js
// Modify the message sent to include the version
Parse.Cloud.define("sendEmailToUser", function(request, response) {
  client.sendEmail({
    to: "email@example.com",
    from: "MyMail@CloudCode.com",
    subject: "Hello from Parse!",
    text: "Using Parse and My Mail Module version " + client.version + " is great!"
  }).then(function(httpResponse) {
    response.success("Email sent!");
  }, function(httpResponse) {
    console.error(httpResponse);
    response.error("Uh oh, something went wrong");
  });
});
```

And that's all there is to know about creating modules for Cloud Code! Make sure to take a look at the

- [Cloud Code guide][cloud-code-guide],
- [JavaScript Modules guide][javascript-modules-guide], and
- [JavaScript SDK Guide][javascript-sdk-guide]

If you have any questions or comments, visit our [Help Center][parse-help].

[tutorial]: #Tutorial
[cloud-code-guide]: https://www.parse.com/docs/cloudcode/guide
[javascript-modules-guide]: https://www.parse.com/docs/cloudcode/guide#cloud-code-advanced-modules
[javascript-sdk-guide]: https://www.parse.com/docs/js/guide
[javascript-sdk-promises]: https://www.parse.com/docs/js/guide#promises
[cloudcode-networking]: https://www.parse.com/docs/cloudcode/guide#cloud-code-advanced-networking
[mailgun]: http://www.mailgun.com/
[commonjs]: http://wiki.commonjs.org/wiki/Modules/1.1.1
[commonjs-implementations]: http://wiki.commonjs.org/wiki/CommonJS#Implementations
[mailgun-api-reference]: http://documentation.mailgun.com/api-sending.html
[parse-help]: https://parse.com/help
