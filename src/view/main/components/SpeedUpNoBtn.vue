<template>
  <div class="ljb-index-left" :class="{'ljb-index-left-animation':isShow}">
    <div class="login-tip" :class="{'error-tip': isShowErrorTip}" v-show="isShowErrorTip">{{ txtTipShow }}</div>
    <div class="login-tip" :class="{'success-tip': isShowSuccessTip}" v-show="isShowSuccessTip">{{ txtTipShow }}</div>
    <div class="btn-container">
      <button :disabled="isDisabled"
        class="speed-up-circle1"
        @click.stop.prevent="clickAccBtn()"
        v-focus="focusStatus"
        @mouseover="focusAccBtn(true)"
        @mouseout="focusAccBtn(false)">
        <div class="speed-up-circle2">
          <div v-show="!isFocusBtn" class="speed-up-circle3">
            <h3 v-html="txtSpeedBtn"></h3>
          </div>
          <div v-show="isFocusBtn" class="speed-up-circle3">
            <h3 style="color: rgba(241, 68, 102, 1)">停止加速</h3>
          </div>
        </div>
        <div :class="{'speed-up-running': isShowSpeedUpPng && !isFocusBtn}" ref="running"></div>
      </button>
      <button class="restart" @click.stop.prevent="apiControlAcc('restart')" v-show="isShowRestartBtn">
        <div :class="{'hover-bg': isShowRestartStatus}"></div>
      </button>
    </div>
    <div class="delay-lost" :class="{'opacity-delay': isShowDelay}">
      <div class="delay">
        <i></i>
        <h4>{{ $t('i18n.DELAY_TIME') }}</h4>
        <p><em>{{ delay }}</em>ms</p>
      </div>
      <div class="lost">
        <i class="lost-icon"></i>
        <h4>{{ $t('i18n.PACKET_LOSS') }}</h4>
        <p><em>{{ loss }}</em>%</p>
      </div>
    </div>
    <div class="nat-upnp-tips">
      <span class="icon-upnp" :class="{'upnp-tip': isUpnpEnable}">
        <div class="upnp-txt-tip">
          <i></i>
          <span>{{ $t('i18n.ONLINE_TEAM_BUFF_TXT') }}</span>
        </div>
      </span>
      <span class="icon-nat" :class="{'nat-tip': isNatEnable}">
        <div class="nat-txt-tip">
          <i></i>
          <span>{{ $t('i18n.NAT_OPEN_BUFF_TXT') }}</span>
        </div>
      </span>
    </div>
  </div>
</template>

<script>
import { accStateGet, accControlAction } from '@/api/speed-manage-api'
import accCause from '@/common/service/accCause'
import { netUpnpGet } from '@/api/network-check-api'
export default {
  name: 'BeforeSpeedUp',
  data: function() {
    return {
      isShow: false,
      txtTipShow: this.$t('i18n.TIP_AUTO_SPEED'), // 提醒的文字
      delay: 34,
      loss: 0,
      focusStatus: true,
      txtSpeedBtn: null,
      isShowDelay: false,
      isShowRestartStatus: false,
      isShowRestartBtn: false,
      isShowErrorTip: false, // 用户加速失败时的错误提醒
      isShowSpeedUpPng: false, // 是否显示加速动画效果的圆圈图片
      isDisabled: false, // 是否禁止button标签的focus效果
      status: null, // 用户的状态
      event: null, // 改变用户状态的事件
      cause: null, // 用于状态发生该事件的原因
      causeFormArr: null, // 从本地整理的文件中获取状态发生该事件的原因
      userConflict: null, // 通道是否被占用
      isOpenRegularPolling: false, // 是否开启常规状态的轮询，默认true,开启
      stopInitSetTimeoutForRP: null, // 定义一个用于页面加载时，延时常规轮询的变量
      stopSetIntervalForRP: null, // 定义一个定时进行常规轮询的变量
      isChangeAccStatusOfRP: false, // 是否改变常规轮询时的各种状态
      stopSetIntervalForSU: null, // 定义一个定时进行加速状态轮询的变量
      isGetInitAccStatus: false, // 是否获取到初始化状态
      isShowSuccessTip: false, // 显示成功状态的顶部提醒
      isStartApi: true, // 解决老版app登录后，web端连续多次调用加速过程控制的接口，使只调用一次
      currentMode: null,
      currentPattern: null,
      isEmitBusCheckStateStatus: false,
      isEmitBusLoginOkStatus: true,
      lastStatus: null, // 记录相对于当前状态的上一次状态
      userType: null, // 记录账户过期
      isOpenInitRG: true, // 是否页面初始化加载时开启常规轮询
      isUpnpEnable: false, // 检测upnp是否开启
      isNatEnable: false, // 检测当前加速是否是p2p
      accInterval: null,
      isResponseAccStatus: true, // acc_state_get接口是否相应值
      accData: null,
      isFocusBtn: false,
      isRestarting: false,
      isStoping: false,
      clickStopBtnNum: 1,
      isDelayApi: true, // 在status变化时，是否延时执行连续两次emit传值的时间间隔
      getAccProcess: null, // 获取acc_state_get接口的过程中是否返回错误状态,true-请求成功|false-请求失败
      handleInterval: null
    }
  },
  methods: {
    apiControlAcc: function(order) {
      // 调用重新加速的接口
      // 处理各种状态
      clearInterval(this.accInterval)
      if (order === 'restart') {
        this.isRestarting = true
        this.txtSpeedBtn = this.$t('i18n.SPEED_RUNNING')
      }
      if (order === 'stop') {
        this.isStoping = true
        this.txtSpeedBtn = '正在停止加速'
        this.isFocusBtn = false
      }
      this.isShowRestartBtn = false
      this.isShow = true
      this.isShowDelay = false
      this.isShowSpeedUpPng = true
      // 调用加速过程控制的接口
      let apiData = {
        action: order
      }
      accControlAction(apiData)
        .then((data) => {
          if (data.ret === 0) {
            // 调取加速中的轮询检测的方法
            this.isRestarting = false
            this.isStoping = false
            this.clickStopBtnNum = 1
            this.getAccStatus()
            // 顶部提醒处理
            this.isShowErrorTip = false
            this.txtTipShow = this.$t('i18n.TIP_AUTO_SPEED')
          }
        })
        .catch(e => {
          clearInterval(this.accInterval)
          this.getAccInterval(5170)
          this.isRestarting = false
          this.isStoping = false
        })
    },
    getAccInterval: function(time = 870) {
      this.accInterval = setInterval(() => {
        if (!this.isResponseAccStatus) {
          return false
        }
        this.isResponseAccStatus = false
        this.getAccStatus()
      }, time)
    },
    getAccStatus: function() {
      accStateGet()
        .then(res => {
          this.isResponseAccStatus = true
          if (!this.isRestarting && !this.isStoping) { // 正在重新加速时或正在停止加速时
            this.busCheckAccStatus(res) // 触发busCheckStateStatus事件
            this.controlParams(res)
            this.getAccProcess = true
          }
        })
        .catch(e => {
          this.isResponseAccStatus = true
          this.errorAccDeal()
          this.getAccProcess = false
        })
    },
    busCheckAccStatus: function(data) { // 触发busCheckStateStatus事件
      if (this.isEmitBusCheckStateStatus) {
        this.$root.Bus.$emit('busCheckStateStatus', data)
        this.$root.Bus.$emit('memberLoginOkSuccess')
        this.isEmitBusCheckStateStatus = false
      }
    },
    controlParams: function(data) {
      this.accData = data
      this.lastStatus = this.status // 记录上一次状态
      this.status = data.status
      this.event = parseInt(data.event)
      this.delay = data.accNodePing
      this.userConflict = parseInt(data.userConflict)
      this.cause = parseInt(data.cause)
      this.causeFormArr = this.cause
      this.$root.Bus.$emit('busGetAccSatateIndexSpeedUp', this.accData)
      this.$root.Bus.$emit('busGetAccState', this.accData)
      if (data.accCurrMode === '') {
        this.currentMode = 0
      } else {
        this.currentMode = parseInt(data.accCurrMode)
      }
      this.currentPattern = data.accNodeZone
      this.userType = parseInt(data.userType)
      // 控制视图
      this.controlView()
      // 判断是否点亮图标
      this.openNat()
    },
    controlView: function() { // 控制视图
      switch (this.status) {
        case 'INIT':
          this.controlInitStatus()
          break
        case 'LOGING':
          this.controlLogingStatus()
          break
        case 'LOGIN_OK':
          this.controlLoginOkStatus()
          break
        case 'ACCING':
          this.controlAccingStatus()
          break
        case 'ACC_OK':
          this.controlAccOkStatus()
          break
        default:
          console.log('status:', this.status)
      }
    },
    controlInitStatus: function() { // 控制init状态的视图
      switch (this.event) {
        case 0:
          this.controlCause(this.cause)
          break
        case 2:
          this.controlCause(this.cause)
          break
        case 4: // 暂不处理
          this.controlCause(this.cause)
          break
        default:
          console.log('event:', this.event, 'cause:', this.cause)
      }
    },
    controlLogingStatus: function() {
      switch (this.event) {
        case 1:
          this.controlCause(this.cause)
          break
        case 3: // 目前不考虑
          this.controlCause(this.cause)
          break
        default:
          console.log('event:', this.event, 'cause:', this.cause)
      }
    },
    controlLoginOkStatus: function() { // 控制login_ok状态的视图
      // console.log(this.event, this.cause, 909)
      switch (this.event) {
        case 1:
          this.controlCause(this.cause)
          break
        case 3: // 目前不考虑
          this.controlCause(this.cause)
          break
        case 6:
          this.controlCause(this.cause)
          break
        default:
          console.log('event:', this.event, 'cause:', this.cause)
      }
    },
    controlAccingStatus: function() { // 控制accing状态的视图
      // console.log(this.event, this.cause, 924)
      switch (this.event) {
        case 5:
          this.controlCause(this.cause)
          break
        case 9:
          this.controlCause(this.cause)
          break
        default:
          console.log('event:', this.event, 'cause:', this.cause)
      }
    },
    controlAccOkStatus: function() { // 控制acc_ok状态的视图
      switch (this.event) {
        case 7:
          this.controlCause(this.cause)
          break
        default:
          console.log('event:', this.event, 'cause:', this.cause)
      }
    },
    controlCause: function(cause) { // 根据对应的cause控制视图
      switch (cause) {
        case 0:
          this.notLoggedIn() // 未登录
          break
        case 1:
          this.logingToLoginOk()
          break
        case 2:
          this.logingToLoginOk()
          break
        case 3:
          this.notLoggedIn() // 未登录
          break
        case 4:
          this.busyLine() // 占线
          break
        case 5:
          this.loginOkToAccing() // 正常加速的状态
          break
        case 6:
          this.loginOkToAccing() // 正常加速的状态
          break
        case 7:
          this.loginOkToAccing() // 正常加速的状态
          break
        case 8:
          this.userStopAccStatus() // 用户停止加速
          break
        case 9:
          this.accingToAccOk() // 加速成功
          break
        case 10:
          // this.abnormalStopAccStatus() // 异常停止加速
          this.userStopAccStatus() // 用户停止加速
          break
        case 11:
          this.restartAccing() // 重新启动加速
          break
        case 12:
          this.restartAccing() // 重新启动加速
          break
        case 13:
          this.accingToAccOk() // 加速成功
          break
        case 14:
          // this.abnormalStopAccStatus() // 异常停止加速
          this.userStopAccStatus() // 用户停止加速
          break
        case 15:
          // this.abnormalStopAccStatus() // 异常停止加速
          this.userStopAccStatus() // 用户停止加速
          break
        case 16:
          this.restartAccing() // 重新启动加速
          break
        case 17:
          this.overdueStopAccStatus() // 账户过期停止加速
          break
        case 18:
          console.log(this.cause) // 暂时不管
          break
        case 19:
          this.loginOkToAccing() // 正常加速的状态
          break
        case 20:
          this.logingFailed() // 登录失败
          break
        case 21:
          // this.abnormalStopAccStatus() // 异常停止加速
          this.userStopAccStatus() // 用户停止加速
          break
        case 22:
          // this.abnormalStopAccStatus() // 异常停止加速
          this.userStopAccStatus() // 用户停止加速
          break
        case 23:
          // this.abnormalStopAccStatus() // 异常停止加速
          this.userStopAccStatus() // 用户停止加速
          break
        case 24:
          // this.abnormalStopAccStatus() // 异常停止加速
          this.userStopAccStatus() // 用户停止加速
          break
        case 25:
          // this.abnormalStopAccStatus() // 异常停止加速
          this.userStopAccStatus() // 用户停止加速
          break
        case 500:
          // this.abnormalStopAccStatus() // 异常停止加速
          this.userStopAccStatus() // 用户停止加速
          break
        default:
          console.log('event:', this.event, 'cause:', this.cause)
      }
    },
    notLoggedIn: function() {
      // 未登录 cause = 0 | 3
      // 未登录时的正常状态
      this.txtSpeedBtn = `${this.$t('i18n.LOGIN')}<br>${this.$t('i18n.QEE_YOU')}`
      this.isShow = false
      this.isShowDelay = false
      this.isShowSpeedUpPng = false
      this.isShowErrorTip = false
    },
    logingFailed: function() {
      // 登录失败 cause = 20
      this.notLoggedIn()
      this.isShowErrorTip = true
      this.txtTipShow = accCause[this.causeFormArr].cause
    },
    busyLine: function() {
      // 占线 cause = 4
      // 弹出强制下线的页面
      this.$router.push({path: '/login-others', query: {status: this.status}})
    },
    logingToLoginOk: function() {
      this.loginOkToAccing()
    },
    loginOkToAccing: function() {
      // 正常加速的状态 cause = 5 | 6 | 7
      this.isShow = true
      this.isShowSpeedUpPng = true
      this.isShowDelay = false
      this.txtSpeedBtn = this.$t('i18n.SPEED_RUNNING')
    },
    userStopAccStatus: function() {
      // 用户停止加速 cause !== 17
      this.isShowDelay = false
      this.isShow = false
      this.txtSpeedBtn = '开始加速'
      if (this.cause === 8) {
        this.isShowErrorTip = false
      } else {
        this.isShowErrorTip = true
      }
      this.isShowSuccessTip = false
      this.isShowSpeedUpPng = false
      // this.isShowRestartBtn = true
      this.txtTipShow = accCause[this.causeFormArr].cause
    },
    abnormalStopAccStatus: function() {
      // 异常停止加速 cause = 10 | 14 | 15
      this.isShowDelay = false
      this.isShow = false
      this.txtSpeedBtn = this.$t('i18n.NO_SPEEDING')
      this.isShowErrorTip = true
      this.isShowSuccessTip = false
      this.isShowSpeedUpPng = false
      this.isShowRestartBtn = false
      this.txtTipShow = accCause[this.causeFormArr].cause
    },
    overdueStopAccStatus: function() {
      // 账户过期停止加速 cause = 17
      this.isShowDelay = false
      this.isShow = false
      this.isShowSpeedUpPng = false
      this.isShowRestartBtn = false
      this.isShowSuccessTip = false
      this.isShowErrorTip = true
      this.txtSpeedBtn = this.$t('i18n.RENEW_NOW')
      this.txtTipShow = accCause[this.causeFormArr].cause
    },
    accingToAccOk: function() {
      // 加速成功
      this.txtSpeedBtn = this.$t('i18n.SPEED_SUCCESS')
      this.isShow = true
      this.isShowDelay = true
      this.isShowSpeedUpPng = false
      this.$root.Bus.$emit('busToShowLoginOutBtn', true) // 显示登出按钮
      this.isShowErrorTip = false
      this.txtTipShow = accCause[this.causeFormArr].cause
    },
    restartAccing: function() {
      // 重新启动加速
      this.loginOkToAccing()
    },
    watchRouter: function() {
      // 监听路由
      let path = this.$route.path
      if (path === '/') {
        this.getAccStatus()
        clearInterval(this.accInterval)
        this.getAccInterval(5170)
        clearInterval(this.handleInterval)
        this.handleAccState()
      } else {
        clearInterval(this.handleInterval)
        clearInterval(this.accInterval)
      }
    },
    watchStatus: function() {
      // 监听acc的状态（status）
      if (this.status === 'ACCING' || !this.status) { // 根据status变化，控制是开启常规轮询还是加速轮询
        this.getAccStatus()
        clearInterval(this.accInterval)
        this.getAccInterval(5170)
      } else {
        clearInterval(this.accInterval)
        this.getAccInterval(5170)
        this.isEmitBusCheckStateStatus = true
      }
      if (this.status === 'ACC_OK') { // 控制主界面加速分区和模式为自动时，是否显示真正的分区和模式
        this.isShowRestartBtn = true
        this.$root.Bus.$emit('busCurrentPM', true, {'mode': this.currentMode, 'pattern': this.currentPattern})
      } else {
        this.isShowRestartBtn = false
        this.$root.Bus.$emit('busCurrentPM', false)
      }
      if (this.status === 'INIT' || !this.status) { // 控制是否显示登出按钮
        this.$root.Bus.$emit('busIndexSpeedUp') // 控制主界面加速模块的登出状态
        this.$root.Bus.$emit('memberLoginOut') // 控制导航栏登出的视图效果
        this.$root.Bus.$emit('busToShowLoginOutBtn', false) // 隐藏登出按钮
        this.$root.Bus.$emit('busMgr128', false) // 未登录状态下，固件升级的按钮不移动
      } else {
        this.$root.Bus.$emit('busToShowLoginOutBtn', true) // 显示登出按钮
        this.$root.Bus.$emit('busMgr128', true) // 登录状态下，将固件升级的按钮向左移动
      }
      if (this.isDelayApi) {
        this.isDelayApi = false
        this.$root.Bus.$emit('memberLoginSuccess', this.accData) // 控制该组件是否显示过期页面
        let aa = setTimeout(() => {
          clearTimeout(aa)
          this.$root.Bus.$emit('busInitAccState', this.accData)
        }, 150)
        this.$root.Bus.$emit('busGetAccState', this.accData)
      }
      let cc = setTimeout(() => {
        clearTimeout(cc)
        this.isDelayApi = true
      }, 2000)
    },
    apiNetUpnpGet: function () { // 判断upnp是否开启
      netUpnpGet()
        .then((res) => {
          let enable = parseInt(res.enable)
          if (enable === 1) {
            this.isUpnpEnable = true
          } else {
            this.isUpnpEnable = false
          }
        })
        .catch((e) => {
          this.isUpnpEnable = false
        })
    },
    openNat: function () { // 判断是否点亮NAT图标
      let reg = /p2p/ig
      let result = reg.test(this.currentPattern)
      if (result) {
        this.isNatEnable = true
      } else {
        this.isNatEnable = false
      }
    },
    focusAccBtn: function(status) { // 控制是否显示停止加速按钮
      if (this.txtSpeedBtn === '正在停止加速' || this.cause === 17 || this.userType === 3 || this.userType === 7 || this.userType === 15 || !this.getAccProcess) {
        this.isFocusBtn = false
      } else {
        if (this.status === 'ACCING' || this.status === 'ACC_OK') {
          this.isFocusBtn = status
        } else {
          this.isFocusBtn = false
        }
      }
    },
    clickAccBtn: function() { // 点击加速按钮
      if (this.isFocusBtn && this.clickStopBtnNum === 1) {
        this.clickStopBtnNum++
        // 点击停止加速按钮,调用停止加速
        this.apiControlAcc('stop')
        return false
      }
      if (this.status === 'INIT') {
        // 进入登录页面
        this.$router.push('/member-login')
        return false
      }
      if (this.status === 'LOGIN_OK' && this.event === 6 && this.cause !== 17 && this.txtSpeedBtn === '开始加速') {
        clearInterval(this.accInterval)
        this.apiControlAcc('restart')
        return false
      }
      // 账户过期
      if (this.cause === 17 || this.userType === 3 || this.userType === 7 || this.userType === 15) {
        window.open('https://www.qiyou.cn/member/opening')
        this.isShowRestartBtn = false
        return false
      }
      return false
    },
    errorAccDeal: function() {
      // let o = typeof (err)
      // if (o === 'object') {
      //   err = err.msg
      // }
      this.isShowErrorTip = true
      this.isShowSuccessTip = false
      this.txtTipShow = '连接到服务器失败或网络错误'
      this.txtSpeedBtn = `未加速`
      this.isShow = false
      this.isShowDelay = false
      this.isShowSpeedUpPng = false
      this.isShowRestartBtn = false
      clearInterval(this.accInterval)
      this.getAccInterval(5170)
    },
    handleAccState: function() {
      this.handleInterval = setInterval(() => {
        this.getAccStatus()
      }, 12272)
    }
  },
  created() {
    this.getAccStatus()
    this.getAccInterval()
    // 用户登录后，直接进入加速状态
    this.$root.Bus.$on('memberLoginSuccessOkBtn', data => {
      // 调用进入加速状态的方法
      this.status = data
      this.loginOkToAccing()
    })
    // 用户登出
    this.$root.Bus.$on('busSpeedUpBtnLoginout', () => {
      console.log(617)
      this.isShow = false
      this.isShowDelay = false
      this.isShowSpeedUpPng = false
      this.status = 'INIT'
      this.accData.status = this.status
      this.txtSpeedBtn = `${this.$t('i18n.LOGIN')}<br>${this.$t('i18n.QEE_YOU')}`
    })
    // 重新加速
    this.$root.Bus.$on('busRestartSpeedUp', () => {
      this.apiControlAcc('restart')
    })
    // 获取upnp
    this.$root.Bus.$on('busUpnpGet', () => {
      this.apiNetUpnpGet()
    })
  },
  mounted () {
    let aa = setTimeout(() => {
      clearTimeout(aa)
      this.apiNetUpnpGet()
    }, 2790)
    clearInterval(this.handleInterval)
    this.handleAccState()
  },
  beforeDestroy() {
    this.$root.Bus.$off('memberLoginSuccessBtn')
    this.$root.Bus.$off('busSpeedUpBtnLoginout')
    this.$root.Bus.$off('busRestartSpeedUp')
    this.$root.Bus.$off('memberLoginSuccessOkBtn')
    this.$root.Bus.$off('busUpnpGet')
    // 清除轮询
    clearInterval(this.accInterval)
    clearInterval(this.handleInterval)
  },
  watch: {
    // $route: 'getRouteFn' // 获取路由
    $route: 'watchRouter', // 监听路由
    status: 'watchStatus' // 监听acc状态
  }
}
</script>

<style lang="scss" scoped>
.ljb-index-left {
  display: flex;
  flex-direction: column;
  justify-content: center;
  width: 100%;
  height: 628px;
  float: left;
  background-color: #fff;
  border-radius: 6px;
  position: relative;
  box-shadow: 0px 20px 60px 0px rgba(0, 0, 0, 0.06);
  & > .btn-cover {
    border: 1px solid red;
    position: absolute;
    left: 0;
    top: 0;
    right: 0;
    bottom: 0;
    border-radius: 6px;
    background-color: red;
    z-index: 99;
    opacity: 0;
    display: none;
    &.display-block {
      display: block;
    }
  }
  .login-tip {
    font-size: 14px;
    line-height: 50px;
    color: rgba(58, 56, 72, 1);
    width: 100%;
    height: 50px;
    background: linear-gradient(
      90deg,
      rgba(0, 159, 232, 0.1),
      rgba(0, 159, 232, 0)
    );
    border-radius: 6px 6px 0px 0px;
    position: absolute;
    top: 0;
    text-align: center;
    &.error-tip {
      background: linear-gradient(
        90deg,
        rgba(241, 68, 102, 0.1),
        rgba(241, 68, 102, 0)
      );
      color: rgba(241, 68, 102, 1);
    }
    &.success-tip {
      background:linear-gradient(90deg,rgba(0,159,232,0.1),rgba(0,159,232,0));
    }
  }
  .btn-container {
    width: 260px;
    height: 260px;
    margin: 0 auto;
    position: relative;
    transition: margin-top .5s;
    & > .restart {
      width: 60px;
      height: 60px;
      border-radius: 100%;
      background-color: #fff;
      position: absolute;
      right: -45px;
      bottom: 0px;
      box-shadow: 0 5px 15px 0 rgba(0, 159, 232, 0.2);
      cursor: pointer;
      border: 2px solid rgba(0, 159, 232, 0.2);
      transition: box-shadow .5s;
      z-index: 99;
      & > div {
        font-size: 0;
        border: none;
        position: absolute;
        width: 28px;
        height: 28px;
        background: url(../../../assets/images/icon/icon_refresh_hover.png) no-repeat;
        background-size: cover;
        left: 50%;
        top: 50%;
        transform: translate(-50%, -50%);
        &.hover-bg {
          background: url(../../../assets/images/icon/icon_refresh_hover.png) no-repeat;
          background-size: cover;
        }
      }
      &:hover {
        border-color: rgba(245, 245, 245, 1);
        background: linear-gradient(
          90deg,
          rgba(57, 193, 255, 1),
          rgba(0, 159, 232, 1)
        );
        box-shadow: 0 6px 20px 0 rgba(0, 159, 232, 0.7);
        & > div {
          background: url(../../../assets/images/icon/icon_refresh3.png) no-repeat;
          background-size: cover;
        }
      }
      &:focus {
        border-color: rgba(245, 245, 245, 1);
        background: linear-gradient(
          90deg,
          rgba(57, 193, 255, 1),
          rgba(0, 159, 232, 1)
        );
        box-shadow: 0 6px 20px 0 rgba(0, 159, 232, 0.7);
        & > div {
          background: url(../../../assets/images/icon/icon_refresh3.png) no-repeat;
          background-size: cover;
        }
      }
    }
  }
  .speed-up-circle1 {
    width: 260px;
    height: 260px;
    border: 5px solid rgba(0, 159, 232, 0.06);
    border-radius: 1000px;
    display: block;
    transition: box-shadow 0.3s, border-color 0.3s, margin-top 0.5s;
    cursor: pointer;
    position: relative;
    outline: none;
    &:focus {
      outline: none;
    }
    &:hover {
      box-shadow: 0px 4px 20px 0px rgba(0, 159, 232, 0.5);
      border-color: rgba(0, 159, 232, 0.8);
      text-decoration: none;
      outline: none;
      .speed-up-circle2 {
        background: linear-gradient(
          0deg,
          rgba(137, 216, 255, 1),
          rgba(0, 175, 255, 1)
        );
      }
    }
    &:focus {
      box-shadow: 0px 4px 20px 0px rgba(0, 159, 232, 0.5);
      border-color: rgba(0, 159, 232, 0.8);
      text-decoration: none;
      .speed-up-circle2 {
        background: linear-gradient(
          0deg,
          rgba(137, 216, 255, 1),
          rgba(0, 175, 255, 1)
        );
      }
    }
  }
  .speed-up-circle2 {
    width: 200px;
    height: 200px;
    border-radius: 1000px;
    margin: -1px auto 0;
    padding-top: 1px;
    opacity: 0.7;
    background: linear-gradient(
      0deg,
      rgba(137, 216, 255, 0.7),
      rgba(0, 175, 255, 0.7)
    );
    filter: progid:DXImageTransform.Microsoft.gradient(
        startColorstr=rgba(137, 216, 255, 0.7),
        endColorstr=rgba(0, 175, 255, 0.7),
        GradientType=0
      );
    transition: background-color linear 0.5s;
    &:hover {
      background: linear-gradient(
        0deg,
        rgba(137, 216, 255, 1),
        rgba(0, 175, 255, 1)
      );
    }
  }
  .speed-up-circle3 {
    width: 168px;
    height: 168px;
    background-color: #fff;
    border-radius: 1000px;
    margin: 15px auto 16px;
    text-align: center;
    line-height: 192px;
    display: block;
    position: relative;
    & > h3 {
      line-height: 25px;
      color: rgba(58, 56, 72, 1);
      font-size: 18px;
      transition: color 0.5s;
      position: absolute;
      top: 50%;
      left: 50%;
      transform: translate(-50%, -50%);
    }
  }
  .delay-lost {
    width: 324px;
    height: 0px;
    transition: height 0.5s, opacity 0.5s;
    overflow: hidden;
    opacity: 0;
    margin: 0 auto;
    & > div {
      width: 150px;
      height: 140px;
      background: linear-gradient(
        180deg,
        rgba(240, 243, 247, 1),
        rgba(255, 255, 255, 1)
      );
      border-radius: 6px;
    }
    & > .delay {
      float: left;
      position: relative;
    }
    & > .lost {
      margin-left: 180px;
      position: relative;
    }
    i {
      display: inline-block;
      width: 24px;
      height: 4px;
      background: rgba(6, 244, 32, 1);
      box-shadow: 0px 2px 8px 0px rgba(53, 255, 182, 1);
      border-radius: 0px 0px 100px 100px;
      position: absolute;
      left: 50%;
      margin-left: -12px;
    }
    .lost-icon {
      background: rgba(255, 34, 34, 1);
      box-shadow: 0px 2px 6px 0px rgba(255, 53, 53, 1);
      border-radius: 0px 0px 100px 100px;
    }
    h4 {
      font-size: 16px;
      color: rgba(131, 137, 161, 1);
      line-height: 22px;
      text-align: center;
      padding-top: 30px;
    }
    p {
      padding-top: 20px;
      text-align: center;
      font-size: 14px;
      color: rgba(141, 139, 142, 1);
      & > em {
        font-size: 36px;
        letter-spacing: 0;
        line-height: 43px;
        color: rgba(44, 42, 56, 1);
      }
    }
    &.opacity-delay {
      opacity: 1;
    }
  }
  & > .nat-upnp-tips {
    position: absolute;
    top: 60px;
    left: 30px;
    width: 46px;
    height: 102px;
    & > span {
      display: block;
      width: 46px;
      height: 46px;
      &:nth-of-type(1) {
        margin-bottom: 10px;
        background: url(../../../assets/images/icon/icon_upnp_normal_index.png) no-repeat;
        background-size: cover;
      }
      &:nth-of-type(2) {
        background: url(../../../assets/images/icon/icon_nat_normal.png) no-repeat;
        background-size: cover;
      }
      &.upnp-tip {
        background: url(../../../assets/images/icon/icon_upnp_hover_index.png) no-repeat;
        background-size: cover;
      }
      &.nat-tip {
        background: url(../../../assets/images/icon/icon_nat_hover.png) no-repeat;
        background-size: cover;
      }
    }
    .icon-upnp {
      position: relative;
      cursor: pointer;
      &:hover {
        & > .upnp-txt-tip {
          display: block;
        }
      }
      & > .upnp-txt-tip {
        position: absolute;
        left: 50px;
        top: -10px;
        width: 205px;
        height: 58px;
        padding: 10px 14px;
        border-radius: 4px;
        display: none;
        background-color: rgba(71,196,255,1);
        & > i {
          position: absolute;
          width: 0;
          height: 0;
          border: 5px solid transparent;
          border-right-color: rgba(71,196,255,1);
          left: -10px;
          top: 50%;
          transform: translateY(-50%);
        }
        & > span {
          font-size: 12px;
          line-height: 17px;
          color: #fff;
        }
      }
    }
    .icon-nat {
      position: relative;
      cursor: pointer;
      &:hover {
        & > .nat-txt-tip {
          display: block;
        }
      }
      & > .nat-txt-tip {
        position: absolute;
        left: 50px;
        top: -10px;
        width: 205px;
        height: 58px;
        padding: 10px 14px;
        border-radius: 4px;
        display: none;
        background-color: rgba(71,196,255,1);
        & > i {
          position: absolute;
          width: 0;
          height: 0;
          border: 5px solid transparent;
          border-right-color: rgba(71,196,255,1);
          left: -10px;
          top: 50%;
          transform: translateY(-50%);
        }
        & > span {
          font-size: 12px;
          line-height: 17px;
          color: #fff;
        }
      }
    }
  }
}
@media screen and (max-width: 1366px) {
  .ljb-index-left {
    height: 500px;
  }
}
.ljb-index-left.ljb-index-left-animation {
  .btn-container {
    transition: margin-top .5s;
    margin-bottom: 30px;
    margin-top: 40px;
  }
  .speed-up-circle2 {
    background: linear-gradient(
      0deg,
      rgba(137, 216, 255, 0.7),
      rgba(0, 175, 255, 0.7)
    );
    background-size: 100% 100%;
  }
  .delay-lost {
    height: 150px;
  }
}
.ljb-index-left {
  -webkit-transform: translateZ(0);
  -moz-transform: translateZ(0);
  -ms-transform: translateZ(0);
  -o-transform: translateZ(0);
  transform: translateZ(0);
  -webkit-backface-visibility: hidden;
  -moz-backface-visibility: hidden;
  -ms-backface-visibility: hidden;
  backface-visibility: hidden;
  -webkit-perspective: 1000;
  -moz-perspective: 1000;
  -ms-perspective: 1000;
  perspective: 1000;
  .speed-up-running {
    position: absolute;
    width: 200px;
    height: 200px;
    top: 25px;
    left: 25px;
    transform: rotate(90deg);
    background: url(../../../assets/images/icon/index-update.png) no-repeat;
    background-size: 100% 100%;
    animation: speed-btn 1s linear infinite;
    @keyframes speed-btn {
      0% {
        transform: rotate(0deg);
      }
      25% {
        transform: rotate(90deg);
      }
      50% {
        transform: rotate(180deg);
      }
      75% {
        transform: rotate(270deg);
      }
      100% {
        transform: rotate(360deg);
      }
    }
  }
}
.btn-center {
  border: 1px solid red;
  width: 60px;
  height: 20px;
}
</style>
