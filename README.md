# fastify-rate-limit

[![Greenkeeper badge](https://badges.greenkeeper.io/fastify/fastify-rate-limit.svg)](https://greenkeeper.io/)
[![js-standard-style](https://img.shields.io/badge/code%20style-standard-brightgreen.svg?style=flat)](http://standardjs.com/)
[![Build Status](https://travis-ci.org/fastify/fastify-rate-limit.svg?branch=master)](https://travis-ci.org/fastify/fastify-rate-limit)

A low overhead rate limiter for your routes. Supports Fastify `2.x` versions.

Please refer to [this branch](https://github.com/fastify/fastify-rate-limit/tree/1.x) and related versions for Fastify 1.x compatibility.

## Install
```
npm i fastify-rate-limit
```

## Usage
Register the plugin pass to it some custom option.<br>
This plugin will add an `onRequest` hook to check if the clients (based on their ip) has done too many request in the given timeWindow.
```js
const fastify = require('fastify')()

fastify.register(require('fastify-rate-limit'), {
  max: 100,
  timeWindow: '1 minute'
})

fastify.get('/', (req, reply) => {
  reply.send({ hello: 'world' })
})

fastify.listen(3000, err => {
  if (err) throw err
  console.log('Server listening at http://localhost:3000')
})
```

In case a client reaches the maximum number of allowed requests, a standard Fastify error will be returned to the user with the status code setted to `429`:
```js
{
  statusCode: 429,
  error: 'Too Many Requests',
  message: 'Rate limit exceeded, retry in 1 minute'
}
```
You can change the response by providing a callback to `errorResponseBuilder`.

The response will have some additional headers:

| Header | Description |
|--------|-------------|
|`x-ratelimit-limit`     | how many request the client can do
|`x-ratelimit-remaining` | how many request remain to the client in the timewindow
|`x-ratelimit-reset`     | how many seconds must pass before the rate limit resets
|`retry-after`           | if the max has been reached, the millisecond the client must wait before perform new requests

### Options

You can pass the following options during the plugin registration:
```js
fastify.register(require('fastify-rate-limit'), {
  global : false, // default true
  max: 3, // default 1000
  ban: 2, // default null
  timeWindow: 5000, // default 1000 * 60
  cache: 10000, // default 5000
  whitelist: ['127.0.0.1'], // default []
  redis: new Redis({ host: '127.0.0.1' }), // default null
  skipOnError: true, // default false
  keyGenerator: function(req) { /* ... */ }, // default (req) => req.raw.ip
  errorResponseBuilder: function(req, context) { /* ... */},
  addHeaders: { // default show all the response headers when rate limit is reached
    'x-ratelimit-limit': true,
    'x-ratelimit-remaining': true,
    'x-ratelimit-reset': true,
    'retry-after': true
  }
})
```

- `global` : indicates if the plugin should apply the rate limit setting to all routes within the encapsulation scope
- `max`: is the maximum number of requests a single client can perform inside a timeWindow. It can be a sync function with the signature `(req, key) => {}` where `req` is the Fastify request object and `key` is the value generated by the `keyGenerator`. The function **must** return a number.
- `ban`: is the maximum number of 429 responses to return to a single client before returning 403. When the ban limit is exceeded the context field will have `ban=true` in the errorResponseBuilder. This parameter is an in-memory counter and could not work properly in a distributed environment.
- `timeWindow:` the duration of the time window. It can be expressed in milliseconds or as a string (in the [`ms`](https://github.com/zeit/ms) format)
- `cache`: this plugin internally uses a lru cache to handle the clients, you can change the size of the cache with this option
- `whitelist`: array of string of ips to exclude from rate limiting. It can be a sync function with the signature `(req, key) => {}` where `req` is the Fastify request object and `key` is the value generated by the `keyGenerator`. If the function return a truthy value, the request will be excluded from the rate limit.
- `redis`: by default this plugins uses an in-memory store, which is fast but if you application works on more than one server it is useless, since the data is store locally.<br>
You can pass a Redis client here and magically the issue is solved. To achieve the maximum speed, this plugins requires the use of [`ioredis`](https://github.com/luin/ioredis). **Note:**: the [default parameters](https://github.com/luin/ioredis/blob/v4.16.0/API.md#new_Redis_new) of a redis connection are not the fastest to provide a rate-limit. We suggest to customize the `connectTimeout` and `maxRetriesPerRequest`.
- `store`: a custom store to track requests and rates which allows you to use your own storage mechanism (such as using an RDBMS, MongoDB, etc...) as well as further customizing the logic used in calculating the rate limits. A simple example is provided below as well as a more detailed example using Knex.js can be found in the [`example/`](https://github.com/fastify/fastify-rate-limit/tree/master/example) folder
- `skipOnError`: if `true` it will skip errors generated by the storage (eg, redis not reachable).
- `keyGenerator`: a function to generate a unique identifier for each incoming request. Defaults to `(req) => req.ip`, the IP is resolved by fastify using `req.connection.remoteAddress` or `req.headers['x-forwarded-for']` if [trustProxy](https://www.fastify.io/docs/master/Server/#trustproxy) option is enabled. Use it if you want to override this behavior
- `errorResponseBuilder`: a function to generate a custom response object. Defaults to `(req, context) => ({statusCode: 429, error: 'Too Many Requests', message: ``Rate limit exceeded, retry in ${context.after}``})`
- `addHeaders`: define which headers should be added in the response when the limit is reached. Defaults all the headers will be shown

`keyGenerator` example usage:
```js
fastify.register(require('fastify-rate-limit'), {
  /* ... */
  keyGenerator: function(req) {
    return req.headers['x-real-ip'] // nginx
    || req.headers['x-client-ip'] // apache
    || req.headers['x-forwarded-for'] // use this only if you trust the header
    || req.session.username // you can limit based on any session value
    || req.raw.ip // fallback to default
})
```

Variable `max` example usage:
```js
// In the same timeWindow, the max value can change based on request and/or key like this
fastify.register(rateLimit, {
  /* ... */
  keyGenerator (req) { return req.headers['service-key'] },
  max: (req, key) => { return key === 'pro' ? 3 : 2 },
  timeWindow: 1000
})
```

`errorResponseBuilder` example usage:
```js
fastify.register(require('fastify-rate-limit'), {
  /* ... */
  errorResponseBuilder: function(req, context) {
    return {
      code: 429,
      error: 'Too Many Requests',
      message: `I only allow ${context.max} requests per ${context.after} to this Website. Try again soon.`,
      date: Date.now()
    }
  }
})
```

Dynamic `whitelist` example usage:
```js
fastify.register(require('fastify-rate-limit'), {
  /* ... */
  whitelist: function(req, key) {
    return req.headers['x-app-client-id'] === 'internal-usage'
  }
})
```

Custom `store` example usage:
```js
function CustomStore (options) {
  this.options = options
  this.current = 0
}

CustomStore.prototype.incr = function (key, cb) {
  const timeWindow = this.options.timeWindow
  this.current++
  cb(null, { current: this.current, ttl: timeWindow - (this.current * 1000) })
}

CustomStore.prototype.child = function (routeOptions) {
  // We create a merged copy of the current parent parameters with the specific
  // route parameters and pass them into the child store.
  const childParams = Object.assign(this.options, routeOptions.config.rateLimit)
  const store = new CustomStore(childParams)
  // Here is where you may want to do some custom calls on the store with the information
  // in routeOptions first...
  // store.setSubKey(routeOptions.method + routeOptions.url)
  return store
}

fastify.register(require('fastify-rate-limit'), {
  /* ... */
  store: CustomStore
})
```

### Options on the endpoint itself

Rate limiting can be configured also for some routes, applying the configuration independently.

For example the `whitelist` if configured:
 - on the plugin registration will affect all endpoints within the encapsulation scope
 - on the route declaration will affect only the targeted endpoint

The global whitelist is configured when registering it with `fastify.register(...)`.

The endpoint whitelist is set on the endpoint directly with the `{ config : { rateLimit : { whitelist : [] } } }` object.

ACL checking is performed based on the value of the key from the `keyGenerator`.

In this example we are checking the IP address, but it could be a whitelist of specific user identifiers (like JWT or tokens):

```js
const fastify = require('fastify')()

fastify.register(require('fastify-rate-limit'),
  {
    global : false, // don't apply these settings to all the routes of the context
    max: 3000, // default global max rate limit
    whitelist: ['192.168.0.10'], // global whitelist access. 
    redis: redis, // custom connection to redis
  })

// add a limited route with this configuration plus the global one
fastify.get('/', {
  config: {
    rateLimit: {
      max: 3,
      timeWindow: '1 minute'
    }
  }
}, (req, reply) => {
  reply.send({ hello: 'from ... root' })
})

// add a limited route with this configuration plus the global one
fastify.get('/private', {
  config: {
    rateLimit: {
      max: 3,
      timeWindow: '1 minute'
    }
  }
}, (req, reply) => {
  reply.send({ hello: 'from ... private' })
})

// this route doesn't have any rate limit
fastify.get('/public', (req, reply) => {
  reply.send({ hello: 'from ... public' })
})

// add a limited route with this configuration plus the global one
fastify.get('/public/sub-rated-1', {
  config: {
    rateLimit: {
      timeWindow: '1 minute',
      whitelist: ['127.0.0.1'],
      onExceeding: function (req) {
        console.log('callback on exceededing ... executed before response to client')
      },
      onExceeded: function (req) {
        console.log('callback on exceeded ... to black ip in security group for example, req is give as argument')
      }
    }
  }
}, (req, reply) => {
  reply.send({ hello: 'from sub-rated-1 ... using default max value ... ' })
})
```

In the route creation you can override the same settings of the plugin registration plus the additionals options:

- `onExceeding` : callback that will be executed each time a request is made to a route that is rate limited
- `onExceeded` : callback that will be executed when a user reached the maximum number of tries. Can be useful to blacklist clients


<a name="license"></a>
## License
**[MIT](https://github.com/fastify/fastify-rate-limit/blob/master/LICENSE)**<br>

Copyright © 2018 Tomas Della Vedova
