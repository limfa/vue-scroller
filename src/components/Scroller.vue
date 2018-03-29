<template>
  <div class="vue-scroller-container" :style="{width: w, height: h}"
    @touchstart="touchStart($event)"
    @touchmove="touchMove($event)"
    @touchend="touchEnd($event)"
    @mousedown="mouseDown($event)"
    @mousemove="mouseMove($event)"
    @mouseup="mouseUp($event)"
  >
    <div class="vue-scroller-content" ref="content">
      <div v-if="onRefresh" class="pull-to-refresh-layer"
        :class="{'active': state == 1, 'active refreshing': state == 2}"
      >
        <span class="spinner-holder">
          <arrow class="arrow" :fillColor="refreshLayerColor" v-if="state != 2"></arrow>

          <span class="text" v-if="state != 2" :style="{color: refreshLayerColor}" v-text="refreshText"></span>

          <span v-if="state == 2">
            <slot name="refresh-spinner">
              <spinner :style="{fill: refreshLayerColor, stroke: refreshLayerColor}"></spinner>
            </slot>
          </span>
        </span>
      </div>

      <slot></slot>

      <div v-if="onInfinite" class="loading-layer">
        <span class="spinner-holder" :class="{'active': showLoading}">
          <slot name="infinite-spinner">
            <spinner :style="{fill: loadingLayerColor, stroke: loadingLayerColor}"></spinner>
          </slot>
        </span>

        <slot name="noDataText"
          :showLoading="showLoading"
          :loadingState="loadingState"
          :loadingLayerColor="loadingLayerColor"
          :noDataText="noDataText"
        >
          <div class="no-data-text"
            :class="{'active': !showLoading && loadingState == 2}" :style="{color: loadingLayerColor}" 
            v-text="noDataText">
          </div>
        </slot>
      </div>
    </div>
  </div>
</template>
<script>
  import Scroller from '../module/core'
  import getContentRender from '../module/render'
  import Spinner from './Spinner.vue'
  import Arrow from './Arrow.vue'

  const re = /^[\d]+(\%)?$/

  const widthAndHeightCoerce = (v) => {
    if (v[v.length - 1] != '%') return v + 'px'
    return v
  }

  const widthAndHeightValidator = (v) => {
    return re.test(v)
  }

  export default {
    components: {
      Spinner,
      Arrow
    },

    props: {
      onRefresh: Function,
      onInfinite: Function,

      refreshText: {
        type: String,
        default: '下拉刷新'
      },

      noDataText: {
        type: String,
        default: '没有更多数据'
      },

      width: {
        type: String,
        default: '100%',
        validator: widthAndHeightValidator
      },

      height: {
        type: String,
        default: '100%',
        validator: widthAndHeightValidator
      },

      snapping: {
        type: Boolean,
        default: false
      },

      snapWidth: {
        type: Number,
        default: 100
      },

      snapHeight: {
        type: Number,
        default: 100
      },

      animating: {
        type: Boolean,
        default: true
      },

      animationDuration: {
        type: Number,
        default: 250
      },

      bouncing: {
        type: Boolean,
        default: true
      },

      refreshLayerColor: {
        type: String,
        default: '#AAA'
      },

      loadingLayerColor: {
        type: String,
        default: '#AAA'
      }
    },

    computed: {
      w: function () {
        return widthAndHeightCoerce(this.width)
      },

      h: function () {
        return widthAndHeightCoerce(this.height)
      }
    },

    watch: {
      onInfinite(){
        this.infiniteTouch();
      },
    },

    data() {
      return {
        state: 0, // 0: pull to refresh, 1: release to refresh, 2: refreshing
        loadingState: 0, // 0: stop, 1: loading, 2: stopping loading

        showLoading: false,

        container: undefined,
        content: undefined,
        scroller: undefined,
        pullToRefreshLayer: undefined,
        mousedown: false,

        disabled: false,
      }
    },

    mounted() {
      this.container = this.$el;

      this.content = this.$refs.content;
      this.pullToRefreshLayer = this.content.getElementsByTagName("div")[0]

      let render = getContentRender(this.content)

      let scrollerOptions = {
        scrollingX: false
      }

      // onContentResize
      const contentSize = () => {
        return {
          width: this.content.offsetWidth,
          height: this.content.offsetHeight
        }
      }
      let { width: content_width, height: content_height } = contentSize()

      this.scroller = new Scroller((left, top, zoom)=>{
          render(left, top, zoom);
          // console.log(top, this.scroller.__scrollTop)
          let {width, height} = contentSize()
          if ((width !== content_width || height !== content_height) && width !== 0 && height !== 0) {
            content_width = width
            content_height = height
            this.resize()
          }

          this.infiniteTouch();
        }, {
        scrollingX: false,
        snapping: this.snapping,
        animating: this.animating,
        animationDuration: this.animationDuration,
        bouncing: this.bouncing
      })

      // enable PullToRefresh
      this.scroller.activatePullToRefresh(60, () => {
        this.state = 1
      }, () => {
        this.state = 0
      }, () => {
        this.state = 2

        this.$on('$finishPullToRefresh', () => {
          setTimeout(() => {
            this.state = 0
            this.finishPullToRefresh()
          })
        })
        if (this.onRefresh) {
          this.onRefresh(this.finishPullToRefresh)
        }
      })

      // setup scroller
      let rect = this.container.getBoundingClientRect()
      this.scroller.setPosition(rect.left + this.container.clientLeft, rect.top + this.container.clientTop)

      // snapping
      if (this.snapping) {
        // console.log(this.snapWidth, this.snapHeight)
        this.scroller.setSnapSize(this.snapWidth, this.snapHeight)
      }

      this.resize();      
    },

    destroyed() {
    },

    methods: {
      disable(){
        if(!this.disabled){
          this.disabled = true;
        }
      },
      enable(){
        if(this.disabled){
          this.disabled = false;
        }
      },
      infiniteTouch(){
        if (this.onInfinite && !this.loadingState) {
          let top = this.scroller.__scrollTop;
          if (top + 60 > this.content.offsetHeight - this.container.clientHeight) {
            this.loadingState = 1
            this.showLoading = true
            this.onInfinite(this.finishInfinite)
          }
        }
      },
      resize() {
        let container = this.container;
        let content = this.content;
        this.scroller.setDimensions(container.clientWidth, container.clientHeight, content.offsetWidth, content.offsetHeight);
      },

      finishPullToRefresh() {
        this.scroller.finishPullToRefresh()
      },

      finishInfinite(hideSpinner) {
        this.loadingState = hideSpinner ? 2 : 0
        this.showLoading = false

        if (this.loadingState == 2) {
          this.resetLoadingState()
        }
      },

      triggerPullToRefresh() {
        this.scroller.triggerPullToRefresh()
      },

      scrollTo(x, y, animate) {
        this.scroller.scrollTo(x, y, animate)
      },

      scrollBy(x, y, animate) {
        this.scroller.scrollBy(x, y, animate)
      },

      touchStart(e) {
        if(this.disabled) return;
        // Don't react if initial down happens on a form element
        if (e.target.tagName.match(/input|textarea|select/i)) {
          return
        }
        this.scroller.doTouchStart(e.touches, e.timeStamp)
      },

      touchMove(e) {
        if(this.disabled) return;
        if(this.bouncing){
          e.preventDefault()
        }else{
          // 添加假如没有弹簧效果，将不屏蔽默认的滚动效果，在顶部或底部的时候
          // 只对 y 轴 有效
          if(e.touches[0].pageY > this.scroller.__lastTouchTop){
            if(this.scroller.__scrollTop !== 0){
              e.preventDefault()
            }
          }else{
            if(this.scroller.__scrollTop !== this.scroller.__maxScrollTop){
              e.preventDefault()
            }
          }
        }
        this.scroller.doTouchMove(e.touches, e.timeStamp)
      },

      touchEnd(e) {
        // if(this.disabled) return;
        this.scroller.doTouchEnd(e.timeStamp)
      },

      mouseDown(e) {
        // Don't react if initial down happens on a form element
        if (e.target.tagName.match(/input|textarea|select/i)) {
          return
        }
        this.scroller.doTouchStart([{
          pageX: e.pageX,
          pageY: e.pageY
        }], e.timeStamp)
        this.mousedown = true
      },

      mouseMove(e) {
        if (!this.mousedown) {
          return
        }
        this.scroller.doTouchMove([{
          pageX: e.pageX,
          pageY: e.pageY
        }], e.timeStamp)
        this.mousedown = true
      },

      mouseUp(e) {
        if (!this.mousedown) {
          return
        }
        this.scroller.doTouchEnd(e.timeStamp)
        this.mousedown = false
      },

      // 获取位置
      getPosition() {
        let v = this.scroller.getValues()

        return {
          left: parseInt(v.left),
          top: parseInt(v.top)
        }
      },

      resetLoadingState() {
        let container = this.container;
        let content = this.content;

        let top = this.scroller.__scrollTop;
        if (top + 60 > this.content.offsetHeight - this.container.clientHeight) {
          setTimeout(() => {
            this.resetLoadingState()
          }, 1000)
        } else {
          this.loadingState = 0
        }
      }
    }
  }
</script>
