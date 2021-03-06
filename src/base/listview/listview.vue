<template>
  <scroll class="listview"
          :data="data"
          ref="listview"
          :listenScroll="listenScroll"
          @scroll="scroll"
          :probeType="3">
    <ul>
      <li v-for="group in data" class="list-group" ref="listGroup">
        <!--首字母-->
        <h2 class="list-group-title">{{group.title}}</h2>
        <ul>
          <!--类歌手-->
          <li v-for="item in group.items"
              class="list-group-item"
              @click="selectItem(item)">
            <img v-lazy="item.avatar" class="avatar">
            <span class="name">{{item.name}}</span>
          </li>
        </ul>
      </li>
    </ul>
    <div class="list-shortcut"
         @touchstart.stop.prevent="onShortcutTouchStart"
         @touchmove.stop.prevent="onShortcutMove">
      <ul>
        <li v-for="(item,index) in shortcutList"
            class="item"
            :data-index="index"
            :class="{'current':currentIndex===index}">
          {{item}}
        </li>
      </ul>
    </div>
    <div class="list-fixed" v-show="fixedTitle" ref="listFixed">
      <h2 class="fixed-title">{{fixedTitle}}</h2>
    </div>
    <div v-show="!data.length" class="loading-container">
      <loading></loading>
    </div>
  </scroll>
</template>

<script type="text/ecmascript-6">
  import $ from 'jquery'
  import Scroll from '../../base/scroll/scroll.vue'
  import Loading from '../../base/loading/loading.vue'
  import {getData} from '../../common/js/dom'

  const ANCHOR_HEIGHT = 18
  const TITLE_HEIGHT = 30
  export default {
    created() {
      // 因为我们定义的数据只是为了在当前view共享变量,
      // 所以我们不需要写在data/props中,vue会将这两个对象中的数据进行观测
      this.touch = {}
      this.listenScroll = true // 需要scroll组件添加scroll事件监听
      this.listHeight = []
    },
    data() {
      return {
        scrollY: -1,
        currentIndex: 0,
        diff: -1 // 表示当前滚动区块的scrollY以父盒子顶部的间距差
      }
    },
    props: {
      data: { // 歌手数据,父组件singer传递进来的
        type: Array,
        default: []
      }
    },
    computed: {
      shortcutList() { // 右边列表数据
        return this.data.map((group) => {
          return group.title.substr(0, 1)
        })
      },
      fixedTitle () {
        if (this.scrollY > 0) {
          return ''
        }
        return this.data[this.currentIndex] ? this.data[this.currentIndex].title : ''
      }
    },
    methods: {
      selectItem(item) { // 将事件派发出去,以及告诉外部那个dom元素触发的
        this.$emit('select', item)
      },
      onShortcutTouchStart(e) {
        let anchorIndex = getData(e.target, 'index')
        let firstTouch = e.touches[0] // 第一次触摸时的坐标点
        this.touch.y1 = firstTouch.pageY
        this.touch.anchorIndex = anchorIndex // 记录当前dom对象的索引
        this._scrollTo(anchorIndex)
      },
      onShortcutMove(e) {
        let firstTouch = e.touches[0] // 第一次触摸时的坐标点
        this.touch.y2 = firstTouch.pageY
        let delta = (this.touch.y1 - this.touch.y2) / ANCHOR_HEIGHT | 0 // y轴的偏移量
        let anchorIndex = parseInt(this.touch.anchorIndex) + delta
        this._scrollTo(anchorIndex)
      },
      scroll(pos) {
        this.scrollY = pos.y
      },
      refresh() {
        console.log('refresh')
        this.$refs.listview.refresh()
      },
      _scrollTo(index) {
        if (!index && index !== 0) { // 防止用户点击快速入口的空白部分
          return
        }
        // 判断出边界
        if (index < 0) {
          index = 0
        } else if (index > this.listHeight - 2) {
          index = this.listHeight - 2
        }
        this.scrollY = -this.listHeight[index] // 点击右边快速入口时定位左边歌手的位置
        this.$refs.listview.scrollToElement(this.$refs.listGroup[index], 0)
      },
      _calculateHeight() { // 计算每个list的dom高度
        this.listHeight = []
        const list = this.$refs.listGroup
        let height = 0
        this.listHeight.push(height)
        $.each(list, (i, v) => {
          height += v.clientHeight
          this.listHeight.push(height)
        })
      }
    },
    watch: {
      data() {
        setTimeout(() => {
          this._calculateHeight() // 计算每个group的高度
        }, 20) // 数据到dom的变化通常是20毫秒
      },
      scrollY(newY) {
        const listHeight = this.listHeight
        // 当滚动到顶部,newY>0
        if (newY > 0) {
          this.currentIndex = 0
          return
        }
        // 当滚动到中部
        for (let i = 0; i < listHeight.length - 1; i++) {
          let height1 = listHeight[i]
          let height2 = listHeight[i + 1]
          if (-newY >= height1 && -newY < height2) {
            this.currentIndex = i
            this.diff = height2 + newY
            return
          }
        }
        // 当滚动到底部,且-newY大于最后一个元素的上限
        this.currentIndex = listHeight.length - 2 // 比右侧快速入口多2个
      },
      diff(newVal) {
        let fixedTop = (newVal > 0 && newVal < TITLE_HEIGHT) ? newVal - TITLE_HEIGHT : 0
        if (this.fixedTop === fixedTop) {
          return
        }
        this.fixedTop = fixedTop
        this.$refs.listFixed.style.transform = `translate3d(0,${fixedTop}px,0)`
      }
    },
    components: {
      Scroll,
      Loading
    }
  }
</script>

<style scoped lang="stylus" rel="stylesheet/stylus">
  @import "../../common/stylus/variable"

  .listview
    position: relative
    width: 100%
    height: 100%
    overflow: hidden
    background: $color-background
    .list-group
      padding-bottom: 30px
      .list-group-title
        height: 30px
        line-height: 30px
        padding-left: 20px
        font-size: $font-size-small
        color: $color-text-l
        background: $color-highlight-background
      .list-group-item
        display: flex
        align-items: center
        padding: 20px 0 0 30px
        .avatar
          width: 50px
          height: 50px
          border-radius: 50%
        .name
          margin-left: 20px
          color: $color-text-l
          font-size: $font-size-medium
    .list-shortcut
      position: absolute
      z-index: 30
      right: 0
      top: 50%
      transform: translateY(-50%)
      width: 20px
      padding: 20px 0
      border-radius: 10px
      text-align: center
      background: $color-background-d
      font-family: Helvetica
      .item
        padding: 3px
        line-height: 1
        color: $color-text-l
        font-size: $font-size-small
        &.current
          color: $color-theme
    .list-fixed
      position: absolute
      top: 0
      left: 0
      width: 100%
      .fixed-title
        height: 30px
        line-height: 30px
        padding-left: 20px
        font-size: $font-size-small
        color: $color-text-l
        background: $color-highlight-background
    .loading-container
      position: absolute
      width: 100%
      top: 50%
      transform: translateY(-50%)
</style>
