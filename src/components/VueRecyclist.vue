<template>
  <div class="vue-recyclist">
    <div ref="list" class="vue-recyclist-items" :style="{height: height + 'px'}">
      <div v-for="(item, index) in visibleItems" class="vue-recyclist-item" :style="{'transform': 'translate3d(0,' + item.top + 'px,0)'}">
        <transition name="v-recyclist">
          <div v-if="tombstone && !item.loaded" :class="{'vue-recyclist-transition': tombstone}">
            <slot name="tombstone"></slot>
          </div>
        </transition>
        <transition name="v-recyclist">
          <div v-if="item.loaded" :class="{'vue-recyclist-transition': tombstone}">
            <slot name="item" :data="item.data" :index="item.index"></slot>
          </div>
        </transition>
      </div>

      <!--get tombstone and item heights from these invisible doms-->
      <div class="vue-recyclist-pool">
        <div :ref="'item'+index" v-for="(item, index) in items" v-if="!item.tomb && !item.height"
          class="vue-recyclist-item vue-recyclist-invisible">
          <slot name="item" :index="item.index" :data="item.data"></slot>
        </div>
        <div ref="tomb" class="vue-recyclist-item vue-recyclist-invisible">
          <slot name="tombstone"></slot>
        </div>
      </div>
    </div>

    <transition name="v-recyclist">
      <div v-if="nomore && !loading"
        class="vue-recyclist-nomore">
        <slot name="nomore">
          <div>End of list</div>
        </slot>
      </div>
    </transition>

    <transition name="v-recyclist">
      <div v-if="spinner && !nomore"
        class="vue-recyclist-loading"
        :style="{visibility: !nomore ? 'visible' : 'hidden'}">
        <slot name="spinner">
          <div class="vue-recyclist-loading-content">
            <div class="cssloading-circle spinner"></div>
          </div>
        </slot>
      </div>
    </transition>

  </div>
</template>
<script>
  const isIOS = !!navigator.userAgent.match(/\(i[^;]+;( U;)? CPU.+Mac OS X/); //ios终端

  function now() {
    return new Date/1;
  }

  export default {
    data() {
      return {
        name: 'VueRecyclist',
        list: [], // 原始数据的列表
        items: [], // 显示列表的 item
        height: 0, // Full list height
        start: 0, // Visible items start index
        startOffset: 0, // Start item offset
        loading: false,
        loadList: [], // 数据加载列表
        nomore: false,
        // 调用 reset 的最后时间
        resetTime: now(),
      }
    },

    computed: {
      visibleItems() {
        return this.items.slice(Math.max(0, this.start - this.size), Math.min(this.items.length, this.start + this.size))
      },
      // containerHeight() {
      //   return this.$el && this.$el.offsetHeight || 0
      // },
      tombHeight() {
        return this.tombstone ? this.$refs.tomb && this.$refs.tomb.offsetHeight : 0
      },
    },

    props: {
      tombstone: {
        type: Boolean,
        default: false // Whether to show tombstones.
      },
      size: {
        type: Number,
        default: 20 // The number of items added each time.
      },
      offset: {
        type: Number,
        default: 200 // The number of pixels of additional length to allow scrolling to.
      },
      loadmore: {
        type: Function,
        required: true // The function of loading more items.
      },
      spinner: {
        type: Boolean,
        default: true // Whether to show loading spinner.
      },
    },

    mounted() {
      this.$el.addEventListener('scroll', this.onScroll.bind(this));
      window.addEventListener('resize', this.onResize.bind(this));
      this.init();
    },

    // <keep-alive>
    activated() {
      this.$el.scrollTop = this.backupScrollTop || 0;
    },
    deactivated() {
      this.backupScrollTop = this.$el.scrollTop || 0;
    },

    // end <keep-alive>
    methods: {
      init() {
        this.reset();
        this.load();
      },

      reset() {
        this.list = [];
        this.items = [];
        this.nomore = false;
        this.loading = false;
        this.height = this.top = this.start = 0;
        this.$el.scrollTop = this.backupScrollTop = 0;
        this.resetTime = now();
      },

      load() {
        if (this.nomore || this.loading) { return; }

        let pageIndex = Math.ceil(this.items.length / this.size);

        // @tombstone 加载，需要特殊处理
        if (this.tombstone) {
          this.items.length = this.list.length + this.size;
          this.loadItems();
        }

        this.getItems(pageIndex);
      },

      getItems(pageIndex, callback) {
        this.loading = true;
        let requestTime = now();
        this.loadmore(pageIndex, (dataList, nomore) => {
          if (this.resetTime > requestTime) { return; }
          this.loading = false;

          // 1. 把 this.list 对应索引的数据，替换为 list 的
          // 2. 重新加载 this.items 对应索引的数据
          let length = dataList.length;
          let size = this.size;

          // @潜规则 如果返回的数据，比尺寸要少，立刻就 nomore 掉
          if (length < size || nomore) {
            this.setNoMoreData(true);
          }

          // 简单的替换
          let end = (pageIndex + 1) * size - (size - length);
          this.list.length = Math.max(this.list.length, end);
          this.items.length = this.list.length;

          let index = 0;
          for (let i = end - length, max = end; i < max; i++) {
            this.list[i] = dataList[index++];
            this.items[i] = null;
          }

          this.loadItems();

          callback && callback();
        });
      },

      loadItems() {
        let loads = [];
        let start = 0;
        let end = this.items.length;

        // 没有内容啦~
        if (end <= 0) {
          this.items = []; // 触发 visibleItems 更新
          this.updateItemTop();
          return;
        }

        for (let i = start; i < end; i++) {
          if (this.items[i] && this.items[i].loaded) {
            continue
          }
          this.setItem(i, this.list[i] || null)
          // update newly added items position
          loads.push(this.$nextTick().then(() => {
            this.updateItemHeight(i);
          }));
        }
        // update items top and full list height
        Promise.all(loads).then(() => {
          this.updateItemTop();
          this.onScroll();
        });
      },

      setItem(index, data) {
        this.$set(this.items, index, {
          index: index,
          data: data ? data : {},
          height: 0,
          top: -1000,
          tomb: !data,
          loaded: !!data
        });
      },

      updateItemHeight(index) {
        // update item height
        let cur = this.items[index];
        if (!cur) { return; }

        let dom = this.$refs['item' + index];
        if (dom && dom[0]) {
          cur.height = dom[0].offsetHeight;
        } else {
          // item is tombstone
          cur.height = this.tombHeight;
        }
      },

      updateItemTop() {
        // loop all items to update item top and list height
        this.height = 0
        for (let i = 0; i < this.items.length; i++) {
          let pre = this.items[i - 1]
          this.items[i].top = pre ? pre.top + pre.height : 0
          this.height += this.items[i].height
        }
        // update scroll top when needed
        if (this.startOffset) {
          this.setScrollTop();
        }
        this.updateIndex();
        this.makeScrollable();
      },

      updateIndex() {
        // update visible items start index
        let top = this.$el.scrollTop
        for (let i = 0; i < this.items.length; i++) {
          if (this.items[i].top > top) {
            this.start = Math.max(0, i - 1)
            break
          }
        }
        // scrolling does not need recalculate scrolltop
        // this.getStartItemOffset()
      },

      getStartItemOffset() {
        if (this.items[this.start]) {
          this.startOffset = this.items[this.start].top - this.$el.scrollTop
        }
      },

      setScrollTop() {
        if (this.items[this.start]) {
          this.$el.scrollTop = this.items[this.start].top - this.startOffset
          // reset start item offset
          this.startOffset = 0
        }
      },

      makeScrollable() {
        // make ios -webkit-overflow-scrolling scrollable
        this.$el.classList.add('vue-recyclist-scrollable');
      },

      onScroll() {
        // @notice 如果是 ios 并且不是 safari 浏览器，需要添加计时器，保证它流畅
        // @notice android 不走计时器!!!!!
        if (isIOS) {
          clearTimeout(this.scrollTimer);
          this.scrollTimer = setTimeout(() => {
            this.checkScroll();
          }, 20);
        } else {
          this.checkScroll();
        }
      },

      checkScroll() {
        if (this.$el.scrollTop + this.$el.offsetHeight > this.height - this.offset) {
          this.load();
        }
        this.updateIndex();
      },

      onResize() {
        clearTimeout(this.resizeTimer);
        this.resizeTimer = setTimeout(() => {
          this.getStartItemOffset();
          this.items.forEach((item) => {
            item.loaded = false;
          });
          this.loadItems();
        }, 200);
      },

      // 重新加载最新一页数据
      loadNewDataAgain() {
        this.nomore = false;
        this.resetTime = now();
        this.load();
      },

      // 删除当前索引的数据
      removeIndex(index) {
        // 删除当前元素
        this.list.splice(index, 1);
        this.items.splice(index, 1);

        // 修正索引
        this.items.forEach((item, i) => {
          item.index = i;
        });

        this.updateItemTop();

        if (this.nomore) { return; }

        // 加载最后一页的数据
        let last = this.items[this.list.length - 1];
        this.reloadPageByIndex(last ? last.index : 0);
      },

      // 根据索引，重新加载某分页
      reloadPageByIndex(index) {
        let pageIndex = Math.floor(index / this.size);
        this.resetTime = now();
        this.getItems(pageIndex, () => {
          this.onScroll();
        });
      },

      // 已经没有数据了
      setNoMoreData(nomore) {
        this.nomore = true;
      },
    },

    destroyed() {
      this.$el.removeEventListener('scroll', this.onScroll.bind(this))
      window.removeEventListener('resize', this.onResize.bind(this))
    }
  }

</script>
<style src="./cssloading.css"></style>
<style lang="less">
  @duration: 300ms;
  .vue-recyclist {
    height: 100%;
    overflow: auto;
    // 解决 android 滚动不流畅的问题
    transform: translateZ(0);
    -webkit-overflow-scrolling: auto;

    &.vue-recyclist-scrollable {
      // 解决 ios safari 浏览器，滚动轴消失的问题
      -webkit-overflow-scrolling: touch;
        .vue-recyclist-items { min-height: 100%; }
    }

    .vue-recyclist-items {
      position: relative;
      margin: 0;
      padding: 0;
      overflow: hidden;
      min-height: 101%;

      .vue-recyclist-invisible {
        top: -1000px;
        visibility: hidden;
      }
      .vue-recyclist-item {
        position: absolute;
        width: 100%;
        .vue-recyclist-transition {
          position: absolute;
          width: 100%;
          // opacity: 0;
          // transition-property: opacity;
          // transition-duration: @duration;
        }
      }
    }

    .vue-recyclist-bottom {
      position: relative;
      .vue-recyclist-loading, .vue-recyclist-nomore {
        // position: absolute;
        // top: 0; left: 0; width: 100%;
      }
    }

    // 默认 loading
    .vue-recyclist-loading-content {
      width: 100%;
      text-align: center;
      .spinner {
        margin: 10px auto;
        width: 20px;
        height: 20px;
      }
    }
  }

  // 切换动画
  .v-recyclist-enter-active, .v-recyclist-leave-active {
    opacity: 1;
    transition: opacity @duration ease;
  }
  .v-recyclist-enter, .v-recyclist-leave-to {
    opacity: 0
  }
</style>
