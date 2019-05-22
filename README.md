# Dynamic Routes for Next.js

Easy to use universal dynamic routes with i18n for [Next.js](https://github.com/zeit/next.js)

- Express-style route and parameters matching
- Support mlti-languages url pattern 
- Request handler middleware for express & co
- `Link` and `Router` that generate URLs by route definition

## How to use

Install:

```bash
npm install next-i18n-routes --save
```

Create `routes.js` inside your project:

```javascript
const routes = require('./next-i18n-routes')

module.exports = routes({
    prefix: '/project-a',
    locales: ['zh-cn', 'en'],
    defaultLocale: 'zh-hk'
})      
    // Name   Pattern    Page            Ignore Languages            
    // ----   ----       -----           -----
.add('hello')              
    // hello  /hello     hello    
.add('user', '/user/:id', 'profile') 
    // user   /user/:id  profile
.add('yeah', '/yeah', 'yeah', ['en']) 
    // user   /user/:id  profile         [en]

// /hello = locale = zh-hk, page = hello
// /zh-hk/hello = 404
// /en/hello = locale = en, page = hello
// /zh-cn/hello = locale = zh-cn, page = hello
// /en/yeah = 404
```

This file is used both on the server and the client.

API:

- `routes.add([name], pattern = /name, page = name, [ignore languages])`
- `routes.add(object)`

Arguments:

- `name` - Route name
- `pattern` - Route pattern (like express, see [path-to-regexp](https://github.com/pillarjs/path-to-regexp))
- `page` - Page inside `./pages` to be rendered

The page component receives the matched URL parameters merged into `query`

```javascript
export default class Blog extends React.Component {
  static async getInitialProps({ Component, ctx, router }) {
    // router.query.lang
  }

  render () {
    // this.props.url.query.slug
  }
}
```

## On the server

```javascript
// server.js
const next = require('next')
const routes = require('./routes')
const app = next({dev: process.env.NODE_ENV !== 'production'})
const handler = routes.getRequestHandler(app)

// With express
const express = require('express')
app.prepare().then(() => {
  express().use(handler).listen(3000)
})

// Without express
const {createServer} = require('http')
app.prepare().then(() => {
  createServer(handler).listen(3000)
})

// With fastify

const fastify = require('fastify')({ logger: { level: 'error' } })
const Next = require('next')

const routes = require('./routes')
const port = parseInt(process.env.PORT, 10) || 3000
const dev = process.env.NODE_ENV !== 'production'



fastify.register((fastify, opts, next) => {
  const app = Next({ dev })
  const handler = routes.getRequestHandler(app)  
  app
    .prepare()
    .then(() => {
      if (dev) {
        fastify.get('/_next/*', (req, reply) => {
          return app.handleRequest(req.req, reply.res).then(() => {
            reply.sent = true
          })
        })
      }

      fastify.get('/*', (req, reply) => {
          console.log('')
        return handler(req.req, reply.res)
      })

      fastify.setNotFoundHandler((request, reply) => {
        return app.render404(request.req, reply.res).then(() => {
          reply.sent = true
        })
      })

      next()
    })
    .catch(err => next(err))
})

fastify.listen(port, err => {
  if (err) throw err
  console.log(`> Ready on http://localhost:${port}`)
})
```

Optionally you can pass a custom handler, for example:

```javascript
const handler = routes.getRequestHandler(app, ({req, res, route, query}) => {
  app.render(req, res, route.page, query)
})
```

Make sure to use `server.js` in your `package.json` scripts:

```json
"scripts": {
  "dev": "node server.js",
  "build": "next build",
  "start": "NODE_ENV=production node server.js"
}
```

## On the client

Import `Link` and `Router` from your `routes.js` file to generate URLs based on route definition:

### `Link` example

```jsx
// pages/index.js
import {Link} from '../routes'

export default () => (
  <div>
    <div>Welcome to Next.js!</div>
    <Link route='blog' params={{slug: 'hello-world'}}>
      <a>Hello world</a>
    </Link>
    or
    <Link route='/blog/hello-world'>
      <a>Hello world</a>
    </Link>
  </div>
)
```

API:

- `<Link route='name'>...</Link>`
- `<Link route='name' params={params}> ... </Link>`
- `<Link route='/path/to/match'> ... </Link>`

Props:

- `route` - Route name or URL to match (alias: `to`)
- `params` - Optional parameters for named routes

It generates the URLs for `href` and `as` and renders `next/link`. Other props like `prefetch` will work as well.

### `Router` example

```jsx
// pages/blog.js
import React from 'react'
import {Router} from '../routes'

export default class Blog extends React.Component {
  handleClick () {
    // With route name and params
    Router.pushRoute('blog', {slug: 'hello-world'})
    // With route URL
    Router.pushRoute('/blog/hello-world')
  }
  render () {
    return (
      <div>
        <div>{this.props.url.query.slug}</div>
        <button onClick={this.handleClick}>Home</button>
      </div>
    )
  }
}
```

API:

- `Router.pushRoute(route)`
- `Router.pushRoute(route, params)`
- `Router.pushRoute(route, params, options)`

Arguments:

- `route` - Route name or URL to match
- `params` - Optional parameters for named routes
- `options` - Passed to Next.js

The same works with `.replaceRoute()` and `.prefetchRoute()`

It generates the URLs and calls `next/router`

---

Optionally you can provide custom `Link` and `Router` objects, for example:

```javascript
const routes = module.exports = require('next-routes')({
  Link: require('./my/link')
  Router: require('./my/router')
})
```

---

##### Related links

- [zeit/next.js](https://github.com/zeit/next.js) - Framework for server-rendered React applications
- [path-to-regexp](https://github.com/pillarjs/path-to-regexp) - Express-style path to regexp
