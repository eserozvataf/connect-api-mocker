connect-api-mocker
==================

[![Build Status](https://travis-ci.org/muratcorlu/connect-api-mocker.svg?branch=master)](https://travis-ci.org/muratcorlu/connect-api-mocker)

`connect-api-mocker` is a [connect.js](https://github.com/senchalabs/connect) middleware that fakes REST API server with filesystem. It will be helpful when you try to test your application without the actual REST API server.

It works with a wide range of servers: [connect][], [express][], [browser-sync][], [lite-server][], [webpack-dev-server][].

## Installation

```
npm install connect-api-mocker --save-dev
```

## Usage

### Using with [Connect][]

```js
var http = require('http');
var connect = require('connect');
var apiMocker = require('connect-api-mocker');

var app = connect();
var restMock = apiMocker('/api', 'mocks/api');

app.use(restMock);
http.createServer(app).listen(8080);
```

### Using with [Express][]

```js
var express = require('express');
var apiMocker = require('connect-api-mocker');

var app = express();
var restMock = apiMocker('/api', 'mocks/api');

app.use(restMock);
app.listen(8080);
```

### Using with [BrowserSync][]

```js
var browserSync = require('browser-sync').create();
var apiMocker = require('connect-api-mocker');

var restMock = apiMocker('/api', 'mocks/api');

browserSync.init({
  server: {
    baseDir: './',
    middleware: [
      restMock,
    ],
  },
  port: 8080,
});
```

### Using with [lite-server][]

`bs-config.js` file:

```js
var apiMocker = require('connect-api-mocker');

var restMock = apiMocker('/api', 'mocks/api');

module.exports = {
  server: {
    middleware: {
      // Start from key `10` in order to NOT overwrite the default 2 middleware provided
      // by `lite-server` or any future ones that might be added.
      10: restMock,
    },
  },
  port: 8080,
};
```

### Using with Grunt

You can use it with [Grunt](http://gruntjs.com). After you install [grunt-contrib-connect](https://github.com/gruntjs/grunt-contrib-connect) add api-mocker middleware to your grunt config. The `mocks/api` folder will be served as REST API at `/api`.

```js

module.exports = function(grunt) {
  var apiMocker = require('connect-api-mocker');

  grunt.loadNpmTasks('grunt-contrib-connect');  // Connect - Development server

  // Project configuration.
  grunt.initConfig({

    // Development server
    connect: {
      server: {
        options: {
          base: './build',
          port: 9001,
          middleware: function(connect, options) {

            var middlewares = [];

            // mock/rest directory will be mapped to your fake REST API
            middlewares.push(apiMocker(
                '/api',
                'mocks/api'
            ));

            // Static files
            middlewares.push(connect.static(options.base));
            middlewares.push(connect.static(__dirname));

            return middlewares;
          }
        }
      }
    }
  });
}
```

After you can run your server with `grunt connect` command. You will see `/api` will be mapped to `mocks/api`.

### Using with Webpack

To use api mocker on your [Webpack][] projects, simply add a setup options to your [webpack-dev-server][] options:

```js
  ...
  setup: function(app) {
    app.use(apiMocker('/api', 'mocks/api'));
  },
  ...
```

## Directory Structure

You need to use service names as directory name and http method as filename. Files must be JSON. Middleware will match url to directory structure and respond with the corresponding http method file.

Example REST service: `GET /api/messages`

Directory Structure:

```
_ api
  \_ messages
     \_ GET.json
```

Example REST service: `GET /api/messages/1`

Directory Structure:

```
_ api
  \_ messages
     \_ 1
        \_ GET.json
```

Example REST service: `POST /api/messages/1`

Directory Structure:

```
_ api
  \_ messages
     \_ 1
        \_ POST.json
```


Example REST service: `DELETE /api/messages/1`

Directory Structure:

```
_ api
  \_ messages
     \_ 1
        \_ DELETE.json
```

## Custom responses

If you want define custom responses you can use `js` files with a middleware function that handles requests.

Example REST service: `POST /api/messages`

Directory Structure:

```
_ api
  \_ messages
     \_ POST.js
```

`POST.js` file:

```js
module.exports = function (request, response) {
  if (!request.get('X-Auth-Key')) {
    response.status(403).send({});
  } else {
    response.sendFile('POST.json', {root: __dirname});
  }
}
```

### Another Example: Respond different json files based on a query parameter:

- Request to `/users?type=active` will be responded by `mocks/users/GET_active.json`
- Request to `/users` will be responded by `mocks/users/GET.json`

`GET.js` file:

```js
module.exports = function (request, response) {
  var targetFileName = 'GET.json';

  // Check is a type parameter exist
  if (request.query.type) {
    // Generate a new targetfilename with that type parameter
    targetFileName = 'GET_' + request.query.type + '.json';

    // If file does not exist then respond with 404 header
    if (!fs.accessSync(targetFileName)) {
      return response.status(404);
    }
  }

  // Respond with targetFileName
  response.sendFile(targetFileName, {root: __dirname});
}
```

## Wildcards in paths

You can use wildcards for paths to handle multiple urls(like for IDs). If you create a folder structure like `api/users/__user_id__/GET.js`, all requests like `/api/users/321` or `/api/users/1` will be responded by custom middleware that defined in your `GET.js`. Also id part of the path will be passed as a request parameter named as `user_id` to your middleware. So you can write a middleware like that:

`api/users/__user_id__/GET.js` file:

```js
module.exports = function (request, response) {
  response.json({
    id: request.params.user_id
  });
}
```

## Defining multiple mock configurations

You can use apiMocker multiple times with your connect middleware server. In example below, we are defining 3 mock server for 3 different root paths:

```js
app.use('/api/v1', apiMocker('target/path'));
app.use('/user-api', apiMocker({
  target: 'other/target/path'
}));
app.use(apiMocker('/mobile/api', {
  target: 'mocks/mobile'
});
```

## Next on not found option

If you have some other middlewares that handles same url(a real server proxy etc.) you can set `nextOnNotFound` option to `true`. In that case, api mocker doesnt trigger a `404` error and pass request to next middleware. (default is `false`)

```js
apiMocker('/api', {
  target: 'mocks/api',
  nextOnNotFound: true
});
```

With that option, you can mock only specific urls simply.

<!-- Definitions -->

[connect]: https://github.com/senchalabs/connect
[express]: https://github.com/expressjs/express
[browsersync]: https://github.com/BrowserSync/browser-sync
[browser-sync]: https://github.com/BrowserSync/browser-sync
[lite-server]: https://github.com/johnpapa/lite-server
[webpack]: https://github.com/webpack/webpack
[webpack-dev-server]: https://github.com/webpack/webpack-dev-server
