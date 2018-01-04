# Permafills
Shipping high-level features as self-hosted JS imports

# Problem
The [Extensible Web Manifesto](https://extensiblewebmanifesto.org/)’s focus on low-level primitives promotes a healthy, well-layered platform that encourages innovation and experimentation in JavaScript. But focusing on low-level primitives means that developers must build most application-level components on their own, creating a high barrier to entry for new web developers. 

This lack of built-in high-level features also bloats page load size. The average site payload is [2.5 MB and takes 19 seconds to load](https://www.doubleclickbygoogle.com/articles/mobile-speed-matters/).

However, shipping high-level features carries its own risks. Standardizing and building the right API becomes exponentially harder as the complexity of a feature increases. The web has a long history of magic features that don't solve the right use cases. When we get something wrong with a high-level feature, developers are often left with no other way to achieve their goals, so they contort the feature in harmful ways to make it work.

Shipping features also incurs an ongoing maintenance and runtime cost - every new feature pollutes the browser namespace, increases JS startup costs, and represents a new surface to introduce bugs throughout the codebase.

# Goal
Enable browsers to ship high-level features such that:

- Runtime/maintenance cost scales
- Experimentation with real world websites is possible before shipping
- Developers are able to build their own version of a high-level feature if ours does not meet their use cases

# Solution
Implement high-level features as unprivileged JavaScript modules, and bundle those libraries with browsers. In all other respects, these features would be like normal web APIs: 

- Follow the normal cross-browser standardization process
- Version with the browser

There are several options to expose these modules to developers:

```javascript
<script type="module" 
        stdsrc="browser:infinite-list"                // Browser provided script
        src="https://some.cdn.com/infinite-list.js">  // Fallback for older browsers
</script>
```

This allows developers to provide a 3rd party library fallback for browsers that have not bundled the feature. Another way to accomplish this could be:

```javascript
<script type="module" 
        src="browser:infinite-list | https://cdn.example.com/infinite-list.js">
</script>
```

This would require no new syntax and could be handled under-the-hood by browser implementations.

Finally, there is a question as to whether we need explicit import at all. We could pursue ways to identify features that are used on the page and automatically import them. Perhaps through a manifest of some sort.

# Benefits
By implementing as unprivileged Javascript, we are forced to identify and ship the appropriate low-level primitives needed to build the high-level feature. This gives web developers the tools they need to build their own versions and it allows us to experiment with the API design as a pure JS library before shipping.

Using modules means that features are explicitly imported, so the global namespace isn't poluted and sites only pay runtime costs for the features they use.

Implementing in Javascript also provides a clean implementation boundary. Changing part of the Javascript implementation can't create thorny bugs throughout the renderer.

Finally, there is the potential for browsers to share these Javascript implementations. 

# Caveats
We believe that the restriction of using unprivileged Javascript, and the benefits that come from this, is necessary in order for the web to responsibly ship high-level features. However, this restriction has its tradeoffs:

- Privacy and security sensitive features could not be implemented using this method
- Developers or libraries that modify the prototypes of primitives could break permafills
- Features that require new low-level primitives would be blocked on those primitives shipping

# FAQ
__Will we ship existing libraries as permafills?__

We believe that existing libraries won’t be a good fit for permafills because permafills will need to version with the browser, be backwards-compatible, and owned by the browsers bundling the code.

Versioning allows traditional libraries to move forward at an aggressive pace, responding to changes in the platform and differentiating themselves within the evolving competitive landscape. However, version skew means that very few sites are using the same set of code for a library. In contrast, with permafills — like all other web platform features — you don't get to choose what version you use; you instead get the one implemented in the current browser version.

Given that they have discrete versions, libraries don't maintain backwards compatibility to the level of web platform features. Much of what allows libraries to present elegant developer experiences is their ability to make a clean break from past versions and build a focused product.

In order for any browser to be comfortable with bundling a piece of code, they must have some final say over changes made that land in the browser (e.g. via code review). External parties will still be heavily involved in the permafills process, but libraries are unlikely to cede this final say to browsers.

Finally, we do not want to pick winners in the ecosystem. While many libraries have a large usage base, they always have competitors. We want to permafill features that represent repeated — not competing — work.

These requirements do not preclude an external library from becoming a permafill. They only make it unlikely. If a library is:

- High usage
- Agreed amongst browsers and developers to be a canonical implementation
- Versionless (i.e. version with the browser)
- Committed to backwards compatibility 
- Willing to cede final code approval to browsers 

… then they may be viable candidates for a permafill. 

__Why not just use libraries on CDNs?__

A common question is how permafills differ from CDNs and aggressive caching. While these two ideas are very similar, a few advantages come from shipping directly with a browser.

Most obviously, shipping with the browser avoids any network cost on page load, whereas a CDN only reduces the cost. In emerging markets, where even a single RTT can be on the order of seconds, this benefit is significant. While aggressive caching could solve this problem for future requests, the cost must still be paid at least once.

Shipping libraries directly with the browser also means they can be more streamlined than a generic library. We can store parsed code, reducing bootup cost by hundreds of milliseconds for large JS features. We can also strip out unnecessary feature detects or polyfills that aren’t needed for that particular browser version.

Another major difference is that because permafills are browser features, they go through the same collaborative cross-browser design process. Falling back to proprietary CDNs would make it more difficult to maintain this open design process.

Finally, "permafills vs. CDNs" is a somewhat misleading framing. Under the hood, permafills could even be implemented as cross-browser, collaboratively-developed libraries that are lazily loaded by the browser from CDNs. Indeed, for rarely-used permafills, this may be more appropriate than bundling them with the initial download. In that case, the delta between permafills and a CDN-based solution is their commitment to a transparent, stable development process and the commitment to first class support within browsers.

