# Layered APIs

A new standards effort for collaborating on high-level features.

## Problem

The [Extensible Web Manifesto](https://extensiblewebmanifesto.org/)’s focus on low-level primitives promotes a healthy, well-layered platform that encourages innovation and experimentation in JavaScript. But focusing on low-level primitives means that developers must build most application-level components on their own, creating a high barrier to entry for new web developers.

This lack of built-in high-level features also bloats page load size. The average site payload is [2.5 MB and takes 19 seconds to load](https://www.doubleclickbygoogle.com/articles/mobile-speed-matters/).

Historically, standards bodies and implementers have been reluctant to work on higher-level APIs. In particular, introducing new capabilities via high-level APIs is dangerous, as when we get something wrong, developers are often left with no other way to access these capabilities. Shipping features also incurs an ongoing maintenance and runtime cost — every new feature pollutes the browser namespace, increases JS startup costs, and represents a new surface to introduce bugs throughout the codebase.

Additionally, the incentive for web developers to adopt higher-level APIs is low due to uneven browser uptake. If a feature does not add some essential new capability, but instead makes your application easier to write in newish browsers, it's rarely seen as worthwhile to go through the feature-detection dance if you have to write the fallback code anyway. Instead, developers often just use libraries built on top of the widely-supported lower-level APIs, incurring the attendant costs on all of their users.

## Goal

Enable the creation of high-level features such that:

- They stay layered on top of low-level features, never getting access to new capabilities unavailable to web developers
- Runtime costs for web developers using the features scale
- Maintenance costs for standardizing and implementing the features scale
- Fallback to polyfills, for browsers that do not support the features, is easy and transparent

## Solution

### Part 1: the infrastructure

We propose a new syntax for accessing certain web platform features, known as _layered APIs_, by importing them from special URLs:

```html
<script type="module"
        src="std:infinite-list|https://some.cdn.com/infinite-list.js">
</script>

<infinite-list>...</infinite-list>
```

```html
<script type="module">
import { get, set } from "std:async-local-storage|https://other.cdn.com/async-local-storage.js";

get("key").then(...);
</script>
```

As shown here, this `std:x|y` URL syntax contains both an _API identifier_ (e.g. "`infinite-list`" or "`async-local-storage`"), and a _fallback URL_. If the browser does not support the layered API specified by the given API identifier, it instead loads the contents of the fallback URL.

See [this document](https://docs.google.com/document/d/1jRQjQP8DmV7RL75u_67ps3SB1sjfa1bFZmbCMfJCvrM/edit) for an exploration of alternate syntax options; the above is our tentative choice for now.

### Part 2: the standards process

Like all web platform features, layered APIs would go through the standards process, producing specifications for their API surface and behavior. However, they would have an important additional constraint: their specifications _must not_ use any "magic" that is inaccessible to web developers. A concrete way of stating this is that a web developer must be able to implement a given layered API's specification, purely in unprivileged JavaScript.

Apart from this additional requirement, layered APIs would be standardized in the same way as other APIs: incubation and explainers; transition to a standards body; TAG review; etc.

## Benefits for web developers

### Cheaper high-level features

Layered APIs will reduce the amount of script developers need to load over the network.

Also, because developers explicitly import the features they use, we don’t bloat the global context of the platform for everyone. You only pay the cost of a feature for features that you use. Because of the requirement to import, implementations can use a variety of implementation strategies, ranging from business-as-usual to lazily-loading the feature from their own servers on every use.

### Encourage layering

By requiring that layered APIs not use any unexposed primitives, we are forced to identify and ship the appropriate low-level primitives needed to build the high-level feature. This gives web developers the tools they need to build their own applications and libraries. And new capabilities are never locked up inside of a higher-level API.

### Built-in fallback

Layered APIs are instantly usable in all browsers via the built-in fallback to polyfill code. At the same time, newer browser versions that do include the feature will not be shipped unnecessary code, thus decreasing page size and JavaScript parsing time.

## Benefits for standardization and implementation

### Healthier platform implementation

Requiring that layered APIs sit on top of the platform's primitives provides a clean implementation boundary. Changing a layered API can't create thorny bugs throughout other specifications or parts of the implementation.

### Decreased maintenance overhead

Implementers often shy away from building high-level features since they can create large, ongoing technical debt. Layered APIs can reduce this risk, as the clean separation means that much less maintenance work will be required. (Generally, maintenance would only be required if the browser purposefully breaks backward-compatibility in a lower-level features the layered API builds on, or fixes some bug upon which the layered API implementation inadvertently depended.)

### Security and privacy

Layered APIs will have an easier time with security and privacy review, since they build on top of other APIs which have already passed security and privacy review. By definition, they are unable to do anything that web developers can't already do themselves, which sets an upper bound on the amount of harm possible.

## Caveats

We believe that the layering restriction, and the benefits that come from it, is necessary in order for the web to responsibly ship high-level features. However, this restriction has its tradeoffs:

- Privacy and security sensitive features could not be implemented using this method
- Features that require new low-level primitives would be blocked on those primitives being standardized and shipped first

## FAQ

__Will we ship existing libraries as layered APIs?__

We believe that existing libraries won't be a good fit for layered APIs because they haven't been through the standardization process. Instead, they are maintained by a small team with a different priority of constituencies.

Additionally, like all web platform features, layered APIs will need to version with the browser, be backwards-compatible, and have their code owned by the browsers shipping it.

The lack of standardization, coupled with the ability to version, allows traditional libraries to move forward at an aggressive pace, responding to changes in the platform and differentiating themselves within the evolving competitive landscape. However, version skew means that very few sites are using the same set of code for a library. In contrast, with layered APIs — like all other web platform features — you don't get to choose what version you use; you instead get the one implemented in the browser you're running.

Given that they have discrete versions, libraries don't maintain backwards compatibility to the level of web platform features. Much of what allows libraries to present elegant developer experiences is their ability to make a clean break from past versions and build a focused product.

In order for any browser to be comfortable with bundling a piece of code, they must have some final say over changes made that land in the browser, traditionally via the standards process. External parties will still be heavily involved in the layered APIs standardization, but libraries are unlikely to cede this final say to standards bodies.

Finally, we do not want to pick winners in the ecosystem. While many libraries have a large usage base, they always have competitors. We want to standardize features that represent repeated — not competing — work.

These requirements do not preclude an external library from becoming a layered API. They only make it unlikely. If a library is:

- High usage
- Agreed amongst the web platform community to be a canonical implementation
- Versionless
- Committed to backwards compatibility
- Willing to cede governance to the standards process

… then it may be viable candidate for becoming a layered API.

__Why not just use libraries on CDNs?__

A common question is how layered APIs differ from placing JavaScript libraries on CDNs and implementing some sort of aggressive caching. While these two ideas are very similar, a CDN-based standard library neglects the role of the standards process in collaborating on web platform features. Additionally, a few advantages come from shipping directly with a browser.

Most obviously, shipping with the browser avoids any network cost on page load, whereas a CDN only reduces the cost. In emerging markets, where even a single RTT can be on the order of seconds, this benefit is significant. While aggressive caching could solve this problem for future requests, the cost must still be paid at least once.

Shipping libraries directly with the browser also means they can be more streamlined than a generic library. Traditionally this is done by implementing features in C++, although alternate strategies (such as Rust, or pre-snapshotted JavaScript) are possible. This can reduce bootup cost by hundreds of milliseconds for large features. Similarly, by being part of the browser codebase, layered APIs don't need to include unnecessary feature detects or polyfills.

Finally, "layered APIs vs. CDNs" is a somewhat misleading framing. Under the hood, layered APIs could even be implemented as cross-browser, standards-process-governed libraries that are lazily loaded by the browser from CDNs. Indeed, for rarely-used layered APIs, this may even be an attractive implementation strategy. In that case, the delta between layered APIs and a CDN-based solution is their commitment to a transparent, standards-based development process that produces specifications, and the commitment to first class support within browsers.
