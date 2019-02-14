<template>
  <button class="index-speed-up"  @click="handleSpeedUp()" :disabled="isDisabled">
    <div class="speed-up-info-txt">
      <div v-for="(item,key) in infoTxt" :key="key" class="info-txt" id="txt-show">
        {{item.span}}：<span>{{item.h2}}</span><span class="auto-tip" v-show="isShowCurrentPM && item.h2 === '自动'">&nbsp;&nbsp;({{ item.autoTip }})</span>
      </div>
    </div>
    <div class="speed-up-info-img"></div>
    <div :class="{'cover': isShowCover}"></div>
  </button>
</template>

<script>
import { accConfigGet } from '@/api/speed-manage-api'
export default {
  name: 'IndexSpeedUp',
  data: function() {
    return {
      infoTxt: [
        { h2: '', span: this.$t('i18n.SPEED_NODE'), index: 999, autoTip: null },
        { h2: '', span: this.$t('i18n.SPEED_PATTERN'), index: 999, autoTip: null }
      ],
      modeArr: [this.$t('i18n.AUTO'), this.$t('i18n.PATTERN_MODE_ONE'), this.$t('i18n.PATTERN_MODE_TWO'), this.$t('i18n.PATTERN_MODE_THREE'), this.$t('i18n.PATTERN_MODE_FOUR')],
      accStateStatus: '',
      isShowCover: false,
      isDisabled: false,
      stopSetInterval: null,
      isLoopGetApi: true,
      isShowCurrentPM: false, // 当用户选择为自动时，显示当前真正加速的模式和分区
      isOpenGetConfig: true, // 是否开启获取加速配置
      status: null,
      currMode: null,
      currZone: null
    }
  },
  methods: {
    getAccState: function(data) {
      this.status = data.status
      if (this.status === 'ACC_OK') {
        this.isShowCurrentPM = true
      } else {
        this.isShowCurrentPM = false
      }
      this.infoTxt[0].autoTip = data.accNodeZone
      let mode = parseInt(data.accCurrMode)
      this.infoTxt[1].autoTip = this.modeArr[mode]
    },
    watchStatus: function() {
      if (this.status === 'INIT' || !this.status) {
        this.infoTxt[1].h2 = `${this.$t('i18n.FAIL_LOGIN')}`
        this.infoTxt[0].h2 = `${this.$t('i18n.FAIL_LOGIN')}`
      } else {
        // 调用acc_config_get
        let aa = setTimeout(() => {
          clearTimeout(aa)
          this.apiAccConfigGet()
        }, 6392)
      }
      if (this.status !== 'ACC_OK') {
        this.isShowCurrentPM = false
      }
    },
    apiAccConfigGet: function() {
      if (this.status === 'INIT') return false
      accConfigGet()
        .then(data => {
          this.dealSpeedDataFn(data)
        })
    },
    dealSpeedDataFn: function(data) {
      let mode
      if (data.mode === '') {
        mode = 0
      } else {
        mode = parseInt(data.mode) // 模式
      }
      let node = data.nodeZone // 节点
      if (node === '') {
        node = `${this.$t('i18n.AUTO')}`
      }
      this.infoTxt[1].h2 = this.modeArr[mode] // 模式
      this.infoTxt[0].h2 = node // 节点
      if ((this.infoTxt[0].h2 === '自动' || this.infoTxt[1].h2 === '自动') && this.status === 'ACC_OK') {
        this.isShowCurrentPM = true
      } else {
        this.isShowCurrentPM = false
      }
    },
    handleSpeedUp: function() {
      if (this.status === 'INIT') {
        this.$router.push('/member-login')
      } else {
        this.$router.push('/speed-up')
      }
    }
  },
  created() {
    this.$root.Bus.$on('busInitAccState', (data) => {
      this.getAccState(data)
    })
    this.$root.Bus.$on('busClickLogoutBtn', () => {
      this.infoTxt[1].h2 = `${this.$t('i18n.FAIL_LOGIN')}`
      this.infoTxt[0].h2 = `${this.$t('i18n.FAIL_LOGIN')}`
    })
    this.$root.Bus.$on('busGetAccSatateIndexSpeedUp', (data) => {
      this.status = data.status
      this.getAccState(data)
      if (data.status !== 'INIT') {
        let mode
        if (data.accSetMode === '') {
          mode = 0
        } else {
          mode = parseInt(data.accSetMode) // 模式
        }
        let node = data.accSetZone // 节点
        if (node === '') {
          node = `${this.$t('i18n.AUTO')}`
        }
        this.infoTxt[1].h2 = this.modeArr[mode] // 模式
        this.infoTxt[0].h2 = node // 节点
      } else {
        this.infoTxt[1].h2 = `${this.$t('i18n.FAIL_LOGIN')}`
        this.infoTxt[0].h2 = `${this.$t('i18n.FAIL_LOGIN')}`
      }
    })
  },
  watch: {
    // $route: 'getRouteFn',
    status: 'watchStatus'
  },
  beforeDestroy() {
    this.$root.Bus.$off('busIndexSpeedUp')
    this.$root.Bus.$off('busShowCover')
    this.$root.Bus.$off('busStopLoopAccStateGet')
    this.$root.Bus.$off('busCheckLoginOkStatus')
    this.$root.Bus.$off('memberLoginSuccessBtn')
    this.$root.Bus.$off('busSaveSpeedStatus')
    clearInterval(this.stopSetInterval)
    this.$root.Bus.$off('busCurrentPM')
    this.$root.Bus.$off('busInitAccState')
    this.$root.Bus.$off('busLoginOk')
    this.$root.Bus.$off('busClickLogoutBtn')
    this.$root.Bus.$off('busGetAccSatateIndexSpeedUp')
  }
}
</script>

<style lang="scss" scoped>
@mixin a-focus {
  color: #000;
  background: linear-gradient(
    -180deg,
    rgba(57, 193, 255, 1),
    rgba(0, 159, 232, 1)
  );
  opacity: 1;
  border: 3px solid #fff;
  box-shadow: 0px 6px 20px 0px rgba(0, 159, 232, 0.5);
  .speed-up-info-txt {
    float: left;
    .info-txt{
      color: #fff;
    }
    span {
      color: #fff;
    }
  }
  .speed-up-info-img {
    width: 162px;
    height: 100px;
    background: url(../../../../../assets/images/icon/img_bar_chart_hover.png)
      no-repeat;
    background-size: 100% 100%;
    float: right;
    margin-right: 30px;
    margin-top: 22px;
  }
  .tip-txt-info {
    opacity: 1;
    overflow: visible;
  }
}
.index-speed-up {
  width: 100%;
  height: 152px;
  background-color: #fff;
  border-radius: 6px;
  margin-bottom: 20px;
  border: 3px solid transparent;
  cursor: pointer;
  box-shadow: 0px 20px 60px 0px rgba(0, 0, 0, 0.06);
  transition: box-shadow 0.5s, opacity 0.3s;
  color: #000;
  position: relative;
  text-align: left;
  // opacity: 0;
  & > .speed-up-info-txt {
    width: 290px;
    overflow: hidden;
    float: left;
    padding-top: 41px;
    padding-bottom: 41px;
  }
  .info-txt {
    margin-left: 30px;
    width: 260px;
    position: relative;
    font-size: 16px;
    color: rgba(140, 139, 142, 1);
    white-space: nowrap;
    overflow: hidden;
    text-overflow: ellipsis;
    &:nth-of-type(1) {
      margin-bottom: 20px;
    }
    & > span {
      font-size: 16px;
      color: rgba(158, 157, 159, 1);
      letter-spacing: 0px;
      line-height: 22px;
      transition: color 0.3s;
      color: rgba(44, 41, 57, 1);
    }
    & > .auto-tip {
      color: rgba(140, 139, 142, 1)
    }
    & > .tip-txt-info {
      position: absolute;
      border-radius: 2px;
      background-color: #fff;
      white-space: nowrap;
      overflow: visible;
      font-size: 14px;
      padding: 2px 4px 3px 3px;
      top: 30px;
      left: 50%;
      transform: translate(-50%);
      opacity: 0;
      transition: opacity 0.3s;
      & > i {
        position: absolute;
        height: 0;
        width: 0;
        border: 4px solid transparent;
        border-top: 5px solid #ffffff;
        bottom: -8px;
        left: 50%;
        transform: translateX(-50%);
      }
    }
  }
  &:hover {
    @include a-focus;
  }
  &:focus {
    @include a-focus;
  }
  .speed-up-info-img {
    width: 162px;
    height: 100px;
    background: url(../../../../../assets/images/icon/img_bar_chart_normal.png)
      no-repeat;
    background-size: 100% 100%;
    float: right;
    margin-right: 30px;
    margin-top: 22px;
  }
  & > .cover {
    position: absolute;
    // border: 1px solid red;
    left: 0;
    top: 0;
    right: 0;
    bottom: 0;
    border-radius: 6px;
    z-index: 99;
  }
}
@media screen and (max-width: 1366px) {
  .index-speed-up {
     height: 120px;
     &:hover {
       .speed-up-info-img {
        width: 105px;
        height: 84px;
        margin-top: 11px;
      }
     }
     .speed-up-info-txt {
      padding-top: 25px;
      padding-bottom: 25px;
     }
     .speed-up-info-img {
      width: 105px;
      height: 84px;
      margin-top: 11px;
     }
  }
}
</style>
