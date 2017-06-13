# Pragmatist's Guide to Service Workers

In this repository, you'll find code examples to illustrate the _Pragmatist's Guide_ presentation, given at Smashing Conference, New York City, 2017.

## Install

Clone this repository, then:

`npm install`

To run a local web server and see the examples in action, run:

`npm start`

You'll want to use a browser that supports Service Worker; I use Chrome but SW is also supported in Firefox and Opera to differing extents.

_Note_: These examples, run locally, rely on the `localhost` exception to the SSL/TLS requirement for Service Workers.

## Examples

1. [Mission 1](missions/01-offline-message): _Offline message_ — Respond to `navigation` requests (i.e. requests for HTML documents). Try to fetch from the network, but if that fails, return a `Response` object that emulates a simple web page with an `Oh, dear` message.
1. [Mission 2](missions/02-offlie-page): _Offline page_ — Same as before, but instead of returning a self-made Response on fetch failure, cache an offline HTML page during the `install` phase and return _that_ upon fetch failure.
1. [Mission 3](missions/03-network-strategies): _Network strategies_ — Respond to fetches for content (HTML) and static assets (images) differently. Use a _network-first_ strategy for content and a _cache-first_ strategy for images.
1. [Mission 4](missions/04-application-shell): _Application shell_ — During the `install` phase, cache a bunch of static assets that we consider to be our application's "shell". Respond to fetches and look in the cache _first_ for requests for those assets. Once the service worker is installed, you can go offline and continue to "request clouds" (images of clouds).
1. [Mission 5](missions/05-versioning): _Cache naming and cleanup_ — Use cache-prefixing to manage versioning of a service worker and, during the `activate` phase, clean up (old) caches that don't match the new cache prefix. To version-bump, you'd want to change the `cachePrefix` value.
1. [Mission 6](missions/06-fancypants): _Fancypants_ — Add a fallback offline image and use a JSON file as a source of URLs to pre-cache during `install`.

## Resources and References

### Web Workers and Related APIs

Service Worker is a type of Web Worker. Web Workers are able to execute on a background thread, staying out of the way of the main execution thread of the browser. Other APIs in the Service Worker/Web Worker universe include:

* [Notifications API](https://developer.mozilla.org/en-US/docs/Web/API/Notifications_API)
* [Push API](https://developer.mozilla.org/en-US/docs/Web/API/Push_API)
* [BackgroundSync](https://wicg.github.io/BackgroundSync/spec/)
* [Channel Messaging API](https://developer.mozilla.org/en-US/docs/Web/API/Channel_Messaging_API)

A Web Worker is created/instantiated by code running in one context, but they execute in a different context. Web Workers consist of a JavaScript file.

### Scopes and Contexts

Service Workers execute in a different _context_ than a web page in a window or tab.

Each window or tab in your browser is a unique _browsing context_ ([MDN glossary](https://developer.mozilla.org/en-US/docs/Glossary/Browsing_context), [HTML specification](https://html.spec.whatwg.org/multipage/browsers.html#windows)). JavaScript in an HTML document executes within the scope of a browsing context. The global scope object in a document in a browsing context is `window`.

Service Workers, like other Web Workers, execute in a separate context from the clients (web pages in browsing contexts, e.g.) that they control. The global scope object in a Service Worker is ServiceWorkerGlobalScope ([MDN](https://developer.mozilla.org/en-US/docs/Web/API/ServiceWorkerGlobalScope), [Spec](https://www.w3.org/TR/service-workers-1/#service-worker-global-scope))

Some of the methods and properties available in `ServiceWorkerGlobalScope` that the presentation takes advantage of include (but are not limited to!):

* `fetch(request)`
* `skipWaiting()`
* `clients` (`clients.claim()` specifically)
* `caches` (`CacheStorage` reference)
* `Request` and `Response` constructors

### Is Service Worker Ready?

A handy-dandy chart of what's shipping where (thanks, Jake!).

[Is Service Worker Ready?](https://jakearchibald.github.io/isserviceworkerready/)

### ServiceWorkerContainer

`navigator.serviceWorker` is a ServiceWorkerContainer ([MDN](https://developer.mozilla.org/en-US/docs/Web/API/ServiceWorkerContainer), [spec](https://www.w3.org/TR/service-workers-1/#service-worker-container)). It:

> ...provides an object...including facilities to register, unregister and update service workers, and access the state of service workers and their registrations.

To register a ServiceWorker, the `navigator.serviceWorker.register(scriptURL, options)` method is used within client code.

### Service Worker Scope

When a client registers a Service Worker, it does so against a _scope_, which is a path or pattern within which the Service Worker can listen for `fetch` events. A service worker cannot respond to fetches outside of its scope.

Scope can be provided as a second (String) argument to [`ServiceWorkerContainer.register(scriptURL, options)`](https://developer.mozilla.org/en-US/docs/Web/API/ServiceWorkerContainer/register). If not provided, scope defaults to `./`, relative to the script's location.

A Service Worker may not have a scope above itself in the directory hierarchy (e.g. `../` or `/` if the Service Worker is not at the top level itself) unless you use a `Service-Worker-Allowed` header.

### FetchEvent

When the browser requests a resource that falls within an active Service Worker's scope, a fetch  event is dispatched on the Service Worker and the SW may listen for it.

`fetch` event _handlers_ are invoked with a [`FetchEvent`](https://developer.mozilla.org/en-US/docs/Web/API/FetchEvent). `FetchEvent` objects contain two very useful things:

* a `Request` object (`fetchEvent.request`) containing many details about the request in question
* a `respondWith()` method that allows the SW to respond to the fetch with its own `Response`

If a Service Worker uses the `fetchEvent.respondWith()` method, it should provide a `Response` or a `Promise` that will resolve to a `Response`. That is, the browser spits out a request and is looking for a response in return.

### fetch API

The [`Fetch` API](https://developer.mozilla.org/en-US/docs/Web/API/Fetch_API) provides an interface for fetching resources from the network. It is similar to `XMLHttpRequest` in ambition.

`fetch(request)` returns a `Promise` that resolves to a `Response` if the fetching is successful.

#### Request

A [Request](https://developer.mozilla.org/en-US/docs/Web/API/Request) is chock full of info about a request for a resource. You can create a `Request` object using the [Request constructor](https://developer.mozilla.org/en-US/docs/Web/API/Request/Request), available in the SW's global scope. A typical instantiation of a `Request` passes a string representing the URL for what's being requested, e.g.:

```
const theRequest = new Request('foo.html');
```

More often, you'll be dealing with a pre-existing request inside of a `fetchEvent`, e.g., rather than creating your own. In this case, looking at details of the `fetchEvent.request` can help you figure out how to handle it. Examples in this presentation include:

* Looking at the request's `Accept` headers to see if the request is for an image (e.g.: `request.headers.get('Accept').indexOf('image') !== -1`)
* Checking `request.mode`: it's value will be `navigate` if this is a request for a web document/page

##### `request.mode` polyfill

`request.mode` isn't supported absolutely everywhere yet. An equivalent check is:

```
request.method === 'GET' && request.headers.get('Accept').includes('text/html')
```

#### Response

A [Response](https://developer.mozilla.org/en-US/docs/Web/API/Response) represents, unsurprisingly, a _response_ to a request.

You can instantiate a `Response` object by using the [`Response` constructor](https://developer.mozilla.org/en-US/docs/Web/API/Response/Response). The constructor takes two arguments: `body`, the response's body, and `init`, which is a confusingly-named catch-all settings argument.

In one presentation example, a Response is created that "looks like a web page". It does this like so:

```
new Response('<p>Oh, Dear!</p>',
  { headers: { 'Content-Type': 'text/html' } });
```

The `body` is a chunk of HTML, and setting a `Content-Type` header to `text/html` makes the browser treat the response as an HTML document.

`Response.ok` is a Boolean that will be `true` if the Response's HTTP code was in the 200-299 range, i.e., it is OK and not an error.

`Response.clone()` creates a clone of the object to deal with the reality that a Response's body can only be used once. This allows the same Response to be given to the browser to use immediately _and_ be cached for later use.

### Promises

A [Promise](https://developers.google.com/web/fundamentals/getting-started/primers/promises) is an object that may produce a value at some point. We generally hope that it will _resolve_ to the type of value that we expect. A _pending_ promise can be _settled_ in one of two ways: it can be _fulfilled_, or resolved; or it can be _rejected_.

Chains of operations with Promises can be created using `Promise.prototype.then()` and `Promise.prototype.catch()`. `Promise.all(promises)` returns a Promise that will resolve if all of the Promises in `promises` resolve, or reject if any one of the Promises in `promises` rejects.

In one example in the presentation, a function given as a `catch` handler itself `Throw`s an error. A subsequent `catch` can be added to the promise chain to handle this, e.g.:

```
doThis().then(doThat).catch(() => {
  // blah blah
  throw Error('...');
}).catch(() => {
  // This function gets invoked if the previous promise rejects or
  // if it throws
});
```

### Lifecycle

A ServiceWorker has a `state` (the value of which can be: parsed, installing, installed, activating, activated or redundant).

When the browser downloads a new service worker file—and this can be an entirely new service worker or a _changed_ service worker file—the service worker is initially `parsed` but moves automatically into `installing` and is `installed` (the _install_ phase).

Later, the service worker moves through `activating` and `activated` states (_activate_ phase). When this happens depends on if the new service worker file is an update to a previously-activated service worker or an entirely new service worker.

If it's an entirely new service worker, it will move into the activate phase automatically after the install phase completes. If it's a changed/updated service worker, it will not do so until all of the clients controlled by the previous service worker have closed.

#### ExtendableEvent

The lifecycle events `install` and `activate` are both [`ExtendableEvent`s](https://developer.mozilla.org/en-US/docs/Web/API/ExtendableEvent).

`ExtendableEvent` objects have a `waitUntil(promise)` method that allow you to make the lifetime of the event "stretch out" until the `promise` provided resolves.

#### install

The _install_ lifecycle phase has an associated event, `install`. This phase is meant for setting the service worker up and pre-caching needed assets for later.

Using `skipWaiting()` at the end of an install handler will cause the service worker to move into activation immediately without having to wait for clients to close.

#### activate

The _activate_ lifecycle phase has an associated event, `activate`. This phase is meant for cleaning up after old versions of the service worker.

Using `clients.claim()` at the end of an activate handler will cause the service worker to take effect immediately without having to wait for clients to reload.

### The cache API

A [`cache`](https://developer.mozilla.org/en-US/docs/Web/API/Cache) is a map of `Request` - `Response` pairs. You can create as many caches as you like and name them whatever you please. These caches are distinct from the browser's built-in caching.

#### CacheStorage

The [`CacheStorage`](https://developer.mozilla.org/en-US/docs/Web/API/CacheStorage) interface serves as a directory for all available caches and is available as `caches` in ServiceWorkerGlobalScope.

* `caches.open('name')`: Returns a Promise that resolves to the Cache requested. If one doesn't exist by `name`, a new one will be created.
* `caches.delete(key)`: Returns a Promise that resolves to `true` if a Cache exists with name `key` and is successfully deleted. It will resolve to `false` if there is no Cache by that name.
* `caches.keys()`: Returns a Promise that resolves to an Array of keys (Strings) for every current Cache.

#### Putting things in cache (Cache objects)

Remember, access a Cache by using `caches.open(name)`.

* `cache.put(request, response)`: Add the given request-response pair to the Cache.
* `cache.add(requestOrURL)`: Fetch the Request given and store it and its resulting Response in the Cache.
* `cache.addAll(urls)`: Fetch all of the URLs in `urls` Array and store the resulting request-response pairs in the Cache.

### Finding things in caches

* `cache.match(requestOrURL)`: Look for a matching `Response` for `requestOrURL` in this Cache object (only this cache). You need to access the Cache first with `caches.open()`.
* `caches.match(requestOrURL)`: Look for a matching `Response` across _all_ Caches. Does not require opening a Cache first.

_Important Note_: Both `cache.match()` and `caches.match()` return a Promise that will resolve to `undefined` if a match is not found. The Promise will not reject. Philosophically, this is because an "answer" (result) _was_ obtained: the answer is that there isn't a match. Promises should only reject when there is a failure in actually obtaining an answer.

### Network Strategies

_Network strategies_ allow you to optimize both online and offline performance by avoiding network round-trips and responding with cached items when the user is offline.

#### Network-First Strategy

A network-first strategy is used for assets that change frequently and should be as fresh as possible. It takes the following steps when responding to a fetch event:

* Try to obtain a fresh copy of the resource from the network.
* If this is successful, store a copy of the resource in cache for potential later use before returning the response to the browser.
* If this fails, check to see if there is a cached copy of the resource and return that, if so.
* In the presentation, the network-first strategy has an additional fallback behavior: return an offline page response if the network is unavailable and there is no cached copy of the resource.

#### Cache-first

A cache-first strategy can be used for static assets that don't change much, reducing network usage. It takes the following steps when responding to a fetch event:

* Try to find the resource in cache.
* If that is successful, respond to the fetch with that cached resource.
* If there is no cached copy of the resource, fetch one from the network.
* If that is successful, cache a copy of the resource for later use before returning the response to the browser.

#### Read-through caching

* See also `Response.ok` and `Response.clone()`

_Read-through caching_ is the technique of caching items as they're fetched in fetch handlers for potential later use, either offline or for cache-first strategies.

### Application Shell

An _application shell_ is those static assets like icons, images, CSS, scripts that are used on every page or nearly every page of a web site or app.

Application shell resources can be pre-cached during service-worker install and subsequent fetches for these resources can be handled in a cache-first manner.

### Versioning

The _activate_ phase of a service worker can be used to clean up after old service worker versions.

One technique for versioning service workers involves using a unique _version string_ or _prefix_ every time you make changes to your service worker. This string can be prefixed to cache names to identify which version of a service worker a given cache is associated with.

When cleaning up after old caches, you can use the `caches.keys()` method to retrieve an Array of all current cache keys (Strings). An example in the presentation then used `Array.prototype.filter` to filter the set of keys down to those that don't match the current version string, that is, those that should be deleted. It then used `Array.prototype.map` to map each key-to-delete to `caches.delete(key)`, resulting in an Array of Promises for those delete operations.

### Clients Interface

The global property `clients` is an interface to the collection of clients (e.g. browsing contexts) that this service worker controls. Using `clients.claim()` at the end of an activate handler allows the newly-activated service worker to take control of its clients immediately instead of having to wait for a reload on each.
