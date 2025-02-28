---
title: Handler - ElysiaJS
head:
    - - meta
      - property: 'og:title'
        content: Handler - ElysiaJS

    - - meta
      - name: 'description'
        content: handler is a function that responds to the request for each route. Accepting request information and returning a response to the client. Handler can be registered through Elysia.get / Elysia.post

    - - meta
      - property: 'og:description'
        content: handler is a function that responds to the request for each route. Accepting request information and returning a response to the client. Handler can be registered through Elysia.get / Elysia.post
---

# Handler

After a resource is located, a function that respond is refers as **handler**

```typescript
import { Elysia } from 'elysia'

new Elysia()
    // the function `() => 'hello world'` is a handler
    .get('/', () => 'hello world')
    .listen(3000)
```

## Context

Context is an request's information sent to server.

```typescript
import { Elysia } from 'elysia'

new Elysia()
    .get('/', ({ path }) => path)
    .listen(3000)
```

We will be covering context property in the next page [context](/essential/context), for now lets see what handler is capable of.

## Set

**set** is a mutable property that form a response accessible via `Context.set`.

- **set.status** - Set custom status code
- **set.headers** - Append custom headers
- **set.redirect** - Append redirect


## Status
We can return a custom status code by using either:

- **error** function (recommended)
- **set.status**

## error
A dedicated `error` function for returning status code with response.

```typescript
import { Elysia } from 'elysia'

new Elysia()
    .get('/', ({ error }) => error(418, "I like tea"))
    .listen(3000)
```

It's recommend to use `error` inside main handler as it has better inference:

- allows TypeScript to check if a return value is correctly type to response schema
- autocompletion for type narrowing base on status code
- type narrowing for error handling using End-to-end type safety (Eden)

## set.status
Set a default status code if not provided by.

It's recommended to use in a plugin that only only need to return a specific status code while allowing user to return a custom value for example, HTTP 201/206 or 403/405 etc.

```typescript
import { Elysia } from 'elysia'

new Elysia()
    .onBeforeHandle(({ set }) => {
        set.status = 418

        return 'I like tea'
    })
    .get('/', () => 'hi')
    .listen(3000)
```

::: tip
HTTP Status indicates the type of response. If the route handler is executed successfully without error, Elysia will return the status code 200.
:::

You can also set a status code using the common name of the status code instead of using a number.

```typescript
import { Elysia } from 'elysia'

new Elysia()
    .get('/', ({ set }) => {
        // with auto-completion
        set.status = "I'm a teapot"

        return 'I like tea'
    })
    .listen(3000)
```

## set.headers
Allowing us to append or delete a response headers represent as Object.

```typescript
import { Elysia } from 'elysia'

new Elysia()
    .get('/', ({ set }) => {
        set.headers['x-powered-by'] = 'Elysia'

        return 'a mimir'
    })
    .listen(3000)
```

## set.redirect
Redirect a request to another resource.

```typescript
import { Elysia } from 'elysia'

new Elysia()
    .get('/', ({ set }) => {
        set.redirect = 'https://youtu.be/whpVWVWBW4U?si=duN5cBbJuWgCrQRA&t=8'
    })
    .listen(3000)
```

When using redirect, returned value is not required and will be ignored. As response will be from another resource.

## Response

Elysia is built on top of Web Standard Request/Response.

To comply with the Web Standard, a value returned from route handler will be mapped into a [Response](https://developer.mozilla.org/en-US/docs/Web/API/Response) by Elysia.

Letting you focus on business logic rather than boilerplate code.

```typescript
import { Elysia } from 'elysia'

new Elysia()
    // Equivalent to "new Response('hi')"
    .get('/', () => 'hi')
    .listen(3000)
```

If you prefer an explicit Response class, Elysia also handles that automatically.

```typescript
import { Elysia } from 'elysia'

new Elysia()
    .get('/', () => new Response('hi'))
    .listen(3000)
```

::: tip
Using a primitive value or `Response` has near identical performance (+- 0.1%), so pick the one you prefer, regardless of performance.
:::

## Static Content

Static Content is a type of handler that always returns the same value, for instance file, hardcoded-value.

In Elysia, static content can be registered by providing an actual value instead of a function.

```typescript
import { Elysia } from 'elysia'

new Elysia()
    .get('/', 'Hello Elysia')
    .get('/video', Bun.file('kyuukurarin.mp4'))
    .listen(3000)
```

This allows Elysia to compile the response ahead of time to optimize performance.

::: tip
Static response is not a cache.

It doesn't append and inherits any cache capability nor behavior of cache headers.
:::
