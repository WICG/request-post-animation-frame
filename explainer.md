# Window.requestPostAnimationFrame

`Window.requestPostAnimationFrame` will act as a bookend to `Window.requestAnimationFrame`. Whereas `requestAnimationFrame` callbacks run just prior to the "update the rendering" step in the [HTML processing model](https://html.spec.whatwg.org/#event-loop-processing-model) (step 10.10), `requestPostAnimationFrame` callbacks will run at the end of the "update the rendering" steps.

Broadly speaking, there are two situations where `requestPostAnimationFrame` will be useful:

## Getting a head start on preparing for the next animation frame

`requestAnimationFrame` is useful for updating the DOM tree in preparation for the next rendering update. However, if the code in a `requestAnimationFrame` callback runs too long, then the rendering will not be complete in time to send new pixels to the display before the next vsync. `requestPostAnimationFrame` provides a way to get the earliest possible start on preparing for the next rendering update, without any possibility of the work being preempted by other less critical tasks. A typical use pattern for a continuously-animating page would be:

```js
function animate() {
  requestPostAnimationFrame(() => {
    // Do potentially long-running work here.
    requestAnimationFrame(() => {
      // Apply final DOM and CSS updates here, if necessary.
      animate();  // Register callbacks for the next frame.
    });
  });
}
```

## Preventing forced layouts

To ensure good performance, web developers are encouraged to avoid scripting patterns that force a synchronous layout (e.g., by modifying a CSS property and then querying the size or position of an element). However, there is currently no way for a developer to schedule code to run at a time when the page layout is guaranteed to be clean. In real world code, a common pattern to minimize the chances of forcing layout is:

```js
addEventListener("message", event => {
  // Query layout information here.
});

requestAnimationFrame(() => {
  postMessage("*", "");
});
```

The `requestAnimationFrame` handler runs just prior to the rendering update, and there's a good chance that the message handler will be the first task to run after the rendering update. However, there is still a non-zero chance that some other code (an expired `setTimeout` or `setInterval`, or an event handler) will run after rendering, but before the message handler.

Because `requestPostAnimationFrame` handlers run immediately after the rendering update, before any other code, it is *guaranteed* that layout will be clean, and querying layout information will not force an additional synchronous layout. Caveat: if css properties are modified inside the `requestPostAnimationFrame` callback, then querying layout information will still force a synchronous relayout.

