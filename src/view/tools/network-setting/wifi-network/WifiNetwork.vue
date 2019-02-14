<template>
  <div class="wifi-network-body">
    <div class="net-way">
      <a href="javascript:;" class="curr-net-way" @click="clickToggleH52Fn()" :class="{'border-radius': isShowH52}">
        <i class="icon-nettype"></i>
        {{$t('i18n.WIFI_NET_INTERNET')}}
        <i class="icon-switch" :class="{'icon-open': isShowH52}"></i>
      </a>
      <a href="javascript:;" class="option-net-way" :class="{'h52': isShowH52}" @click="clickToggleNetType()">
        <i class="icon-nettype"></i>
        {{$t('i18n.WIRED_NET_INTERNET')}}
      </a>
    </div>
    <div class="wifi-name">
      <p>{{ $t('i18n.WIFI_NETWORK_NAME') }}：</p>
      <div class="input-wifi-name">
        <input type="text" :placeholder="wifiName.placeholder" class="beauty-placeholder" v-model="wifiName.value" autocomplete="new-password" disabled>
        <a href="javascript:void(0)" @click="clickSearchWifiFn()">{{ $t('i18n.SEARCH_WIFI') }}</a>
      </div>
    </div>
    <div class="pdt30">
      <p>{{ $t('i18n.WIFI_NETWORKING_PASSWORD') }}：</p>
      <div class="input-wifi-psd">
        <input type="password" :placeholder="wifiPassword.placeholder" class="beauty-placeholder" v-model="wifiPassword.value" :class="{'error': !wifiPassword.valid}" @focus="pswFocus()" autocomplete="new-password">
        <div :class="{'input-error': !wifiPassword.valid}" v-show="!wifiPassword.valid">{{ $t('i18n.PASSWORD_CANNOT_BE_EMPTY') }}</div>
      </div>
    </div>
    <p class="pdt30 p">{{ $t('i18n.CONNECTION_STATUS') }}：
      <span v-if="connectingStatus == 0">{{ $t('i18n.BEING_TESTED') }}...</span>
      <span v-if="connectingStatus == 1">{{ $t('i18n.NOT_CONNECTED') }}</span>
      <span class="connecting" v-if="connectingStatus == 2">{{ $t('i18n.CONNECTING') }}...</span>
      <span v-if="connectingStatus == 3">{{ $t('i18n.CONNECT_FAILED') }}</span>
      <span class="connecting" v-if="connectingStatus == 4">{{ $t('i18n.CONNECT_SUCCESS') }}</span>
    </p>
    <!-- <div>连接</div> -->
    <primary-button class="connent-btn" v-show="isShowConnectBtn" @btnClicked="clickNetWifiFn()" :text="$t('i18n.CONNECT')" :disabled="waiting" :loading="waiting"></primary-button>
    <primary-button class="connent-btn" v-show="!isShowConnectBtn" @btnClicked="clickCloseWifiFn()" :text="$t('i18n.CLOSE_CONNECT')" :disabled="waiting" :loading="waiting"></primary-button>
    <search-wifi></search-wifi>
    <fade-tips :msg="tip.msg" :type="tip.type" :show="tip.show" @hide="tip.show = false"></fade-tips>
  </div>
</template>

<script>
import {
  netWlanGet,
  netRepeaterSet,
  getNetRepeaterStatusApi,
  netTypeSet,
  netRepeaterGetApi
} from '@/api/network-check-api'
import { getInitStatusApi } from '@/api/login-api'
import { PrimaryButton } from '@/common/components/buttons/index'
import SearchWifi from '@/common/components/search-wifi/SearchWiFi'
import getNetTypeAndWifiSsidService from '@/common/service/getNetTypeAndWifiSsid'
import FadeTips from '@/common/components/fade-tips/FadeTips'
export default {
  name: 'WifiNetwork',
  data: function() {
    return {
      isShowH52: false,
      showWifiLists: false,
      wifiName: {
        value: '',
        valid: true,
        placeholder: 'QeeYou-Game'
      },
      wifiData: {},
      getStatusInterValTime: '',
      connectingStatus: 1,
      waiting: false,
      wifiPassword: {
        value: '',
        valid: true,
        placeholder: this.$t('i18n.PASSWORD_PLACEHOLDER')
      },
      errNumber: 1,
      temPromise: null,
      tip: {
        msg: '',
        type: 'success',
        show: false
      },
      getNetrepeterType: null,
      isShowConnectBtn: true, // 是否显示连接按钮
      sysInitGetNum: 1 // 检测网络成功的次数
    }
  },
  components: {
    PrimaryButton,
    SearchWifi,
    FadeTips
  },
  methods: {
    nameFocus: function() {
      this.wifiName.valid = true
    },
    pswFocus: function() {
      this.wifiPassword.valid = true
    },
    clickSearchWifiFn: function() {
      this.showWifiLists = !this.showWifiLists
      this.$root.Bus.$emit('busShowWifiLists', this.showWifiLists)
    },
    getWifiDataFn: function(data) {
      // 从search组件获取数据选中的wifi数据
      this.wifiName.value = data.ssid
      this.wifiData = data
      this.wifiPassword.value = ''
      this.connectingStatus = 1
      this.isShowConnectBtn = true
      // this.wifiPassword.valid = false
    },
    apiNetWlanGetFn: function() {
      netWlanGet({ ifname: 'wlan0' })
        .then(data => {})
        .catch(e => {})
    },
    clickNetWifiFn: function() {
      if (this.wifiName.value === '') {
        this.wifiName.valid = false
        return
      }
      // if (this.wifiPassword.value === '') {
      //   this.wifiPassword.valid = false
      //   return
      // }
      // 调用netRepeaterSet接口
      this.sysInitGetNum = 1
      this.apiNetTypeSetFn()
      let msgTxt = null
      if (parseInt(this.temPromise.network) === 2) {
        // msgTxt = '完成设置后需要连接到本机WiFi:' + this.temPromise.wifiSsid + ',并刷新页面'
        msgTxt = `${this.$t('i18n.NETWORK_SETTING_W_2')}:${this.temPromise.wifiSsid},${this.$t('i18n.NETWORK_SETTING_W_3')}`
      } else if (parseInt(this.temPromise.network) === 1) {
        msgTxt = `${this.$t('i18n.NETWORK_SETTING_W_1')}`
      }
      let blurData = {
        // msg: `${this.$t('i18n.WIFI_SETTING_TIP')}`,
        msg: msgTxt,
        show: true
      }
      this.$root.Bus.$emit('waitingMaskShow', blurData)
    },
    apiNetTypeSetFn: function() {
      let apiData = {
        nettype: 'wifi'
      }
      netTypeSet(apiData)
        .then((data) => {
          this.apiNetRepeaterSetFn()
        })
        .catch((e) => {
          this.$root.Bus.$emit('waitingMaskShow', { show: false })
          this.waitingDy = false
          this.tip.msg = e
          this.tip.type = 'error'
          this.tip.show = true
        })
    },
    apiNetRepeaterSetFn: function() {
      this.waiting = true
      let apiData = {}
      let setData = {
        enable: '1',
        radio: '2.4G',
        ssid: this.wifiName.value,
        bssid: this.wifiData.bssid,
        password: this.wifiPassword.value
      }
      Object.assign(apiData, this.wifiData, setData)
      netRepeaterSet(apiData)
        .then(data => {
          if (data) {
            // 检查初始化状态
            this.checkInitStatusFn()
            this.connectingStatus = 2
          }
        })
        .catch(e => {
          this.connectingStatus = 3
          let blurData = {
            msg: '',
            show: false
          }
          this.$root.Bus.$emit('waitingMaskShow', blurData)
        })
    },
    checkInitStatusFn: function() {
      this.getStatusInterValTime = window.setInterval(() => {
        getInitStatusApi().then(res => {
          if (' ' + res && this.sysInitGetNum === 1) {
            this.sysInitGetNum++
            clearInterval(this.getStatusInterValTime)
            let aa = setTimeout(() => {
              this.getNetRepeaterStatusFn()
              clearTimeout(aa)
            }, 10000)
          }
        })
      }, 3000)
    },
    getNetRepeaterStatusFn: function() {
      getNetRepeaterStatusApi()
        .then(data => {
          this.waiting = false
          if (data.ret === 0) {
            this.connectingStatus = 4
            this.isShowConnectBtn = false // 显示断开连接
            let blurData = {
              msg: '',
              show: false
            }
            this.$root.Bus.$emit('waitingMaskShow', blurData)
          } else {
            // this.connectingStatus = 3
            this.errNumber++
            if (this.errNumber <= 3) {
              let aa = setTimeout(() => {
                this.getNetRepeaterStatusFn()
                clearTimeout(aa)
              }, 10000)
            } else {
              this.errNumber = 1
              this.sysInitGetNum = 1
              this.connectingStatus = 3
              this.isShowConnectBtn = true // 显示连接
              let blurData = {
                msg: '',
                show: false
              }
              this.$root.Bus.$emit('waitingMaskShow', blurData)
            }
          }
        })
        .catch(e => {
          this.connectingStatus = 3
          this.isShowConnectBtn = true // 显示连接
          this.waiting = false
          let blurData = {
            msg: '',
            show: false
          }
          this.$root.Bus.$emit('waitingMaskShow', blurData)
        })
    },
    clickToggleH52Fn: function () {
      this.isShowH52 = !this.isShowH52
    },
    clickToggleNetType: function() {
      this.$root.Bus.$emit('busShowWifiLists', false)
      this.$root.Bus.$emit('busToggleNetType', 'ethernet')
      this.isShowH52 = false
    },
    apiGetNetrepeterFn: function() {
      this.connectingStatus = 0
      netRepeaterGetApi()
        .then(data => {
          let enable = parseInt(data.enable)
          if (enable === 1) {
            this.wifiName.value = data.ssid
            this.wifiPassword.value = data.password
            this.wifiData.bssid = data.bssid
            if (data.status === true) {
              this.connectingStatus = 4
              this.isShowConnectBtn = false // 显示断开连接
            } else {
              this.connectingStatus = 1
              this.isShowConnectBtn = true // 显示连接
            }
          } else {
            this.connectingStatus = 1
            this.wifiName.value = ''
            this.wifiPassword.value = ''
            this.wifiData.bssid = ''
            this.isShowConnectBtn = true // 显示连接
          }
        })
        .catch(e => {
          this.connectingStatus = 1
          // this.tip.msg = e
          // this.tip.type = 'error'
          // this.tip.show = true
          this.isShowConnectBtn = true // 显示连接
        })
    },
    clickCloseWifiFn: function() {
      let apiData = {
        radio: '2.4',
        enable: '0'
      }
      netRepeaterSet(apiData)
        .then(data => {
          this.tip.msg = this.$t('i18n.WIFI_CLOSE_SUCCESS')
          this.tip.type = 'success'
          this.tip.show = true
          this.isShowConnectBtn = true // 显示连接
          this.connectingStatus = 1
          this.wifiName.value = ''
          this.wifiPassword.value = ''
          this.wifiData.bssid = ''
        })
        .catch(e => {
          this.tip.msg = this.$t('i18n.WIFI_CLOSE_FAILED')
          this.tip.type = 'error'
          this.tip.show = true
          this.isShowConnectBtn = false // 显示断开连接
        })
    }
  },
  created() {
    this.$root.Bus.$on('busWifiName', data => {
      this.getWifiDataFn(data)
    })
    this.$root.Bus.$on('busChangeFalse', data => {
      this.showWifiLists = data
    })
    getNetTypeAndWifiSsidService()
      .then((data) => {
        this.temPromise = data
      })
    this.$root.Bus.$on('busNetType', (data) => {
      this.getNetrepeterType = data
      if (data === 'wifi') {
        // this.connectingStatus = 0
      } else {
        this.connectingStatus = 1
      }
    })
    this.apiGetNetrepeterFn()
    // this.getNetrepeterTypeFn()
  },
  beforeDestroy() {
    this.$root.Bus.$off('busWifiName')
    this.$root.Bus.$off('busChangeFalse')
    this.$root.Bus.$off('busNetType')
  }
}
</script>

<style lang="scss" scoped>
@import '../../../../common/sass/_mixin';
@import '../../../../common/sass/_constant';
.error {
  border: 1px solid #f14466 !important;
  background: rgba(241, 68, 102, 0.1);
}
.wifi-network-body {
  padding: 40px 60px 60px;
  .curr-net-way {
    width: 480px;
    height: 52px;
    border: 1px solid rgba(230, 230, 231, 1);
    border-radius: 8px;
    background-color: #fff;
    color: rgba(58, 56, 72, 1);
    font-size: 14px;
    padding-left: 45px;
    padding-right: 20px;
    line-height: 52px;
    position: relative;
    transition: border-color .3s;
    &:focus {
      border-color: #39c1ff;
    }
    &:hover {
      border-color: #39c1ff;
    }
    &.border-radius {
      border-radius: 8px 8px 0 0;
      border-bottom-width: 0;
    }
    & > .icon-nettype {
      position: absolute;
      left: 20px;
      top: 50%;
      margin-top: 1px;
      transform: translateY(-50%);
      width: 16px;
      height: 16px;
      background: url(../../../../assets/images/icon/icon_wifi_setting.png) no-repeat;
      background-size: cover;
    }
    & > .icon-switch {
      position: absolute;
      top: 50%;
      right: 20px;
      margin-top: 1px;
      transform: translateY(-50%);
      width: 12px;
      height: 8px;
      background: url(../../../../assets/images/icon/icon_arrow_down_1.png) no-repeat;
      background-size: cover;
      &.icon-open {
        background: url(../../../../assets/images/icon/icon_arrow_down_2.png) no-repeat;
        background-size: cover;
      }
    }
  }
  .option-net-way {
    width: 480px;
    height: 0;
    border: 1px solid rgba(230, 230, 231, 1);
    border-width: 0;
    border-radius: 0 0 8px 8px;
    background-color: #fff;
    color: rgba(58, 56, 72, 1);
    font-size: 14px;
    padding-left: 45px;
    padding-right: 20px;
    line-height: 52px;
    position: relative;
    transition: border-color .3s, height .3s, color .2s;
    overflow: hidden;
    &.h52 {
      height: 52px;
      border-width: 1px;
    }
    & > .icon-nettype {
      position: absolute;
      left: 20px;
      top: 50%;
      margin-top: 1px;
      transform: translateY(-50%);
      width: 16px;
      height: 16px;
      background: url(../../../../assets/images/icon/icon_wired_setting.png) no-repeat;
      background-size: cover;
    }
    &:focus {
      border-color: #39c1ff;
      color: #39c1ff;
    }
    &:hover {
      border-color: #39c1ff;
      color: #39c1ff;
    }
  }
  .wifi-name {
    padding-top: 20px;
  }
  & > .pdt30 {
    padding-top: 30px;
  }
  p {
    font-size: 14px;
    color: #6f7592;
    line-height: 20px;
    margin-bottom: 10px;
    & > span {
      color: rgba(44, 42, 56, 1);
    }
  }
  .beauty-placeholder {
    &:focus {
      border-color: #009fe8;
      box-shadow: 0px 6px 20px 0px rgba(0, 159, 232, 0.5);
    }
    &:-moz-placeholder {
      color: #b3b3b3;
    }
    &:-ms-input-placeholder {
      color: #b3b3b3;
    }
    &::-webkit-input-placeholder {
      color: #b3b3b3;
    }
  }
  .input-wifi-name {
    & > input {
      width: 364px;
      height: 46px;
      border: 1px solid rgba(230, 230, 231, 1);
      border-radius: 4px;
      padding: 0 20px;
      background: #fff;
    }
    & > a {
      @include startBtn-primaryHover;
      float: right;
      width: 96px;
      height: 46px;
      border-radius: 4px;
      line-height: 46px;
      font-size: 14px;
    }
  }
  .input-wifi-psd {
    position: relative;
    @include inputWarnTip;
    input {
      height: 46px;
      border-color: #e6e6e7;
    }
  }
  .p {
    & > span {
      color: rgba(234, 42, 83, 1);
    }
    & > .connecting {
      color: rgba(0, 141, 234, 1);
    }
  }
  & > .connent-btn {
    // @include startBtn-primaryHover;
    width: 480px;
    height: 56px;
    line-height: 56px;
    margin-top: 40px;
  }
}
</style>
