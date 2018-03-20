# Layered APIs

A new standards effort for collaborating on high-level features.

## Problem

The [Extensible Web Manifesto](https://extensiblewebmanifesto.org/)’s focus on low-level primitives promotes a healthy, well-layered platform that encourages innovation and experimentation in JavaScript. But focusing on low-level primitives means that developers must build most application-level components on their own, creating a high barrier to entry for new web developers.

This lack of built-in high-level features also bloats page load size. The average site payload is [2.5 MB and takes 19 seconds to load](https://www.doubleclickbygoogle.com/articles/mobile-speed-matters/).

Historically, standards bodies and implementers have been reluctant to work on higher-level APIs. In particular, introducing new capabilities via high-level APIs is dangerous, as when we get something wrong, developers are often left with no other way to access these capabilities. Shipping features also incurs an ongoing maintenance and runtime cost — every new feature pollutes the browser namespace, increases JS startup costs, and represents a new surface to introduce bugs throughout the codebase.

Additionally, the incentive for web developers to adopt higher-level APIs is low due to uneven browser uptake. If a feature does not add some essential new capability, but instead makes your application easier to write in newish browsers, it's rarely seen as worthwhile to go through the feature-detection dance if you have to write the fallback code anyway. Instead, developers often just use libraries built on top of the widely-supported lower-level APIs, incurring the attendant costs on all of their users.

## Goal

Enable the creation of high-level features such that:

- They stay layered on top of low-level features, never making access to new capabilities unavailable to web developers
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

## What makes a good layered API candidate?

When judging whether a feature is a good fit for the layered APIs effort, here are some criteria to consider:

- **Does this feature need new low-level capabilities to work successfully?** If so it, it isn't a good candidate yet; we need to fill in those gaps first.

- **Can this feature stand on its own, or does it require integration into existing APIs?** For example, adding methods or properties to existing web platform objects like `Array` or `HTMLElement` is potentially tricky, and we're not sure yet whether we should create layered APIs that, upon importing, have global side effects. For now, features that require such integration are not a good candidate for layered APIs.

- **Is this feature tricky to implement performantly or correctly?** It's better for the platform if such features can be standardized once, and implemented by browsers, instead of requiring developers to get them right every time independently. This criteria motivates the potential infinite list or [tasklets](https://github.com/GoogleChromeLabs/tasklets) permafills.

- **Do the APIs for this feature vary wildly across the JS ecosystem, or have they mostly settled down?** Layered APIs will be less successful when they try to pick a winner that excludes popular styles or paradigms. For example, a virtual DOM layered API would likely be a poor idea at this time.

- **Does this feature involve a lot of styling choices for its UI?** If so, we're still figuring that out (see below), so the feature is probably not (yet) a good fit for the layered APIs effort.

- **Will this feature be used commonly, or rarely?** In the long term, layered APIs are a good fit for both cases. But for the initial batch of layered APIs, we'd like to focus on ones that will be used widely to show their value in terms of bringing down code size and making it easier to build web apps out of the box.

## Styling and UI-component layered APIs

Several potentially good layered APIs, including the [infinite list component](https://github.com/domenic/infinite-list-study-group), are UI components. Such components should generally come with minimal styling—at least as minimal, if not more, than existing standard HTML UI components. It would not be appropriate to encode a specific UI styling, like Material Design (Google) or Cupertino (Apple), into the layered APIs.

At the same time, layered APIs should be extremely styleable: authors should be able to make them fit into their pages, ideally with only CSS modifications. In the current landscape, this will require care; e.g. we cannot over-use shadow DOM, since it cannot be styled inside. In the future, [CSS shadow parts](https://tabatkins.github.io/specs/css-shadow-parts/) will greatly help with this.

That said, it’s important that built in UI components on the platform look and feel good by default. So there’s an open problem we’ll have to figure out for how UI components should be themed by default such that they can be consistent with OS-specific expectations. We can avoid dealing with this problem by starting with UI components that have no visual aspect to them (e.g. infinite list, which has no expectations, vs. new form controls, which do).
