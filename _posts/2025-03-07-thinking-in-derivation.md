---
title: Thinking In Derivation
published: true
draft: true
---

this is a post about handling asyncronous flows in a derived manner instead
of an imperative manner.

pull-based rendering = derived
push-pased action responses = imperative

while we're specifically talking about the patterns WarpDrive encourages apps
to use with Ember, this topic transcends both WarpDrive and Ember and applies
to all frameworks, especially those utilizing reactive signals.

one of the reasons that WarpDrive only needs computed and signal from the TC39 spec
and does not require either effects or relay is that while we support imperative
code, we enable all work to be done in a reactive derived manner.

Further, we actively want to steer apps away from async computeds and feel
such a primitive would be an unmitigated disaster https://github.com/tc39/proposal-signals/issues/30 
representing the totality of the worst-parts of javascript and EmberData historically.

This was the underlying message of the introduction of the methods and components
for Reactive Control Flow.

some of the thoughts I want to summarize are available here: https://discord.com/channels/480462759797063690/1335930918229246003/1335930918229246003 

I would like for this post to primarily target the ember audience and answer questions
like

- how do I do this without EmberConcurrency
- how do I do this without resources
- how do I do this without async/await

`<Request />` is like if the best parts of react suspense, tanstack/query, relay and EmberData
all got together and gave birth to the most beautiful child ever conceived.


