---
title: Exploring Advanced Request Handlers in WarpDrive
published: true
draft: true
---

# Exploring Advanced Request Handlers in WarpDrive

[WarpDrive](https://warp-drive.io) provides applications with a managed fetch pipeline
following the chain-of-responsibility principle. When a request is issued it triggers the
registered pipline.

Pipelines can be simple, for instance below is the setup for a pipeline that just calls
fetch with minimal processing.

```ts
const manager = new RequestManager()
  .use([Fetch]);
```

> [!TIP]
> For a good introduction to managed fetch in WarpDrive, read [this post](https://runspired.com/2024/01/31/modern-ember-data.html) about modern request paradigms in EmberData (WarpDrive is the modern evolution of EmberData).

Most often, handlers should be simple. Providing minimal decoration for universal responsibilities
like setting up authentication tokens or adding tracing for observability tooling.

```ts
const API_ROOT = `https://${location.hostname}/api/`;

function isOwnAPI(url: string): boolean {
	return url.startsWith(API_ROOT) || url.startsWith('/');
}

function decorateOwnRequest(session, request) {
	const headers = new Headers(request.headers);
	headers.set('X-Amzn-Trace-Id', `Root=${crypto.randomUUID()}`);

	const token = session?.token;
	if (token) {
		headers.set('token', token);
	}

	return Object.assign({}, request, { headers });
}


export class AuthHandler {
  constructor(sessionService) {
    this.session = sessionService;
  }

  async request({ request }, next) {
		if (!request.url || !isOwnAPI(request.url)) {
			return next(request);
		}

    return next(decorateOwnRequest(request));
  }
}
```

In even this simple example, we can start to see the rumblings of why handlers can be advanced:

- Requests passed to handlers are immutable, and so `decorateOwnRequest` generates a new request
from the original to apply the additional headers.
- requests have the ability to see the result of calling `next` (this is why we return the result of the call).
- next can be called with a different request than the handler received

There's a fourth feature of handlers thats less obvious: a handler can call `next` as many times
as it wants, but it can only return a single response.

These four features – immutability, access to the response, ability to pass along a different request,
and the ability to call `next` any number of times – work together to allow applications to build highly
advanced handlers to  fulfill requests however they best see fit.

Below, lets dive into two such advanced handlers that I recently worked on: a `pagination engine` and
a `recommendation engine`.

## Implementing a Pagination Engine

When using [JSON:API](https://jsonapi.org/format/#fetching-pagination), pagination is implemented using
links at the top level of the response which describe where to get other pages of data related to the
current response.

```json
{
  "links": {
    "self": "https://example.com/api/users",
    "prev": null,
    "next": "https://example.com/api/users?page[offset]=25&page[limit]=25",
    "first": "https://example.com/api/users?page[offset]=0&page[limit]=25",
    "last": "https://example.com/api/users?page[offset]=75&page[limit]=25"
  },
  "data": [
    // ... list of users ... //
  ]
}
```

WarpDrive adopted this convention as part of its Cache API. When a cache receives a request, it returns a
"response document" that describes what it found in the content for that request. While the cache does not
need to be in JSON:API format, this response document needs to be in a universal format and so WarpDrive
adopted the top-level structure from JSON:API to use due to its expressiveness.

When the WarpDrive Store creates a reactive wrapper for this response document, it exposes utility methods
for working with the request. For instance, to fetch the next page, we can call `next`

```ts
const currentPage = await store.request({ url: '/api/users' });
const nextPage = await currentPage.next();
```

This feature allows applications to construct advanced pagination logic in a highly conventional
manner by abstracting the mechanics of pagination as an unimportant detail of the underlying request,
but it seemingly breaks down if an API does not provide pagination links 🙈

Handlers to the rescue!

Our PaginationEngine is going to process our requests for two separate scenarios:

- GET based pagination using queryParams
- POST based pagination using params in the body (for QUERY via POST semantics)

For this example, lets assume our `GET` endpoints (e.g. `GET /api/users`) accept
the params `limit` and `offset` and in the response meta we receive a total:

```
{ meta: { total: number } }
```

Lets assume our `POST` endpoints (e.g. `POST /api/users`) accepts the same `limit` and `offset`
top level, e.g.

```ts
await fetch('/api/users', {
  method: 'POST',
  body: JSON.stringify({ limit: 25, offset: 0 })
}).toJSON();
```

In both cases, we assume that other parameters unrelated to pagination may be present. Let's dive
into a implementing the simpler `GET` case first.




## Implementing a Recommendation Engine

pagination-engine

multi-plexing

