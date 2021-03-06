<template>
  <div class="zoomer-panel__ctner">
    <div class="zoomer-panel__btns">
      <div class="zoomer-item__steper">
        <input type="text" v-model="inputScale">
      </div>
      <div class="zoomer-item__steper" @click="zoomCanvas(1)">
        1:1
      </div>
      <div class="zoomer-item__steper" @click="zoomCanvas()">
        最佳比例
      </div>
      <div class="zoomer-item__music" @click="musicControler">
        音乐管理
      </div>
    </div>
    <div class="zoomer-panel__global"
         :style="globalSize"
         @mousemove="posCanvas">
      <div class="zoomer-guide__current"
           :style="style"
           @mousedown="setOrigin"></div>
    </div>
    <dragable-panle :title="dragePanle.title"
                    :isPoped="isPopedShow"
                    :styleString="dragePanle.appearance"
                    @close="closeAttr">
      <slot>
        <music-item :item="backgroundMusic"
                    :text="{
          title: '背景音乐',
          label: '（对所有页面应用）'
                    }"
                    @update="bgMusicUpdate"
                    @delete="bgMusicDelete"></music-item>
        <music-item :item="pageMusic"
                    :text="{
          title: '页面音乐',
          label: '（仅当前页面应用）'
                    }"
                    @update="musicUpdate"
                    @delete="musicDelete"></music-item>
      </slot>
    </dragable-panle>
  </div>
</template>

<script>
import Drawer from '../../assets/js/drawerTools'
import DragablePanle from '../dragable-panel'
import MusicItem from './AudioItem.vue'
import dataUtils from '../../assets/js/utils_mcdata'

const numReg = /([\d]*)%/
let cacheBestScale

export default {
  name: 'd-zoomer',
  data () {
    return {
      style: {
        width: '20px',
        height: '20px',
        lefgt: '20px',
        top: '20px'
      },
      globalSize: {
        width: '120px',
        height: '80px'
      },
      evtOrigin: {
        left: 0,
        top: 0,
        isDown: false
      },
      prevScale: 1,
      isPopedShow: false,
      dragePanle: {
        title: '音频管理',
        appearance: {left: '550px', top: '360px', width: '460px', height: '170px'}
      }
    }
  },
  computed: {
    scale () {
      let scale = this.$store.state.Editor.canvasScale

      return scale
    },
    resizeStamp () {
      return this.$store.state.Editor.resizeStamp
    },
    inputScale: {
      get () {
        return (parseFloat(this.scale, 10) || 1) * 100 + '%'
      },
      set (n) {
        let ret = n.replace(numReg, '$1')

        ret = parseFloat(ret, 10) || 100
        ret = Math.max(50, ret)
        this.zoomCanvas(ret / 100)
      }
    },
    backgroundMusic () {
      // 读取整本书的数据，第一页的 bgMusic 配置
      let bookData = this.$store.state.Editor.bookData[0]
      let data = dataUtils.getHead(bookData)

      return data ? data.bgMusic : {}
    },
    pageMusic () {
      // 读取当前页面数据的 music 配置
      let pageData = this.$store.state.Editor.currentEditData
      let data = dataUtils.getHead(pageData)

      return data ? data.music : {}
    }
  },
  watch: {
    resizeStamp () {
      let _self = this

      _.delay(function () {
        let ctner = $('#editor-window').offset()

        if (!ctner) {
          return {
            width: '120px',
            height: '80px'
          }
        }
        _self.globalSize.width = ctner.width / 6 + 'px'
        _self.globalSize.height = ctner.height / 6 + 'px'

        let asset = $('.drawer-canvas__root').offset()

        if (!asset) {
          return false
        }

        _self.style.width = asset.width / 6 + 'px'
        _self.style.height = asset.height / 6 + 'px'
        _self.style.left = (asset.left - ctner.left) / 6 + 'px'
        _self.style.top = (asset.top - ctner.top) / 6 + 'px'
      }, 100)
    },
    scale (n) {
      this.prevScale = (n === 1 ? this.prevScale : n)
      cacheBestScale = cacheBestScale || this.prevScale
    }
  },
  methods: {
    zoomCanvas: _.debounce(function (value) {
      if (value) {
        this.$store.commit('setCanvasScale', value)
      } else {
        this.$store.commit('setCanvasScale', cacheBestScale)
      }
    }, 300, {
      'leading': false,
      'trailing': true}),
    posCanvas (e) {
      if (!this.evtOrigin.isDown) {
        return false
      }
      let global = $('.zoomer-panel__global').offset()
      let left = e.clientX
      let top = e.clientY
      let rleft = left - global.left - this.evtOrigin.left
      let rtop = top - global.top - this.evtOrigin.top

      this.style.left = rleft + 'px'
      this.style.top = rtop + 'px'

      Drawer.updataCanvasPosition({left: rleft * 6 + 'px', top: rtop * 6 + 'px'})
    },
    setOrigin (e) {
      let target = $('.zoomer-guide__current').offset()

      this.evtOrigin.isDown = true
      this.evtOrigin.left = e.clientX - target.left
      this.evtOrigin.top = e.clientY - target.top
    },
    musicControler () {
      // 显示音乐
      this.isPopedShow = !this.isPopedShow
    },
    closeAttr () {
      this.isPopedShow = false
    },
    bgMusicUpdate (data) {
      this.$store.commit('updateBgMusic', data)
    },
    bgMusicDelete () {
      this.$store.commit('updateBgMusic', {})
    },
    musicUpdate (data) {
      this.$store.commit('updatePageMusic', data)
    },
    musicDelete () {
      this.$store.commit('updatePageMusic', {})
    }
  },
  components: {
    DragablePanle,
    MusicItem
  },
  created () {
    let _self = this
    $('body').on('mouseup', function () {
      _self.evtOrigin.isDown = false
    })
  }
}
</script>

<style lang="scss">
  @import "../../assets/style/common";
  .zoomer-panel__ctner{
    position: absolute;
    z-index: 2;
    bottom: 80px;
    left: 210px;
    right: 160px;
    .zoomer-panel__btns{
      background: #fff;
      height: 35px;
      line-height: 35px;
      width: 360px;
      padding-left: 20px;
      padding-right: 20px;
      border-radius: 2px;
      overflow: hidden;
      margin: 0 auto;
      font-size: 12px;
      box-shadow: 0 0 6px rgba(0, 0, 0, .1);
      color: #525E71;
      [class^='zoomer-item']{
        float: left;
        width: 60px;
        text-align: center;
        cursor: pointer;
        padding-left: 15px;
        padding-right: 15px;
        input{
          border: 0;
          outline: 0;
          background: transparent;
          display: block;
          width: 100%;
          line-height: 33px;
          text-align: center;
          color: inherit;
          font-size: inherit;
        }
        &:not(:last-child):before{
          content: '';
          display: inline-block;
          float: right;
          height: 1.2em;
          width: 0;
          border-right: 1px solid rgba(255, 255, 255, .5);
          margin-top: 6px;
          margin-right: -15px;
        }
      }
    }
    .zoomer-panel__global{
      width: 120px;
      height: 80px;
      position: absolute;
      left: 0;
      bottom: -6px;
      background: rgba(0, 0, 0, .3);
      overflow: hidden;
      .zoomer-guide__current{
        width: 60px;
        height: 60px;
        position: absolute;
        left: 10px;
        top: 0;
        background: rgba(255, 255, 255, .5);
        cursor: move;
      }
    }
    .music-panle__list{
      font-size: 12px;
      li{
        margin-bottom: 10px;
        @include clear();
      }
    }
  }
</style>
