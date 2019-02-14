<template>
<div class="wrapper">
  <div class="login-others-wrapper">
    <h2 class="title">{{$t('i18n.LOGIN_OTHERS')}}</h2>
    <fade-tips
    :msg="tips.msg"
    :show="tips.show"
    :type="tips.type"
    @hide="tips.show = false"></fade-tips>
  </div>
  <div class="sys-check-failed">
    <button class="restart" @click="clearAccStatusFn()">
      {{$t('i18n.YES_KNOW')}}
    </button>
    <button class="go-index" @click="restartToLoginFn()">
      <span><i :class="{'display': isShowWaitting}">q</i></span>
      {{$t('i18n.RELOGIN')}}
    </button>
  </div>
</div>
</template>

<script>
import { PrimaryButton } from '../../../common/components/buttons'
import FadeTips from '../../../common/components/fade-tips/FadeTips'

import { clearUserInfo, UserLoginApi } from '../../../api/user-manger-api'
import { accStateGet } from '@/api/speed-manage-api'

export default {
  name: 'LoginOthers',
  data() {
    return {
      waiting: false,
      text: this.$t('i18n.SURE'),
      tips: {
        msg: '',
        type: 'success',
        show: false
      },
      isShowWaitting: false
    }
  },
  components: {
    PrimaryButton,
    FadeTips
  },
  methods: {
    restartToLoginFn: function() {
      let status = this.$route.query.status
      switch (status) {
        case 'INIT':
          this.toLoginViewFn()
          break
        case 'LOGIN_OK':
          // this.loginokToLogoutFn()
          break
        default:
          console.log(status)
      }
      this.$router.push('/member-login')
    },
    clearAccStatusFn: function() {
      let status = this.$route.query.status
      switch (status) {
        case 'INIT':
          this.clearUserInfoFn()
          break
        case 'LOGIN_OK':
          // this.initToLogoutFn()
          break
        default:
          console.log(status)
      }
    },
    clearUserInfoFn: function() {
      clearUserInfo()
        .then(res => {
          this.$router.push('/')
        })
        .catch(e => {
          this.tips.msg = this.$t('i18n.FAILED_CLICK_BTN_ONCE')
          this.tips.type = 'error'
          this.tips.show = true
        })
    },
    initToLogoutFn: function() {
      accStateGet()
        .then(res => {
          let status = res.status
          if (status === 'INIT') {
            this.clearUserInfoFn()
          } else {
            this.tips.msg = this.$t('i18n.FAILED_CLICK_BTN_ONCE')
            this.tips.type = 'error'
            this.tips.show = true
          }
        })
        .catch(e => {
          this.tips.msg = this.$t('i18n.FAILED_CLICK_BTN_ONCE')
          this.tips.type = 'error'
          this.tips.show = true
        })
    },
    loginokToLogoutFn: function() {
      UserLoginApi({ action: 'logout' })
        .then((data) => {
          this.$root.Bus.$emit('busSpeedUpBtnLoginout')
          this.$root.Bus.$emit('busIndexSpeedUp')
          this.$root.Bus.$emit('busShowUpdateBtn', true)
          this.$root.Bus.$emit('memberLoginOut')
          this.$router.push('/member-login')
        })
        .catch(e => {
          this.tips.msg = this.$t('i18n.FAILED_CLICK_LOGIN_ONCE')
          this.tips.type = 'error'
          this.tips.show = true
        })
    },
    toLoginViewFn: function() {
      this.$router.push('/member-login')
    }
  }
}
</script>

<style lang="scss" scoped>
@import '../../../common/sass/_constant.scss';
@import '../../../common/sass/_mixin.scss';
.login-others-wrapper {
  width: 400px;
  margin: 0 auto;
  background:rgba(255,255,255,1);
  border-radius:6px;
  padding: 60px 46px 40px;
  position: relative;
  .title {
    font-size:22px;
    color:rgba(58,56,72,1);
    margin-bottom: 25px;
  }
  .login-button {
    width: 280px;
    margin: 0 auto;
  }
}
.wrapper {
  width: 400px;
  margin: 0 auto;
  & > .sys-check-failed {
    // overflow: hidden;
    height: 56px;
    margin-top: 30px;
    & > button {
      @include startBtn-primaryHover();
      height: 56px;
      width: 48%;
      &.restart {
        float: left;
      }
      &.go-index {
        float: right;
        color:rgba(58,56,72,1);
        background: white;
        position: relative;
        &:hover {
          &::after {
            border-color: #31BCFC;
          }
        }
        &:focus {
          &::after {
            border-color: #31BCFC;
          }
        }
        & > span {
          // display: inline-block;
          font-size: 0;
          position: absolute;
          width: 20px;
          height: 20px;
          left: 40px;
          top: 50%;
          transform: translateY(-50%);
          & > i {
            font-size: 0;
            position: absolute;
            left: 0;
            top: 0;
            right: 0;
            bottom: 0;
            background: url(../../../assets/images/icon/blue-waitting.png) no-repeat;
            background-size: cover;
            animation: waitting 1s linear infinite;
            display: none;
            @keyframes waitting {
              0% {
                transform: rotateZ(0deg)
              }
              50% {
                transform: rotateZ(180deg)
              }
              100% {
                transform: rotateZ(360deg)
              }
            }
            &.display {
              display: block;
            }
          }
        }
      }
    }
  }
}
</style>
