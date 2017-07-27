<template>
  <div>
    <div class="header">
      <div v-show="!touching&&isLoading && isPullDown">
        <slot name="spinner">
          <i class="loading-bubbles" style="top:20px;"></i>
        </slot>
      </div>
      <div class="infinite-status-prompt" v-show="touching && isPullDown">
        <slot name="pull-refresh">{{downTipMsg}}</slot>
      </div>
    </div>
    <slot name="content"></slot>
    <div class="infinite-loading-container">
      <div v-show="isLoading && !isPullDown">
        <slot name="spinner">
          <i :class="spinnerType"></i>
        </slot>
      </div>
      <div class="infinite-status-prompt" v-show="isNoData">
        <slot name="no-results">没有数据</slot>
      </div>
      <div class="infinite-status-prompt" v-show="!isLoading && isComplete && hasScrollBar">
        <slot name="no-more">没有更多数据</slot>
      </div>
    </div>
</template>
<script>
const spinnerMapping = {
  bubbles: 'loading-bubbles',
  circles: 'loading-circles',
  default: 'loading-default',
  spiral: 'loading-spiral',
  waveDots: 'loading-wave-dots',
};
/**
 * get the first scroll parent of an element
 * @param  {DOM} elm    the element which find scorll parent
 * @return {DOM}        the first scroll parent
 */
function getScrollParent(elm) {
  if (elm.tagName === 'BODY') {
    return window;
  } else if (['scroll', 'auto'].indexOf(getComputedStyle(elm).overflowY) > -1) {
    return elm;
  }
  return getScrollParent(elm.parentNode);
}

function getScrollTop(element) {
  let scrollTop;
  if (element.tagName !== 'BODY') {
    scrollTop = element.scrollTop;
  } else {
    scrollTop = document.documentElement.scrollTop;
  }
  return scrollTop;
}

function getScrollHeight(element) {
  let scrollHeight;
  if (element.tagName !== 'BODY') {
    scrollHeight = element.scrollHeight;
  } else {
    scrollHeight = document.documentElement.scrollHeight;
  }
  return scrollHeight;
}

function getVisibleHeight(element) {
  let offsetHeight;
  if (element.tagName !== 'BODY') {
    offsetHeight = element.offsetHeight;
  } else {
    offsetHeight = document.documentElement.offsetHeight;
  }
  return offsetHeight;
}

function isDownTrigger(element) {
  return getScrollTop(element) + getVisibleHeight(element) + 5 >= getScrollHeight(element);
}

export default {
  data() {
    return {
      scrollParent: null,
      scrollHandler: null,
      isLoading: false,
      isComplete: false,
      isFirstLoad: true, // save the current loading whether it is the first loading
      isNoData: false,
      startY: 0,
      currentY: 0,
      startScroll: 0,
      moveY: 0,
      moveDirection: 'up',
      pullDownheight: 0,
      touching: false,
      operateType: 1,//operateType=0 表示下拉刷新，operateType=1 表示滚动加载。
      downTipMsg: '',
    };
  },
  computed: {
    spinnerType() {
      return spinnerMapping[this.spinner] || spinnerMapping.default;
    },
    isPullDown() {
      return this.moveDirection === 'down';
    },
    hasScrollBar() {
      return getScrollHeight(this.scrollParent) - getVisibleHeight(this.scrollParent) > 0;
    },
  },
  props: {
    distance: {
      type: Number,
      default: 100,
    },
    onInfinite: {
      type: Function,
      required: true,
    },
    spinner: String,
    onRefresh: {
      type: Function,
      required: false,
    },
    pullDownDistance: {
      type: Number,
      default: 40,
    },
  },
  ready() {
    this.scrollParent = getScrollParent(this.$el);

    this.scrollHandler = function scrollHandlerOriginal() {
      if (!this.isLoading) {
        this.attemptLoad();
      }
    }.bind(this);

    setTimeout(this.scrollHandler, 1);
    this.scrollParent.addEventListener('scroll', this.scrollHandler);
    if (this.onRefresh) {
      this.scrollParent.addEventListener('touchstart', this.touchStart);
      this.scrollParent.addEventListener('touchmove', this.touchMove);
      this.scrollParent.addEventListener('touchend', this.touchEnd);
    }
  },
  events: {
    '$InfiniteLoading:refreshComplete': function refreshComplete() {
      this.operateType = 0;
      this.isLoading = false;
      this.moveDirection = 'up';
      this.isFirstLoad = false;
      this.complete = false;
      const header = this.scrollParent.querySelector('.header');
      if (header) {
        header.style.height = '0px';
        header.style.display = 'none';
      }
    },
    '$InfiniteLoading:loaded': function loaded() {
      this.isFirstLoad = false;
      this.operateType = 1;
      if (this.isLoading) {
        this.$nextTick(this.attemptLoad);
      }
    },
    '$InfiniteLoading:complete': function complete() {
      this.isLoading = false;
      this.isComplete = true;
      this.scrollParent.removeEventListener('scroll', this.scrollHandler);
    },
    '$InfiniteLoading:completeButNoData': function complete() {
      this.isNoData = true;
      this.isLoading = false;
      this.isComplete = true;
      this.scrollParent.removeEventListener('scroll', this.scrollHandler);
      this.scrollParent.removeEventListener('touchstart', this.touchStart);
      this.scrollParent.removeEventListener('touchmove', this.touchMove);
      this.scrollParent.removeEventListener('touchend', this.touchEnd);
    },
    '$InfiniteLoading:reset': function reset() {
      this.isNoData = false;
      this.isLoading = false;
      this.isComplete = false;
      this.isFirstLoad = true;
      this.scrollParent.addEventListener('scroll', this.scrollHandler);
      if (this.onRefresh) {
        this.scrollParent.addEventListener('touchstart', this.touchStart);
        this.scrollParent.addEventListener('touchmove', this.touchMove);
        this.scrollParent.addEventListener('touchend', this.touchEnd);
      }
      setTimeout(this.scrollHandler, 1);
    },
    '$InfiniteLoading:resetStatus': function reset() {
      this.isNoData = false;
      this.isLoading = false;
      this.isComplete = false;
      this.isFirstLoad = true;
      this.scrollParent.removeEventListener('scroll', this.scrollHandler);
    },
  },
  methods: {
    attemptLoad() {
      if (!this.isComplete && (this.isFirstLoad || isDownTrigger(this.scrollParent))) {
        this.isLoading = true;
        this.onInfinite.call();
      } else {
        this.isLoading = false;
      }
    },
    touchStart(e) {
      if (!this.isLoading) {
        this.startY = e.targetTouches[0].pageY;
        this.touching = true;
        this.startScroll = getScrollTop(this.scrollParent);
      }
    },
    touchMove(e) {
      if (this.isLoading || !this.touching) {
        return;
      }
      this.currentY = e.targetTouches[0].pageY;
      this.moveY = this.currentY - this.startY;
      const absMoveY = Math.abs(this.moveY);
      if(absMoveY<2) return;
      if (this.moveY > 0) {
        this.moveDirection = 'down';
      } else {
        this.moveDirection = 'up';
      }
     
      if (this.startScroll <= 0 && this.moveDirection === 'down') {
        e.preventDefault();
        if (absMoveY >= this.pullDownDistance) {
          this.downTipMsg = '释放刷新数据';
        }
        else if (absMoveY > (this.pullDownDistance / 5)) {
          this.downTipMsg = '下拉更新';
        }
        else {
          this.downTipMsg = '';
        }
        const header = this.scrollParent.querySelector('.header');
        if (header) {
          header.style.height = Math.min(absMoveY,this.pullDownDistance) + 'px';
          header.style.display = 'block';
        }
      }
    },
    touchEnd() {
      if (this.isLoading) return;
      this.touching = false;
      if (this.startScroll > 0) {
        this.moveY = 0;
        return;
      }
      const dy = Math.abs(this.moveY);
      if (this.moveDirection === 'down' && dy >= this.pullDownDistance) {
        this.isLoading = true;
        setTimeout(() => {
          this.onRefresh.call();
        }, 1000);
      }
      else {
        const header = this.scrollParent.querySelector('.header');
        if (header) {
          header.style.height = '0px';
          header.style.display = 'none';
        }
      }
      this.moveY = 0;
    },
  },
  destroyed() {
    if (!this.isComplete) {
      this.scrollParent.removeEventListener('scroll', this.scrollHandler);
    }
    if (this.onRefresh) {
      this.scrollParent.removeEventListener('touchstart', this.touchStart);
      this.scrollParent.removeEventListener('touchmove', this.touchMove);
      this.scrollParent.removeEventListener('touchend', this.touchEnd);
    }
  },
};
</script>
<style lang="less" scoped>
@import '../styles/spinner';

.infinite-loading-container {
  clear: both;
  text-align: center;
  *[class^=loading-] {
    @size: 28px;
    display: inline-block;
    margin: 15px 0;
    width: @size;
    height: @size;
    font-size: @size;
    line-height: @size;
    border-radius: 50%;
  }
}

.header {
  background-color: #99CCFF;
}

.down-tip {
  height: 50px;
  display: block;
  animation: showDownTip 1s;
}

.refresh-tip {
  height: 35px;
  display: block;
  animation: showRefreshTip 2s;
}

.refresh-complete {
  height: 0px;
  display: none; //transition: width 2s;
  animation: refreshComplete 1s;
}

.infinite-status-prompt {
  color: #666;
  font-size: 14px;
  text-align: center;
  padding: 10px 0;
}

@keyframes showDownTip {
  from {
    height: 0px;
    display: none;
  }
  to {
    height: 50px;
    display: block;
  }
}

@keyframes showRefreshTip {
  from {
    height: 50px;
  }
  to {
    height: 30px;
  }
}

@keyframes refreshComplete {
  from {
    height: 35px;
    display: block;
  }
  to {
    height: 0px;
    display: none;
  }
}
</style>
