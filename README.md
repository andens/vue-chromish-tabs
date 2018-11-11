# vue-chromish-tabs
Chrome-inspired tabs with horizontal scroll and client-defined contents.

**Note:**
This code is written with intent to use it in Electron applications.
Browser compatibility is for that reason not considered.

## Notes to self

* I had some issues with popping z-index as the dragged tab was dropped.
At first I thought it was something wrong with `vue-dnd-list`, but it turned out to simply be that z-index was transitioned.
Silly, but worth keeping in mind.
