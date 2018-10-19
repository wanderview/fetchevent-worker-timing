FetchEvent Worker Timing Explained

# What’s All This Then

This document proposes a new web platform API that allows a service worker script to record timing information associated with a particular FetchEvent.  This timing information then becomes available on the associated network request’s PerformanceResourceTiming object.

When the browser navigates or loads a subresource for a page it may dispatch a FetchEvent to a service worker.  This allows the service worker script to dynamically determine how that navigation or subresource request is handled.  This provides the site the flexibility to dynamically handle poor network conditions or other situations.

This flexibility, however, can come at a cost.  Dispatching an event to a javascript thread while fetching a network request adds some overhead.  This overhead may vary based on browser and device characteristics.  In addition, the service worker script itself may inadvertently perform work that takes longer than expected to complete.  These delays can add up over the course of a page’s many resource requests.

The platform already exposes the `PerformanceResourceTiming` interface [0] to help sites quantify network performance.  Historically this interface has focused on resources loaded from the network or http cache.  The `workerStart` attribute was added to measure the time required to start the service worker thread.  Beyond that, however, there is currently no information about the time required to process the FetchEvent in the service worker.

This leaves some large, complex sites in the unfortunate situation where they encounter some amount of regression in page load times with service workers enabled, but no clear way to diagnose the problem.

The API proposed in the document will provide additional insight into FetchEvent processing time for each resource request.

# Goals

**Goal 1:** Help large, complex sites build high quality service worker scripts with efficient FetchEvent handles.

**Goal 2:** Help sites provide better bug reports to browsers about FetchEvent performance problems in the platform.

# Non-Goals

Service workers support many different features in addition to the FetchEvent.  This proposal is focused solely on the FetchEvent.  Measuring the behavior of other features is explicitly a non-goal.

# Getting Started / Example Code

Consider an html document that contains an image element.  The page also runs a script on load that reports how long the image took to load.

<!-- example document -->

```
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

If this document is loaded while controlled by a service worker, then the browser will dispatch a FetchEvent on the service worker thread.

```
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

This service worker performs multiple actions for each FetchEvent.  First it determines a strategy for handling the event.  This is done asynchronously to represent potentially reading a manifest from IDB, etc.  Next the handler implements a cache-only, network-only, or fallback behavior depending on which strategy was chosen.

With the proposed API the service worker script would be modified to mark the time at each step.  This is done using an API similar to the User Timing API [1] exposed on the FetchEvent.

```
// service worker script
self.addEventListener(fetchEvent => {
  fetchEvent.respondWith(async _ => {
    fetchEvent.performanceMark("strategyLookupStart");
    let strategy = await getStrategy(fetchEvent.request.url);
    fetchEvent.performanceMark("strategyLookupEnd");

    let response;
    if (strategy !== "network-only") {
      fetchEvent.performanceMark("offlineCacheStart");**
      response = await caches.match(fetchEvent.request);
      fetchEvent.performanceMark("offlineCacheEnd");**
      if (response) {
        return response;
      } else if (strategy === "cache-only") {
        return new Response("error");
      }
    }
    fetchEvent.performanceMark("networkFetchStart");**
    return fetch(fetchEvent.request);
  }());
});
```

The page would then be able to easily read and report these values.

```
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

      // Record the PerformanceEntry values created by using
      // FetchEvent.performanceMark() in the service worker
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

# Key Scenarios

## Scenario 1: Measuring Navigations

The performance of the initial navigation request for a page is a critical component in the overall load time.  Any delays block further loading work and directly translate to the bottom line.  Therefore, it's important to be able to measure service worker impacts on navigation requests.

The browser exposes navigation timing via the PerformanceNavigationTiming interface [2].  Since this interface extends PerformanceResourceTiming the proposed workerTiming attribute will be automatically exposed for navigations as well.

The example code previously shown in this document should work for both subresource and navigation requests.

## Scenario 2: Duplicate Requests

While many sites only load each resource once, it is certainly possible to load the same URL multiple times.  For example, the site may be using a REST API that they need to invoke multiple times.  The performance of these requests should be tracked separately unambiguously.

Since the FetchEvent.performanceMark() method directly populates a specific PerformanceResourceTiming object duplicate requests are easily distinguished.  The API for access PerformanceResourceTiming objects already supports duplicate requests in a standard way.

The example code previously shown in this document should work for sites that load the same URL multiple times.

## Scenario 3: Measuring a ReadableStream Body

While service workers will often produce a Response using fetch() or caches.match(), they may also programmatically create a synthetic Response.  The service worker can even dynamically generate the body data by using a ReadableStream.  It should be possible measure this kind of ReadableStream body generation.

To support this scenario FetchEvent.performanceMark() can be called up until the responseEnd value is recorded for the request.

A service worker implementing this case might look like this:

```
// service worker script
self.addEventListener(fetchEvent => {
  let endWaitUntil;
  fetchEvent.waitUntil(
    new Promise(resolve => endWaitUntil = resolve));

  let count = 0;
  let body = new ReadableStream({
    pull: async controller => {
      count += 1;
      fetchEvent.performanceMark(`startComputeChunk-${count}`);
      let chunk = await computeNextChunk();
      fetchEvent.performanceMark(`endComputeChunk-${count}`);
      if (chunk) {
        controller.enqueue(chunk);
      } else {
        fetchEvent.performanceMark(`dynamicBodyComplete`);
        controller.close();
        fetchEvent.performanceMark will start being ignored
        // after the waitUntil() promise is resolved.
        endWaitUntil();
      }
    }
  });
  fetchEvent.respondWith(new Response(body));
});
```

# Detailed Design Discussion

## Issue 1: Timing-Allow-Origin Considerations

One of the primary safety mechanisms on the web is the same-origin policy (SOP).  This policy allows sites to load some cross-origin resources, but prevents script from inspecting the content of cross-origin resources.  Sites that want to allow their resources to be read cross-origin can opt-in to the behavior via CORS headers.

Even if content is hidden from cross-origin script, it is still possible for a site to infer information based on the timing of the load.  Because of this some PerformanceResourceTiming information is blocked unless the resource is served with the Timing-Allow-Origin header.

In the case of FetchEvent Worker Timing the Timing-Allow-Origin header is not necessary.  Service workers are only allowed to control pages of the same origin.  FetchEvents may be dispatched for cross-origin subresources, but the service worker can only decorate the load with same-origin information.  The service worker does not have any additional access to cross-origin information.  Also, any information produced by the service worker can already be shared with the page via postMessage(), IDB, or other mechanisms.

Because of the strict same-origin design of service workers we do not believe this feature needs to be guarded by Timing-Allow-Origin.

## Issue 2: Late FetchEvent.performanceMark() Calls

FetchEvent objects have a natural life cycle based on how the network request is satisfied.  For example, if the handler does not call respondWith() synchronously then the request is allowed to fall back to network.  Or the handler could pass a Response and the browser could read the body.  In both cases the FetchEvent essentially loses the ability to effect the request any more.

The FetchEvent, however, does provide a waitUntil() method which can be used to perform extended actions after the respondWith() promise has been resolved.  Since these actions are still associated with the FetchEvent we would like to make the performanceMark() API available during this extended time period.

To support this we propose that performanceMark() function normally up until the last FetchEvent.waitUntil() promise is settled.  After that point calls will be ignored.

In order to support reading these late values we also propose that the PerformanceObserver notification for intercepted requests be delayed until their last waitUntil() promise is settled.  If this is considered too breaking, this behavior could be made an opt-in feature.

## Issue 3: Metadata Limits

The FetchEvent Worker Timing API is essentially a per-request information side-channel.  It’s conceivable that some sites might try to pass unexpected data through this channel.  For example, passing large json data structures for the mark names.

We could try to place restrictions on the size of names and the number of entries that could be created for each FetchEvent.  It turns out, however, that User Timing level 3 is likely to propose an API to pass arbitrary metadata along with a mark or measure.  There is already interest from both web developers and implementers for passing this kind of information through User Timing.

Since the FetchEvent Worker Timing API is based on User Timing it would make sense to adopt this metadata API when it is implemented in User Timing.  Given this likely outcome it seems unnecessary to impose limits on the proposed API to block passing larger amounts of data.

# Considered Alternatives

## Alternative 0: Combining Results on the Server

The most basic alternative is to not try to connect service worker performance directly to resources on the client.  Instead, the page and service worker script could send results to a server-side reporting service to "stitch" the performance data back together.

While this is a valid approach, it is not always reasonable to implement.  In particular, it tends to be easier when there is a unique identifier, such as for a logged in user, that can be used to associated records on the server.  For CDNs and analytics providers this kind of identifier is often not available.

In addition, the only piece of common data between the FetchEvent in the service worker and the request is the URL.  This provides some ability to match records up, but its difficult to precisely say which FetchEvent was associated with which request.

We have been moving more information into the client to help reduce the need for complex server data combination.  For example, the Server Timing specification [4] now allows server-side performance data to be included in the PerformanceResourceTiming object in the browser.

The goal is to give more sites accurate performance data for their entire stack.  The FetchEvent Worker Timing proposal builds on the Server Timing approach to further this goal.

## Alternative 1: Client.postMessage()

The main alternative to this proposal is the existing Client.postMessage() API.  The idea is that the service worker script would track timing information in its own data structures and then postMessage() the information to the page.  The page would then merge this information with the PerformanceResourceTiming data.

There are a couple issues with this approach.

First, browsers currently do not expose a Client id for navigation requests.  So it's not possible to get a handle to the Client to call postMessage() for navigations.  Browsers are starting to implement FetchEvent.resultingClientId now, so this problem should be resolved in the future.

Second, it is difficult to track data for duplicate requests with this approach.  It may not be clear with PerformanceResourceTiming object corresponds to the MessageEvent data from the service worker.  This makes it difficult to build general purpose measurement systems.

Third, this approach imposes additional costs on the site in order to measure other code.  Additional javascript must be loaded in both the service worker script and the main page.  It also requires running additional tasks on the main thread to receive the MessageEvent.  While these costs are not excessive, it would be nice to avoid them if possible.

## Alternative 2: Timing Attributes Based on Response Source

Another alternative would be to automatically record some timing information on the Response object based on its source.  For example, a Response produced from the Cache API might record when the database lookup started and ended.  These values could then be placed on the PerformanceResourceTiming object in specified attributes.

For example, this service worker:

```
self.addEventListener("fetch", fetchEvent => {
  fetchEvent.respondWith(caches.match(fetchEvent.request));
});
```

Might produce values like this on the main thread:

```
let entry = performance.getEntriesByName(url)[0];
let cacheElapsed = entry.cacheMatchEnd - entry.cacheMatchStart;
```

The main downside of this alternative is that it only provides information about the particular Response that was passed to the FetchEvent.  If the service worker script does anything beyond a simple cache or pass-through operation then it will miss important information.

For example, if a service worker implements a fallback strategy then this alternative will provide information about how long it took to fetch() the network Response.  It will completely ignore the time required to access the Cache API to determine the resource was not available locally.

Given that we need to help sites with large complex service worker scripts we chose to forego this alternative approach.

# References & Acknowledgements

[0]: [https://w3c.github.io/resource-timing/#sec-performanceresourcetiming](https://w3c.github.io/resource-timing/#sec-performanceresourcetiming)

[1]: [https://w3c.github.io/user-timing/#extensions-performance-interface](https://w3c.github.io/user-timing/#extensions-performance-interface)

[2]: [https://w3c.github.io/navigation-timing/#sec-PerformanceNavigationTiming](https://w3c.github.io/navigation-timing/#sec-PerformanceNavigationTiming)

[3]: [https://docs.google.com/document/d/1hltt8z9C4PaI5Qu1YMIp1wOGdbJPJPoJwBarEeCY6xQ/edit?usp=sharing](https://docs.google.com/document/d/1hltt8z9C4PaI5Qu1YMIp1wOGdbJPJPoJwBarEeCY6xQ/edit?usp=sharing)

[4]: [https://w3c.github.io/server-timing/](https://w3c.github.io/server-timing/)
