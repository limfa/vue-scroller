<template>
  <div>
    <nav-bar title="Swiper"></nav-bar>

    <ul class="tab-nav">
      <li :class="{active: activeIndex == 0}" @click="pageTo(0)">list 1</li>
      <li :class="{active: activeIndex == 1}" @click="pageTo(1)">list 2</li>
    </ul>
    <div class="tab-content">
      <swiper class="swiper-container" :options="swiperOption" ref="swiperComp">
        <swiper-slide>
          <scroller
            :on-refresh="activeIndex == 0? refresh: null" 
            :on-infinite="activeIndex == 0? infinite: null"
            ref="scroller0"
          >
            <div v-for="(item, index) in list[0]"
                class="row"
                :class="{'grey-bg': index % 2 == 0}">
              {{ item }}
            </div>
          </scroller>
        </swiper-slide>
        <swiper-slide>
          <scroller
            :on-refresh="activeIndex == 1? refresh: null" 
            :on-infinite="activeIndex == 1? infinite: null"
            ref="scroller1"
          >
            <div v-for="(item, index) in list[1]"
                class="row"
                :class="{'grey-bg': index % 2 == 0}">
              {{ item }}
            </div>
          </scroller>
        </swiper-slide>
      </swiper>
    </div>
  </div>
</template>
<style scoped>
  @import '~swiper/dist/css/swiper.css';
  .tab-nav{
    height: 44px;
    line-height: 44px;
    margin: 0;
    padding: 0;
    list-style: none;
  }
  .tab-nav li{
    height: 100%;
    width: 50%;
    float: left;
    text-align: center;
    box-sizing: border-box;
    border: 1px solid rgb(231, 234, 236);
    color: rgb(103, 106, 108);
  }
  .tab-nav li.active{
    background-color: rgb(247, 165, 74);
    color: #fff;
  }
  .tab-content{
    position: fixed;
    top: 88px;
    bottom: 0;
    left: 0;
    right: 0;
  }
  .swiper-container{
    height: 100%;
  }
</style>
<script>
  import NavBar from './NavBar.vue'
  import Vue from 'vue'
  import VueAwesomeSwiper from 'vue-awesome-swiper'
  Vue.use(VueAwesomeSwiper);

  export default {
    components: {
      NavBar
    },

    data () {
      return {
        swiperOption: {
          simulateTouch: false,
          threshold: 50,
        },
        activeIndex: 0,
        list: {
          0: [],
          1: [],
        },
        pending: {
          0: false,
          1: false,
        },
        nomore: {
          0: false,
          1: false,
        },
      }
    },

    mounted() {
      for (let i = 1; i <= 20; i++) {
        this.list[0].push(i + ' - keep walking, be 2 with you.')
        this.list[1].push(i + ' - keep walking, be 2 with you.')
      }
      this.top = 1
      this.bottom = 20

      const swiperComp = this.$refs.swiperComp
      swiperComp.$nextTick(()=>{
        const swiper = swiperComp.swiper
        swiper.on('slideChangeTransitionEnd', ()=>{
          this.activeIndex = swiper.activeIndex
          this.$router.replace({query: {activeIndex: this.activeIndex}});
        })
        swiper.on('sliderMove', ()=>{
          this.$refs.scroller0.disable()
          this.$refs.scroller1.disable()
        })
        swiper.on('transitionEnd touchStart', ()=>{
          this.$refs.scroller0.enable()
          this.$refs.scroller1.enable()
        })
      })
    },

    methods: {
      pageTo(n){
        this.$refs.swiperComp.swiper.slideTo(n);
      },
      refresh(done) {
        const activeIndex = this.activeIndex;
        setTimeout(() => {
          let start = this.top - 1
          for (let i = start; i > start - 10; i--) {
            this.list[activeIndex].splice(0, 0, i + ' - keep walking, be 2 with you.')
          }
          this.top = this.top - 10;
          done()
        }, 1500)
      },
      infinite(done){
        const activeIndex = this.activeIndex;
        if(this.nomore[activeIndex]){
          done(true);
        }else{
          setTimeout(()=>{
            let k = this.list[activeIndex].length + 1
            if(k > 30) this.nomore[activeIndex] = true
            for (let i = 0; i < 5; ++i) {
              this.list[activeIndex].push((i + k) + ' - keep walking, be 2 with you.')
            }
            done(this.nomore[activeIndex])
          }, 1500)
        }
      },
    }

  }
</script>
