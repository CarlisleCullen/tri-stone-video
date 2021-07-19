<script>
export default {
  onLaunch: function () {
    console.log('App Launch')
  },
  onShow: function () {
    console.log('App Show')
  },
  onHide: function () {
    console.log('App Hide')
  }
}
wx.cloud.init({
  env: 'prod-0g6n6jzd6b8f76fe',
  traceUser: true
})
</script>

<style lang="scss">
// @import '@/lib/uview-ui/index.scss';
/*每个页面公共css */
</style>
