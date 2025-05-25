---
title: In Defense of Machine Exchangeable Formats
published: true
---

# In Defense of Machine Exchangeable Formats

> Or: **Why a custom DSL is the last thing you should add to your project**

I've been thinking on DSLs lately and how many of them invert the problem space.

In [*Warp***Drive**](https://warp-drive.io), we chose to remove our custom DSL in favor of JSON, a decision I'll discuss a bit after detailing some of the underlying thought processes that led to this decision.

<img src="../../../assets/images/machine-exchange-format.png" width="100%" alt="Lots of machines all talking with each other">

## Stability of the Output is more important than Stability of the Input

Often DSLs are seen as a way to improve authoring DX and enable a stable syntax overtop a shifting underlying output.

For instance, take an ORM using decorators to define
fields on classes to generate schemas for records.

```ts
import { hasOne, hasMany, primaryKey, text } from '@orm/schema';
import { TrackedUpdates } from '@orm/traits';

@TrackedUpdates
class User {
  @primaryKey({ type: 'uuid-v7' })
  id;

  @text name;

  @hasOne('team', { fk: 'manager' })
  managedTeam;

  @hasOne('user', { fk: 'reports' })
  manager;

  @hasMany('user', { fk: 'manager' })
  reports;
}

export { User };
```

One reason server-side ORMs approach schemas via DSL is to be able to normalize the input into multiple potential outputs (table schemas for SQLite, PostgreSQL, MySQL, MongoDB etc.).

This works because for a given input + given database there is a stable underlying format the DSL decomposes to such as a flavor of SQL.

If there wasn't a stable underlying output, this decision on the part of server-side ORMs would be a disaster. It generally is not, because when the ORM's DSL becomes a point of friction teams can eject and write SQL.

The ability to both **access** and **directly use** the actual format is key.

While DSLs are nice for authoring sugar, if they don't decompose to an agnostic machine exchangeable format - or they don't expose that format for direct use - they become too restrictive and present a significant long-term maintenance risk.

As is the order of operations: these custom ORM DSLs only became possible *because* there were **existing** stable formats for them to decompose into.

## Great DSLs

The crucial point the above discussion is teasing is that the DSL should not be the spec. Stable representations come first and define a spec, and DSLs produce spec compliant output.

This is one of several common patterns that in my experience all great DSLs follow.

- they document the underlying representation they use
- they allow access to the underlying representation
- they allow direct use of the underlying representation instead of the DSL when needed
- they use an underlying representation that is broadly exchangeable
- the DSL is not the spec, the underlying representation is
- there underlying representation can be produced by any number of DSLs, its not specific to one.

Note how none of these is at all about the authoring DX the DSL provides: it doesn't matter how great the authoring experience is if its also an obstacle.

Which brings me to my next point..

## Authoring DX Is (sometimes) Vastly Overrated

DSLs work best when they simplify something done regularly.

Defining a schema for a table in your database isn't something done regularly, as compared to say GraphQL queries where you might find yourself writing several new queries every day.

I see a lot of projects create custom DSLs for something done once as part of configuration and then rarely touched. To me, this seems bad.

I've also seen a lot of developers evaluate a tool or library based on whether they like the DSL for the configuration thing they will do once. This also seems bad.

This isn't to say a great DSLs can't make it easier to get started configuring a project - they absolutely can - just that the human addiction to sugar shouldn't be trusted. Sugar is sweet, but it can also kill you over time.

## Embracing JSON 

Handy-wavy over-simplification, *Warp***Drive** is a library intended for use in web clients that combines a [fetch](https://developer.mozilla.org/en-US/docs/Web/API/Fetch_API) pipeline with ORM-like capabilities for relational data. If you are unfamiliar, the project [introduction](https://docs.warp-drive.io/guide/) gives a better synopsis.

This means *Warp***Drive** encounters two of the most common areas in which DSLs arise:

- request building (think sql generators in your ORM of choice, or GraphQL queries)
- resource schemas

While these are the same problems server ORMs must often solve, there is a crucial distinction to make.

*Warp***Drive** is a client-side library. Requests aren't constructing SQL they are constructing messages to be sent via http or websocket. And since there's no underlying database there's no underlying SQL format that schemas decompose to and which you could eject and use.

If *Warp***Drive** wants to give developers the same escape valves to *just use the platform* it MUST similarly expose the ability for users to interact with the actual underlying representation when the DSL doesn't cut it.

In both of these areas *Warp***Drive**'s predecessor *Ember***Data** used to provide a DSL

- Methods such as `store.findRecord` that would magically build a request for your API
- Models that were used to provide resource schemas

Both of these DSLs repeatedly proved to be overly restrictive. They both decomposed to JSON, but that underlying representation was neither accessible nor directly usable.

Eventually escape hatches were built, but they felt like clumsy work arounds because they in fact were: for instance `adapterOptions` as a way to channel arbitrary information through to adapters on requests to enable folks to eject and write their own behaviors.

This is ultimately why schemas in [*Warp***Drive**](https://warp-drive.io) ejected from the more common ORM pattern of using decorators on classes. We were decomposing to JSON anyway, and by making the exchangeable format the spec we ensure greater flexibility and easier maintenance.

It is also why `store.request` just takes a JSON object following the `RequestInit` interface with minimal additional decoration for *Warp***Drive** specific behaviors.

For both of these APIs, folks that liked the DSL approach have complained about authoring in JSON. Generally, if you liked the prior DSLs it meant your usage fell in a very narrow range that didn't hit the friction of their restrictive nature.

This isn't to say we couldn't have a DSL for these things. In fact we will likely ship one eventually,
both for requests and for schemas. We had one planned for schemas early in the development of our new reactive objects, but opted against in the interest of time and because we found it added little value.

Why little value?

Because creating resource schemas is something you don't do regularly. But also because the JSON schemas are fully typed which results in both editor autocomplete and is AI prompt completions assisting you far more easily than they could in the class approach.

This is beside the point though, DSLs are still allowed. It's just important that the underlying representation is JSON and is directly usable. By doing this, we've set up *Warp***Drive** for greater short term and long term success.

Say (for instance) that you wanted to use GraphQL with WarpDrive. To make this happen you need three things:

- for the template-tag function implementation to produce a valid `RequestInit` (and ideally type!)

```ts
store.request(gql`{
    user(id: 5) {
      firstName
      lastName
    }
}`);
```

- for the parsed query to provide resource schemas for the contained data that can be loaded into the store's schema service
- a request handler that normalizes the GraphQL response into a more cacheable format (this sounds harder than it is)

In the GraphQL example, GraphQL is serving as both the schema and the request DSL. Because *Warp***Drive** operates on a JSON representation this DSL can easily decompose to, integration with GraphQL doesn't require the library itself to know anything about GraphQL. Imagine trying to bolt this onto Models...

And the same goes for tons of other DSLs and mechanisms of producing schemas. What happens when you want to supply your data schemas from SQL table schemas or OpenAPI specs, or compile them from types, or deliver them on-demand from your API to ensure they are always consistent with what your API is providing?

While its likely that *Warp***Drive** will eventually ship with one or more DSLs for requests and schemas, this architecture ensures flexibility and maintainability over time. 

A custom DSL is the last thing you should add to your project. Some folks might interpret this to mean you shouldn't use a DSL or should never add a DSL. That isn't what I'm saying. DSLs are like optimizing the last 100ft of a delivery coming from a hundred miles away. It matters, but the spec and underlying representation need to be solid first.

## A Parting Thought on MCP in the Context of DSLs

My suspicion is that the Model Context Protocol is shaping up to be a great DSL. Not based on usage or rigourous study of the specification, but because it chose to explicitly define the underlying representation as JSON, following a schema, and communicated via json-rpc.

There are those who wonder if HTML or XML wouldn't have been a better choice. They may be right in some ways, but personally I've found that working with structured JSON is a better choice due to the ease with which tooling supports consuming JSON across languages and ecosystems and in how JSON maps close-enough to the object instances a language will parse it into.

MCP follows most of the patterns that I feel lead to great DSLs, so I see no reason why it couldn't become one (or one couldn't develop for it). I am far less worried around the speed of its development given these things than the speed at which the JavaScript language was invented.
