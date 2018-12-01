# vue-chromish-tabs
Chrome-inspired tabs with horizontal scroll and client-defined contents.

**Note:**
This code is written with intent to use it in Electron applications.
Browser compatibility is for that reason not considered.

Inspired by and adapted from [`chrome-tabs`](https://github.com/adamschwartz/chrome-tabs).

TODO: Demo/example code.
Possibly a gif of some kind.

## Theming
TODO: Write a sentence or two about how to theme it.
For now, take a look at [`chrome-tabs-dark-theme.css`](./src/css/chrome-tabs-dark-theme.css) for an idea of what can be done.

## How it works
TODO: How tabs are rendered in three parts for allowing other heights and varying width.

TODO: row-reverse flex-direction. For avoiding z-index issues with transition.

TODO: Delegation of interation logic to `vue-dnd-list`.

### Transitions/animations

Because the behavior of `transition-group` in Vue did not satisfy the requirements of `vue-dnd-list` (mid-transition swapping using custom transition values and -times), it had to use individual `transition` components with custom move logic.
As it turns out, they don't work too well when used in `v-for`, preventing both `enter` and `leave` transitions from working properly.
For that reason, `vue-chromish-tabs` handles opening/closing transitions itself.
_It is the intention to remove the use of `transition` from `vue-dnd-list`, hopefully making it more of a renderless component that simply extracts sorting logic away from `vue-chromish-tabs`._

The opening animation is simply done by applying the animation class and removing it when the animation is done.

The closing transition is more complicated.
A negative margin is set in JS according to the tab width to have the following items shift into place.
While doing this, the tab content is itself shifted.
This looks like the entire tab is shifted, but it stays in place, which allows using a mask to prevent larger tabs from going through smaller ones and the leftmost tab to not be abruptly cut off.
A timeout removes the tab when the transition is over.

## Notes to self

* I had some issues with popping z-index as the dragged tab was dropped.
At first I thought it was something wrong with `vue-dnd-list`, but it turned out to simply be that z-index was transitioned.
Silly, but worth keeping in mind.
