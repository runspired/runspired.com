# Web Reflections

I recently came back upon this 2017 blog post from Tom Dale considering whether [compilers are the new frameworks](https://tomdale.net/2017/09/compilers-are-the-new-frameworks/). If you haven't read it, its short, I recommend doing so and before popping back over here to continue.

A few of his points:
- We’ve reached the point of diminishing returns on improving runtime JS performance
- We'll transpile, compile and minify *even more* as time goes on
- JS (in 2017) is about to have the building blocks for new app paradigms: WebAssembly, SharedArrayBuffer, Atomics, etc
- To predict the future of the web, look at what high-performance native systems do
- This will lead to better performance and bundle size

What is fun about a post like this, now seven years old, is we can see the results of those predictions and insights.

In my opinion, Tom was about 90% right on how things were going to evolve, but way off on the conclusion of it leading to a better more performant web.

The investment in compilers and tooling pipelines has largely not led to a reduction in bytes sent to the browser, but an increase. Performance hasn't improved from the move to compilation, and its maybe decreased.

In fact, the bigger your toolchain for compilation the more likely it has become that you ship more JS than you should. Why did this happen? And was this outcome forseeable? I think it was, and I think three things were at play that Tom didn't seem to consider:

### 1. Human Factors

We tend to get what we want, and what we want is generally to ship features and product. More powerful toolchains allowing app sizes to scale with the size of the product and team just meant we could ship more features and product.

This is basically the same observation that even while CPUs have gotten more powerful our programs seem to be slowing down. These tools and compilers can result in better performance, but only if we *don't look at the new space in the budget and immediately overspend it.*

### 2. The Platform

Looking back now, its evident the platform had not stopped evolving, and the performance and bundle size impact of many newer platform APIs is not only massive, but significantly larger than a compiler could ever achieve.

Animations are really great example of this. The APIs for WebAnimations, ViewTransitions and generators together replaced the need for the animation libraries that previously were among our largest dependencies.

There's an important lesson here: *No compiler can beat shipping zero-bytes.*

> No compiler can beat shipping zero-bytes

The examples of this sort of platform win are everywhere: CSS layers, variables, advanced selectors, improved media query capabilities. Each of these replaced JS-runtime or CSS-compiler based solutions that had high runtime or bundlesize costs.

These wins happened even at the language level: browsers finally implemented generator functions, fields, and private fields natively. Currently they've started implementation of decorators and accessors. These have played a massive role in reducing the amount of bytes we need to ship in addition to generally being faster implementations.

And at the JS SDK Level: 10 different implementations of UUID in your bundle? Use crypto.randomUUID. A lot of logic to diff arrays and sets? There are APIs for unions, intersections, and more now.

I work on an SPA that is over 12 years old. More often than not when I'm refactoring something it looks like deleting a lot of code and replacing it with a simpler, smaller platform API.

So to sum up, I think betting on compilers has done much less than betting on the platform. And personally I believe there are still a lot of gains to be had in that area!

Maybe Tom was right though, and maybe we've done exactly as he thought we should: *we looked at what made high-performance native code great, and realized it was the platform.*

### 3. Complexity

There's a key point missing in Tom's post. A point I think was able to be seen then, and even addressed then but was nonetheless missed. A point that I think the web community as a whole missed for the better part of a decade: *If you want faster apps, reduce their complexity.*

> If you want faster apps, reduce their complexity.

Tom's ideas mostly related to how to optimize complexity that already exists: but in the same way that no compiler can ever beat shipping 0 bytes, no compiler can optimize a for-loop to be faster than no-loop (unless of course for the case where the loop has no outputs and can just be safely deleted).

This incidentally has been where my own chips have been pushed. Complex frontend JavaScript apps are just a symptom of an underlying disease: JS is not the problem.

I think a lot of our performance issues trace back to poor architectural decisions around how we model, retrieve, transform and mutate state. I don't just mean within apps either: there's been a huge failure of imagination when it comes to designing API Frameworks and formats.

It's amazing how we'll spend hours fighting over which framework renders a component millionths of a second faster, and then dump in a MB of JSON we've given almost no thought to the shape of without a care in the world, probably in a waterfall pattern.

The rise of RSCs, HTMX, Qwik, LiveWire and (to a lesser extent) Astro are in my mind at least in part a reaction to realizing that the performance issue wasn't actually a JS issue but a data issue.

So too are the many local-first DB offerings like Replicache: they just went the other way on the same problem and decided "if we're sending it all there anyway, why not just keep it there".

But in my opinion these choices are too often trying to sidestep the problem without really solving it. This does not make these bad solutions or bad tools in any way, some of them I even think are amazing and will shape the architecture of apps for the next decade in a positive way. But this does mean there's more out there for us to work to solve still.

### Compilers Are OK

Don't mistake this for a screed against compilers. I think compilers are doing amazing things and will continue to play a critical role in the evolution of the web. Its a mistake though to think we can compile away all our problems or to treat them as the panacea for performance.

So here’s my advice for anyone who wants to make a dent in the future of web development: invest in the platform, and be thoughtful in your data design.
