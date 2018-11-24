# vue-chromish-tabs
Chrome-inspired tabs with horizontal scroll and client-defined contents.

**Note:**
This code is written with intent to use it in Electron applications.
Browser compatibility is for that reason not considered.

## How it works

### Transitions/animations

Because the behavior of `transition-group` in Vue did not satisfy the requirements of `vue-dnd-list` (mid-transition swapping using custom transition values and -times), it had to use individual `transition` components with custom move logic.
As it turns out, they don't work too well when used in `v-for`, preventing both `enter` and `leave` transitions from working properly.
For that reason, `vue-chromish-tabs` handles opening/closing transitions itself.
__
It is the intention to remove the use of `transition` from `vue-dnd-list`, hopefully making it more of a renderless component that simply extracts sorting logic away from `vue-chromish-tabs`.
__

The opening animation is simply done by applying the animation class and removing it when the animation is done.

## Notes to self

* I had some issues with popping z-index as the dragged tab was dropped.
At first I thought it was something wrong with `vue-dnd-list`, but it turned out to simply be that z-index was transitioned.
Silly, but worth keeping in mind.
