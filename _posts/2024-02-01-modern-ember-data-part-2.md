---
title: Not Your Parent's EmberData (part 2)
published: false
---

# Not Your Parent's EmberData (part 2)

> This is Part 2 of a series exploring architectural decisions within EmberData. If you haven't read [Part 1](./2024-01-31-modern-ember-data.md), you should read that first.

So every request should be handled by EmberData. Now what?

The RequestManager shifted interaction with EmberData from being *resource centric* to being *request centric*. However, this does
not mean that EmberData stopped caring about resources. Far from it.

The way to think about this shift is "yes, and".

Does EmberData still care about resources? Yes, and it now cares about requests.

Does EmberData still allow you to model those resources? Yes, and it provides greater flexibility now in how to do so.

Our goal with this architectural change was to unlock power, flexibility and composability. That goal is not achievable if the result is somehow
not powerful enough, not flexible enough, or not composable enough to support the historical feature set.

## Data Integrity

I mentioned in Part 1 that the redesign of EmberData really began in earnest with changes to the cache. In particular, the introduction of Identifiers ([RFC#403](https://rfcs.emberjs.com/id/0403-ember-data-identifiers/)).

In order to safely and accurately maintain a cache, you need to be able to safely and 
accurately determine cache keys for the content the goes into that cache. In the case 
of a library like EmberData, that need extends beyond a simple mapping between a key
and some resource data.

A good example of where being able to work safely via identity alone (without any 
associated resource data) is important is relationships.

Relationships are links between two potential cache keys. Either or both sides of the 
relationship may not have loaded resource data yet, and so we need to trust that we 
have a cache-key that correctly points to where that data would be placed in the 
future.

We also have to be able to store relationship information we receive from other 
resources for a resource that has not been loaded yet key'd to the same cache-key.

EmberData being very good at this cache-key management for graphs of resources has
been fundamental to its value proposition. This capability is what allows EmberData to 
seamlessly merge together the data from multiple requests, and allows it to quickly
ensure that wherever the same resource is referenced the same object reference is handed
out to you.

It is even at the heart of how structural polymorphism works in EmberData, 
as ultimately polymorphism is one resource that acts as though it has multiple 
identities but has only one true cache-key.

I'll dive more into how the Graph and Polymorphism each works in future posts.

In this post, I want to explain one thing we got *wrong* with resource identity in the
many years before RequestManager, and how RequestManager not only fixes that mistake but
in doing so increases the reliability of the cache.

## Until RequestManager, the Resource CacheKey was the Request CacheKey.

A key insight into both the development of Identifiers *and* the development of 
RequestManager was that to be *powerful*, *flexible* and *composable* our core
architecture needed to be *lossless* about how it handled your data.

It is always ok for a developer to decide that certain information is not relevant to
what they need from request. For instance, response headers, or extra top-level meta 
properties on JSON in the response body. But it is *never* ok for your data library to 
assume that for you.

But in a *resource centric* world this is exactly what happens. The body is extracted
from the response, and the rest of it is discarded. Then the resources are extracted
from the body, and anything that is not a resource is discarded.

Even assuming that there is no loss to individual resource data (hint, there often is 
in this design), this *loses* tons of valuable information the request contains. Not 
just headers, but meta, which associations were included, and what order they were 
returned in are all among the pieces of information discarded.

Fundamentally, being lossy was what was wrong with this core EmberData API the most:

```ts
class Store {
    findRecord(type: string, id: string, params: object): Promise<RecordInstance>;
}
```

`findRecord` is lossy because we have no idea whether we resolved from cache or 
network, and no access any returned information that wasn't on the resource.

As a design principle, ***lossless* applies equally to the intent of a request as it does to specific handling of the information received from your API.**

### Lossless Intent

`findRecord` as an API is also an example of lossy intent. Consider the below example:

```ts
const user = await store.findRecord('user', '1', {
  include: ['friends', 'company', 'pets']
});
```

While not immediately evident, this example highlights the mistake in the older 
*resource centric* design and is one of many reasons why lossy APIs are being phased
out of EmberData.

To the reader, the intent of this request might seem to be clear: "fetch the user
resource with ID 1 and make sure to also fetch that user's friends, company and pets".

Except that isn't what happens in a *resource centric* library. Instead, the steps
go like this:

1. determine the cache-key for the resource of type `user` and id `'1'`
2. check if we have loaded the resource for that cache-key.
3. if so, resolve the resource from the cache. If not, fetch the resource.
4. return the record for this resource

Did you catch the mistake?

In the *resource centric* world the resource cache-key is the same as the request
cache-key. Information like `include` isn't part of the cache-key determination.

Could we make it part of the cache-key determination? Not naturally. The contract
between the store and adapters is pretty simple. For `findRecord` there are three
methods that are a part of this contract:

```ts
interface Adapter {
    shouldReloadRecord?(store: Store, snapshot: Snapshot): boolean;
    shouldBackgroundReloadRecord?(store: Store, snapshot: Snapshot): boolean;
    
    findRecord(store: Store, schema: Schema, id: string, snapshot: Snapshot): Promise<JSON>;
}
```

It has always been possible albeit extremely unergonomic for an application to implement
`shouldReloadRecord` and `shouldBackgroundReloadRecord` in a way that would have taken into
account additional information like includes. But it solves *only* findRecord and not 
relationship or query requests, and it only solves them for a very narrow view of what and how
someone might want to issue a request.

More importantly, it means that there is nothing EmberData could do to more generally handle
requests efficiently. Without a cache-key, there is no request de-duping, no request updating,
no request caching. The list goes on and on.

So even while for many requests the cache-key for the resource and the cache-key for the request
to get that resource might be 1:1, the inumerable scenarios where this is not the case meant
that combining these things leads to an untrusty cache.

With the above call to `findRecord`, were we to resolve from cache there is no guarantee that
the requested includes are also available. Loss of intent immediately results in a loss of 
trust.

The solution for many apps was `reload`.

```ts
const user = await store.findRecord(
    'user', '1',
    { include: ['friends'] },
    { reload: true }
);
```

`reload: true` is a poison pill in apps. Once you need it, you start needing it everywhere.
There's still value in a cache even when you trust the cache so little that every request is
sent to network like this results in, which is perhaps its own intersting blog post sometime
to delve into, but suffice it to say wow. What a failure of expectations of a library.

These shortcomings are why the `query` and `queryRecord` methods on store **always** hit 
network. Without a cache-key for the request and a request cache using it there's no ability
to do anything but. Users have often added their own cache in front of these methods to try
to account for it with application specific cache-keys, but it rightfully feels like something
the store should just do for you. **And now it does!**

So Now What?

## Request Cache Keys

In Part 1 we worked on this builder:

```ts
import { buildBaseURL, buildQueryParams } from '@ember-data/request-utils';

export function queryData(query, resourcePath) {
    const url = buildBaseURL({ resourcePath });
    const queryData = structuredClone(query);
    const key = `${url}?${buildQueryParams(queryData)}`;

    return {
        url,
        op: 'query',
        identifier: { type: resourcePath },
        cacheOptions: { key },
        method: 'POST',
        headers: {
            'Content-Type': 'application/json'
        },
        body: JSON.stringify(query)
    }
}
```

At the time I mentioned that `op` and `identifier` were two pieces of meta that we could
use to help manage cache lifetimes, and that we would use the `key` set on `cacheOptions`
as the key for the request.

When using store's CacheHandler, if no `cacheOptions.key` is present AND the request is a GET
request and has a URL, the URL will be used as the cache key. Using the url as the cache-key
in this way is powerful, because not only is it *usually* 1:1 with a request, it also *usually*
captures all the state necessary to consider whether two requests are the same. By embracing
the URL, urls used for relationships and pagination become first class mechanisms of identity
capable of driving powerful features EmberData was missing before.

There are of course always exceptions though and this is why request identity is designed to
be very flexible. In addition to `cacheOptions.key` and the abstraction that builders provide
for auto-generating cache-keys, EmberData provides a configuration hook for advanced use cases
I won't get into here that enables configuration of how EmberData looks to determine identity
in a request by default.

What does this key get us out-of-the-box?