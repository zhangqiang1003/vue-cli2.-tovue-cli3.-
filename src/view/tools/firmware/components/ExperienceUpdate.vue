<template>
<div>
  <div class="experience-update" :class="{ 'blur': isShowReleaseMask}">
    <h5 class="title">{{ $t('i18n.EXPERIENCE_VERSION') }}</h5>
    <div class="container">
      <div class="wrapper" ref="logWrapper">
        <div class="txt-wrapper">
          <p>{{ $t('i18n.EXPERIENCE_VERSION_INTRO') }}</p>
          <p class="curr-version">{{ currVersion }}</p>
          <p class="change-log" v-html="changLog"></p>
        </div>
      </div>
      <primary-button
        @btnClicked="betaUpdate()"
        :disabled="disabled"
        :loading="updateing"
        :isBetaVer="isBetaVer"
        :text="text"></primary-button>
      <div class="recovery-release" @click="showReleaseMask()">{{ $t('i18n.RECOVERY_RELEASE_VERSION_ONCE') }}</div>
    </div>
  </div>
  <div class="mask-wrapper" :class="{'display-block': isShowReleaseMask}">
    <div class="recovery-mask" @click="hideReleaseMask()"></div>
    <div class="con-wrapper">
      <div class="con">
        <p>{{ $t('i18n.RECOVERY_RELEASE_TIP') }}</p>
        <div class="feedback">
          <i class="feedback-left"></i>
          <div class="feedback-right">
            <p class="pdt10">{{ $t('i18n.FEEDBACK_BY_QCODE') }}</p>
            <p class="p-txt">{{ $t('i18n.FEEDBACK_BY_URL') }}</p>
            <a :href="urlTxt" target="_blank" class="url-txt">{{ urlTxt }}</a>
          </div>
        </div>
      </div>
      <primary-button
        @btnClicked="releaseUpdate()"
        :disabled="relDisabled"
        :loading="relLoading"
        :text="recoveryTxt"></primary-button>
    </div>
  </div>
  <fade-tips :msg="tips.msg" :show="tips.show" :type="tips.type" @hide="tips.show = false"></fade-tips>
</div>
</template>

<script>
import { PrimaryButton } from '@/common/components/buttons/index'
import { compareVersion } from '@/common/service/utils'

import {
  sysFirmwareInfo,
  sysFirmwareCheck,
  sysOnlineUpgradeFirmware
} from '@/api/system-manage-api'

import getNetTypeAndWifiSsidService from '@/common/service/getNetTypeAndWifiSsid'
import { getInitStatusApi } from '@/api/login-api'

import FadeTips from '@/common/components/fade-tips/FadeTips'
import jsonp from '@/plugin/jsonp'

export default {
  name: 'ExperienceUpdate',
  data: function() {
    return {
      updateing: false,
      disabled: true,
      relLoading: false,
      relDisabled: false,
      text: this.$t('i18n.EXPERIENCE_VERSION_UPDATE'),
      changLog: null,
      recoveryTxt: this.$t('i18n.RECOVERY_RELEASE_VERSION'),
      urlTxt: 'https://wj.qq.com/s/2895728/d5a1/',
      isShowReleaseMask: false,
      currVersion: this.$t('i18n.CURRENT_EXPERIENCE_VERSION') + this.$t('i18n.GETTING_VERSION_INFO'),
      objVersionInfo: null,
      updateIntervalTime: null,
      urlT: null,
      isBetaVer: false,
      tips: {
        msg: '',
        type: 'success',
        show: false
      },
      waiting: {
        msg: this.$t('i18n.FIRMWARE_UPDATE_TIPS'),
        show: true
      }
    }
  },
  components: {
    PrimaryButton,
    FadeTips
  },
  methods: {
    betaUpdate: function() { // 升级尝鲜固件
      // 判断是否获取到固件信息
      if (!this.objVersionInfo === null) {
        this.tips.type = 'tip'
        this.tips.msg = this.$t('i18n.GETTING_VERSION_INFO')
        this.tips.show = true
        return
      }
      // 判断当前固件是否是最新固件
      if (this.objVersionInfo.currVer === this.objVersionInfo.betaVer) {
        this.tips.type = 'tip'
        this.tips.msg = this.$t('i18n.LATEST_BETA_VERSION')
        this.tips.show = true
        return
      }
      // 调用接口升级最新尝鲜固件
      this.updateing = true
      this.onlineUpdateBeta()
    },
    createBar() {
      this.$nextTick(() => {
        this.bar = new this.$geminiScrollbar({
          element: this.$refs.logWrapper
        }).create()
      })
    },
    showReleaseMask: function() {
      this.isShowReleaseMask = true
      this.$root.Bus.$emit('showBlur', true)
      this.$root.Bus.$emit('busIsOpenWifiList', true)
    },
    hideReleaseMask: function() {
      this.isShowReleaseMask = false
      this.$root.Bus.$emit('showBlur', false)
      this.$root.Bus.$emit('busIsOpenWifiList', false)
    },
    releaseUpdate: function() { // 恢复正式固件
      if (!this.objVersionInfo.currVer || !this.objVersionInfo.releaseVer) {
        this.relLoading = true
        this.relDisabled = true
        new Promise((resolve) => {
          sysFirmwareCheck({ image_type: 'release' })
            .then((res) => {
              this.objVersionInfo.releaseVer = res.version
              resolve()
            })
            .catch((e) => {
              this.tips.type = 'error'
              this.tips.msg = this.$t('i18n.RELEASE_UPDATE_FAILED')
              this.tips.show = true
              this.relLoading = false
              this.relDisabled = false
              return false
            })
        })
          .then(() => {
            sysFirmwareInfo()
              .then((res) => {
                this.objVersionInfo.currVer = res.version
              })
              .catch((e) => {
                this.tips.type = 'error'
                this.tips.msg = this.$t('i18n.RELEASE_UPDATE_FAILED')
                this.tips.show = true
                this.relLoading = false
                this.relDisabled = false
                return false
              })
          })
          .then(() => {
            // 判断检查当前版本是否为最新固件
            if (this.objVersionInfo.currVer === this.objVersionInfo.releaseVer) {
              // 当前是最新固件
              this.tips.type = 'tip'
              this.tips.msg = this.$t('i18n.LATEST_RELEASE_VERSION')
              this.tips.show = true
              this.relLoading = false
              this.relDisabled = false
              return false
            }
            // 判断需要升级最新正式固件
            this.onlineUpdateRelease()
          })
      } else {
        // 判断检查当前版本是否为最新固件
        if (this.objVersionInfo.currVer === this.objVersionInfo.releaseVer) {
          // 当前是最新固件
          this.tips.type = 'tip'
          this.tips.msg = this.$t('i18n.LATEST_RELEASE_VERSION')
          this.tips.show = true
          return false
        }
        // 判断需要升级最新正式固件
        this.onlineUpdateRelease()
      }
    },
    getVersionInfo: function() {
      let currVer = null
      let betaVer = null
      let releaseVer = null
      let log = null
      let betaSaveConfig = null
      let releaseSaveConfig = null
      let that = this
      function getBetaVer() {
        sysFirmwareCheck({ image_type: 'beta' })
          .then((res) => {
            betaVer = res.version
            betaSaveConfig = parseInt(res.save_config)
            log = res.changlog.replace(/\r\n/g, '<br>')
            getReleaseVer()
          })
          .catch(() => {
            that.checkVersionType({currVer, betaVer, log, releaseVer, betaSaveConfig, releaseSaveConfig})
          })
      }
      getBetaVer()
      function getReleaseVer() {
        sysFirmwareCheck({ image_type: 'release' })
          .then((res) => {
            releaseVer = res.version
            releaseSaveConfig = parseInt(res.save_config)
            getCurrVer()
          })
          .catch(() => {
            that.checkVersionType({currVer, betaVer, log, releaseVer, betaSaveConfig, releaseSaveConfig})
          })
      }
      function getCurrVer() {
        sysFirmwareInfo()
          .then((res) => {
            currVer = res.version // 当前版本信息
            that.checkVersionType({currVer, betaVer, log, releaseVer, betaSaveConfig, releaseSaveConfig})
            that.objVersionInfo = {currVer, betaVer, releaseVer, log, betaSaveConfig, releaseSaveConfig}
          })
          .catch(() => {
            that.checkVersionType({currVer, betaVer, log, releaseVer, betaSaveConfig, releaseSaveConfig})
          })
      }
    },
    checkVersionType: function(obj) {
      console.log('curr:', obj.currVer, 'beta:', obj.betaVer, 'release:', obj.releaseVer)
      // 如果因为接口调用失败，没有获取到beta版或正式版的版本信息，则显示暂无尝鲜固件
      if (!obj.betaVer || !obj.releaseVer || !obj.currVer) {
        this.currVersion = this.$t('i18n.HAVE_NOT_BETA_VERSION')
        this.changLog = null
        this.disabled = true
        return false
      }
      // 检查系统最新版的版本类型（beta or release）
      // obj.currVer = '2.2.0'
      // obj.betaVer = '2.2.1.20181120'
      // obj.releaseVer = '2.2.0'
      let result = this.comparisonVersion(obj.currVer, obj.betaVer) // true - 表示可能有尝鲜固件 ； false - 表示可能无尝鲜固件
      let c = obj.currVer.substring(0, 5)
      let b = obj.betaVer.substring(0, 5)
      if (c === b && !result && (obj.currVer.length === obj.betaVer.length)) {
        let curr = parseInt(obj.currVer.substring(6))
        let beta = parseInt(obj.betaVer.substring(6))
        if (curr < beta) { // 表示有尝鲜固件（当前版本和在线尝鲜版本比较，为true，可能有尝鲜版本）
          result = true
        } else { // 表示无尝鲜固件
          result = false
        }
      }
      if (result) { // 当前版本和在线尝鲜版本比较后，进一步比较在线尝鲜版和在线正式版
        result = this.comparisonVersion(obj.releaseVer, obj.betaVer)
      }
      if (result) { // 视图显示有尝鲜固件
        this.currVersion = this.$t('i18n.CURRENT_EXPERIENCE_VERSION') + obj.betaVer
        this.changLog = obj.log
        this.disabled = false
        if (this.bar) {
          this.bar.update()
        } else {
          this.createBar()
        }
      } else { // 视图显示没有尝鲜固件
        this.currVersion = this.$t('i18n.HAVE_NOT_BETA_VERSION')
        this.changLog = null
        this.disabled = true
      }
    },
    comparisonVersion: function(ver1, ver2) { // 比较版本信息
      let compare = compareVersion(ver1, ver2)
      return compare
    },
    initCheckVersion: function() { // 组件初始化判断是否有固件版本信息
      this.objVersionInfo = null
      this.getVersionInfo()
    },
    onlineUpdateBeta: function() { // 升级尝鲜固件
      let ssid = null
      let defaultSsid = null
      let network = null
      getNetTypeAndWifiSsidService()
        .then((res) => {
          ssid = res.wifiSsid
          defaultSsid = res.defaultWifiSsid
          network = parseInt(res.network)
          if (network === 2) { // 判断本机联网方式
            this.waiting.msg =
              this.$t('i18n.USE_WIFI_SERVER_UPDATE') +
              (this.objVersionInfo.betaSaveConfig === 1 ? ssid : defaultSsid)
          }
          this.$root.Bus.$emit('waitingMaskShow', this.waiting)
          sysOnlineUpgradeFirmware({image_type: 'beta'})
            .then((res) => {
              if (parseInt(res.ret) === 0) {
                this.checkCompleted('beta')
              } else {
                return Promise.reject(res.msg)
              }
            })
        })
    },
    onlineUpdateRelease: function() { // 升级正式固件
      let ssid = null
      let defaultSsid = null
      let network = null
      this.relLoading = true
      this.relDisabled = true
      getNetTypeAndWifiSsidService()
        .then((res) => {
          ssid = res.wifiSsid
          defaultSsid = res.defaultWifiSsid
          network = parseInt(res.network)
          if (network === 2) { // 判断本机联网方式
            this.waiting.msg =
              this.$t('i18n.USE_WIFI_SERVER_UPDATE') +
              (this.objVersionInfo.betaSaveConfig === 1 ? ssid : defaultSsid)
          }
          this.$root.Bus.$emit('waitingMaskShow', this.waiting)
          sysOnlineUpgradeFirmware({image_type: 'release'})
            .then((res) => {
              if (parseInt(res.ret) === 0) {
                this.checkCompleted('release')
              } else {
                this.relLoading = false
                this.relDisabled = false
                return Promise.reject(res.msg)
              }
            })
        })
        .catch(() => {
          this.relLoading = false
          this.relDisabled = false
        })
    },
    /**
     * 检查是否完成升级(保存配置升级)
     */
    checkCompleted(data) {
      window.sessionStorage.removeItem('versionInfo')
      if ((data === 'beta' && this.objVersionInfo.betaSaveConfig === 1) ||
        (data === 'release' && this.objVersionInfo.releaseSaveConfig === 1)) { // 保存配置
        this.updateIntervalTime = setInterval(() => {
          getInitStatusApi().then(res => {
            if ('' + res.ret) {
              this.waiting.show = false
              this.$emit('disableClick', false) // 解除禁止点击切换升级类型
              this.$root.Bus.$emit('waitingMaskShow', this.waiting)
              window.location.href = this.urlT
              this.relLoading = false
              this.relDisabled = false
            }
          })
        }, 4000)
      } else { // 不保存配置
        this.updateIntervalTime = setInterval(() => {
          this.checkServers()
        }, 4000)
      }
    },
    /**
     * 检测是否完成(不保存配置升级)
     */
    checkServers() {
      // jsonp
      const url = `${this.urlT}/cgi-bin/luci/sys_init_get`
      jsonp(
        url,
        {},
        {
          format: 'jsonp'
        }
      )
        .then(res => {
          window.clearInterval(this.updateIntervalTime)
          window.location.href = this.urlT
          this.relLoading = false
          this.relDisabled = false
        })
        .catch(e => {
          window.clearInterval(this.updateIntervalTime)
          window.location.href = this.urlT
          this.relLoading = false
          this.relDisabled = false
        })
    }
  },
  created() {
    this.initCheckVersion()
    this.isBetaVer = true // 判断进入尝鲜固件页面
  },
  mounted() {
    const randNum = Math.random() * 1000
    this.urlT = window.location.protocol + '//' + window.location.host + window.location.pathname + `?updateNum=${randNum}/`
  },
  beforeDestroy() {
    this.isBetaVer = false // 判断离开尝鲜固件页面
  }
}
</script>

<style lang="scss" scoped>
@import '../../../../common/sass/_constant';
@import '../../../../common/sass/_mixin';

.experience-update {
  width: 100%;
  border-radius: $startContentRaduis;
  background: linear-gradient(
    180deg,
    rgba(240, 243, 247, 1),
    rgba(255, 255, 255, 1)
  );
  box-shadow: 0px 20px 60px 0px rgba(0, 0, 0, 0.15);
  & > .title {
    padding-left: 30px;
    height: 76px;
    color: rgba(58, 56, 72, 1);
    font-size: 20px;
    line-height: 76px;
    background: white;
    font-weight: bold;
    border-radius: $startContentRaduis $startContentRaduis 0 0;
  }
  & > .container {
    padding: 40px 60px 10px 60px;
    background: linear-gradient(
      180deg,
      rgba(240, 243, 247, 1),
      rgba(255, 255, 255, 1)
    );
    border-radius: $startContentRaduis;
    & > .wrapper {
      height: 185px;
      overflow: hidden;
    }
  }
  .recovery-release {
    margin-top: 15px;
    width: 480px;
    height: 56px;
    line-height: 56px;
    text-align: center;
    text-decoration: none;
    cursor: pointer;
    color: rgba(131, 137, 161, 1);
    &:hover {
      // text-decoration: underline;
      color: rgba(58, 56, 72, 1);
    }
  }
  .txt-wrapper {
    background-color: #fff;
    border-radius: 6px;
    padding: 10px;
    color: rgba(58, 56, 72, 1);
    font-size: 14px;
    & > .curr-version {
      margin-top: 15px;
    }
    & > .change-log {
      margin-top: 15px;
      line-height: 22px;
    }
  }
}
.mask-wrapper {
  position: fixed;
  left: 0;
  right: 0;
  bottom: 0;
  top: 0;
  z-index: 99;
  padding-top: 170px;
  display: none;
  &.display-block {
    display: block;
  }
  & > .recovery-mask {
    position: absolute;
    left: 0;
    top: 0;
    right: 0;
    bottom: 0;
    z-index: -1;
    background-color: rgba(255, 255, 255, 0.4);
  }
  .con-wrapper {
    width: 450px;
    margin: 0 auto;
    z-index: 1;
    color: rgb(42, 42, 44);
  }
  .con {
    width: 100%;
    height: 240px;
    background-color: #fff;
    border-radius: $startContentRaduis;
    padding: 40px 40px 30px;
    box-shadow:0px 20px 60px 0px rgba(0,0,0,0.06);
  }
  .feedback {
    margin-top: 20px;
  }
  .feedback-left {
    float: left;
    width: 100px;
    height: 100px;
    background: url(../../../../assets/images/icon/feedback_qcode.png) no-repeat;
    background-size: 100% 100%;
    border: 1px solid rgba(0, 159, 232, 1);
    border-radius: 8px;
  }
  .feedback-right {
    // border: 1px solid red;
    margin-left: 130px;
  }
  .pdt10 {
    padding-top: 5px;
  }
  .p-txt {
    margin-top: 15px;
  }
  .url-txt {
    display: block;
    border: 1px solid rgba(0, 0, 0, 0.15);
    margin-top: 8px;
    height: 30px;
    line-height: 30px;
    padding-left: 5px;
    overflow: hidden;
    cursor: pointer;
    color: rgba(58, 56, 72, 1);
  }
}
.hide-textarea {
  height: 0px;
  width: 0px;
  opacity: 0;
}
</style>
