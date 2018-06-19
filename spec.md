# Layered APIs Proto-spec

This document contains the basics of a spec for the [layered APIs infrastructure](./README.md#part-1-the-infrastructure) discussed in the README. It is by no means finalized or formalized, but it gives implementers something to look at and possibly prototype with.

We expect that if this spec gains acceptance it would eventually be incorporated in the HTML Standard's script loading sections.

## Introduction

Although layered APIs are loaded as JavaScript modules, they cannot have web-exposed source code. Their source code is an implementation detail, and depending on the browser, may not be in JavaScript.

This means we need to modify the module loading infrastructure slightly, as currently you are only able to load and evaluate module source code that you can also read with `fetch()`, due to the requirement for module scripts to be shared via CORS.

## Layered API specifications

Layered API specifications define the following key concepts:

<dl>
  <dt id="api-identifier">API identifier
  <dd>A unique string by which this layered API will be imported

  <dt id="exports">Exports
  <dd>An <a href="https://infra.spec.whatwg.org/#ordered-map">ordered map</a> of strings to JavaScript objects, where the keys must be <a href="https://tc39.github.io/ecma262/#prod-IdentifierName">IdentifierName</a>s

  <dt id="evaluation-steps">Evaluation steps
  <dd>An optional set of steps to run when the layered API is evaluated, i.e. when it is first imported
</dl>

## Layered API module records

We introduce a new type of [Module Record](https://tc39.github.io/ecma262/#sec-abstract-module-records), here called a Layered API Module Record.

We could define this in one of two ways:

1. **Hand-wave**: reusing the existing Source Text Module Record (STMR) structure, and saying that implementations should create a STMR as if by parsing source text that would have the observable exports and evaluation steps as specified in the layered API's specification
2. **Formalize**: actually create a new Module Record specification type; details below

We're not sure that formalization would actually be beneficial to implementers, but here we sketch out what such a module record would look like:

<details>
<summary>Formalized Layered API Module Record (LAMR)</summary>

<dl>
  <dt>[[Realm]]
  <dd>Every Realm contains a full set of LAMRs

  <dt>[[Environment]]
  <dd>A synthetic Module Environment created and populated with bindings corresponding to the exports defined in the layered API's specification

  <dt>[[Namespace]]
  <dd>No special treatment needed; the ECMAScript specification machinery will lazily fill this in for us

  <dt>[[HostDefined]]
  <dd>Not used

  <dt>GetExportedNames(exportStarSet)
  <dd>Returns the list of exported names defined in the layered API's specification. (The <var>exportStarSet</var> can be ignored in our case, as layered APIs cannot participate in module cycles.)

  <dt>ResolveExport(exportName, resolveSet)
  <dd>Checks if <var>exportName</var> is present in the list of exported names defined in the layered API's specification; returns { [[Module]]: this LAMR, [[BindingName]]: <var>exportName</var> } if so, or null otherwise. (The <var>resolveSet</var> can be ignored since layered APIs cannot participate in module cycles.)

  <dt>Instantiate()
  <dd>Sets up [[Environment]] as described above

  <dt>Evaluate()
  <dd>Evaluates any setup code described in the layered API's specification
</dl>
</details>

## Pre-populating the module map

We would specify that on realm initialization, the [module map](https://html.spec.whatwg.org/multipage/webappapis.html#concept-settings-object-module-map) is pre-populated with mappings from every `std:x` URL to the corresponding layered API module record. This must include `std:blank`, which is detailed below

A browser could implement this by lazily populating the map when the appropriate layered API is imported; this would be observably equivalent. Note that the map is only populated; _moduleRecord_.Evaluate() is not called until `import` time.

## Reserved layered APIs

`std:blank` is a layered API that is guaranteed to be supported if layered APIs are supported by a browser. It has an API identifier of "`blank`", an empty list of exports, and no initialization steps. Its primary use case is feature detection.

`std:none` denotes a layered API that is never supported; that is, layered API specifications must not use "`none`" as their API identifier. It is also useful for tests and feature detection.

## Modifications to module loading

We make the following changes to ensure that when loading modules, we retrieve the LAMR from the module map if the corresponding LAPI is implemented, or use the fallback URL if not.

### Resolve a module specifier

Modify [resolve a module specifier](https://html.spec.whatwg.org/multipage/webappapis.html#resolve-a-module-specifier)'s first step as follows:

1. Let _parsed_ be the result of applying the [URL parser](https://url.spec.whatwg.org/#concept-url-parser) to specifier. If _parsed_ is not failure, then return the [layered API fetching URL](#user-content-layered-api-fetching-url) given _parsed_ and _script_'s [base URL](https://html.spec.whatwg.org/multipage/webappapis.html#concept-script-base-url).

This impacts `import` statements and `import()` calls.

### `<script>`'s prepare a script

In [prepare a script](https://html.spec.whatwg.org/multipage/scripting.html#prepare-a-script), change step 23.6's "`module`" case to the following:

1. Set _url_ to the [layered API fetching URL](#user-content-layered-api-fetching-url) given _url_ and _settings object_'s [API base URL](https://html.spec.whatwg.org/multipage/webappapis.html#api-base-url).
1. If _url_ is failure, [queue a task](https://html.spec.whatwg.org/multipage/webappapis.html#queue-a-task) to [fire an event](https://dom.spec.whatwg.org/#concept-event-fire) named [`error`](https://html.spec.whatwg.org/multipage/indices.html#event-error) at the element, and return.
1. [Fetch a module script graph](https://html.spec.whatwg.org/multipage/webappapis.html#fetch-a-module-script-tree) given _url_, _settings object_, "`script`", and _options_.

This impacts `<script type="module">`'s `src=""` resolution.

### `<link rel="modulepreload">`'s obtain the resource

In the "obtain the resource" algorithm for [link type "`modulepreload`"](https://html.spec.whatwg.org/#link-type-modulepreload), move the current step 5 (which creates the _settings object_ variable) up to before the current step 4. Then, insert the following steps after the current step 4:

1. Set _url_ to the [layered API fetching URL](#user-content-layered-api-fetching-url) given _url_ and _settings object_'s [API base URL](https://html.spec.whatwg.org/multipage/webappapis.html#api-base-url).
1. If _url_ is now failure, return.

This impacts `<link rel="modulepreload">`'s `href=""` resolution.

### Supporting algorithm

<p id="layered-api-fetching-url">The above modifications delegate to the following algorithm for determining the <dfn>layered API fetching URL</dfn> given a URL <var>url</var> and a URL <var>baseURLForFallback</var>:

1. If _url_'s [scheme](https://url.spec.whatwg.org/#concept-url-scheme) is not "`std`", return _url_.
1. Let _serialized_ be the result of running the [URL serializer](https://url.spec.whatwg.org/#concept-url-serializer) on _url_.
1. Remove the leading "`std:`" from _serialized_.
1. Let _identifier_ be the portion of _serialized_ before the first U+007C (|), or all of _serialized_ if no U+007C is present.
1. Let _fallback_ be the portion of _serialized_ after the first U+007C, or null if no U+007C is present.
1. If the layered API identified by _serialized_ is implemented by this user agent, return the result of [parsing](https://url.spec.whatwg.org/#concept-url-parser) the concatenation of "`std:`" with _identifier_.
1. If _fallback_ is null, return failure.
1. Return the result of [parsing](https://url.spec.whatwg.org/#concept-url-parser) _fallback_ with the base URL _baseURLForFallback_.

This operation maps URLs of the form `std:x|y` to a specific absolute URL: either `std:x`, or `y` (resolved relative to the given base).

## Specification consequences

Here are some things to note about the above proto-spec:

### Only impacts module loading

Because this specification only modifies module specifier resolution and module fetching, its impact is limited. Unlike `blob:` URLs that wrap `Blob` objects, `std:` URLs are not something that can be seen by `fetch()`, `<img>`, etc.

Indeed, it's not really accurate to treat `std:x|y` URLs as URLs at all. They are more just mechanisms for interfacing with the platform's module resolution system. They do not have an origin; they are not seen by service workers; they are not affected by CSP.

This is in some ways similar to `blob:` URLs that wrap `MediaSource` objects: those are handled specially in HTML's [media resource fetch algorithm](https://html.spec.whatwg.org/#concept-media-load-resource), and never make it to platform's general fetching infrastructure (e.g. the [scheme fetch](https://fetch.spec.whatwg.org/#scheme-fetch) algorithm asserts that it never sees such `blob:` URLs).

Future work may make it possible to reuse JavaScript import specifiers in other URL-accepting contexts, e.g. people have proposed `<img src="package:bootstrap/arrow.svg">` to [integrate with package name maps](https://github.com/domenic/package-name-maps/issues/23). However, this would need separate work to integrate with `std:x|y` URLs, since we would need to preserve the property of these resources being opaque to hide their implementation details.

### Feature detection

Feature detection of the layered API mechanism is possible using code such as the following:

```html
<script type="module">
(async () => {
  let layeredAPIsSupported = false;
  try {
    await import("std:blank");
    layeredAPIsSupported = true;
  } catch {}

  if (!layeredAPIsSupported) {
    // Load polyfills the old-fashioned way.
  }
})();
```

This feature detection is asynchronous, but does not require any network round-trip.

### Relative fallback URLs work

Although all of our examples tend to use absolute fallback URLs, since we anticipate fallbacks being located on CDNs or similar, relative URLs also work:

```js
import { storage } from "std:async-local-storage:lib/polyfills/als.mjs";
```

```html
<script type="module" src="std:virtual-list|/node_modules/virtual-list/element.mjs">
```
