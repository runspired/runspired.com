# Not Your Parent's EmberData

> How RequestManager flips the script on everything

---

EmberData's legacy [turns 18 this year](https://github.com/sproutcore/sproutcore/commit/f6248b1650a688a401cc6eea135fbe983e20cd12).

What is remarkable (besides a Javascript project surviving for this long) is how long the 
project lasted without any significant revisiting of its architecture before RequestManager
was added in 2022 for the 4.12 release.

Ok, that's not totally the truth. We paved the way to RequestManager years before, and began
the internal evolution that would allow for it with the
[RFC for identifiers](https://rfcs.emberjs.com/id/0403-ember-data-identifiers/) in 2018. 
But until RequestManager, much of the power that identifiers unlocked remained largely out 
of sight.

In this post, I want to walk through one of the key changes that comes with RequestManager
that highlights the advantages of the changing architecture, as well as tease a bit of what
is still to come.

**All requests should use EmberData**

But first, a short overview of the shift to RequestManager.

## What Changed

In the past, the request layer in EmberData was an abstraction hidden from the application
developer. Whether and when the store would decide to build and issue a request via an 
adapter vs resolve from cache was a bit of magic.

The good was that this allowed for a uniform, stable API for requesting data to work with.
The bad was that how to update the cache or invalidate it was a bit mysterious.. and at times
downright frustrating.

Regardless of whether you were experiencing the fun or the frustrating aspects of working
with the store, these interactions were *resource* centric: "find me this record", "query for
records matching X", "give me all records of this type", "save this record".

Consider the (simplified) signatures of the historical approach:

```ts
interface Store {
    findRecord(type: string, id: string): Promise<RecordInstance>;
    findAll(type: string): Promise<RecordInstance[]>;
    queryRecord(type: string, query: object): Promise<RecordInstance>;
    query(type: string, query: object): Promise<RecordInstance[]>;
}
```

Compared to the (again, simplified) signature of the new approach:

```ts
interface Result {
    request: FetchInit;
    response: Response;
    content: Document;
}

interface Store {
    request(options: FetchInit): Promise<Result>
}
```

These interfaces present a rough sketch of the shape of the change, take note these are not
the exact types 😅!

As its name implies: the RequestManager is instead *request* centric. Instead of answering
questions about specific records or types of records like we used to ask the store in the past, we ask it about the status of a specific request.

Ok, so how does this new API change how we build our applications?

## All Requests Should Use EmberData

> Me: If your application makes even one fetch request, your app should use EmberData.
> 
> Person 2: Wait, really?!
>
> Me: Yes. Really.
>
> Person 2: Explain...

I will sum up: convert this:

```ts
class MyService {
  async queryData(query) {
    const response = await fetch(`/api/v1/my-data`, {
        method: 'POST',
        headers: {
          'content-type': 'application/json'
        },
        body: JSON.stringify(query)
    });
    return await response.json();
  }
}

```

Into this:

```ts
class MyService {
    @service requestManager;

    async queryData(query) {
      const response = await this.requestManager.request({
        url: `/api/v1/my-data`,
        method: 'POST',
        headers: {
          'content-type': 'application/json'
        },
        body: JSON.stringify(query)
      });
      return response.content;
    }
}
```

Note, the above is the 1:1 conversion to keep things "exactly" as they were in your app, and as
it stands it already provides huge albeit hidden value. Below we will unpack that value and
then begin to iteratively migrate to unlock even more value.

Because of its history, there is a temptation to think of EmberData as a *resource centric*
library, along the lines of an ORM (or worse, a full fledged database).

But as we discussed in the last section, EmberData is now a *request centric* library. The
extent to which this is true goes far beyond just porting the requests that EmberData used to make magically via adapters into the new paradigm.

If you are doubting right now how EmberData being *request centric* means you should use it
for all your requests then first, I applaud you for your skepticism. It is invaluable to 
critically analyze the claims a library (or its author) makes.

Second, I feel you. 18 years of doing things one way is a *lot* of history, built context
and emotions to suddenly toss away. But before you head over to my threads account 
to drag me for how bad a take this is, hear me out. Then please do so at your earliest 
convenience.

You probably fall into one of two camps:

  1. You use EmberData but you have requests for which you do not use it. If this is you then probably you are more willing to hear me out.
  2. You don't use EmberData – perhaps because you ripped it out of your app in a burning fit of passion, and you have no idea outside of either sick revenge or disaster porn why you are reading this post ... I don't know either but I hope you keep reading ... –  

Those of you in the first camp, this will probably be an easier sell: you can drop usage of ember-ajax, jquery.ajax, ember-fetch and probably a dozen homegrown internal things you have and instead use a nice uniform interface for managing all manner of requests. I know that doesn't sway your skepticism that this is possible, but hopefully its at least a small carrot.

**But WHYyyyyyy**

### Lets start with what you gained in this simple migration

I mentioned huge albeit hidden value.

The most basic answer is that RequestManager provides a simple, stable unified interface for 
how we request or mutate data. Developers that love EmberData historically have largely loved
it for this aspect. Yes, you still have to construct fetch requests, but we'll get to that 
part below in a moment.

A stable unified interface gives you three things.

First, an easier ability to refactor (by lots of different mechanisms).

Similarly – second, an easy integration point to make things happen universally when needed: whether a shared abstraction to reduce cognitive overhead or for implementing a sweeping API change that you can make feel like a tiny bump instead of a major rewrite.

Third, unified expectations of behavior.

In this code example we see all three of these things in play.

First, by unifying on the platform, it was quick to convert our standard fetch request into the request manager paradigm.

Second, we gained all the functionality of whatever handlers we registered for use. Most commonly, this will be the [Fetch Handler](https://github.com/emberjs/data/blob/5a48f52a08587e59cb529575577880daf678ae00/packages/request/src/fetch.ts).

This means that immediately our code example started handling a number of scenarios that it didn't before, despite no seeming change. Lets unpack what we get that we didn't have before. In no particular order:

- The fetch code is SSR ready
- We account for libraries like mirage that will try to dynamically swap out `fetch` for its own version
- A test waiter is automatically installed to help prevent leaky or flakey tests
- AbortController is automatically wired in
- Streaming Responses are automatically prepared for
- Vague network level errors are converted into meaningful error objects
- The `date` header is autoset on the response if not already present to ensure the ability to check request age
- Responses that represent errors are converted into meaningful error objects
  - statusText is normalized for http1.1/2/3
  - the error message is set to a meaningful string that will be helpful to observability tooling (like Sentry) and avoids common pitfalls of errors being too similar
- the handling of parsing the response to JSON is done for you

Before this change, had our fetch request failed, the most likely
outcome in many applications would have been no error surfaced to
the application or to observability tooling like Sentry (because `fetch` always resolves), or a very confusing and poorly differentiated error
along the lines of `cannot parse token < at json line:0`.

And third, we gained expectations of behavior.

Everything that the Fetch handler does is something every developer 
otherwise must do each time individually. But often in the interests of time, terseness, overconfidence in network stability, or due to lack of awareness these things will not be done.

The value of an abstraction like EmberData is that it is able to reduce cognitive and implementation burden on product engineers for these sorts
of considerations, in many cases eliminating that burden entirely.

Whether or not you use EmberData to manage your requests doesn't change the fact that *they need managed*. Even just a single, relatively simple request has this need.

But the value doesn't end here. Lets take our migration further

### Taking our Migration Even Further

Above we had refactored our service into what is copied again below:

```ts
class MyService {
    @service requestManager;

    async queryData(query) {
      const response = await this.requestManager.request({
        url: `/api/v1/my-data`,
        method: 'POST',
        headers: {
          'content-type': 'application/json'
        },
        body: JSON.stringify(query)
      });
      return response.content;
    }
}
```

It might have been invoked something like this:

```ts
class Route {
    @service myService;

    async model({ search, offset }) {
        const query = {
            search,
            sort: 'name:asc',
            limit: 50,
            offset: Number(offset)
        };
        const data = await this.myService.queryData(query);

        return {
            search,
            offset,
            data
        };
    }
}
```

This service is doing three things for us:

1. choosing the url
2. generating the fetch options
3. ensuring a json response

Lets start by refactoring it to make use of the RequestManager's encouraged
pattern of builders and handlers.

> Builders setup requests, they are functions that may understand the app state and context in which a request is being generated
>
> Handlers help to fulfill requests, by processing requests or responses in ways that are broadly applicable

In our current service, everything within the call to `request` is something that is immediately a candidate for a builder or a handler.

Lets write just a builder for today. Generally we name builders to follow the natural
meaning of what the request being constructed is intended to do.

```ts
export function queryData(query, resourcePath) {
    return {
        url: `/api/v1/${resourcePath}`,
        method: 'POST',
        headers: {
            'Content-Type': 'application/json'
        },
        body: JSON.stringify(query)
    }
}
```

That makes our service now look like this:

```ts
import { queryData } from './builders';

class MyService {
    @service requestManager;

    async queryData(query) {
      const response = await this.requestManager.request(
        queryData(query, 'my-data')
      );
      return response.content;
    }
}

```

At which point we realize this custom fetch-wrapping service is no longer useful.

Lets delete it and use RequestManager directly.

```diff
+import { queryData } from './builders';
+
class Route {
-    @service myService;
+    @service requestManager;

    async model({ search, offset }) {
-        const query = {
+        const query = queryData({
            search,
            sort: 'name:asc',
            limit: 50,
            offset: Number(offset)
-        };
+        }, 'my-data');
-        const data = await this.myService.queryData(query);
+        const result = await this.requestManager.request(query);

        return {
            search,
            offset,
-            data
+            data: result.content
        };
    }
}
```

Which results in:

```ts
import { dataQuery } from './builders';

class Route {
    @service requestManager;

    async model({ search, offset }) {
        const query = dataQuery({
            search,
            sort: 'name:asc',
            limit: 50,
            offset: Number(offset)
        }, 'my-data');
        const result = await this.requestManager.request(query);

        return {
            search,
            offset,
            data: result.content
        };
    }
}
```

While very slightly more verbose, this is immediately better than the custom service we had
before in a bunch of ways.

Now the developer knows this is a managed request instead of that context being hidden
behind the extra service, so it level sets expectations of what they can expect to need
to do.

The developer also now gets direct access to the response of the request, which gives them access
to the wired in AbortController, additional request and response information, and the ability
to stream the response if they choose. Our wrapper while useful was previously discarding the capabilities from being accessible to the developer. "Use the platform" has a friend: "Use the framework". These things feel invisible, but they are there ready for when they are needed, no workarounds necessary.

### Incrementally Migrating With Builders

Finally, we've now introduced a very nifty refactoring nicety. When we had our custom service, had we wanted to migrate the requests to use a new API we had three choices available.

1. We could have migrated all requests simultaneously
2. We could have introduced an overload to the method to take in whether or not to use the updated API
3. We could have added an additional method name. E.g. `queryDataV2`.

The trouble with (1) is it carries inherent risk for anything but small apps. The trouble with (2) is that you quickly grow the cognitive and implementation complexity of your method.
The trouble with (3) is that if you don't choose a good name, you introduce even more cognitive complexity, and even if you do choose a good name the odds are that the original method name is both easier to remember, faster to autocomplete, and intuitively preferred.

We've all been there with (3). Naming things is hard, and teaching folks to migrate their habits is too.

The neat thing about RequestManager is that its a simple chain-of-command executor. It doesn't care much about what your requests are, just that it can execute them. It is *interface* driven instead of imperative. Thus we remove the complexity attached to the method name and signature, replacing it instead with an interface that rarely if ever will change.

Which means our refactor still may take three forms, but the change looks different.

1. We could update the existing builder to migrate all requests simultaneously
2. We could change the builder signature to have an options argument to which we pass the version
3. We could implement a whole new builder.

Which answer is best for you will vary. In my app at work I have three API migrations planned for the year that utilize builders in their migration strategy.

The first will migrate all requests simultaneously. It is simply updating the underlying format in a way that won't be product affecting.

The second will migrate requests incrementally to a new format, it is updating the underlying transfer format in a way that affects product code, but not changing the overall semantics of the API. For this I have considered passing in an option to the builder, but I am more likely to take the new-builder approach as it makes tracking the status of the incremental migration with static analysis much easier. Either approach is very valid.

The third will migrate requests to a new API version that changes semantics significantly. This is a high risk migration, and so I will use a completely new builder.

With this in mind, lets iterate on the builder we have in our example above and show how this pattern provides value. First, by refactoring in a way that doesn't result in a product code change, then in one which does.

As a reminder, this was where we left off with our builder before:

```ts
export function queryData(query, resourcePath) {
    return {
        url: `/api/v1/${resourcePath}`,
        method: 'POST',
        headers: {
            'Content-Type': 'application/json'
        },
        body: JSON.stringify(query)
    }
}
```

The first thing we probably want to do is have our builder respect a configurable default host
and namespace. This prepares us for our site and API not being on the same domain (a decision you do not want to have to piecemeal figure out how to account for later).

EmberData provides users a global config mechanism for host and namespace. Typically you will want to do this either in your store file or app file.

```ts
import { setBuildURLConfig } from '@ember-data/request-utils';

setBuildURLConfig({
  host: 'https://api.example.com', // no trailing slash, though '/' is valid
  namespace: 'api/v1', // no leading slash and again no trailing slash
});
```

What this does is set the default host and namespace for use by the request-utils package,
which provides a number of utility methods for constructing builders.

Next, lets update our builder to make use of this:

```ts
import { buildBaseURL } from '@ember-data/request-utils';

export function queryData(query, resourcePath) {
    return {
        url: buildBaseURL({ resourcePath }),
        method: 'POST',
        headers: {
            'Content-Type': 'application/json'
        },
        body: JSON.stringify(query)
    }
}
```

> Note: the signature of what is passed to buildBaseURL is simpler above than in the current
> EmberData 5.3 release, and reflects updates that relaxed the signature which will be in 5.4

Awesome! Now we are ready for deployment to CDNs! Or maybe your application is single-tenant
and has a per-customer api domain that needs to be configured globally, easy-peasy. This can all be handled without the product code needing to be changed or consider it.

Now lets say we want this request builder to build requests that are *also* able to be cached when using EmberData's CacheHandler?

For that, we need a cacheKey, and for good measure an op-code (some op-codes are special –this one is not– but I will explain more about op-codes in another post) as well as an identifier.

```ts
import { buildBaseURL, buildQueryParams } from '@ember-data/request-utils';

export function queryData(query, resourcePath) {
    const url = buildBaseURL({ resourcePath });
    const queryData = structuredClone(query);
    const key = `${url}?${buildQueryParams(queryData)}`;

    return {
        url: ,
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

> Note: whether resourcePath and identifier.type match is API specific, in many APIs they do, in equally many they do not. A simple mapping function is useful if they do not.

Ok so what all did we do here?

First, we add `op: 'query'`, which is a hint to the CacheHandler about how to treat this request. Then we added `identifier: { type: resourcePath },` which will hint to it the
primary resource type this request pertains to, which is useful for many cache invalidation strategies. There are additional properties that may be added if desired for that reason, I won't go into them all here.

Finally, we activated caching and told it the key to use. This was important because by default the CacheHandler will only cache GET requests with a url by their url, but we want
to use POST as QUERY here. If we blindly used the URL we would have a bug in our application.

So we serialize the query to be part of the cache-key, and we do this using the `buildQueryParams` utility. This utility performs a bit of wizardy to produce a stable key.

In case you didn't know `JSON.stringify({ a: '1', b: '2' })` is not the same as `JSON.stringify({ b: '2', a: '1' })`. Key insertion order is respected during JSON serialization. However, this is rarely useful for creating a cache key because objects are often dynamically generated. What we care about is whether they have a (deep) equivalent value, which this function helps us to achieve in more cases. Yet another thing where an annoying and complicated problem vanishes with a good framework and good infra.

What if you don't want to trust serializing the query like this to get a cache key? Use any string key you would like, just make sure that its uniqueness validly describes the query.

The reasons for why this is so important will go into my next post which will dive into caching.

#### Migrations That Affect Product

Lets say we wanted to change our API from using ActiveModel to using JSON:API as its format. The response in both cases is JSON but the shape is very different. How would
we handle this with builders? For this exercise, lets assume the API version stays the
same and the new format is controlled by JSON:API's expected header.

> Note: We are glossing over that JSON:API doesn't have a post-as-query capability in
> the spec, most real-world implementations still implement it. Here we care only about
> updating headers to get the new API response format, no other changes in API semantics.

For this I would write a new builder. I would start by copying the original, and then
adjust the headers as desired. I would also account for the format in the cache-key
because it is something that affects the response but is not captured by the URL itself.

```diff
import { buildBaseURL, buildQueryParams } from '@ember-data/request-utils';

export function queryData(query, resourcePath) {
    const url = buildBaseURL({ resourcePath });
    const queryData = structuredClone(query);
-    const key = `${url}?${buildQueryParams(queryData)}`;
+    const key = `[JSON:API]${url}?${buildQueryParams(queryData)}`;

    return {
        url: ,
        op: 'query',
        identifier: { type: resourcePath },
        cacheOptions: { key },
        method: 'POST',
        headers: {
+            'Accepts': 'application/vnd.api+json',
-            'Content-Type': 'application/json'
+            'Content-Type': 'application/vnd.api+json'
        },
        body: JSON.stringify(query)
    }
}
```

Now lets migrate our product code usage:

```diff
-import { dataQuery } from './builders';
+import { dataQuery } from './builders-v2';

class Route {
    @service requestManager;

    async model({ search, offset }) {
        const query = dataQuery({
            search,
            sort: 'name:asc',
            limit: 50,
            offset: Number(offset)
        }, 'my-data');
        const result = await this.requestManager.request(query);

        return {
            search,
            offset,
            data: result.content
        };
    }
}
```

Obviously we will then need to make additional changes to our code to account for the changed json shape, but our request code is stable, our migration state is easy to statically analyze, and our brain doesn't hate any weirdly named methods.

Of note: if you were using EmberData's Store here and not just the RequestManager, you
would not need to migrate code using `result.content` as that would already be
record instances!

The only difference in code to have seamlessly absorbed such a major migration would be this!

```diff
import { dataQuery } from './builders-v2';

class Route {
-    @service requestManager;
+    @service store;

    async model({ search, offset }) {
        const query = dataQuery({
            search,
            sort: 'name:asc',
            limit: 50,
            offset: Number(offset)
        }, 'my-data');
-        const result = await this.requestManager.request(query);
+        const result = await this.store.request(query);

        return {
            search,
            offset,
            data: result.content
        };
    }
}
```


### Where To From Here?

Builders and RequestManager are but the first stepping stone in what promises to be a big evolution in how you manage querying and mutating data in your application.

We see the future as one that is schema and spec driven. Specs describe API endpoints and explain how they operate on the resources your schemas describe. This information then feeds
into tooling to automatically produce your API mocks for tests, eliminates Models, provides
strong end-to-end typing guarantees, and allows lint and runtime verification of query validity.

The hints of this are already throughought the EmberData codebase. Alpha versions of ideas like the request mocking library (which can do things no other mocking library does due to RequestManager) are under construction and even being used by the library's own test suite to dogfood their development.

All this to say, we think builders will end up a typechecked, typed response thing that for
most apps looks a bit like this for the example we've been using.

```ts
import { dataQuery } from './builders';
import { aql } from '@warp-drive/aql';

class Route {
    @service requestManager;

    async model({ search, offset }) {
        const result = await this.requestManager.request(aql`
          QUERY my-data {
            data {}
            filter {
              @arg search = ${search}
            }
            sort [
                name = "asc"
            ]
            page {
                @arg offset = ${Number(offset)}
                limit = 50
            }
          }
        `);

        return {
            search,
            offset,
            data: result.content
        };
    }
}
```

With benefits that stretch for miles beyond the simplicity of the interface.