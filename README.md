# FetchController-spec

A preliminary document outlining extensions to window.fetch for control and observation.

This is an attempt to consolidate most of the discussion in [whatwg/fetch#447][] one consistent actionable proposal.

[whatwg/fetch#447]: https://github.com/whatwg/fetch/issues/447
[whatwg/fetch#448]: https://github.com/whatwg/fetch/issues/448

This is the first time I've really taken a whack at writing a specification, so it's admittedly somewhat sloppy, and missing all the markdown of what sections are "normative" and [what sections are "informative"](https://twitter.com/geddski/status/814296358714605568) and all that fancy jazz that a proper WHATWG document would have.

## Terminology

For the purposes of this document, "a fetch" refers to all actions entailed with an in-flight request by the user agent, as instantiated by `fetch()` (or the browser plumbing as handed to a Service Worker via the `fetch` event), including the user agent's processing and handling of that request's response. (See also the preface to [whatwg/fetch#448][].)

## Constructed-or-revealing-constructor options

A recurring pattern in this specification. which I generally spell out in longer terms instead of referring to it by name (as I haven't given it a pithy one), is an option on a call/constructor that may take either a function that works as a revealing constructor (ie. an object is constructed and then passed to the given callback), or an object that has *already* been constructed (ie. by calling its constructor directly).

This design pattern is in place to address developer sentiments that one flow would be less natural with the way their code is factored over another. Accomodating both of these flows doesn't cause any conflicts in the specification's design, and, indeed, respecting *both* of them, *cooperatively*, allows for better factoring of implementations to *interoperate* according to whichever form factor makes more sense for their implementation. (For example, there are many points in this specification describing behaviors that should be followed when a function call employs *both* approaches *simultaneously*.)

## Two new classes

This proposal adds two new classes, `FetchController` and `FetchObserver`.

### `FetchObserver`

The `FetchObserver` class provides passive properties and methods to *observe the state* of a fetch, from its instantiation, through any upload and download traffic, up until the fetch's completion.

`FetchObserver` features an `addEventListener` method to listen to events from a fetch, such as `progressup`: it also features read-only Promise properties, such as `complete`, that resolve when the corresponding one-time event has fired (or reject in situations where it never will, such as when the fetch is aborted).

### `FetchController`

`FetchController` features methods to control a fetch, such as `abort()` to abort the fetch (rejecting any related promises pending from it). It also includes a read-only `.observer` property, containing an associated `FetchObserver` for monitoring the fetch (which may be constructed before the `FetchController` and associated on the `FetchController` object's construction, as described below).

## `FetchEvent.observer`

`FetchEvent` objects passed to Service Workers' `fetch` event listeners will include a read-only `FetchObserver` property that allows the lifecycle of the fetch to be observed (from the perspective of the client initiating the fetch being handled: see notes on "`FetchObserver` events" below).

## Two new `fetch()` options

The `window.fetch()` function's dictionary of options recognizes two new properties, `controller` and `observer`. A call to `fetch()` may specify none, one, or both of these properties, with either/both accepting either a pre-constructed object (using the constructors described below), or a revealing-constructor callback function (which will receive a constructed object associated with the fetch).

To be clear, these are the internal steps `fetch()` follows:

- Let "the fetch" be the fetch actions that a Controller and/or Observer will be associated with.
- If there is a value defined for the `observer` option:
  - If the `observer` option is a pre-constructed `FetchObserver` object:
    - If the `controller` option is a pre-constructed `FetchController` object, and the `.observer` property of that controller is not the `FetchObserver` defined by the `observer` option, throw a TypeError.
    - Else, let "the observer" be that `FetchObserver`.
  - Else, if the `observer` option is a function, let "the observer" be a newly-constructed `FetchObserver`.
  - Else, throw a TypeError.
- If there is a value defined for the `controller` option:
  - If the `controller` option is a pre-constructed `FetchController` object, let "the controller" be that `FetchController`.
  - Else, if the `controller` option is a function, let "the controller" be a newly-constructed `FetchController`:
    - If "the observer" has already been defined, then let the observer "the controller" is associated with on construction be "the observer".
    - Else, let "the observer" be the implicitly-constructed observer constructed with "the controller".
  - Else, throw a TypeError.
- Associate "the controller" and "the observer", if either are present, with "the fetch".
- If there is a function present as the `observer` option, call it with "the observer" as its argument.
- If there is a function present as the `controller` option, call it with "the controller" as its argument.
- Perform "the fetch", as defined in the steps of https://fetch.spec.whatwg.org/#fetching

While I've refrained from using this kind of "actual steps" dissection in the rest of this loose proposal, the steps above were the clearest way I could think to explain that "revealing constructors" in `fetch()` are actually called *after the objects in question have been constructed*, specifically so that they are associated with the fetch *before the callbacks* (to avoid footguns like code thinking it can associate the controller with a newly-instantiated fetch *within the revealing constructor callback*).

## Class constructors

The `FetchController` and `FetchObserver` classes may be constructed directly by calling their respective contructors.

### `new FetchObserver(options)`

Constructs a new `FetchObserver`, to be associated with a corresponding fetch and/or `FetchController` at some point after the `FetchObserver` object's construction.

The `options` object is only reserved for the purposes of future extension. This docuemnt specifies no options for the `FetchObserver` constructor at this time. (Note that any options that would be passed to this constructor, for the purposes of parity with revealing-constructor use cases, should only be considered if they may be mirrored as an option to `fetch()` and the standalone `new FetchController(options)` constructor.)

### `new FetchController(options)`

Constructs a new `FetchController`, for use with a fetch at some point after construction.

The only `options` property specified at this time is `observer`, which may take either a pre-constructed `FetchObserver` and associate it with the new `FetchController` as its `.observer` property, or a revealing-constructor function that will receive the newly-constructed `FetchObserver` associated with the `FetchController` as its `.observer` property.

## Properties and methods

### FetchObserver.pending, FetchController.pending

Read-only property. A boolean introspecting whether the `FetchController` or `FetchObserver` is associated with a fetch (`false` if associated, `true` if not).

This boolean is mostly only meaningful in the context of a pre-constructed `FetchController` or `FetchObserver` (for determining whether it has been used or not); for `FetchController` or `FetchObserver` objects constructed via revealing constructor or provided with a `FetchEvent`, this will always be `false`.

### FetchObserver.begin, FetchController.begin

Read-only property. A Promise that is resolved when the `FetchController` or `FetchObserver` is associated with a fetch, with the value of the object itself (the `FetchObserver` or `FetchController` that the `begin` Promise is a property of).

This Promise is mostly only meaningful in the context of a pre-constructed `FetchController` or `FetchObserver` (for taking action once it is used); for `FetchController` or `FetchObserver` objects constructed via revealing constructor or provided with a `FetchEvent`, this will always be resolved.

### FetchObserver.request, FetchController.request

Read-only property. The `Request` that instantiated the fetch that this `FetchObserver` observes and/or `FetchController` controls.

`FetchObserver.request` is admittedly redundant with the `request` property of whatever the `FetchObserver`'s parent may be (ie. a `FetchController` or `FetchEvent`): however, it's a sensible property to allow code that *only takes `FetchObserver` arguments* to operate normally.

### FetchController.fetch(options)

Per [this comment](https://github.com/whatwg/fetch/issues/447#issuecomment-270739989).

If a `FetchController` has been constructed and is pending (see `FetchController.pending` above), this performs a `fetch()` with all the given `options` as `window.fetch()` would, using the `FetchController` as the fetch's controller.

If a function is specified for the `controller` or `observer` options (ie. "revealing constructors"), they are passed the already-constructed `FetchController` or corresponding `.observer`, respectively. If any other value is defined for either these options, the fetch fails with a `TypeError`.

Note that, although such a method *could* be specified, this document *does not* specify a corresponding `FetchObserver.fetch()` method, as methods on `FetchObserver` are meant to be uniformly *passive*.

### FetchController.abort()

Aborts the corresponding fetch, causing any pending promises on the fetch to be rejected with an `AbortError`.

If the `FetchController` has been constructed without an associated fetch and is still pending (see `FetchController.pending` above), this is an error.

If the fetch is already complete, this is an error. See [whatwg/fetch#448][].

## FetchObserver events

These are patterned after [the progress events of XMLHttpRequest][Monitoring progress], but with the names changed, to avoid the ironically-overloaded use of the word "load", and to consolidate upload and download events into the same stream (rather than having all but one kind of event mirrored across two objects, with the one differing kind of event having the same name).

[Monitoring progress]: https://developer.mozilla.org/en-US/docs/Web/API/XMLHttpRequest/Using_XMLHttpRequest#Monitoring_progress

Note that, for a `FetchObserver` provided as part of a `FetchEvent` for a Service Worker, these events correspond to the point of view of fetch *being handled*: for instance, the `progressdown` events will fire for whatever data the Service Worker provides via `respondWith`, *even if that data is being produced by code within the Service Worker from a non-network source*.

### `start`

Fired when the fetch's request is initiated. This is analogous to XHR's "loadstart" event.

### `upload`

Fired when the fetch sends data in the request. This is analogous to the "progress" event on an XMLHttpRequest's `upload` object.

### `response`

Fired when the fetch receives a response and headers from the server. This is analogous to the "readystatechange" event when an XMLHttpRequest's `readyState` changes to `2` (`HEADERS_RECEIVED`).

### `download`

Fired when the fetch receives data in the response. This is analogous to the "progress" event on an XMLHttpRequest's *base* object.

### `complete`

Fires when the fetch completes successfully. This is analogous to XHR's "load" event.

### `abort`

Fires if the fetch is aborted, either by a `FetchController` or by the user agent. (More data about the source of the abort will probably be available on the event.) This is analogous to XHR's "abort" event.

### `error`

Fires if the fetch fails, for whatever non-abortive reason (such as if the connection is unexpectedly terminated). This is analogous to XHR's "error" event.

### `end`

Fires after `complete`, `abort`, or `error`. This is analogous to XHR's "loadend" event.

## Restrictions

Due to the corresponding one-time Promise properties on these objects, neither a `FetchController` nor a `FetchObserver` may be reused for multiple fetches, nor may a `FetchController` or `FetchObserver` have their associated fetch changed (or vice versa) after the fetch's instantiation.

(Likewise, the `FetchObserver` associated with a `FetchController`, as its read-only `.observer` property, many not be modified, before *or* after the instantiation of a corresponding fetch.)
