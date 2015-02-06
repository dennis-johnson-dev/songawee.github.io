---
layout: post
comments: true
title:  "Intro to testing a basic hapi.js server"
date:   2015-2-5
categories: jekyll update
---

In this post, we will cover how to test a server built with [hapi.js](http://hapijs.com/) using [jasmine](https://github.com/mhevery/jasmine-node). Our server will consist of a few simple routes that we will then test. The code of this example can be found [here](https://github.com/songawee/testing_hapijs).

First, let's get our project set up and install our dependencies. Here is what we will need:

```bash
npm install --save hapi
npm install --save-dev jasmine-node
npm install --save-dev nodemon
```

Let's configure our package.json so we can run `npm start` and `npm test`. Here are the entries we will need:

```json
  {
    ...
    "scripts": {
      "start": "./node_modules/nodemon/bin/nodemon.js",
       "test": "./node_modules/jasmine-node/bin/jasmine-node spec"
    }
    ...
  }
```

> **Note:** `nodemon` is a module that watches your files and restarts the server on changes.

We can place our server in a directory named `server` and call it `server.js`. We can create a data fixture to simulate the result of some data fetching.

```js
//./spec/fixtures/data.js
module.exports = [
  {
    message: 'hello'
  },
  {
    message: 'world'
  }
];
```

```js
// ./server/server.js
var Hapi = require("hapi");
var data = require("../spec/fixtures/data");

function createServer(port) {
  var server = new Hapi.Server(port);

  var healthCheck = {
    path: "/",
    method: "GET",
    handler: function(request, reply) {
      reply("hello world");
    }
  };

  var getData = {
    path: "/data/{index}",
    method: "GET",
    handler: function(request, reply) {
      var index = parseInt(request.params.index);
      var isValidKey = typeof data[index] !== "undefined";
      if (isValidKey) {
        reply(data[index]);
      } else {
        reply("Not Found").code(404);
      }
    }
  };

  server.route([
    healthCheck,
    getData
  ]);

  return server;
}

module.exports = {
  createServer: createServer
};

```

Our server has two routes. One that just replies with a simple string and one that replies with some data specified by the user. One important thing to mention about routes in hapi.js is that it doesn't matter which order you add them to the routes array. They are sorted by specificity as mentioned [here](http://hapijs.com/api#path-matching-order).

We can see in the `getData` route that *index* is a user specified string. We can grab this value off of the request with `request.params.index`. We check if it is a valid index of the data array and respond accordingly. hapi.js will infer the correct MIME type for your asset. So if you `reply('hi')`, it will respond with *'text/html'* and if you `reply({ some: 'object' })`, it will respond with *'application/json'*.

Now, we can set up a wrapper for our server. This will allow us to pass parameters (such as port number) into the server. This is useful for testing and keeping the server api clean. We will put this wrapper in `index.js`.

```js
// ./index.js
var hapi_server = require('./server/server.js');
var port = 3000;

var server = hapi_server.createServer(port);
server.start();
console.log('hapi.js server running at port: ' + port);
```

If you run `npm start` at the root of the project directory and navigate to [localhost:3000](localhost:3000), you should see this:

<img src="/img/hello_world.png" width="200" />

Now, we can start testing our simple server with jasmine. First, we need to create a `spec` directory that jasmine will look for in order to find the tests. Here are the basic elements of our testfile.

```js
// ./spec/HealthCheckSpec.js

var server = require("../server/server.js").createServer(3000);

describe('Health Check', function () {

  it("responds with status code 200 and hello world text", function(done) {
    var options = {
      method: "GET",
      url: "/"
    };  

    server.inject(options, function(response) {
      expect(response.statusCode).toBe(200);
      expect(response.result).toBe('hello world');
      done();
    });
  });

});

```

This test uses a hapi.js `server.inject` function off of the server object that injects an HTTP request in the form of options and gives us a response object to make assertions on. In this case, we inspect that we receive a 200 status code and a result of 'hello'.

>Note that we must call the `done` function at the end of our assertions to tell jasmine that this is an asynchronous test and we need to wait before moving onto the next test. If this line is missing, a timeout error will occur.

Now we can look at the tests for the `getData` function.

```js
// ./spec/getDataSpec.js

var server = require("../server/server.js").createServer(3000);
var data = require("./fixtures/data.js");

describe("get Data", function () {

  it("responds with data object", function(done) {
    var options = {
      method: "GET",
      url: "/data/1"
    };

    server.inject(options, function(response) {
      expect(response.statusCode).toBe(200);
      expect(response.result).toEqual(data[1]);
      done();
    });
  });

  it("responds with 404 - Not Found", function(done) {
    var options = {
      method: "GET",
      url: "/data/two"
    };

    server.inject(options, function(response) {
      expect(response.statusCode).toBe(404);
      expect(response.result).toBe("Not Found");
      done();
    });
  });

});
```

We can see a similar pattern with the health check test. In this case, we pass the server a path that will return data and a path that respond with a 404.

In the first case, we set the url to "/data/1". This will set `request.params.index` to "1". We expect the JSON returned to be equal (deep comparison) to the second object in the data array.

In the second case, we set the url so that `request.params.index` will be undefined. This will cause our server to respond with a 404.

When we run `npm test`, we should see something similar to the following in the terminal window.

<img src="/img/tests_pass.png" width="400" />

I hope this has been informative and a good first step in setting up your repo for testing a hapi.js server. The code of this example can be found [here](https://github.com/songawee/testing_hapijs).
