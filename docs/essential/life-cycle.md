---
title: Life Cycle - ElysiaJS
head:
    - - meta
      - property: 'og:title'
        content: Life Cycle - ElysiaJS

    - - meta
      - name: 'description'
        content: Lifecycle event is a concept for each stage of Elysia processing, "Life Cycle" or "Hook" is an event listener to intercept, and listen to those events cycling around. Hook allows you to transform data running through the data pipeline. With the hook, you can customize Elysia to its fullest potential.

    - - meta
      - property: 'og:description'
        content: Lifecycle event is a concept for each stage of Elysia processing, "Life Cycle" or "Hook" is an event listener to intercept, and listen to those events cycling around. Hook allows you to transform data running through the data pipeline. With the hook, you can customize Elysia to its fullest potential.
---

# Life Cycle

Also knows as middleware with name in Express or Hook in Fastify.

Imagine we want to return a text of HTML.

We need to set **"Content-Type"** headers as **"text/html"** to for browser to render HTML.

Explicitly specify that response is HTML could be repetitive if there are a lot of handlers, says ~200 endpoints.

We can see a duplicated code for just to specify that response is HTML.

But what if after we sent a response, we could detect if a response is an HTML string then append headers automatically?

That's when the concept of Life Cycle comes into play.

---

Life Cycle allows us to intercept important events, and customize the behavior of Elysia, like adding an HTML header automatically.

Elysia's Life Cycle event can be illustrated as the following.
![Elysia Life Cycle Graph](/assets/lifecycle.webp)

You don't have to understand/memorize all of the events in one go, we will be covering each on the next chapter.

## Events

Most of the events you are going to use are highlighted in the blue area but to summarize:

Elysia does the following for every request:

1. **Request**
    - Notify new event is received, providing only the most minimal context to reduce overhead
    - Best for:
        - Caching
        - Analytics
2. **Parse**
    - Parse body and add to `Context.body`
    - Best for:
        - Providing custom body-parser
3. **Transform**
    - Modify `Context` before validation
    - Best for:
        - Mutate existing context to conform with validation.
        - Adding new context (derive this)
4. **Validation** (not interceptable)
    - Strictly validate incoming request provided by `Elysia.t`
5. **Before Handle**
    - Custom validation before route handler
    - **If value is returned, route handler will be skipped**
    - Best for:
        - Providing custom requirements to access route, eg. user session, authorization.
6. **Handle** (Route Handler)
    - A callback function of each route
7. **After Handle**
    - Map returned value into a response
    - Best for:
        - Add custom headers or transform the value into a new response
8. **Error**
    - Capture error when thrown
    - Best for:
        - Provide a custom error response
        - Catching error response
9. **Response**
    - Executed after response sent to the client
    - Best for:
        - Cleaning up response
        - Analytics

These events are designed to help you decouple code into smaller reusable pieces instead of having long, repetitive code in a handler.

## Hook

We refer to each function that intercepts the life cycle event as **"hook"**, as the function hook into the lifecycle event.

Hooks can be categorized into 2 types:

1. Local Hook: Execute on a specific route
2. Global Hook: Execute on every route

::: tip
The hook will accept the same Context as a handler, you can imagine you adding a route handler but at a specific point.
:::

## Local Hook

The local hook is executed on a specific route.

To use a local hook, you can inline hook into a route handler:

```typescript
import { Elysia } from 'elysia'
import { isHtml } from '@elysiajs/html'

new Elysia()
    .get('/', () => '<h1>Hello World</h1>', {
        afterHandle({ response, set }) {
            if (isHtml(response))
                set.headers['Content-Type'] = 'text/html; charset=utf8'
        }
    })
    .get('/hi', () => '<h1>Hello World</h1>')
    .listen(3000)
```

The response should be listed as follows:

| Path | Content-Type             |
| ---- | ------------------------ |
| /    | text/html; charset=utf8  |
| /hi  | text/plain; charset=utf8 |

## Global Hook

Register hook into **every** handler that came after.

To add a global hook, you can use `.on` followed by a life cycle event in camelCase:

```typescript
import { Elysia } from 'elysia'
import { isHtml } from '@elysiajs/html'

new Elysia()
    .get('/none', () => '<h1>Hello World</h1>')
    .onAfterHandle(({ response, set }) => {
        if (isHtml(response))
            set.headers['Content-Type'] = 'text/html; charset=utf8'
    })
    .get('/', () => '<h1>Hello World</h1>')
    .get('/hi', () => '<h1>Hello World</h1>')
    .listen(3000)
```

The response should be listed as follows:

| Path  | Content-Type             |
| ----- | ------------------------ |
| /     | text/html; charset=utf8  |
| /hi   | text/html; charset=utf8  |
| /none | text/plain; charset=utf8 |

Events from other plugins are also applied to the route so the order of code is important.

## Order of code

The order of Elysia's life-cycle code is very important.

Elysia's life-cycle event is stored as a queue, aka first-in first-out. So Elysia will **always** respect the order of code from top-to-bottom followed by the order of life-cycle events.

```typescript
import { Elysia } from 'elysia'

new Elysia()
    .onBeforeHandle(() => {
        console.log('1')
    })
    .onAfterHandle(() => {
        console.log('3')
    })
    .get('/', () => 'hi', {
        beforeHandle() {
            console.log('2')
        }
    })
    .listen(3000)
```

Console should log as the following:

```bash
1
2
3
```

## Hook type
Starting from Elysia 1.0 introduce a **hook type**, to specify if the hook should be local-only, or global.

Elysia hook type are as the following:
- local (default) - apply to only current instance and descendant only
- scoped - apply to only 1 ascendant, current instance and descendants
- global - apply to all instance that apply the plugin (all ascendants, current, and descendants)

If not specified, hook is local by default.

::: tip
Starting from Elysia 1.0 hook is local by default while Elysia < 1.0 will be global only.

This is a breaking change.
:::

To specify hook's type, add a `{ as: hookType }` to hook.

To apply hook to globally, we need to specify hook as global.
```typescript
const plugin = new Elysia()
    .onBeforeHandle(() => { // [!code --]
    .onBeforeHandle({ as: 'global' }, () => { // [!code ++]
        console.log('hi')
    })
    .get('/child', () => 'log hi')

const main = new Elysia()
    .use(plugin)
    .get('/parent', () => 'log hi')
```

Let's create a plugin to illustrate how hook type work.

```typescript
// ? Value base on table value provided below
const type = 'local'

const child = new Elysia()
    .get('/child', () => 'hello')

const current = new Elysia()
    .onBeforeHandle({ as: type }, () => {
        console.log('hi')
    })
    .use(child)
    .get('/current', () => 'hello')

const parent = new Elysia()
    .use(current)
    .get('/parent', () => 'hello')

const main = new Elysia()
    .use(parent)
    .get('/main', () => 'hello')
```

By changing the `type` value, the result should be as follows:

| type       | child | current | parent | main |
| ---------- | ----- | ------- | ------ | ---- |
| 'local'    | ✅    | ✅       | ❌     | ❌   | 
| 'scope'    | ✅    | ✅       | ✅     | ❌   | 
| 'global'   | ✅    | ✅       | ✅     | ✅   | 
