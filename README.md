<!-- markdownlint-disable first-line-h1 line-length -->

[![CircleCI](https://circleci.com/gh/mutant-ws/blocks.svg?style=svg)](https://circleci.com/gh/mutant-ws/blocks)
[![npm package version](https://badge.fury.io/js/%40mutant-ws%2Fblocks.svg)](https://badge.fury.io/js/%40mutant-ws%2Fblocks)
[![Coverage Status](https://coveralls.io/repos/github/mutant-ws/blocks/badge.svg)](https://coveralls.io/github/mutant-ws/blocks)

# blocks

> REST API framework for Node.js. `request` |> `think hard` |> `response`.

![Request-Response cycle](docs/bin/req-res-cycle.svg "Request-Response cycle")

---

<!-- vim-markdown-toc GFM -->

* [Features](#features)
  * [Validate input](#validate-input)
  * [Permissions](#permissions)
  * [Promises](#promises)
  * [Other](#other)
* [Install](#install)
* [Example](#example)
* [Configuration](#configuration)
* [Routes](#routes)
  * [Default "/ping" route](#default-ping-route)
  * [Custom route](#custom-route)
  * [Custom JSON schema](#custom-json-schema)
* [Plugins](#plugins)
  * [Custom plugin](#custom-plugin)
* [Develop](#develop)
* [Changelog](#changelog)

<!-- vim-markdown-toc -->

## Features

### Validate input

> Pass request data (headers, body, query parameters, URL parameters) through custom JSON Schemas defined for each route. Make sure no unwanted data gets in, de-clutter the route logic and make the API behave more consistent.  
If validation fails, an automatic `409 Conflict` response will be sent.

See [`ajv`](https://github.com/epoberezkin/ajv) and [JSON Schema docs](https://json-schema.org) for more on data validation.

### Permissions

> Simple function outside of main route logic.  
If it returns false, an automatic `403 Forbidden` response will be sent.

### Promises

> `async/await` in Plugins and Routes

### Other

* File upload and form parsing for `multipart/form-data` - [`busboy`](https://github.com/mscdex/busboy)
* Middleware support of existing package - [`connect`](https://github.com/senchalabs/connect)
* JSON Web Token - [`jsonwebtoken`](https://github.com/auth0/node-jsonwebtoken)
* Query string parsing - [`qs`](https://github.com/ljharb/qs)
* Route param parsing - [`path-to-regexp`](https://github.com/pillarjs/path-to-regexp)
* Cross-origin resource sharing - [`cors`](https://github.com/expressjs/cors)
* Secure your API with various HTTP headers - [`helmet`](https://github.com/helmetjs/helmet)

## Install

```bash
npm install @mutant-ws/blocks
```

## Example

`src/index.js`

```js
import http from "http"
import glob from "glob"
import { block } from "@mutant-ws/blocks"

// Initialize `block` app
const app = block({
  plugins: glob.sync("./plugins/*.js", { cwd: __dirname, absolute: true }),
  routes: glob.sync("./**/*.route.js", { cwd: __dirname, absolute: true }),
})


// After plugins successfully initialize, start http server
app.then(([middleware, plugins]) => {
    const server = http.createServer(middleware)

    server.listen({
      port: process.env.PORT,
    })

    server.on("error", error => {
      console.log("Server error", error)
    })

    server.on("listening", () => {
      console.log(`Server started on port ${process.env.PORT}`)
    })
  })
  .catch(error => {
    console.log("Application could not initialize", error)
  })
```

## Configuration

`blocks` uses a set of `process.env` variables for configuration. See [`_env`](_env) file for all available options and defaults.

Use [`dotenv`](https://github.com/motdotla/dotenv) for easy local development.

## Routes

### Default "/ping" route

`GET: /ping`

```js
{
  "name": "foo",
  "ping": "pong",
  "aliveFor": {
    "days": 2, "hours": 1, "minutes": 47, "seconds": 46
  }
}
```

### Custom route

`src/something.route.js`

```js
module.exports = {
  method: "GET",
  path: "/something/:id",

  // 409 if invalid req.query, req.headers, req.params or req.body
  schema: require("./something.schema"),

  // 401 if returns false or throws
  authenticate: (/* plugins */) => (/* req */) => true,

  // 403 if returns false or throws
  authorize: (/* plugins */) => (/* req */) => true,

  /**
   * After schema validation and permission checking
   *
   * @param  {Object}  plugins  Plugins
   * @param  {Object}  req      Node request
   *
   * @return {mixed}
   */
  action: (/* plugins */) => req => {
    return {
      message: req.ctx.params.id
    }
  },
}
```

### Custom JSON schema

A schema can contain only 4 (optional) keys:

* `headers` validates `req.headers`
* `params` validates `req.ctx.params` parsed from URL with [`path-to-regexp`](https://github.com/pillarjs/path-to-regexp)
* `query`: validates `req.ctx.query` parsed from URL with [`qs`](https://github.com/ljharb/qs)
* `body` validates `req.ctx.body` parsed from `req` with `JSON.parse`

See [`src/plugins/route-default.schema.js`](src/plugins/route-default.schema.js) for default values.

Each key needs to be a [`ajv`](https://github.com/epoberezkin/ajv) compatible schema object.

`src/something.schema.js`

```js
module.exports = {
  headers: {
    type: "object",
    required: ["authorization"],
    properties: {
      authorization: {
        type: "string",
      },
    },
  },

  params: {
    type: "object",
    additionalProperties: false,
    required: ["id"],
    properties: {
      id: {
        type: "string",
        pattern: "^[a-z0-9-]+$",
        maxLength: 25,
        minLength: 25,
      }
    }
  },

  query: {
    type: "object",
    additionalProperties: false,
    properties: {
      offset: {
        type: "integer",
        minimum: 0,
        default: 0,
      },
      limit: {
        type: "integer",
        minimum: 1,
        maximum: 100,
        default: 20,
      },
    },
  },
}
```

## Plugins

Separate code interfacing with 3rd party libraries or services. [pluginus](https://github.com/mutant-ws/pluginus) dependency injection library is used.

Plugins are accesible in other plugins, middleware and routes.

### Custom plugin

A plugin consists of a constructor function and a list of other plugins that is dependent on.

Whatever the `create` function returns will be considered as the plugin's content and is what will be exposed to the routes, middleware and other plugins.

`src/plugins/database.js`

```js
const Sequelize = require("sequelize")

module.exports = {
  /**
   * Array of plugins to wait for before running `create`.
   * Name is constructed from the filename by removing the extension and
   * turning it into CammelCase.
   *
   * Ex. "test__name--BEM.plugin.js" => "TestNameBemPlugin"
   */
  depend: ["Lorem"],

  /**
   * Constructor, return value will be considered the plugin's content exposed
   * to routes, middleware and other plugins.
   *
   * @returns  {Promise<any>}  Plugin content
   */
  create: => Lorem => {
    console.log("Checking DB connection")

    // Database connection, model loading etc
    ...
    return {
      Todos: ...,
      Comments: ...,
    }
  }
}
```

## Develop

```bash
git clone git@github.com:mutant-ws/blocks.git && \
  cd blocks && \
  npm run setup
```

Run all `*.test.js` in `tests` folder

```bash
npm test
```

Watch `src` and `tests` folders and re-run tests

```bash
npm run tdd
```

## Changelog

See the [releases section](https://github.com/mutant-ws/blocks/releases) for details.
