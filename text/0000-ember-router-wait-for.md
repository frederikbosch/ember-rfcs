---
stage: accepted
start-date: 2026-05-11T07:21:00.000Z
release-date: # In format YYYY-MM-DDT00:00:00.000Z
release-versions:
teams: # delete teams that aren't relevant
  - framework
prs:
  accepted: # Fill this in with the URL for the Proposal RFC PR
project-link:
suite: 
---

# Ember Router public Transition.waitFor method

## Summary

> Add a public method `Transition.waitFor` to the Ember Router and thereby supporting the new native
ViewTransition API.

## Motivation

> At the moment the router cannot await a promise when switching from the old route to the new one. In order 
to implement the new browser native ViewTransition API you need the router's transition to happen during the 
native view transition. The `waitFor` method allows the Ember router to pause its transition until the native
view transition has finished its asynchronous `document.startViewTransition` call.

## Detailed design

Inside the `Transition` object inside the Ember Router a new method `waitFor` will be added.

```js
waitFor(promise: Promise<any>) {
  this._pausingPromise = promise;
}
```

When called, the method will cause the transition to wait for the passed promise to resolve before
continuing the transition. The promise awaited after the `routeWillChange` event and before the 
`beforeModel` hook in the new route. This allows users to call the `waitFor` sinde `routeWillChange` 
listeners.

```js
router.on('routeWillChange', async (transition) => {
  const { promise, resolve, reject } = Promise.withResolvers();
  transition.waitFor(promise);

  const viewTransition = document.startViewTransition(async () => {
    resolve();
    await transition.promise;
  });
  await viewTransition.updateCallbackDone;
});
```

This is particularly useful to implement the new browser-native [ViewTransition API](https://developer.mozilla.org/en-US/docs/Web/API/View_Transition_API)
as shown in the example above. In order to start a view transition you need to call `document.startViewTransition`, 
the passed callback is executed asynchronously.

> The callback is invoked once the API has taken a snapshot of the current page. When the promise returned 
by the callback fulfills, the view transition begins in the next frame. If the promise returned by the callback 
rejects, the transition is abandoned.

At the moment the Ember Router will not await `routeWillChange` listeners. So the method below will not cause the
transition to pause. The router will continue entering the new route, even though the snapshot by the view transition
as referred to above was not yet finished.

```js
router.on('routeWillChange', async (transition) => {
  const viewTransition = document.startViewTransition(async () => {
  });
  await viewTransition.updateCallbackDone;
});
```

An alternative solution to the `waitFor` method would be let the router await `routeWillChange` listeners. But
with probably many listeners being used by current users, it is hard to foresee if this would possibly break
current applications. A new method allows users to opt-in to an asynchronous switch between old and new route.

Finally, the `waitFor` accepts a single promise that replaces a possible other pausing promise. In the example 
below only promise2 is waited for by the router. There is also no possibility to remove pausing promises. If 
users would want to wait for multiple Promise, it could create an aggregate promise via `Promise.all()`.

```js
const { promise: promise1 } = Promise.withResolvers();
transition.waitFor(promise1);

const { promise: promise2 } = Promise.withResolvers();
transition.waitFor(promise2);
```


## How we teach this

Since animation libraries liquid-fire and Ember animated have always been popuplar within Ember, there should a 
separate section inside the [Ember Routing guides](https://guides.emberjs.com/release/routing/). It should be
a separate page View Transition between Asynchronous Routing and Controllers. The page should document how to
use view transitions within Ember.

## Drawbacks

A drawback might be that there is a team considering replacing the current Ember Router with a new one. However,
there is a need within the Ember community right now to implement view transitions in a non-hacky manner.

## Alternatives

Alternatives are already discussed within the design and include awaiting `routeWillChange` listeners. This
alternative has not beed chosen because the author of this RFC cannot foresee if this would break current
applications.

## Unresolved questions

`waitFor` or `waitUntil`, I have chosen former but the method might reconsidered.
