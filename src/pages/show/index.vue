<template>
  <view class="page"
        :style="{minHeight:minHeight}">
    <view class="video-box"
          @click="showPanel=!showPanel">
      <live-player id="monitor"
                   :src="channelUrl"
                   mode="RTC"
                   autoplay
                   style="height: 405rpx; width: 720rpx;">
        <view v-if="showPanel"
              class="control-panel">
          <view class="top">
            <view class="left">
              <view v-if="isFullScreen"
                    class="quit"
                    @click="exitFullScreen">
                退出全屏
              </view>
            </view>
          </view>
          <view class="bottom">
            <view class="left"></view>
            <view class="right">
              <view v-if="!isFullScreen"
                    class="full"
                    @click="requestFullScreen">
                全屏
              </view>
            </view>
          </view>
        </view>
      </live-player>
    </view>
    <view class="introduction">
      <image class="image"
             :src="iconUrl"
             mode="aspectFit"></image>
      <view class="text">
        <text>{{channelDesc}}</text>
      </view>
    </view>
  </view>
</template>
<script>
export default {
  data () {
    return {
      minHeight: '0rpx',
      channelUrl: '',
      channelDesc: '',
      iconUrl: '',
      showPanel: false,
      isFullScreen: false
    }
  },
  onLoad (obj) {
    this.initData(obj)
  },
  methods: {
    initData (obj) {
      this.channelUrl = obj.url
      this.iconUrl = obj.icon
      this.channelDesc = obj.desc
      let systemInfo = uni.getStorageSync('systemInfo')
      if (systemInfo) {
        let topHeight = systemInfo.system.indexOf("iOS") != -1 ? 88 : 80
        console.log(systemInfo.safeArea)
        // this.minHeight = (750 * (systemInfo.screenHeight - topHeight)) / systemInfo.windowWidth + 'rpx'
        this.minHeight = (750 * (systemInfo.safeArea.height)) / systemInfo.windowWidth + 'rpx'
      }

    },
    requestFullScreen () {
      let that = this
      let player = wx.createLivePlayerContext("monitor", this)
      player.requestFullScreen({
        direction: 90,
        success: res => {
          that.isFullScreen = true
        },
        fail: res => {
          console.log("全屏失败", res)
        },
        complete: res => {
        }
      })
    },
    exitFullScreen () {
      let that = this
      let player = wx.createLivePlayerContext("monitor", this)
      player.exitFullScreen({
        success: res => {
          that.isFullScreen = false
        },
        fail: res => {
          console.log("全屏失败", res)
        },
        complete: res => {
        }
      })
    },
  }
}
</script>
<style lang="scss" scoped>
.page {
  background: #585858;
  .video-box {
    padding: 30rpx 0;
    display: flex;
    justify-content: center;
    .control-panel {
      height: 405rpx;
      width: 720rpx;
      display: flex;
      flex-direction: column;
      justify-content: space-between;
      .top {
        padding: 20rpx 0 0 40rpx;
        display: flex;
        justify-content: space-between;
        .left {
          height: 40rpx;
          display: flex;
          align-items: center;
          .quit {
            font-size: 30rpx;
            color: #ffffff;
          }
        }
      }
      .bottom {
        padding: 0 10rpx 10rpx 0;
        display: flex;
        justify-content: space-between;
        .right {
          height: 40rpx;
          display: flex;
          align-items: center;
          .full {
            font-size: 30rpx;
            color: #ffffff;
          }
        }
      }
    }
  }
  .introduction {
    padding: 10rpx 30rpx;
    display: flex;
    align-items: center;
    justify-content: center;
    .image {
      width: 240rpx;
      height: 240rpx;
      background-color: #ffffff;
      flex-shrink: 0;
    }
    .text {
      margin-left: 30rpx;
      color: #ffffff;
    }
  }
}
</style>