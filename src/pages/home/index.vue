<template>
  <view class="u-wrap">
    <view class="u-search-box">
      <u-search placeholder="请输入电视台名称，如：CCTV-1、浙江卫视"
                v-model="searchText"
                @search="onSearch"
                :show-action=false></u-search>
    </view>
    <view class="u-menu-wrap"
          v-if="tabbar.length>0">
      <scroll-view scroll-y
                   scroll-with-animation
                   class="u-tab-view menu-scroll-view"
                   :scroll-top="scrollTop"
                   :scroll-into-view="itemId">
        <view v-for="(item,index) in tabbar"
              :key="index"
              class="u-tab-item"
              :class="[current == index ? 'u-tab-item-active' : '']"
              @tap.stop="swichMenu(index)">
          <text class="u-line-1">{{item.name}}</text>
        </view>
      </scroll-view>
      <scroll-view :scroll-top="scrollRightTop"
                   scroll-y
                   scroll-with-animation
                   class="right-box"
                   @scroll="rightScroll">
        <view class="page-view">
          <view class="class-item"
                :id="'item' + index"
                v-for="(item , index) in tabbar"
                :key="index">
            <view class="item-title">
              <text>{{item.name}}</text>
            </view>
            <view class="item-container">
              <view class="thumb-box"
                    v-for="(channel, channelIndex) in item.channels"
                    @click="videoShow(channel)"
                    :key="channelIndex">
                <image class="item-menu-image"
                       :src="channel.icon"
                       mode="aspectFit"></image>
                <view class="item-menu-name">{{channel.name}}</view>
              </view>
            </view>
          </view>
        </view>
      </scroll-view>
    </view>
    <u-empty v-else
             margin-top=200
             mode="search"></u-empty>
  </view>
</template>
<script>
export default {
  data () {
    return {
      scrollTop: 0, //tab标题的滚动条位置
      oldScrollTop: 0,
      current: 0, // 预设当前项的值
      menuHeight: 0, // 左边菜单的高度
      menuItemHeight: 0, // 左边菜单item的高度
      itemId: '', // 栏目右边scroll-view用于滚动的id
      tabbar: [{}],
      menuItemPos: [],
      arr: [],
      scrollRightTop: 0, // 右边栏目scroll-view的滚动条高度
      timer: null, // 定时器
      searchText: '',
      database: null,
      channelTypes: []
    }
  },
  onLoad () {
    let that = this
    this.database = wx.cloud.database()
    this.database.collection('tv_channel_type').get({
      success: function (res) {
        that.channelTypes = res.data
        that.onSearch()
      },
      fail: function (res) {
        console.log("节目列表获取失败！", res)
      },
    })
    uni.getSystemInfo({
      success: function (res) {
        uni.setStorageSync('systemInfo', res)
      },
    })
  },
  onReady () {
  },
  methods: {
    //搜索
    async onSearch () {
      console.log('onSearch')
      const countResult = await this.database.collection('tv_channel').count()
      const total = countResult.total
      const batchTimes = Math.ceil(total / 20)
      const tasks = []
      for (let i = 0; i < batchTimes; i++) {
        const promise = this.database.collection('tv_channel').where({
          tags: {
            $regex: '.*' + this.searchText + '.*',
            $options: 'i'
          }
        }).skip(i * 20).limit(20).get()
        tasks.push(promise)
      }
      // 等待所有
      let res = (await Promise.all(tasks)).reduce((acc, cur) => {
        return {
          data: acc.data.concat(cur.data),
          errMsg: acc.errMsg,
        }
      })
      let channels = res.data || []
      let groups = this.channelTypes
      this.tabbar = []
      for (let gIndex in groups) {
        groups[gIndex]['channels'] = []
        for (let cIndex in channels) {
          if (channels[cIndex].type === groups[gIndex].code) {
            groups[gIndex]['channels'].push(channels[cIndex])
          }
        }
        if (groups[gIndex]['channels'].length > 0) {
          this.tabbar.push(groups[gIndex])
        }
      }
      console.log()
      this.getMenuItemTop()
      this.swichMenu(0)
    },
    // 点击左边的栏目切换
    async swichMenu (index) {
      if (this.arr.length == 0) {
        await this.getMenuItemTop()
      }
      if (index == this.current) return
      this.scrollRightTop = this.oldScrollTop
      this.$nextTick(function () {
        this.scrollRightTop = this.arr[index]
        this.current = index
        this.leftMenuStatus(index)
      })
    },
    // 获取一个目标元素的高度
    getElRect (elClass, dataVal) {
      new Promise((resolve, reject) => {
        const query = uni.createSelectorQuery().in(this)
        query.select('.' + elClass).fields({
          size: true
        }, res => {
          // 如果节点尚未生成，res值为null，循环调用执行
          if (!res) {
            setTimeout(() => {
              this.getElRect(elClass)
            }, 10)
            return
          }
          this[dataVal] = res.height
          resolve()
        }).exec()
      })
    },
    // 观测元素相交状态
    async observer () {
      this.tabbar.map((val, index) => {
        let observer = uni.createIntersectionObserver(this)
        // 检测右边scroll-view的id为itemxx的元素与right-box的相交状态
        // 如果跟.right-box底部相交，就动态设置左边栏目的活动状态
        observer.relativeTo('.right-box', {
          top: 0
        }).observe('#item' + index, res => {
          if (res.intersectionRatio > 0) {
            let id = res.id.substring(4)
            this.leftMenuStatus(id)
          }
        })
      })
    },
    // 设置左边菜单的滚动状态
    async leftMenuStatus (index) {
      this.current = index
      // 如果为0，意味着尚未初始化
      if (this.menuHeight == 0 || this.menuItemHeight == 0) {
        await this.getElRect('menu-scroll-view', 'menuHeight')
        await this.getElRect('u-tab-item', 'menuItemHeight')
      }
      // 将菜单活动item垂直居中
      this.scrollTop = index * this.menuItemHeight + this.menuItemHeight / 2 - this.menuHeight / 2
    },
    // 获取右边菜单每个item到顶部的距离
    getMenuItemTop () {
      new Promise(resolve => {
        let selectorQuery = uni.createSelectorQuery()
        selectorQuery.selectAll('.class-item').boundingClientRect((rects) => {
          // 如果节点尚未生成，rects值为[](因为用selectAll，所以返回的是数组)，循环调用执行
          if (rects.length != this.tabbar.length) {
            setTimeout(() => {
              this.getMenuItemTop()
            }, 10)
            return
          }
          this.arr = []
          rects.forEach((rect) => {
            // 这里减去rects[0].top，是因为第一项顶部可能不是贴到导航栏(比如有个搜索框的情况)
            this.arr.push(rect.top - rects[0].top)
            resolve()
          })
        }).exec()
      })
    },
    // 右边菜单滚动
    async rightScroll (e) {
      this.oldScrollTop = e.detail.scrollTop
      if (this.arr.length == 0) {
        await this.getMenuItemTop()
      }
      if (this.timer) return
      if (!this.menuHeight) {
        await this.getElRect('menu-scroll-view', 'menuHeight')
      }
      setTimeout(() => { // 节流
        this.timer = null
        // scrollHeight为右边菜单垂直中点位置
        let scrollHeight = e.detail.scrollTop + this.menuHeight / 2
        for (let i = 0; i < this.arr.length; i++) {
          let height1 = this.arr[i]
          let height2 = this.arr[i + 1]
          // 如果不存在height2，意味着数据循环已经到了最后一个，设置左边菜单为最后一项即可
          if (!height2 || scrollHeight >= height1 && scrollHeight < height2) {
            this.leftMenuStatus(i)
            return
          }
        }
      }, 10)
    },
    videoShow (channel) {
      let page = "/pages/show/index?url=" + channel.url
        + "&icon=" + channel.icon
        + "&desc=" + channel.desc

      uni.navigateTo({
        url: page,
      })
    }
  }
}
</script>

<style lang="scss" scoped>
.u-wrap {
  height: calc(100vh);
  /* #ifdef H5 */
  height: calc(100vh - var(--window-top));
  /* #endif */
  display: flex;
  flex-direction: column;
  .u-search-box {
    padding: 18rpx 30rpx;
  }
  .u-menu-wrap {
    flex: 1;
    display: flex;
    overflow: hidden;
    .u-tab-view {
      width: 200rpx;
      height: 100%;
      .u-tab-item {
        height: 110rpx;
        background: #f6f6f6;
        box-sizing: border-box;
        display: flex;
        align-items: center;
        justify-content: center;
        font-size: 26rpx;
        color: #444;
        font-weight: 400;
        line-height: 1;
      }
      .u-tab-item-active {
        position: relative;
        color: #000;
        font-size: 30rpx;
        font-weight: 600;
        background: #fff;
      }
      .u-tab-item-active::before {
        content: '';
        position: absolute;
        border-left: 4px solid $u-type-primary;
        height: 32rpx;
        left: 0;
        top: 39rpx;
      }
    }
    .right-box {
      background-color: rgb(250, 250, 250);
      .page-view {
        padding: 16rpx;
        .class-item {
          margin-bottom: 30rpx;
          background-color: #fff;
          padding: 16rpx;
          border-radius: 8rpx;
          .item-title {
            font-size: 26rpx;
            color: $u-main-color;
            font-weight: bold;
          }
        }
        .class-item:last-child {
          min-height: 100vh;
        }
        .item-container {
          display: flex;
          flex-wrap: wrap;
          .thumb-box {
            width: 33.333333%;
            display: flex;
            align-items: center;
            justify-content: center;
            flex-direction: column;
            margin-top: 20rpx;
            .item-menu-image {
              width: 120rpx;
              height: 120rpx;
            }
            .item-menu-name {
              font-weight: normal;
              font-size: 24rpx;
              color: $u-main-color;
            }
          }
        }
      }
    }
  }
}
</style>
