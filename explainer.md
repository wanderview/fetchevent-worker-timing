# FetchEvent Worker Timing Explained

## What’s All This Then

This document proposes a new web platform API that allows a [service worker][ServiceWorker] script to record timing information associated with a particular [`FetchEvent`][FetchEvent].  This timing information then becomes available on the associated network request’s `PerformanceResourceTiming` object.

When the browser navigates or loads a subresource for a page it may [dispatch][HandleFetch] a `FetchEvent` to a service worker.  This allows the service worker script to dynamically determine how that navigation or subresource request is handled.  This provides the site the flexibility to dynamically handle poor network conditions or other situations.

This flexibility, however, can come at a cost.  Dispatching an event to a javascript thread while fetching a network request adds some overhead.  This overhead may vary based on browser and device characteristics.  In addition, the service worker script itself may inadvertently perform work that takes longer than expected to complete.  These delays can add up over the course of a page’s many resource requests.

The platform already exposes the [`PerformanceResourceTiming`][0] interface to help sites quantify network performance.  Historically this interface has focused on resources loaded from the network or http cache.  The `workerStart` attribute was added to measure the time required to start the service worker thread.  Beyond that, however, there is currently no information about the time required to process the `FetchEvent` in the service worker.

This leaves some large, complex sites in the unfortunate situation where they encounter some amount of regression in page load times with service workers enabled, but no clear way to diagnose the problem.

The API proposed in the document will provide additional insight into `FetchEvent` processing time for each resource request.

## Goals

**Goal 1:** Help large, complex sites build high quality service worker scripts with efficient `FetchEvent` handlers.

**Goal 2:** Help sites provide better bug reports to browsers about `FetchEvent` performance problems in the platform.

## Non-Goals

Service workers support many different features in addition to the `FetchEvent`.  This proposal is focused solely on the `FetchEvent`.  Measuring the behavior of other features is explicitly a non-goal.

## Getting Started / Example Code

Consider an html document that contains an image element.  The page also runs a script on load that reports how long the image took to load.

```html
<!-- example document -->
<html>
  <body>
    <img src="foo.jpg”/>
  </body>
  <script>
    function onLoad() {
      let imageURL = new URL("foo.jpg", self.location).href;
      let entry = performance.getEntriesByName(imageURL)[0];
      let record = {
        fetchStart: entry.fetchStart,
        workerStart: entry.workerStart,
        requestStart: entry.requestStart,
        responseStart: entry.responseStart,
        responseEnd: entry.responseEnd,
      };
      reportLoadTime(record);
    }
    self.addEventListener("load", onLoad, { once: true });
  </script>
</html>
```

If this document is loaded while controlled by a service worker, then the browser will dispatch a `FetchEvent` on the service worker thread.

```javascript
// service worker script
self.addEventListener(fetchEvent => {
  fetchEvent.respondWith(async _ => {
    let strategy = await getStrategy(fetchEvent.request.url);
    let response;
    if (strategy !== "network-only") {
      response = await caches.match(fetchEvent.request);
      if (response) {
        return response;
      } else if (strategy === "cache-only") {
        return new Response("error");
      }
    }
    return fetch(fetchEvent.request);
  }());
});
```

This service worker performs multiple actions for each `FetchEvent`.  First it determines a strategy for handling the event.  This is done asynchronously to represent potentially reading a manifest from `IndexedDB`, etc.  Next the handler implements a cache-only, network-only, or fallback behavior depending on which strategy was chosen.

With the proposed API the service worker script would be modified to mark the time at each step.  This is done by first using the existing [User Timing API][3] to create `PerformanceEntry` objects and then using the new API to associate these entries with the `FetchEvent`.

```javascript
// service worker script
self.addEventListener(fetchEvent => {
  fetchEvent.respondWith(async _ => {
    fetchEvent.addPerformanceEntry(mark("strategyLookupStart"));
    let strategy = await getStrategy(fetchEvent.request.url);
    fetchEvent.addPerformanceEntry(mark("strategyLookupEnd"));

    let response;
    if (strategy !== "network-only") {
      fetchEvent.addPerformanceEntry(mark("offlineCacheStart"));
      response = await caches.match(fetchEvent.request);
      fetchEvent.addPerformanceEntry(mark("offlineCacheEnd"));
      if (response) {
        return response;
      } else if (strategy === "cache-only") {
        return new Response("error");
      }
    }
    fetchEvent.addPerformanceEntry(mark("networkFetchStart"));
    return fetch(fetchEvent.request);
  }());
});

// The performance entry could be generated in a few different ways.
function mark(name) {
  // User Timing Level 2
  performance.mark(name);
  let entries = performance.getEntriesByName(name);
  return entries[entries.length - 1];
  
  // User Timing Level 3
  // return performance.mark(name);
  
  // Or if we could use a constructor:
  // return new PerformanceMark(name);
}
```

The page would then be able to easily read and report these values.

```html
<!-- example document -->
<html>
  <body>
    <img src="foo.jpg”/>
  </body>
  <script>
    function onLoad() {
      let imageURL = new URL("foo.jpg", self.location).href;
      let entry = performance.getEntriesByName(imageURL)[0];
      let record = {
        startTime: entry.startTime,
        fetchStart: entry.fetchStart,
        workerStart: entry.workerStart,
        requestStart: entry.requestStart,
        responseStart: entry.responseStart,
        responseEnd: entry.responseEnd,
      };

      // Record the PerformanceEntry values associated via
      // FetchEvent.addPerformanceEntry() in the service worker.
      // script.
      for (let workerEntry in entry.workerTiming) {
        record[workerEntry.name] = workerEntry.startTime;
      }
      reportLoadTime(record);
    }
    self.addEventListener("load", onLoad, { once: true });
  </script>
</html>
```

The site can now distinguish between different possible reasons for poor loading performance.  For example, slow strategy determination due to manifest loading can be separated from poor network conditions.

In addition, the site can observe the actual paths taken through the service worker to identify configuration problems.  If all requests are going to the network, perhaps there is a problem with the code which populates the offline cache, etc.

## Key Scenarios

### Scenario 1: Measuring Navigations

The performance of the initial navigation request for a page is a critical component in the overall load time.  Any delays block further loading work and directly translate to the bottom line.  Therefore, it's important to be able to measure service worker impacts on navigation requests.

The browser exposes navigation timing via the [`PerformanceNavigationTiming`][2] interface.  Since this interface extends `PerformanceResourceTiming` the proposed `workerTiming` attribute will be automatically exposed for navigations as well.

The example code previously shown in this document should work for both subresource and navigation requests.

### Scenario 2: Duplicate Requests

While many sites only load each resource once, it is certainly possible to load the same URL multiple times.  For example, the site may be using a REST API that they need to invoke multiple times.  The performance of these requests should be tracked separately unambiguously.

Since the `FetchEvent.addPerformanceEntry()` method directly targets a specific PerformanceResourceTiming object duplicate requests are easily distinguished.  The API for access `PerformanceResourceTiming` objects already supports duplicate requests in a standard way.

The example code previously shown in this document should work for sites that load the same URL multiple times.

### Scenario 3: Measuring a ReadableStream Body

While service workers will often produce a `Response` using `fetch()` or `caches.match()`, they may also programmatically create a synthetic `Response`.  The service worker can even dynamically generate the body data by using a `ReadableStream`.  It should be possible measure this kind of `ReadableStream` body generation.

To support this scenario `FetchEvent.addPerformanceEntry()` can be called up until the last `waitUntil()` or `respondWith()` promise is settled.

A service worker implementing this case might look like this:

```javascript
// service worker script
self.addEventListener(fetchEvent => {
  let endWaitUntil;
  fetchEvent.waitUntil(
    new Promise(resolve => endWaitUntil = resolve));

  let count = 0;
  let body = new ReadableStream({
    pull: async controller => {
      count += 1;
      fetchEvent.addPerformanceEntry(mark(`startComputeChunk-${count}`));
      let chunk = await computeNextChunk();
      fetchEvent.addPerformanceEntry(mark(`endComputeChunk-${count}`));
      if (chunk) {
        controller.enqueue(chunk);
      } else {
        fetchEvent.addPerformanceEntry(mark(`dynamicBodyComplete`));
        controller.close();
        // fetchEvent.addPerformanceEntry() will start being ignored
        // after the waitUntil() promise is resolved.
        endWaitUntil();
      }
    }
  });
  fetchEvent.respondWith(new Response(body));
});
```

## Detailed Design Discussion

### Issue 1: Timing-Allow-Origin Considerations

One of the primary safety mechanisms on the web is the same-origin policy (SOP).  This policy allows sites to load some cross-origin resources, but prevents script from inspecting the content of cross-origin resources.  Sites that want to allow their resources to be read cross-origin can opt-in to the behavior via CORS headers.

Even if content is hidden from cross-origin script, it is still possible for a site to infer information based on the timing of the load.  Because of this some `PerformanceResourceTiming` information is blocked unless the resource is served with the `Timing-Allow-Origin` header.

In the case of FetchEvent Worker Timing the `Timing-Allow-Origin` header is not necessary.  Service workers are only allowed to control pages of the same origin.  `FetchEvent` objects may be dispatched for cross-origin subresources, but the service worker can only decorate the load with same-origin information.  The service worker does not have any additional access to cross-origin information.  Also, any information produced by the service worker can already be shared with the page via `postMessage()`, `IndexedDB`, or other mechanisms.

Because of the strict same-origin design of service workers we do not believe this feature needs to be guarded by `Timing-Allow-Origin`.

### Issue 2: Late FetchEvent.addPerformanceEntry() Calls

`FetchEvent` objects have a natural life cycle based on how the network request is satisfied.  For example, if the handler does not call `respondWith()` synchronously then the request is allowed to fall back to network.  Or the handler could pass a `Response` and the browser could read the body.  In both cases the `FetchEvent` essentially loses the ability to effect the request any more.

The `FetchEvent`, however, does provide a `waitUntil()` method which can be used to perform extended actions after the `respondWith()` promise has been resolved.  Since these actions are still associated with the `FetchEvent` we would like to make the `addPerformanceEntry()` API available during this extended time period.

To support this we propose that `addPerformanceEntry()` function normally up until the last `FetchEvent.waitUntil()` or `FetchEvent.respondWith()` promise is settled.  After that point calls will be ignored.

In order to support reading these late values we also propose that the `PerformanceObserver` notification for intercepted requests be delayed until their last `waitUntil()` promise is settled.  If this is considered too breaking, this behavior could be made an opt-in feature.

### Issue 3: Metadata Limits

The FetchEvent Worker Timing API is essentially a per-request information side-channel.  It’s conceivable that some sites might try to pass unexpected data through this channel.  For example, passing large json data structures for the mark names.

We could try to place restrictions on the size of names and the number of entries that could be created for each FetchEvent.  It turns out, however, that User Timing level 3 is likely to propose an API to pass arbitrary metadata along with a mark or measure.  There is already interest from both web developers and implementers for passing this kind of information through User Timing.

Since the FetchEvent Worker Timing API is based on User Timing it would make sense to adopt this metadata API when it is implemented in User Timing.  Given this likely outcome it seems unnecessary to impose limits on the proposed API to block passing larger amounts of data.

## Issue 4: Redirects

Currently its possible to measure some information about redirected navigation and subresource loads via their resulting `PerformanceResourceTiming` entry.  This information is protected by the `Timing-Allow-Origin` check.  While this same restriction may not apply to the service worker, we still need to handle redirects cautiously.

In the case of a subresource redirecting we can simply allow all `FetchEvent` related entries to be accumulated on the `PerformanceResourceTiming` object.  The subresource may trigger multiple `FetchEvent` dispatches, but they will always be sent to the same service worker.  Preserving the previous entries from this same service worker is not a problem.

For a navigation, however, redirects may trigger `FetchEvent` dispatches on different service workers.  In addition, some of these may be on completely different origins.  At a minimum any previous `FetchEvent` worker timing entries must be cleared on a cross-origin navigation redirect.  To be conservative we could clear these entries on all navigation redirects to start.

## Issue 5: Privacy Considerations

Another important issue to consider is whether this new API allows sites to collect more information about the user than they currently can.  Typically this concern is focused on the capabilities of "3rd party" sites in the form of cross-origin iframes to collect information.  Data collection can take the form of explicitly exposed information or information that can be passively inferred.  For example, two well known class of problems are information side channels due to precise timing and device fingerprinting.

In this case we feel this API does not produce any additional privacy concerns.  As mentioned in the Timing-Allow-Origin section above, the FetchEvent Worker Timing is make it more convenient to access and organize information that is already available to the site.  It only allows the service worker to associate data with a FetchEvent, but does not expose additional information not already available.  While this data will often be timing related, those times are already available and the precision of the values will not be increased by this API.

Therefore at this time we believe this API should not worsen privacy for the user.

## Considered Alternatives

### Alternative 1: Combining Results on the Server

The most basic alternative is to not try to connect service worker performance directly to resources on the client.  Instead, the page and service worker script could send results to a server-side reporting service to "stitch" the performance data back together.

While this is a valid approach, it is not always reasonable to implement.  In particular, it tends to be easier when there is a unique identifier, such as for a logged in user, that can be used to associated records on the server.  For CDNs and analytics providers this kind of identifier is often not available.

In addition, the only piece of common data between the FetchEvent in the service worker and the request is the URL.  This provides some ability to match records up, but its difficult to precisely say which FetchEvent was associated with which request.

We have been moving more information into the client to help reduce the need for complex server data combination.  For example, the [Server Timing][4] specification now allows server-side performance data to be included in the `PerformanceResourceTiming` object in the browser.

The goal is to give more sites accurate performance data for their entire stack.  The FetchEvent Worker Timing proposal builds on the Server Timing approach to further this goal.

### Alternative 2: Client.postMessage()

The main alternative to this proposal is the existing `Client.postMessage()` API.  The idea is that the service worker script would track timing information in its own data structures and then `postMessage()` the information to the page.  The page would then merge this information with the `PerformanceResourceTiming` data.

There are a couple issues with this approach.

First, browsers currently do not expose a Client id for navigation requests.  So it's not possible to get a handle to the Client to call `postMessage()` for navigations.  Browsers are starting to implement FetchEvent.resultingClientId now, so this problem should be resolved in the future.

Second, it is difficult to track data for duplicate requests with this approach.  It may not be clear with PerformanceResourceTiming object corresponds to the MessageEvent data from the service worker.  This makes it difficult to build general purpose measurement systems.

Third, this approach imposes additional costs on the site in order to measure other code.  Additional javascript must be loaded in both the service worker script and the main page.  It also requires running additional tasks on the main thread to receive the MessageEvent.  While these costs are not excessive, it would be nice to avoid them if possible.

### Alternative 3: Service Worker Sets Server-Timing Headers

The [Server Timing][4] spec allows servers to include a header that will then result in a value being exposed on the `PerformanceResourceTiming` object.  One alternative would be to require the service worker to craft these headers and attach them to the `Response`.

The main problem with this is that it only allows the service worker to measure cases where there is a `Response` object.  Its possible for the service worker to not call `FetchEvent.respondWith()` at all and fallback to the browser's default network request.  In this case there is no `Response` object to add headers to.  Ideally any measurement API should support measuring this case as well.

Also, the [Fetch API][Fetch] allows headers to be added to a `Response` object, but it does not make it very ergonomic.  Since the `Response` from a `fetch()` or `cache.match()` is immutable the service worker would have to create a new synthetic `Response` in order to add headers.  This in turn will lose some information, such as the response URL, from the original `Response`.

Finally, it seems sub-optimal to require the service worker to serialize data to a header string just for the browser to turn around and parse it immediately.  Ideally a performance measurement mechanism should have as little overhead as possible.  Using headers to communicate from the service worker would add unnecessary overhead.

### Alternative 4: Timing Attributes Based on Response Source

Another alternative would be to automatically record some timing information on the `Response object` based on its source.  For example, a Response produced from the Cache API might record when the database lookup started and ended.  These values could then be placed on the `PerformanceResourceTiming` object in specified attributes.

For example, this service worker:

```javascript
self.addEventListener("fetch", fetchEvent => {
  fetchEvent.respondWith(caches.match(fetchEvent.request));
});
```

Might produce values like this on the main thread:

```javascript
let entry = performance.getEntriesByName(url)[0];
let cacheElapsed = entry.cacheMatchEnd - entry.cacheMatchStart;
```

The main downside of this alternative is that it only provides information about the particular `Response` that was passed to the `FetchEvent`.  If the service worker script does anything beyond a simple cache or pass-through operation then it will miss important information.

For example, if a service worker implements a fallback strategy then this alternative will provide information about how long it took to `fetch()` the network `Response`.  It will completely ignore the time required to access the Cache API to determine the resource was not available locally.

Given that we need to help sites with large complex service worker scripts we chose to forego this alternative approach.

## References & Acknowledgements

* https://w3c.github.io/ServiceWorker/
* https://w3c.github.io/resource-timing/#sec-performanceresourcetiming
* https://w3c.github.io/user-timing/#extensions-performance-interface
* https://w3c.github.io/navigation-timing/#sec-PerformanceNavigationTiming
* https://docs.google.com/document/d/1hltt8z9C4PaI5Qu1YMIp1wOGdbJPJPoJwBarEeCY6xQ/edit?usp=sharing
* https://w3c.github.io/server-timing/
* https://fetch.spec.whatwg.org/

Thank you to Ralph Chelala, Tim Dresser, Aaron Nelson, Dan Seminara, Jimmy Shen, Yoav Weiss, and Clay Woolam for reviewing earlier drafts of this proposal.

[ServiceWorker]:https://w3c.github.io/ServiceWorker/
[FetchEvent]:https://w3c.github.io/ServiceWorker/#fetchevent
[HandleFetch]:https://w3c.github.io/ServiceWorker/#on-fetch-request-algorithm
[0]:https://w3c.github.io/resource-timing/#sec-performanceresourcetiming
[1]:https://w3c.github.io/user-timing/#extensions-performance-interface
[2]:https://w3c.github.io/navigation-timing/#sec-PerformanceNavigationTiming
[3]:https://docs.google.com/document/d/1hltt8z9C4PaI5Qu1YMIp1wOGdbJPJPoJwBarEeCY6xQ/edit?usp=sharing
[4]:https://w3c.github.io/server-timing/
[Fetch]:https://fetch.spec.whatwg.org/
