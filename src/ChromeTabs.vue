<template>
  <div class="chrome-tabs">
    <div class="fixed-content">
      <slot name="fixed-content" />
    </div>
    <div class="chrome-tabs-bottom-bar"></div>
    <div class="tabs-container-wrapper">
      <!--
        I tried some pre-existing dnd libraries, but they were usually limited
        in some way. Some things they didn't cover well enough were:
        * mid-transition swapping (linear, respect current progress)
        * swap determination logic (swap when an edge crosses neighbor center)
        * autoscroll behavior (outside container, limited speed)
        * customizability (mostly animation control)
        * avoid manipulating irrelevant items (don't transform all)
        vue-dnd-list is not perfect, but it does a decent job making drag and
        drop functionality work while at the same time playing nicely with Vue
        DOM manipulations.
      -->
      <sortable-container
        ref="tabsContent"
        v-model="tabs"
        orientation="x"
        transitionName="chrome-tab-transition"
        class="chrome-tabs-container"
        scrollContainerClass="chrome-tabs-content"
        itemKeyProperty="id"
        :activationDistance=10
        :helperZ="tabs.length + 2"
        :maxItemTransitionDuration=150
        :scrollContainerEvents="{
          'mousewheel': scrollTabs, // Originally: @mousewheel.native='...'
        }"
        :lockAxis=true
        @sort-end="sortEnd"
      >
        <!-- slot-scope injects data from vue-dnd-list into this slot content -->
        <chrome-tab
          slot-scope="{listItem: tab, index, isGhost, isHelper, sorting, settling, startDrag}"
          :class="{ 'chrome-tab-current': selected === tab, 'ghost': isGhost && (sorting || settling) }"
          :style="{
            zIndex: `${selected === tab || isHelper ? tabs.length + 2 : tabs.length - index}`,
          }"
          @auxclick.middle.native="closeTab(index)"
          @mousedown.middle.native.prevent.stop
          @mousedown.left.native.prevent="e => {
            if (canSelectTab) {
              selectTab(index);
              startDrag(e);
              canSelectTab = false;
            }
          }"
          @close="closeTab(index)"
        >
          <!-- Include user tab content, passing tab data to the user -->
          <slot v-bind="tab.data" />
        </chrome-tab>
      </sortable-container>
    </div>
  </div>
</template>

<script>
import SortableContainer from "vue-dnd-list";

const ChromeTab = {
  // The tab side slant has a 14:29 ratio.
  template: `
    <div class="chrome-tab">
      <!-- Adapted from https://stackoverflow.com/questions/26028370/multiple-aspect-ratios-in-a-single-svg -->
      <!-- This works fine with changing width as long as the height remains fixed, but that is intended for the tabs anyway -->
      <!-- This works by defining a source image and then rendering parts of it in three pieces, where the middle piece consumes remaining flexbox space and is stretched without preserving aspect ratio -->
      <div class="chrome-tab-background">
        <svg width="0" height="0" xmlns="http://www.w3.org/2000/svg" xmlns:xlink="http://www.w3.org/1999/xlink">
          <defs>
            <g id="source">
              <path d="M0,29h57l-14,-29h-29z"/>
            </g>
          </defs>
        </svg>
        <svg class="edge-piece" viewBox="0 0 14 29">
          <use y="1" xlink:href="#source" class="chrome-tab-background"/>
          <use xlink:href="#source" class="chrome-tab-shadow"/>
        </svg>
        <svg class="middle" viewBox="14 0 29 29" preserveAspectRatio="none">
          <use y="1" xlink:href="#source" class="chrome-tab-background"/>
          <use xlink:href="#source" class="chrome-tab-shadow"/>
        </svg>
        <svg class="edge-piece" viewBox="43 0 14 29">
          <use y="1" xlink:href="#source" class="chrome-tab-background"/>
          <use xlink:href="#source" class="chrome-tab-shadow"/>
        </svg>
      </div>
      <div class="chrome-tab-content">
        <slot />
      </div>
      <div class="chrome-tab-close" @mousedown.left.prevent.stop @click="$emit('close')" />
    </div>
  `,
};

export default {
  data() {
    return {
      tabs: [],
      scroll: {
        targetScrollPosition: 0,
        animationFrameId: null,
        scrollDelta: 0,
        iteration: 0,
      },
      selected: null,
      draggabillyInstances: [],
      accumulatedTabCount: 0,
      canSelectTab: true,
    }
  },

  methods: {
    scrollTabs(event) {
      event.preventDefault();
      event.stopPropagation();

      const scrollContainer = this.$refs.tabsContent.getScrollContainer();

      // If we are not already scrolling, reset the targetScrollPosition because
      // we might have autoscrolled by dragging tabs.
      if (!this.scroll.animationFrameId) {
        this.scroll.targetScrollPosition = scrollContainer.scrollLeft;
      }

      // Early exit: scroll left while at leftmost end.
      if (this.scroll.targetScrollPosition <= 0 && event.deltaY < 0) {
        return;
      }

      // Early exit: scroll right while at rightmost end.
      if (this.scroll.targetScrollPosition >= (scrollContainer.scrollWidth - scrollContainer.clientWidth) && event.deltaY > 0) {
        return;
      }

      // The number of iterations over which the scroll should reach its target
      // position. Should probably be calculated using the update frequency
      // (which likely was 60 fps when this was written) so that the time is
      // always correct.
      const iterationCount = 15;

      // The idea now is that when rolling the mouse wheel we manually scroll
      // the horizontal direction. Using `requestAnimationFrame` allows us to
      // scroll smoothly by reaching the scroll target (which accumulates, thus
      // speeding up with fast scrolls) over multiple iterations. As we continue
      // scrolling, the delta is recalculated and the iteration count is reset
      // to give enough iterations to reach the target.
      this.scroll.targetScrollPosition += event.deltaY;
      let delta = this.scroll.targetScrollPosition - scrollContainer.scrollLeft;
      this.scroll.scrollDelta = Math.sign(delta) * Math.max(Math.abs(delta), 100) / iterationCount;
      this.scroll.iteration = 0;

      let self = this;

      function doScroll() {
        scrollContainer.scrollLeft += self.scroll.scrollDelta;
        self.$refs.tabsContent.synchronizeHelperTranslation();

        if (++self.scroll.iteration === iterationCount) {
          cancelAnimationFrame(self.scroll.animationFrameId);
          self.scroll.animationFrameId = null;
          self.scroll.targetScrollPosition = scrollContainer.scrollLeft;
          self.scroll.iteration = 0;
        }
        else {
          self.scroll.animationFrameId = requestAnimationFrame(doScroll);
        }
      }

      if (!this.scroll.animationFrameId) {
        doScroll();
      }
    },

    // Intended to be called by user
    addTab(customData, select = true) {
      let t = { id: this.accumulatedTabCount++, data: customData };
      this.tabs.push(t);
      if (select) {
        this.selectTab(this.tabs.length - 1);
      }
    },

    selectTab(index) {
      if (index === null) {
        this.selected = null;
        return;
      }

      let tab = this.tabs[index];
      if (this.selected !== tab) {
        this.selected = tab;
        this.$emit("select", {
          index: index,
          data: tab.data,
        });
      }
    },

    closeTab(index) {
      let tab = this.tabs[index];
      if (this.selected === tab) {
        // Closing last tab: no selection
        if (this.tabs.length === 1) {
          this.selectTab(null);
        }
        // Closing rightmost tab: select previous
        else if (index + 1 === this.tabs.length) {
          this.selectTab(index - 1);
        }
        // Next tab takes the place of the closed one
        else {
          this.selectTab(index + 1);
        }
      }

      let deleted = this.tabs.splice(index, 1);

      this.$emit("close", {
        index: index,
        data: deleted[0].data,
      });
    },

    sortEnd() {
      this.canSelectTab = true;
    },
  },

  components: {
    SortableContainer,
    ChromeTab,
  },
}
</script>

<style scoped>
@import url("./css/chrome-tabs.css");
</style>
