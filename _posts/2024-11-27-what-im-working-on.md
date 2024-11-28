---
title: Polaris | What I'm Working On
published: true
---

# Polaris: What I'm Working On

You'd think being on a core team (emberjs) I'd know what other core team members are exploring, but the reality is often I know just as much as the next person about where other team members have wandered off to poke and prod. So it was refreshing to read [Yehuda's blog post](https://yehudakatz.com/2024/10/28/polaris-what-im-working-on/) covering his recent meanderings, and I figured I'd add my own musings to the mix.

## Shipping ember-data 4.13

Polaris isn't a version, its a concept. A set of APIs, paradigms, and patterns that flow together cohesively. If it were just a point-in-time version what I'm going to say next would be *really strange*.

I've been working hard on shipping the Polaris story for [ember-data v4](https://github.com/emberjs/data/pull/9598). Yes, V4 - the prior major cycle that ended 20 months ago.

As we've worked to evolve EmberData into WarpDrive in the 5.x cycle we've been keeping a steady flow of improvements into 4.12 to attempt to maintain enough alignment that apps can make the jump seamlessly.

But two of the biggest improvements in the 5.x cycle (namely types and vite support) couldn't be easily backported: and because they couldn't be backported it was getting exponentially harder to backport fixes and improvements to the cache, schema service, and request-manager needed to enable applications using 4.12 to be compatible with SchemaRecord and the full feature suite offered by `@warp-drive/ember`.

We want apps stuck in 4.x to be able to use as many Polaris idioms as possible, being able to do so makes the upgrade path and adoption process simpler, and it gives extended breathing room to larger apps that need more time to upgrade.

And so, we have reactivated the v4 release line and will soon release 4.13. 4.13 will

- have its own native types
- be compatible with vite
- contain all bug fixes and improvements from the 5.x series
- maintain all deprecated features from the 4.x series (duh)
- allow activation of deprecations from the 5.x series (these are off by default, so no new deprecations are getting added, but you can choose to opt-in early)

## Shipping ember-data 5.4

The 5.4 release has been long overdue, but we didn't want to run away from 4.x users looking for an easier migration path until we had more of a cohesive vision implemented of what the final destination for them was going to look like.

Now we do: we're putting the finishing touches on the first release of SchemaRecord and expect to ship by EOY.

What may have gone unnoticed though is that while we haven't shipped a new minor of `ember-data` in a long time, we have continued to ship new patches of `5.3`. In fact, every improvement and fix in `5.4` is already in `5.3`! This is because `5.4` is symbolic more than anything, when we ship and start regular minor cadence again it will signify that the new paradigm is ready for you to try out.

This means that 5.3 supports vite 💜

## Planning the replacement for unloadAll and unloadRecord

`unloadRecord` and `unloadAll` are to APIs that barely made sense in the resource-centric past of EmberData and make no-sense in the request-centric future of WarpDrive. They've been crutches for apps that really needed to care about 

But there's been no design push to replace them because its not been totally clear what a good general-purpose replacement for them would be that doesn't immediately fall victim to the same issues they have around accidentally creating broken state in the app.

With that in mind, I've been exploring an alternative idea around how to implement a robust but performant Garbage Collection (GC) feature for WarpDrive on-and-off for the past several years.

Two of the sticking points were:
- in the legacy setup, relationships MUST be retainers which often makes GC near impossible
- it seems to require a reference-counting approach, which is known to be algorithmically slow and error prone
- freeing instances and data from the cache isn't a great decision if you're going to want that data again really quickly thereafter.

To that end, I finally realized that fully-embracing the request-centric world allows us to treat requests as roots and thus implement a tracing-gc approach. Moreover, in this paradigm we no longer need to strictly consider relationships as retainers (there's a couple of edge cases around mutation where you do).

If you are curious I wrote up a length explanation of how this will work [here](https://github.com/emberjs/data/pull/9612/commits/3352349217368bdc9635175586bc06923273f7f1), which I plan to turn into an RFC soon after we ship 5.4

## Thinking about Schemas

When I first started talking about SchemaRecord (then called SchemaModel) I envisioned pairing it with an optional Schema DSL to make authoring schemas and types a quick unified experience.

In the interest of time, that DSL has been indefinitely delayed. I've come to think of it as a "nice to have" but not an essential, because the experience of authoring JSON schemas and standalone types has so far felt pretty easy and enjoyable.

I'm not sure yet if that feeling I have will extend to everyone coming from the Model world though, and if others want to help take on the work I still think it has a lot of utility.

## Thinking about CRDTs

So far: I feel they are more hype than useful. My mental model of them is probably not good enough yet. From where I stand though, value-added to the diffs you still need seems low while the implementation and mental model costs feel high. This might change with more familiarity.

## Thinking about PersistedCache

I really want to get mutations into the DataWorker and PersistedCache permitives soon, to do so we need named stores. (There's also some cache API changes we will want, but those can be experimented with separately).

It would be a fairly small RFC and implementation, I'm tempted to ship one in the next few weeks.

## The Scheduler

I remain convinced that the [Render Aware Scheduler Interface](https://github.com/emberjs/rfcs/pull/957) is one of the largest lifts to DX ember can deliver for Polaris. And I don't think it would be hard, but none of us have had the time to focus on it. Imagine: no more backburner, no more run loops, no more error traces and call stacks deep in framework code trying to find where your own code with the mistake is.

## Routing

The state of routing in Ember has become a huge bottleneck to shipping new features in WarpDrive. I have so many things waiting in the wings dependent on it: so I've been doing a lot of ruminating on how to push exploration there forward: I'm extremely tempted to ship a whole new router, and I don't think that's a good thing.

A few of the things I'm looking to explore:

- An EdgeServer that eliminates fetch waterfalls and allows any app to utilize the big-pipe approach
- route request pre-fetch via the DataWorker
- Turning Ember apps into MPAs
- using context to provide route requests to component trees

```ts
{% raw %}
import { Route } from '@warp-drive/ember';

export function fetch(params) {
  // return value can either be a request Future, promise or an object.
  // if it is an object, each key of the object should point at
  // either a request Future, promise or a value.
  // the function should not be marked `async`
  // the return will become the `@route` arg provided to the template.
}

const MyRoute = <template>
  access the result of the fetch function (unresolved)
  {{@route}}

  component trees invoked here or within
  the yield would be able to access the
  route object via `consume('@route')`
  standard yield also works

  {{yield}}
</template>;
{% endraw %}
```

- using context to provide request results to a component tree

```hbs
{% raw %}
const MyRoute = <template>
  <Request @request={{@route.someRequest}} @key="awesomeSauce">
    <AwesomeSauceConsumer />
  </Request>
</template>;
{% endraw %}
```

## Testing

WarpDrive should offer an integrating testing experience. I've been exploring ideas for that with [Holodeck](https://github.com/emberjs/data/tree/main/packages/holodeck#readme) and [Diagnostic](https://github.com/emberjs/data/tree/main/packages/diagnostic#readme). The next step is to add a simple router (likely I'll just use something layer on honojs, which is what I've used for a POC of this) and re-use WarpDrive as the ORM Layer.

There's a few approaches to the setup with differing tradeoffs, all of them still better than the state of things with Mirage or MSW. A big unanswered question is around how to handle API specs. I was hoping to generate from OpenAPI Specs if an API has them, but these seem to lose way too much information vital to robust mocking.