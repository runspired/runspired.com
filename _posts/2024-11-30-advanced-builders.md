---
title: Adventures in WarpDrive | Advanced Builders
published: true
draft: true
---

# Adventures in WarpDrive | Advanced Builders

One of the bigger risks with the introduction of new patterns for requests in WarpDrive has been the inclusion of request builders for common active-record, rest and json:api endpoint setups.

Why is their inclusion a risk?

Because one of the goals of the request paradigm is to shift users away from the many limitations adapters and serializers had.

Over the years we saw adapters and serializer lead to *a lot* of bad code, badly performing code, or complex workaround to solve for what ought to have been relatively simple cases.

The inclusion of "sensible default" builders that won't work for everyone carries this same risk. Users might invoke the builder just to significantly change the output in their own builder, or decide that WarpDrive doesn't support their API because the builders don't align with theirs perfectly.

One of the biggest risks they carry is in making developers think that these "sensible defaults" are representative of all – or even most – of what can be done with builders. And well ... not only is that just not true, being able to do more is why we shifted to a request-centric approach in the first place!

And I don't just mean "a little more", I mean "a lot more".

## Typing Builders

Before we get started, there's a type utility we're going to use in this post that is not yet part of WarpDrive (it will be, probably by the time you read this) which is worth showing and explaining here:

```ts
import { withBrand, type TypedRequestInfo } from './request';
import type { TypeFromInstance } from './record';
import type { SingleResourceDataDocument } from './spec/document';

function exampleFindRecord<T>(
  type: TypeFromInstance<T>,
  id: string
): TypedRequestInfo<T, SingleResourceDataDocument<T>> {
  return withBrand({
    url: `/api/${type}/${id}`,
    method: 'GET',
    cacheOptions: { backgroundReload: true },
    op: 'findRecord',
  });
}
```

When used with the store the following will happen:

```ts
type User = {
  id: string;
  name: string;
  [Type]: 'user';
};

// ...

const userRequest = exampleFindRecord<User>('user', '1');
const { content } = await store.request(userRequest);

// ...

content.data; // type User!
```

You may have seen the ability for builders to assign type signatures before, but previously this often required either a `ts-expect-error` or a cast to add the brand. The magic here is the relationship between `withBrand` and `TypedRequestInfo`.

`TypedRequestInfo` provides our request's return type, while `withBrand` sets up the response object in a way that lets typescript infer that the brand applies instead of requiring the cast or ignoring a type error.

Useful, ok let's take a dive into eight things builders make easy that adapters made hard.

## 1. Write as many of them as you want!

Builders are useful even when you're only making a request once. Why? They provide a nice abstraction around ensuring the
type signature is setup correctly, make it easier to write tests or share the request later if it turns out you need to, and often clean up the readability of your code.

Some common questions though are "what should be builders" or its cousin "how many builders should I have". A negative effect of the "sensible defaults" mapping so closely to the methods the store and adapter used to have (`findRecord`, `createRecord`, `deleteRecord`, `query` etc.) is that it leads people to believe builders should be highly generic, or use just these few recognizable names.

On the contrary, use as many builders as seems reasonable. This can be anywhere from a builder per-request or per-endpoint to a small number of carefully curated builders that work against most endpoints targeting a highly conventional API.

For instance, I started playing around with creating [a BlueSky client using WarpDrive](https://github.com/warp-drive-data/embersky.app/pull/4) and I'm currently opting to create a builder for every single RPC action that API exposes.

I really don't need to create so many builders, the RPC endpoints largely follow the same pattern, but by mapping 1:1 I'm able to take an API I don't know much about yet and have types, docs, and autocomplete for every action I might take at my fingertips. In other words, in this case builders make an API with a lot of convention but also a lot of nuance much easier to quickly integrate with!

## 2. Ad Hoc Requests

How many of you have either ejected from EmberData entirely or called `adapterFor('<some-type>').buildURL(...)` or `adapterFor('<some-type>').aCustomMethod()`? Or pushed a ton of configuration into a request's `adapterOptions` in order to change how the request was going to be made?

And then had to figure out how to get the response normalized and inserted into the store? Leaving a trail of `serializerFor` and `pushPayload` calls that seem a bit dicey but your tech-lead says this is the pattern you've been using so you follow along?

With builders, it becomes easy to construct ad-hoc requests or less common requests that you'll re-use only a few times and have everything look and function just the same as any other request. The response is automatically inserted into the cache if that is what you want, all your normal handlers for processing the data are utilized (if you want, its also easy to set them up to be skipped), all the same typescript support, all the same request utilities like `<Request />` and `getRequestState` and advanced error handling...

## 3. RPC Calls

I touched on this in point-1 above but I really want to call this out specifically. With builders, RPC-like patterns are easy to implement. For instance, let's say you want to "add a like" in a highly scalable system where the likes count is eventually consistent. You don't want to model this as a `post hasMany like` because loading millions of likes just to display a count and the state of the heart button would overhwlem the system. Instead you might model this (similar to bluesky) as a [post with a viewer object](https://docs.bsky.app/docs/api/app-bsky-feed-get-feed), where the viewer object contains data about you, the user (have you liked it? etc.).

When you like or unlike the post, you want to:

- toggle the state of `post.viewer.liked`
- increment/deprecate the `post.likeCount` field
- make an RPC request that either creates or deletes a user like.

```ts
function createViewerLike<T extends Post>(post: T): TypedRequestOptions<T, void> {
  return withBrand({
    url: `/api/app.post.createViewerLike`
    method: 'POST',
    op: 'app.post.createViewerLike',
    body: JSON.stringify({
      postId: post.id,
    }),
  })
}
```

Great, say the response here is a 202 or 204 and not a 201? What should we do about updating those two fields?

This is where WarpDrive leaves a lot of decisions to you. You could:

- have your builder optimistically update the cache state
- have your builder mutate the cache state
- have your builder tell the cache this is an update to the post record
- use have a handler which pessimistically updates the cache state
- use a handler that runs a callback passed in via options to update the cache state

Each of these has tradeoffs. The first, you risk being in an odd state if you fail.

## 4. Operations

## 5. Transactional Saves

## 6. Sharing Queries

## 7. Lazy Paginated Selects

## 8. DSLs like GraphQL / SQL
