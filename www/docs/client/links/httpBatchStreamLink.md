---
id: httpBatchStreamLink
title: HTTP Batch Stream Link
sidebar_label: HTTP Batch Stream Link
slug: /client/links/httpBatchStreamLink
---

`httpBatchStreamLink` is a [**terminating link**](./overview.md#the-terminating-link) that batches an array of individual tRPC operations into a single HTTP request that's sent to a single tRPC procedure (equivalent to [`httpBatchLink`](./httpBatchLink.md)), but doesn't wait for all the responses of the batch to be ready and streams the responses as soon as any data is available.

## Options

Options are identical to [`httpBatchLink options`](./httpBatchLink.md#options).

## Usage

> All usage and options are identical to [`httpBatchLink`](./httpBatchLink.md).

:::note
If you require the ability to change/set response headers (which includes cookies) from within your procedures, make sure to use `httpBatchLink` instead! This is due to the fact that `httpBatchStreamLink` does not support setting headers once the stream has begun. [Read more](https://trpc.io/docs/client/links/httpBatchLink).
:::

You can import and add the `httpBatchStreamLink` to the `links` array as such:

```ts title="client/index.ts"
import { createTRPCClient, httpBatchStreamLink } from '@trpc/client';
import type { AppRouter } from '../server';

const client = createTRPCClient<AppRouter>({
  links: [
    httpBatchStreamLink({
      url: 'http://localhost:3000',
    }),
  ],
});
```

After that, you can make use of batching by setting all your procedures in a `Promise.all`. The code below will produce exactly **one** HTTP request and on the server exactly **one** database query:

```ts
const somePosts = await Promise.all([
  trpc.post.byId.query(1),
  trpc.post.byId.query(2),
  trpc.post.byId.query(3),
]);
```

## Streaming mode

When batching requests together, the behavior of a regular `httpBatchLink` is to wait for all requests to finish before sending the response. If you want to send responses as soon as they are ready, you can use `httpBatchStreamLink` instead. This is useful for long-running requests.

```ts title="client/index.ts"
import { createTRPCClient, httpBatchStreamLink } from '@trpc/client';
import type { AppRouter } from '../server';

const client = createTRPCClient<AppRouter>({
  links: [
    httpBatchStreamLink({
      url: 'http://localhost:3000',
    }),
  ],
});
```

Compared to a regular `httpBatchLink`, a `httpBatchStreamLink` will:

- Cause the requests to be sent with a `trpc-accept: application/jsonl` header
- Cause the response to be sent with a `transfer-encoding: chunked` and `content-type: application/jsonl`
- Remove the `data` key from the argument object passed to `responseMeta` (because with a streamed response, the headers are sent before the data is available)

## Async generators and deferred promises {#generators}

You can try this out on the homepage of tRPC.io: [https://trpc.io/?try=minimal#try-it-out](/?try=minimal#try-it-out)

```ts twoslash
// @target: esnext
// @filename: trpc.ts
import { initTRPC } from '@trpc/server';

const t = initTRPC.create({});

export const router = t.router;
export const publicProcedure = t.procedure;

// ---cut---
// @filename: server.ts
import { publicProcedure, router } from './trpc';

const appRouter = router({
  examples: {
    iterable: publicProcedure.query(async function* () {
      for (let i = 0; i < 3; i++) {
        await new Promise((resolve) => setTimeout(resolve, 500));
        yield i;
      }
    }),
  },
});

export type AppRouter = typeof appRouter;


// @filename: client.ts
import { createTRPCClient, httpBatchStreamLink } from '@trpc/client';
import type { AppRouter } from './server';

const trpc = createTRPCClient<AppRouter>({
  links: [
    httpBatchStreamLink({
      url: 'http://localhost:3000',
    }),
  ],
});
const iterable = await trpc.examples.iterable.query();
//      ^?

for await (const value of iterable) {
  console.log('Iterable:', value);
  //                         ^?
}
```

## Compatibility (client-side)

### Browsers

Browser support should be identical to [`fetch`](https://caniuse.com/fetch) support.

### Node.js / Deno

For runtimes other than the browser ones, the `fetch` implementation should support streaming, meaning that the response obtained by `await fetch(...)` should have a `body` property of type `ReadableStream<Uint8Array> | NodeJS.ReadableStream`, meaning that:

- either `response.body.getReader` is a function that returns a `ReadableStreamDefaultReader<Uint8Array>` object
- or `response.body` is a `Uint8Array` `Buffer`

This includes support for `undici`, `node-fetch`, native Node.js fetch implementation, and WebAPI fetch implementation (browsers).

### React Native

Receiving the stream relies on the `TextDecoder` and `TextDecoderStream` APIs, which is not available in React Native. It's important to note that if your `TextDecoderStream` polyfill does not automatically polyfill `ReadableStream` and `WritableStream` those will also need to be polyfilled. If you still want to enable streaming, you need to polyfill those.

You will also need to overide the default fetch in the `httpBatchStreamLink` configuration options. In the below example we will be using the [Expo fetch](https://docs.expo.dev/versions/latest/sdk/expo/) package for the fetch implementation.

```typescript
httpBatchStreamLink({
  fetch: (url, opts) =>
    fetch(url, {
      ...opts,
      reactNative: { textStreaming: true },
    }),
  ...restOfConfig,
});
```

## Compatibility (server-side)

> ⚠️ for **aws lambda**, `httpBatchStreamLink` is not supported (will simply behave like a regular `httpBatchLink`). It should not break anything if enabled, but will not have any effect.

> ⚠️ for **cloudflare workers**, you need to enable the `ReadableStream` API through a feature flag: [`streams_enable_constructors`](https://developers.cloudflare.com/workers/platform/compatibility-dates#streams-constructors)

## Reference

You can check out the source code for this link on [GitHub.](https://github.com/trpc/trpc/blob/main/packages/client/src/links/httpBatchStreamLink.ts)

## Configure a ping option to keep the connection alive

When setting up your root config, you can pass in a `jsonl` option to configure a ping option to keep the connection alive.

```ts
import { initTRPC } from '@trpc/server';

const t = initTRPC.create({
  jsonl: {
    pingMs: 1000,
  },
});
```
