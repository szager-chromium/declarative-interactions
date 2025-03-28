# Declarative Interactions

## Overview

One of the cornerstones of interactive web design is providing prompt visual feedback in response to input events; however, that can be difficult to achieve reliably. One of the enduring obstacles is that the only way to apply visual changes in response to an input event is via an event handler run by the window event loop, which may be congested. This puts the web platform at a disadvantage compared to modern native platforms, which do not have this constraint.

A common pattern in web design is for an input event handler to start an animation that provides immediate visual feedback to the user. All major browser implementations provide a way for certain animations, once started, to continue updating smoothly, even while the window event loop is blocked or busy. Thus an effective strategy to achieve smooth UI is to start an animation from an input handler, while postponing expensive application logic until after the animation is running. However -- crucially -- an animation can only be started from a task running in the window event loop, making it susceptible to arbitrary delay if the window event loop is blocked when the input event arrives.

This proposal provides a declarative syntax for specifying that an input event targeting a particular element should start an animation on another specified element. The syntax is designed to make it possible for an implementation to opportunistically start the animation independently of the window event loop, thereby eliminating a significant source of input response delay.

## Solution Sketch

```html
<style>
  #spinner {
    animation: spin 0.25s forwards paused;
    animation-trigger: click(spin-trigger) once;
  }
  #spin-trigger {
    animation-trigger-name: spin-trigger;
  }
  @keyframes spin {
    from { rotate: 0deg; }
    to { rotate: 180deg; }
  }
</style>
<div id="spin-trigger">
  Details
  <svg id="spinner" style="width: 24px; height: 24px;">
    <path d="M16.59 8.59 12 13.17 7.41 8.59 6 10 12 16 18 10Z"></path>
  </svg>
</div>
```

The above is almost exactly equivalent to:

```html
<style>
  #spinner {
    animation: spin 0.25s forwards paused;
  }
  @keyframes spin {
    from { rotate: 0deg; }
    to { rotate: 180deg; }
  }
</style>
<div id="spin-trigger" onclick="trigger()">
  Details
  <svg id="spinner" style="width: 24px; height: 24px;">
    <path d="M16.59 8.59 12 13.17 7.41 8.59 6 10 12 16 18 10Z"></path>
  </svg>
</div>
<script>
function trigger() {
  spinner.getAnimations().find(a => a.id == "spin").play();
}
</script>
```

This builds on an existing [proposal](https://github.com/w3c/csswg-drafts/issues/8942#issuecomment-1602924213) for the `animation-trigger` CSS property. That proposal is focused on triggering an animation in response to a change in the intersection between a target element and the document-level scroll viewport. This proposal builds on that syntax by allowing an input event to act as the trigger. Because the `animation-trigger` syntax is declarative and does not run script or depend on global state, an implementation can potentially set up the animation in advance and start running it in response to an input event without waiting for an event handler to run on the window event loop.