# Soft Navigations

## Overview

“Soft navigations” are JS-driven same-document navigations that are using the history API or the new Navigation API, triggered by a user gesture and modifies the DOM, modifying the previous content, as well as the URL displayed to the user.

[PerformanceTimeline#168](https://github.com/w3c/performance-timeline/issues/168) outlines the desire to be able to better report performance metrics on soft navigation. Heuristics for detecting soft navigations can ensure that developers can measure their SPA’s performance metrics, and optimize them to benefit their users.

## Motivation

Why would we want to web-expose soft navigation at all, you ask?

Well, a few reasons:
* Developers would like to attribute various performance entries to specific “soft navigation” URLs. For example, layout shifts caused in one URL can currently be attributed to the corresponding landing page, resulting in mis-attribution and trouble finding the real cause and fixing it.
* Developers would like to receive various “load” performance entries for soft navigations. Specifically, paint timing entries seem desired for such navigations.

From a user's perspective, while they don't necessarily care about the architecture of the site they're visiting, they likely care about it being fast. This specification would enable alignment of the measurements with the user experience, improving the chances of SPA sites being perceived as fast by their users.

## Goals

* Enable measurement of Single-Page app performance in the wild for today's apps.
* Enable such measurement at scale - allowing the team that owns the measurement to be decoupled from the team that owns the app's logic.
* Not rely on developers annotating soft navigations themselves.

## Proposed Heuristics

* The user initiated a soft navigation, by clicking on a DOM element, or using an unfocused "keydown" event.
  - We considered using only semantic elements, but it seems to not match current real-world practices.
* That operation resulted in an event handler firing (either a “click” event or a “navigate” event)
* We then follow the tasks triggered by the event handler:
  - If it’s a “navigate” event, those tasks are part of the Promise passed to traverseTo()
  - If it’s a “click” or a "keydown" event, those tasks are spawned by the event handler itself
* In case of a “click”  or a "keydown" event, the handler triggered tasks that included History.pushState() or History.replaceState() calls, or a change to the document’s location
* The tasks modify DOM elements.
  - We may try to limit that to specific DOM elements or some other heuristic regarding "meaningful" DOM modifications, in case we'd see the heuristic is too broad and captures modifications which should not be reasonably be considered 
navigations".
* The next paint that contains a contentful element will be considered the soft navigation’s FCP.
* The next largest contentful element will trigger LCP entries.
* Finally, we should consider limiting the amount of soft navigation detected in a certain timeframe (e.g. X per Y seconds).

### [Task attribution](https://bit.ly/task-attribution)

The above heuristics rely on the ability to keep track of tasks and their provenance. We need to be able to tell that a certain task was posted by another, and be able to create a causality chain between DOM dirtying and URL modifications to the event handler that triggered the soft navigation.

Note: We would need to specify TaskAttribution as part of the event loop's processing in order to properly specify the heuristics above.

## Proposed API shape
```
SoftNavigationEntry : PerformanceEntry {
   unsigned long NavigationId;
}
```
[NavigationID](https://pr-preview.s3.amazonaws.com/w3c/performance-timeline/192/ca6936d...clelland:6e5497e.html#dom-performanceentry-navigationid) will be defined in Performance Timeline. ([explainer](https://docs.google.com/document/d/1sUPyBLPYtKPyyuu1-3XqJT1QUE6n0cRBZP5IfH8cjQE/edit?resourcekey=0-PpjDpPLPsFk8ips35GXj1w))

The inheritance from `PerformanceEntry` means that the entry will have `startTime`, `name`, `entryType` and `duration`:
* `startTime` would be defined as the time in which the user's interaction was received. See [discussion](https://bugs.chromium.org/p/chromium/issues/detail?id=1369680).
* `name` would be the URL of the history entry representing the soft navigation.
* `entryType` would be "soft-navigation".
* `duration` - it's not currently clear what a useful `duration` value would be that enables the entry to fire early enough. See [issue #4](https://github.com/yoavweiss/soft-navigations/issues/4).

## Examples

That's all neat, but how would developers use the above? Great question!

### Reporting a soft navigation

If developers want to augment their current reporting with soft navigations, they'd need to do something like the following:

```javascript
const soft_navs = await new Promise(resolve => {
  (new PerformanceObserver( list => resolve(list.getEntries()))).observe(
    {type: 'soft-navigation', buffered: true});
  });
```

Or by using `getEntriesByType`:

```javascript
const soft_navs = performance.getEntriesByType('soft-navigation');
```

That would give them a list of past and future soft navigations they can send to their server for processing.

They would be able to also get soft navigations as they come (similar to other performance entries):

```javascript
const soft_navs = [];
(new PerformanceObserver( list => soft_navs.push(...list.getEntries()))).observe(
    {type: 'soft-navigation'});
```

Or to include past soft navigations:

```javascript
const soft_navs = [];
(new PerformanceObserver( list => soft_navs.push(...list.getEntries()))).observe(
    {type: 'soft-navigation', buffered: true});
```

### Correlating performance entries with a soft navigation

For that developers would need to collect `soft_navs` into an array as above.
Then they can, for each entry (which can be LCP, FCP, or any other entry type), find its corresponding duration as following:

```javascript
const lcp_entries = [];
(new PerformanceObserver( list => lcp_entries.push(...list.getEntries()))).observe(
    {type: 'largest-contentful-paint', includeSoftNavigationObservations: true});

for (entry of lcp_entries) {
  const id = entry.navigationId;
  const nav = soft_navs.filter(entry => entry.navigationId == id)[0];
  entry.lcp_duration = entry.startTime - nav.startTime;
}
```

## Required spec changes

* This relies on performance timeline's navigationID
* We'd need to specify Task Attribution
* We would need to modify PaintTiming and LCP to restart their reporting once a soft navigation was encountered.
* We need to add `PerformanceObserverInit` option named "includeSoftNavigationObservations", that will indicate that post-soft-navigation FP, FCP and LCP entries should be observed.

## Privacy and security considerations

This API exposes a few novel paint timestamps that are not available today, after a soft navigation is detected: the first paint, the first contentful paint and the largest contentful paint.
It is already possible to get some of that data through Element Timing and requestAnimationFrame, but this proposal will expose that data without the need to craft specific elements with the `elementtiming` attribute.

### Mitigations

* The LCP timestamp is subject to the same constraints as current LCP entries, and doesn't expose cross-origin rendering times without an explicit opt-in.
* The FCP timestamp doesn't necessary waits until a cross-origin image is fully loaded in order to fire, minimizing the cross-origin information exposed.
* Soft navigations detected are inherently user-driven, preventing programatic scanning of markup permutations and their impact on paint timestamps.
* Soft Navigation detection can be time limited, further limiting the scalability of information exposure.

Given the above mitigations, attacks such as [history sniffing attacks](https://krebsonsecurity.com/2010/12/what-you-should-know-about-history-sniffing/) are not feasible, given that `:visited` information is not exposed.
`:visited` only modifications are not counting as DOM modifications, so soft navigations are not detected. Whenever other DOM modifications are included alongside visited changes, the next paint would include both modifications, and hence won't expose visited state.

Furthermore, cross-origin imformation about images or font resources is not exposed by Soft Navigation LCP, similarly to regular LCP.

## Open Questions

* Do we need to define FCP/LCP as contentful paints that are the result of the soft navigation?
* Could we augment the heuristic to take both DOM additions and removals into account?
  - Currently, interactions such as Twitter/Gmail's "compose" button would be considered soft navigations, where one could argue they are really interactions.
  - A heuristic that requires either a modification of existing DOM nodes or addition *and* removal of nodes may be able to catch that, without increasing the rate of false negatives.
* `<your questions here>`

## I want to take this for a spin!!

I like how you're thinking!

You can do that by:
* Joining the [Origin Trial](https://developer.chrome.com/origintrials/#/trials/active), and [enabling it on your site](https://developer.chrome.com/en/docs/web-platform/origin-trials/).
* Alternatively, you can:
  - [enable "Experimental Web Platform features"](chrome://flags/#enable-experimental-web-platform-features) in Chrome
  - Browsing to the site you want to test!
  - Opening the devtools' console
  - Looking for "A soft navigation has been detected" in the console logs
  - Alternatively, running the example code above in your console to observe `SoftNavigationEntry` entries
 
The Chrome team [have published an article about its implementation](https://developer.chrome.com/blog/soft-navigations-experiment/), and how developers can use this to try out the proposed API to see how it fits your needs.

And remember, if you find bugs, https://crbug.com is the best way to get them fixed!

# FAQs

## Should this rely on the [Navigation API](https://html.spec.whatwg.org/multipage/nav-history-apis.html#navigation-api)?

If this effort were to rely on the Navigation API, that would mean that it can only cover future web apps, or require web apps to completely rewrite their routing libraries in order to take advantage of Soft Navigation measurement.
That would go against the goal of being able to measure such navigations at scale.

On top of that, the Navigation API does not make any distinction between "real" navigations and interactions, so even if we were to rely on the Navigation API, extra heuristics would still be needed.

With that said, this effort works great with the Navigation API, as well as with the older history API.
