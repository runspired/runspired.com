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

Let's take a dive into seven things builders make easy that adapters made hard.

## 1. Ad Hoc Requests

## 2. RPC Calls

## 3. Operations

## 4. Transactional Saves

## 5. Sharing Queries

## 6. Lazy Paginated Selects

## 7. DSLs like GraphQL / SQL
