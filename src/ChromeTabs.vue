<template>
  <div class="chrome-tabs">
    <div class="fixed-content">
      <slot name="fixed-content" />
    </div>
    <!--
      Wrap the tab container in a relative positioned flex item that spans
      the remaining width, allowing it to be absolutely positioned, which in
      turn makes the clip property usable to prevent rendering the dragged
      tab outside the container even if fixed positioned.
      https://stackoverflow.com/a/23859719
    -->
    <div class="tabs-container-wrapper">
      <div class="tabs-container-edge-left" :style="`z-index: ${tabs.length + 3};`">
        &nbsp;
      </div>
      <chrome-tabs-container
        ref="tabsContent"
        v-model="tabs"
        axis="x"
        lockAxis="x"
        :transitionDuration=150
        :distance=10
        :appendHelperToContainer=true
        :draggedSettlingDuration=150
        @mousewheel.native="scrollTabs"
        @sort-end="sortEnd"
      >
        <chrome-tab
          v-for="(tab, index) in tabs" :key="tab.id"
          ref="tabs"
          :index="index"
          :class="{ 'chrome-tab-current': selected === tab }"
          :style="`z-index: ${selected === tab ? tabs.length + 2 : tabs.length - index};`"
          @auxclick.middle.native="closeTab(index)"
          @mousedown.middle.native.prevent.stop
          @mousedown.left.native="selectTab(index)"
          @close="closeTab(index)"
        >
          <slot v-bind="tab.data" />
        </chrome-tab>
      </chrome-tabs-container>
      <div class="tabs-container-edge-right" :style="`z-index: ${tabs.length + 3};`">
        &nbsp;
      </div>
    </div>
    <div class="chrome-tabs-bottom-bar" :style="`z-index: ${tabs.length + 1}`"></div>
  </div>
</template>

<script>
import { ContainerMixin, ElementMixin } from "vue-slicksort";

const ChromeTabsContainer = {
  mixins: [ContainerMixin],
  template: `
    <transition-group
      name="chrome-tab-transition"
      tag="div"
      class="chrome-tabs-content"
      @beforeLeave="beforeLeave"
    >
      <slot />
    </transition-group>
  `,
  methods: {
    beforeLeave(el) {
      // When an element is removed from the transition group, set the margin
      // to the element width, which will then be transitioned in CSS.
      el.style.marginLeft = `-${el.offsetWidth}px`;
    },
  },
};

const ChromeTab = {
  mixins: [ElementMixin],
  template: `
    <div class="chrome-tab">
      <div class="chrome-tab-background">
        <svg version="1.1" xmlns="http://www.w3.org/2000/svg">
          <defs>
            <symbol id="chrome-tab-geometry-left" viewBox="0 0 214 29">
              <path d="M14.3 0.1L214 0.1 214 29 0 29C0 29 12.2 2.6 13.2 1.1 14.3-0.4 14.3 0.1 14.3 0.1Z"/>
            </symbol>
            <symbol id="chrome-tab-geometry-right" viewBox="0 0 214 29">
              <use xlink:href="#chrome-tab-geometry-left"/>
            </symbol>
            <clipPath id="crop">
              <rect class="mask" width="100%" height="100%" x="0"/>
            </clipPath>
          </defs>
          <svg width="50%" height="100%">
            <use xlink:href="#chrome-tab-geometry-left" width="214" height="29" class="chrome-tab-background"/>
            <use xlink:href="#chrome-tab-geometry-left" width="214" height="29" class="chrome-tab-shadow"/>
          </svg>
          <g transform="scale(-1, 1)">
            <svg width="50%" height="100%" x="-100%" y="0">
              <use xlink:href="#chrome-tab-geometry-right" width="214" height="29" class="chrome-tab-background"/>
              <use xlink:href="#chrome-tab-geometry-right" width="214" height="29" class="chrome-tab-shadow"/>
            </svg>
          </g>
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
    }
  },

  methods: {
    scrollTabs(event) {
      event.preventDefault();
      event.stopPropagation();

      // If we are not already scrolling, reset the targetScrollPosition because
      // we might have autoscrolled by dragging tabs.
      if (!this.scroll.animationFrameId) {
        this.scroll.targetScrollPosition = this.$refs.tabsContent.$el.scrollLeft;
      }

      // Early exit: scroll left while at leftmost end.
      if (this.scroll.targetScrollPosition <= 0 && event.deltaY < 0) {
        return;
      }

      // Early exit: scroll right while at rightmost end.
      if (this.scroll.targetScrollPosition >= (this.$refs.tabsContent.$el.scrollWidth - this.$refs.tabsContent.$el.clientWidth) && event.deltaY > 0) {
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
      let delta = this.scroll.targetScrollPosition - this.$refs.tabsContent.$el.scrollLeft;
      this.scroll.scrollDelta = Math.sign(delta) * Math.max(Math.abs(delta), 100) / iterationCount;
      this.scroll.iteration = 0;

      let self = this;

      function doScroll() {
        self.$refs.tabsContent.$el.scrollLeft += self.scroll.scrollDelta;

        if (++self.scroll.iteration === iterationCount) {
          cancelAnimationFrame(self.scroll.animationFrameId);
          self.scroll.animationFrameId = null;
          self.scroll.targetScrollPosition = self.$refs.tabsContent.$el.scrollLeft;
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

    sortEnd({oldIndex, newIndex}) {
      if (oldIndex !== newIndex) {
        this.$emit("move", { oldIndex, newIndex });
      }
    },
  },

  components: {
    ChromeTabsContainer,
    ChromeTab,
  },
}
</script>

<style scoped>
@import url("./css/chrome-tabs.css");
</style>
