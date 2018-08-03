# Layered APIs

A new standards effort for collaborating on high-level features.

## Problem

The [Extensible Web Manifesto](https://extensiblewebmanifesto.org/)’s focus on low-level primitives promotes a healthy, well-layered platform that encourages innovation and experimentation in JavaScript. But focusing on low-level primitives means that developers must build most application-level components on their own, creating a high barrier to entry for new web developers.

This lack of built-in high-level features also bloats page load size. The average site payload is [2.5 MB and takes 19 seconds to load](https://www.doubleclickbygoogle.com/articles/mobile-speed-matters/).

We believe that, to be a competitive platform, the web needs an easier "out of the box experience" for developers. It needs more high-level APIs, which work together nicely, meet a known quality bar, and are widely known and accessible without having to go looking for them. At the same time, they should be framework-agnostic (in the way that current web APIs are), forming a common base layer that frameworks and applications can build on top of.

Historically, standards bodies and implementers have been reluctant to work on higher-level APIs. In particular, introducing new capabilities via high-level APIs is dangerous, as when we get something wrong, developers are often left with no other way to access these capabilities. Shipping features also incurs an ongoing maintenance and runtime cost — every new feature pollutes the browser namespace, increases JS startup costs, and represents a new surface to introduce bugs throughout the codebase.

Additionally, the incentive for web developers to adopt higher-level APIs is low due to uneven browser uptake. If a feature does not add some essential new capability, but instead makes your application easier to write in newish browsers, it's rarely seen as worthwhile to go through the feature-detection dance if you have to write the fallback code anyway. Instead, developers often just use libraries built on top of the widely-supported lower-level APIs, incurring the attendant costs on all of their users.

## Goal

Enable the creation of high-level features such that:

- They stay layered on top of low-level features, never getting access to new capabilities unavailable to web developers
- Runtime costs for web developers using the features scale
- Maintenance costs for standardizing and implementing the features scale
- Fallback to polyfills, for browsers that do not support the features, is easy and transparent

## Solution

### Part 1: the standards process

We propose a renewed focus in the web-standards space on higher-level APIs. In particular, we'd propose a marketing-brand of "layered APIs" for high-level API standardization efforts which follow these two constraints:

- **Layered**: Their specifications _must not_ use any "magic" that is inaccessible to web developers, i.e. they must be layered on top of the platform's existing lower-level features. A concrete way of stating this is that a web developer must be able to implement a given layered API's specification, purely in unprivileged JavaScript.
- **Pay as you go**: Their specifications must expose the APIs via JavaScript modules, instead of via globals. This allows implementations to load the API on demand, ensuring web applications don't pay the cost of features they don't use.

Apart from these requirements, a "layered API" specification would be standardized in the same way as other APIs: incubation and explainers; transition to a standards body and interfacing with such bodies' existing workstreams; TAG review; etc.


### Part 2: importing infrastructure

Per part 1, the only web-developer-visible feature of layered APIs is that they are imported as JavaScript modules. This section outlines the details of such a setup.

The summary is that web developers would be able to import these modules using identifiers of the form `std/async-local-storage`, and would be able to use the [package name maps](https://github.com/domenic/package-name-maps) proposal to customize several aspects of the resolution.

Now, let's discuss how we get there:

#### Base layer: `layeredapi:` URLs for the modules

Since modules on the web are URL-keyed, this means we need special URLs identifying these LAPI modules. We propose using the `layeredapi:` scheme in this regard. This scheme would be more like `blob:` than like `https:`, in that it would be handled specially in the module resolution algorithm (similar to `blob:`'s special handling in the [media resource fetch algorithm](https://html.spec.whatwg.org/#concept-media-load-resource)). It would not be possible to fetch such URLs directly, which makes sense, as it's not clear what they would contain; they're just locators into the user agent's internal implementation.

We further propose that each layered API be located at a specific subset of the URL space, such as `layeredapi:virtual-scroller`. Within this package, individual modules would be present at URLs like `layeredapi:virtual-scroller/main` for the main module, or `layeredapi:virtual-scroller/virtual-content` for a secondary module.

This means you could import a layered API like so in a HTML context:

```html
<script type="module" src="layeredapi:virtual-scroller/main"></script>
```

or like so in a JS context:

```js
import "layeredapi:virtual-scroller/main";
```

But, we view these URLs as the "raw" location of the LAPIs. The way most users interface with them will be as packages:

#### Exposing as built-in packages using package name maps

For a variety of reasons explained in the next few subsections, it's advantageous to abstract away the raw `layeredapi:` URLs behind the facade of a built-in "package". The concept of a "package" of JavaScript code comes from the [npm](https://www.npmjs.com/) ecosystem, which is popular for front-end code today, and we'd bring it to the browser generically via the [package name maps](https://github.com/domenic/package-name-maps/) proposal.

Consider a package name map such as the following:

```json
{
  "packages": {
    "std/virtual-scroller": {
      "path": "layeredapi:virtual-scroller",
      "main": "main"
    }
  }
}
```

This tells the browser that the module specifier `std/virtual-scroller` should map to the URL `layeredapi:virtual-scroller/main`, and furthermore that any module specifier under the package, e.g. `std/virtual-scroller/*`, should map to the corresponding URL, `layeredapi:virtual-scroller/*`.

**We propose that layered APIs add to the browser's built-in package name map**, so that they each are exposed as a package. Thus, the more idiomatic way of using a LAPI would be as

```html
<script type="module" src="import:std/virtual-scroller"></script>
```

(assuming we do [domenic/package-name-maps#23](https://github.com/domenic/package-name-maps/issues/23)) or

```js
import "std/virtual-scroller";
```

This layer of abstraction gives the following benefits:

##### Changing the mapping

A web developer's explicit package name map should take precedence over the browser's built-in package name map. In that case, a web developer could redirect `std/virtual-scroller` to their own implementation with just the following code:

```html
<script type="packagemap">
{
  "packages": {
    "std/virtual-scroller": {
      "path": "/js/libs/my-scroller",
      "main": "index.mjs"
    }
  }
}
</script>
```

##### Providing fallbacks

In browsers that do support package name maps, but don't support a given layered API, a [proposed idea for package name maps](https://github.com/domenic/package-name-maps#supplying-fallbacks-for-host-supplied-standard-library-packages) would allow fallbacks to user-provided polyfills. That could look something like:

```json
{
  "packages": {
    "std/virtual-scroller": [
      { "path": "layeredapi:virtual-scroller", "main": "main" },
      { "path": "https://backupcdn3.com/virtual-scroller", "main": "index.mjs" },
      { "path": "/node_modules/std-virtual-scroller-polyfill", "main": "index.js" }
    ]
  }
}
```

This would mostly be useful in some years, once all target browsers supports package name maps, but not all support every LAPI.

##### Backward-compatibility

In browsers that don't support package name maps, [another proposed idea](https://github.com/domenic/package-name-maps) inverts the previous idea by allowing you to use the polyfill URL as the import identifier, but replace it with the contents of the layered API if that is present. That could look something like:

```json
{
  "packages": {
    "std/virtual-scroller": {
      "replaces": [
        { "path": "/node_modules/std-virtual-scroller-polyfill", "main": "index.js" }
      ]
    }
  }
}
```

Then web developers could do

```js
import "/node_modules/std-virtual-scroller/polyfill/index.js";
```

which in a browser that supports package name maps and has a mapping for `std/virtual-scroller` would be interpreted as `import "std/virtual-scroller"`, but in older browsers would import the fallback as written.

This would likely be the initial deployment strategy for web developers depending on layered APIs.

## Benefits for web developers

### Cheaper high-level features

Layered APIs will reduce the amount of script developers need to load over the network.

Also, because developers explicitly import the features they use, we don’t bloat the global context of the platform for everyone. You only pay the cost of a feature for features that you use. Because of the requirement to import, implementations can use a variety of implementation strategies, ranging from business-as-usual to lazily-loading the feature from their own servers on every use.

### Encourage layering

By requiring that layered APIs not use any unexposed primitives, we are forced to identify and ship the appropriate low-level primitives needed to build the high-level feature. This gives web developers the tools they need to build their own applications and libraries. And new capabilities are never locked up inside of a higher-level API.

### Built-in fallback

Layered APIs are instantly usable in all browsers via the package name map-based fallback to polyfill code. At the same time, newer browser versions that do include the feature will not fetch unnecessary code, thus decreasing page size and JavaScript parsing time.

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

## Layered API design considerations

### What makes a good layered API candidate?

When judging whether a feature is a good fit for the layered APIs effort, here are some criteria to consider:

- **Does this feature need new low-level capabilities to work successfully?** If so it, it isn't a good candidate yet; we need to fill in those gaps first.

- **Can this feature stand on its own, or does it require integration into existing APIs?** For example, adding methods or properties to existing web platform objects like `Array` or `HTMLElement` is potentially tricky, and we're not sure yet whether we should create layered APIs that, upon importing, have global side effects. For now, features that require such integration are not a good candidate for layered APIs.

- **Is this feature tricky to implement performantly or correctly?** It's better for the platform if such features can be standardized once, and implemented by browsers, instead of requiring developers to get them right every time independently. This criteria motivates the potential infinite virtual list or [tasklets](https://github.com/GoogleChromeLabs/tasklets) layered APIs.

- **Do the APIs for this feature vary wildly across the JS ecosystem, or have they mostly settled down?** Layered APIs will be less successful when they try to pick a winner that excludes popular styles or paradigms. For example, a virtual DOM layered API would likely be a poor idea at this time.

- **Does this feature involve a lot of styling choices for its UI?** If so, we're still figuring that out (see below), so the feature is probably not (yet) a good fit for the layered APIs effort.

- **Will this feature be used commonly, or rarely?** In the long term, layered APIs are a good fit for both cases. But for the initial batch of layered APIs, we'd like to focus on ones that will be used widely to show their value in terms of bringing down code size and making it easier to build web apps out of the box.

### Styling and UI-component layered APIs

Several potentially good layered APIs, including the [infinite virtual scroller component](https://github.com/domenic/infinite-list-study-group), are UI components. Such components should generally come with minimal styling—at least as minimal, if not more, than existing standard HTML UI components. It would not be appropriate to encode a specific UI styling, like Material Design (Google) or Cupertino (Apple), into the layered APIs.

At the same time, layered APIs should be extremely styleable: authors should be able to make them fit into their pages, ideally with only CSS modifications. In the current landscape, this will require care; e.g. we cannot over-use shadow DOM, since it cannot be styled inside. In the future, [CSS shadow parts](https://tabatkins.github.io/specs/css-shadow-parts/) will greatly help with this.

That said, it’s important that built in UI components on the platform look and feel good by default. So there’s an open problem we’ll have to figure out for how UI components should be themed by default such that they can be consistent with OS-specific expectations. We can avoid dealing with this problem by starting with UI components that have no visual aspect to them (e.g. virtual list, which has no expectations, vs. new form controls, which do).
