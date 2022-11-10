# Adding a NavigationID to Performance Timeline Events
## Or, Exposing user-perceived navigations for the purpose of analyzing user-perceived performance on the web.

Currently, web performance APIs such as the Performance Timeline, and associated timeline entries such as Navigation Timing, Paint Timing, Resource Timing and others, are designed around a distinctly "Web 1.0" vision of the user's experience. Performance issues are very often subjective[^1], and are relative to the user's expectations, and so it is important that performance be analyzed with respect to those expectations. Specifically, these APIs implicitly expect that the user's first experience of the page occurs after they load if from the network (or the HTTP cache), and that they believe themselves to be interacting with that page until they navigate away, loading the next page from the network.

These assumptions are broken on the modern web by features such as the back-forward cache, which causes a page *not* to be unloaded when a user navigates away, and by single-page-apps, which are often designed to appear to the user like a transition has occurred, without needing to actually navigate to a new page. In both cases, the page continues running, and the performance timeline APIs still report events relative to the network or cache load, while the user's experience may be no different than if they had performed a traditional navigation to a new page.

### Proposal

This, then, is a proposal to add a "navigation id" to the performance timeline events, which is intended to provide a unique identifier[^2] to each "user-perceived" navigation-like transition, regardless of whether or not that represents an actual HTTP network-or-cache fetch and a new browsing context. Performance APIs can tag every entry with the id of the most recent navigation, and then tools analyzing page performance as it relates to user experience can segment the entries by navigation id. Each new id change should be accompanied by a timeline entry which represents that transition.

Initially, this navigation id would be set when a page is loaded, and it would be changed in response to the following events:

* The page is restored from the back-forward cache
    * This is accompanied by a `BackForwardCacheRestoration` `PerformanceEntry`
* An SPA navigation is detected[^3]
    * This is accompanied by a `SoftNavigation` `PerformanceEntry`

Other transitions may need to be added to this set in the future, but this is the inital propsal.

#### Proposed IDL Changes

This would require a change to the [PerformanceEntry interface](https://w3c.github.io/performance-timeline/#dom-performanceentry), adding a navigationId member:

```webidl
[Exposed=(Window,Worker)]
interface PerformanceEntry {
  readonly    attribute DOMString           name;
  readonly    attribute DOMString           entryType;
  readonly    attribute DOMHighResTimeStamp startTime;
  readonly    attribute DOMHighResTimeStamp duration;
  readonly    attribute unsigned long       navigationId;
  [Default] object toJSON();
};
```

[^1]: This subjectivity is the reason that the Cumulative Layout Shift metric, for instance, differentiates between expected and unexpected layout shifts(https://web.dev/cls/#expected-vs-unexpected-layout-shifts), exempting shifts which occur when the user will legitimately expect some content on the page to change. Similarly, the Largest Contentful Paint metric currently excludes paints which happen after the user interacts with the page, with the idea that paint delays after that point are less critical than those which delay the initial user understanding of the page.

[^2]: In the limit, a simple incrementing counter could work, but maybe an opaque UUID is better?

[^3]: SPA navigations would be governed by [heuristics](https://github.com/yoavweiss/soft-navigations/), but these should be well defined and predictable to developers.
