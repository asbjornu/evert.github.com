---
title: "Curveball - A typescript microframework"
date: "2020-02-26 20:19:04 UTC"
draft: true
tags:
  - http2
  - hypermedia
  - api
  - rest
  - push
  - hal
  - curveball
  - typescript
  - microframework
  - koa
  - expressjs
  - nodejs
  - javascript
  - lambda
---

Since mid-2018 I've been working on a new micro-framework, written in
typescript. The framework competes with [Express][1], and takes heavy
inspiration from [Koa][2].

If you only ever worked with Express, I feel that for most people this
project will feel like a drastic step up. Express was really written in an
earlier time of Node.js, before Promises and async/await were commonplace,
so first and foremost the biggest change is the use of async/await middlewares
throughout.

If you came from Koa, that will already be familiar. Compared to Koa, these
are the major differences:

* Curveball is written in Typescript
* It has strong built-in support HTTP/2 push.
* Native support for running servers on AWS Lambda, without the use of
  [strange hacks][3].
* Curveball's request/response objects are decoupled from the Node.js `http`
  library.

We've been using this in a variaty of (mostly API) projects for the past
few years, and it's been working really well for us. We're also finding that
it tends to be a pretty 'sticky' product. People exposed to it, tend to want
to use it for their next project too.

Examples
--------

### Hello world

```typescript
import { Application } from '@curveball/core';

const app = new Application();
app.use( ctx => {
  ctx.response.type = 'text/plain';
  ctx.body = 'hello world';
});

app.listen(80);
```

### Hello world on AWS Lambda

```typescript
import { Application } from '@curveball/core';
import { handler } from '@curveball/aws-lambda';

const app = new Application();
app.use( ctx => {
  ctx.response.type = 'text/plain';
  ctx.response.body = 'hello world';
});

exports.handler = handler(app);
```

### HTTP/2 Push

```typescript
const app = new Application();
app.use( ctx => {
  ctx.response.type = 'text/plain';
  ctx.body = 'hello world';

  ctx.push( pushCtx => {

    pushCtx.path = '/sub-item';
    pushCtx.response.type = 'text/html';
    pushCtx.response.body = '<h1>Automatically pushed!</h1>';

  });


});
```

### Resource-based controllers

Controllers are optional and opiniated. A single controller should only ever
manage one type of resource, or one route.


```typescript
import { Application, Context } from '@curveball/core';
import { Controller } from '@curveball/controller';

const app = new Application();

class MyController extends Controller {

  get(ctx: Context) {

    // This is automatically triggered for GET requests

  }

  put(ctx: Context) {

    // This is automatically triggered for PUT requests

  }

}

app.use(new MyController());
```

### Routing

The recommended pattern is to use exactly 1 controller per route.

```typescript
import { Application } from '@curveball/core';
import router from '@curveball/router';

const app = new Application();

app.use(router('/articles', new MyCollectionController());
app.use(router('/articles/:id', new MyItemController());
```

### Content-negotation in controllers

```typescript
import { Context } from '@curveball/core';
import { Controller, method, accept } from '@curveball/controller';

class MyController extends Controller {

  @accept('html')
  @method('GET')
  async getHTML(ctx: Context) {

    // This is automatically triggered for GET requests with
    // Accept: text/html

  }

  @accept('json')
  @method('GET')
  async getJSON(ctx: Context) {

    // This is automatically triggered for GET requests with
    // Accept: application/json

  }

}
```

### Emitting errors

To emit a HTTP error, it's possible to set `ctx.status`, but easier to just
throw a related exception.


```typescript
function myMiddleware(ctx: Context, next: Middleware) {

  if (ctx.method !== 'GET') {
    throw new MethodNotAllowed('Only GET is allowed here');
  }
  await next();

}
```

The project also ships with a [middleware][5] to automatically generate
[RFC7807][4] `application/problem+json` responses.

### Transforming HTTP responses in middlewares

With express middlewares it's easy to do something *before* a request was
handled, but if you ever want to transform a response in a middleware, this
can only be achieved through a complicated hack.

This is due to the fact that responses are immediately written to the TCP
sockets, and once written to the socket it's effecftively gone.

So do do things like gzipping responses, Express middleware authors needs to
mock the response stream and intercept any bytes sent to it. This can be
clearly seen in the express-compression source:
<https://github.com/expressjs/compression/blob/master/index.js>.

Curveball does not do this. Response bodies are buffered and available by
middlewares.

For example, the following middleware looks for a HTTP Accept header of
`text/html` and automatically transforms JSON to a simple HTML output:

```typescript
app.use( async (ctx, next) => {

  // Let the entire middleware stack run
  await next();
  
  // HTML encode JSON responses if the client was a browser.
  if (ctx.accepts('text/html') && ctx.response.type ==== 'application/json') {
    ctx.response.type = 'text/html';
    ctx.response.body = '<h1>JSON source</h1><pre>' + JSON.stringify(ctx.response.body) + '</pre>';
  }

});
```

To achieve the same thing in express would be quite complicated.

You might wonder if this is bad for performance for large files. You would be
completely right, and this is not solved yet.

However, instead of writing directly to the output stream the intent for this
is to allow users to set a callback on the `body` property, so writing the
body will not be buffered, just deferred. The complexity of implementing these
middlewares will not change.

Curveball also ships with an [API browser][6] that automatically transforms
JSON in to traversable HTML, and automatically parses HAL links and HTTP Link
headers.

Stable release
--------------

We're currently on the 11'th beta, and closing in on a stable release. Changes
will at this point be minor.

If you have thoughts or feedback on this project, it would be really helpful
to hear. Don't hesitate to leave comments, questions or suggestions as a
[Github Issue][7].


[1]: https://expressjs.com/
[2]: https://koajs.com/
[3]: https://github.com/awslabs/aws-serverless-express/blob/master/src/index.js
[4]: https://tools.ietf.org/html/rfc7807
[5]: https://github.com/curveball/problem/
[6]: https://github.com/curveball/hal-browser
[7]: https://github.com/curveball/core/issues