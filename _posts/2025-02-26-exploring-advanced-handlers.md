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

> **TIP**
>
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
the params `limit` and `offset` and in the response meta we receive back this
limit, offset and total:

```ts
{ meta: { total: number; limit: number; offset: number; } }
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

```ts
const API_ROOT = `https://${location.hostname}/api/`;

function isOwnAPI(url: string): boolean {
  return url.startsWith(API_ROOT) || url.startsWith('/');
}

function maybeAddPaginationLinks(request, doc) {
  if (doc.content.meta?.total) {
    const links = doc.content.links = doc.content.links ?? {};
    const { limit, offset, total } = doc.content.meta;
    const originalUrl = new Url(request.url);
    
    // add current
    originalUrl.queryParams.set('limit', limit);
    originalUrl.queryParams.set('offset', offset);
    links.self = String(originalUrl);

    // add first
    if (offset === 0) {
      links.first = links.self;
    } else {
      originalUrl.queryParams.set('offset', 0);
      links.first = String(originalUrl);
    }

    // add prev
    if (offset > 0) {
      originalUrl.queryParams.set('offset', offset - limit);
      links.prev = String(originalUrl);
    } else {
      links.prev = null;
    }
    
    // add next
    if (offset + limit < total) {
      originalUrl.queryParams.set('offset', offset + limit);
      links.next = String(originalUrl);
    } else {
      links.next = null;
    }

    // add last
    const lastOffset = (Math.ceil(51 / 25) - 1) * limit;
    originalUrl.queryParams.set('offset', lastOffset);
    links.last = String(originalUrl);
  }

  return doc;
}

export const BasicPaginationHandler = {
  request({ request }, next) {
    if (!request.url || !isOwnAPI(request.url)) {
      return next(request);
    }

    return next(request)
      .then(doc => maybeAddPaginationLinks(request, doc));
  }
}
```

And that's it! Now our response has the full range of links it can use for pagination
features derived from our params based contract. In many cases, an API may not provide
access to meta like this, but we can use limit and offset information from the original
URL, and provide `null` for the `last` link, allowing cursor based navigation via next/prev
as well as a return to start, which for most features is more than enough.

But what happens when our endpoint requires the use of `POST` ? Let's explore!

For this challenge, we are going to take advantage of the fact that the *first* request
issued by the APP is issued directly (e.g. not via `await currentPage.next()` or similar
but via `store.request({ url: '/users', method: 'POST', body })`).

This gives us a key distinction to make when handling a request

- requests that are the *first* in a series
- requests that are a *continuation* of a series

In this approach we'll want some state for book-keeping so we are going to implement a class
for our handler and store a Map that holds on to a bit of info from first requests that
we will need for a continuation request.

The general strategy is that we generate a link (really just a unique string) that the app can
use to make a `GET` request conceptually that works with `await page.next()`, but switch the
request out for a `POST` request when we encounter the generated link in our handler.

We also need a way to know that a request wants to opt into this pagination behavior,
there's a number of heuristics we could use but for this example we're going to be
explicit and pass an option into the initial request that tells us to make use of this
feature.

```ts
store.request({
  url: '/api/users',
  method: 'POST',
  body: JSON.stringify({ limit: 25, offset: 0 }),
  options: { usePaginationEngine: true }
})
```

```ts
class QUERYPaginationEngine {
  urlMap = new Map();

  request({ request }, next) {
    if (!request.url || !isOwnAPI(request.url)) {
      return next(request);
    }
  
    if (request.options?.usePaginationEngine) {
      return handleInitialRequest(request, next, this.urlMap);
    }

    if (this.urlMap.has(request.url)) {
      return handleContinuationRequest(request, next, this.urlMap);
    }

    return next(request);
  }
}
```

Above, if the request targets our own API and has requested to use the
pagination engine, we handle the request as an "initial" request.

If the url is a url in our url map, we handle it as a "continuation request",
else we pass along this request since we aren't being asked to handle it.

For this blog post I am only going to implement `next` link behavior, though
as with the first `GET` example above we can follow this pattern for every
pagination link we may want.

```ts

function handleInitialRequest(request, next, urlMap) {
  if (request.method !== 'POST') {
    throw new Error(
      `The PaginationEngine handler expects usePaginationEngine to only be used with POST requests`
    );
  }

  return handleRequest(request, next, urlMap);
}

async function handleRequest(request, next, urlMap) {
  const identifier = request.store.identifierCache.getOrCreateDocumentIdentifier(request);
  if (!identifier) {
    throw new Error(
      'The PaginationEngine handler expects the request to utilize a cache-key, but none was provided',
    );
  }

  const response = await next(request);
  const nextLink = `{@psuedo-link:next}//${identifier.lid}`;

  const links = response.content.links = response.content.links ?? {};
  links.next = nextLink;
  urlMap.set(nextLink, request);

  return response;
}
```

Above, we create a fake url for our "next" link by generating a string that roughly says "I represent the next link for the quest with the following cache-key". Then we key the original request in our map to that
link and move on. (Sidenote: if you want this handler to work with the PersistedCache experiment we will
also need to setup a way to restore this map from cache, this is left for discussion at another time.)

When the app calls `await page.next()`, it will effectively generate the request:

```ts
store.request({
  method: 'GET',
  url: `{@psuedo-link:next}//${identifier.lid}`
})
```

When we see this link in the handler, we invoke `handleContinuationRequest`.

```ts
async function handleContinuationRequest(request, next, urlMap) {
  const requestInfo = urlMap.get(request.url);
  const upgradedRequest = buildPostRequest(request, requestInfo);

  return handleRequest(upgradedRequest, next, urlMap);
}

function buildPostRequest(request: StoreRequest, parentRequest: PaginatedRequestInfo): StoreRequest {
  const { url } = parentRequest;

  const overrides: Record<string, unknown> = {
    url,
    method: 'POST',
  };

  if (parentRequest.headers) {
    const headers = new Headers(parentRequest.headers);
    request.headers?.forEach((value, key) => headers.set(key, value));
    overrides.headers = headers;
  }

  // update the offset
  const body = JSON.parse(parentRequest.body);
  body.offset += body.limit;
  overrides.body = JSON.stringify(body);

  return Object.assign({}, request, overrides);
}
```

And there we have it, our POST based API convention now plays along seamlessly with the
simpler mental model of calling `await page.next()` on any collection.

## Implementing a Recommendation Engine

On the surface, it seems like WarpDrive's managed fetch pipeline treats requests as 1:1
with a call to an API endpoint or a query on some other source (like a local database).

But the reality is more nuanced: WarpDrive thinks about requests as 1:1 with a response. 

Catch that? The nuance is very subtle. Possibly imperceptible.

It helps to explore this via a real world example.

Lets say you want to request options for a select, and in that list of options you also
want to include a few "recommended" options near the top that based on some criteria are
more likely to be what the user is searching for.

The naive way of implementing this is as two requests, one for options (perhaps filtered
and sorted) and another for recommendations. Then in the app we splice the two arrays
together and pass them into our select component.

Its not bad, but it is often a lot of work to transform the shapes of the two responses into
the same shape, and doing so defeats much of the power of using reactive cache-driven resources.

Taking a step back though we see that while we have two requests, conceptually we really have
just one "request" for options with recommendations and one "response" or result being the
merger of these.

Lets implement a handler that understands this requirement, and does so in a way that any
request can make a "recommendations" request alongside the "options" request.

For this example, we'll assume we have two endpoints:

- `POST /api/recommendations` which returns recommendations for resources of type <entity> in `JSON:API` format.
- `GET /api/<entity>` which returns a page of results of resource type `<entity>` in `JSON:API` format

When we receive a request that wants to also fetch recommendations, we'll issue two
requests and "mux" (combine) the response.

  
```ts
function shouldMakeMLRecommendationsRequest(request) {
  return request.headers?.get('X-Include-Recommendations') === 'fetch';
}

function invokeAndProcessMuxSuccess(request, next) {
  // ... implemented below later ... //
}

const RecommendationsHandler = {
  request({ request }, next) {
    if (!request.url || !isOwnAPI(request.url)) {
      return next(request);
    }

    if (shouldMakeRecommendationsRequest(request)) {
      return invokeAndProcessMuxSuccess(request, next);
    }

    return next(request);
  }
}
```


