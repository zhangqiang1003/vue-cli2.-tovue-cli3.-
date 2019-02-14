<template>
  <div class="login">
    <p class="title">{{ $t('i18n.USER_LOGIN') }}</p>
    <label class="email" for="email" :class="{'error': !userName.right}">
      <input
        type="text"
        @focus="inputFoucs(userName)"
        @blur="inputFoucs(userName)"
        @keyup.enter="login"
        name="email" id="email"
        :class="{'error':!userName.right}"
        :placeholder="$t('i18n.MEMBER_ACCOUNT_PLACEHOLDER')"
        autocomplete="new-password"
        v-model.trim="userName.value">
      <div class="input-error" v-show="!userName.right">{{ userNameErrorTip }}</div>
      <div class="clear" @click="clearInputFn()" v-show="userName.right"><i></i></div>
    </label>
    <label class="pass" for="pass" :class="{'error': !userPass.right}">
      <input
        :type="inputType"
        @focus="inputFoucs(userPass)"
        @blur="inputFoucs(userName)"
        @keyup.enter="login"
        name="pass" id="pass"
        :class="{'error': !userPass.right}"
        :placeholder="$t('i18n.PASSWORD_PLACEHOLDER')"
        autocomplete="new-password"
        v-model.trim="userPass.value">
      <div class="input-error" v-show="!userPass.right">{{ $t('i18n.PASSWORD_PLACEHOLDER') }}</div>
      <div class="switch-psd" v-show="userPass.right" @click="switchPsdFn()">
        <i :class="{'close': isCloseStatus, 'open': !isCloseStatus}"></i>
      </div>
    </label>
    <primary-button
      :disabled="waitting"
      :loading="waitting"
      :text="btnText"
      height="46"
      @btnClicked="login()"
      class="login-btn">
    </primary-button>
    <a class="forget-pass-btn" type="button" href="https://www.qiyou.cn/forgot-password" target="_blank">{{ $t('i18n.FORGOT_PASSWORD') }}？</a>
    <fade-tips :msg="tips.msg" :show="tips.show" :type="tips.type" @hide="tips.show = false"></fade-tips>
  </div>
</template>

<script>
import VaildTools from '../../../common/service/regular'
import { forceOfflineApi, UserLoginApi } from '../../../api/user-manger-api'

// import dealErrorCode from '../../../common/service/dealErrorCode'

import { PrimaryButton } from '../../../common/components/buttons/index'
import FadeTips from '@/common/components/fade-tips/FadeTips'
import { trimEmpty } from '@/common/service/utils'
export default {
  name: 'LoginComponent',
  data() {
    return {
      userName: {
        value: '',
        right: true
      },
      userPass: {
        value: '',
        right: true
      },
      tips: {
        msg: '',
        type: 'success',
        show: false
      },
      waitting: false,
      btnText: this.$t('i18n.LOGIN'),
      userDetails: null,
      userNameErrorTip: this.$t('i18n.MEMBER_ACCOUNT_PLACEHOLDER'),
      isCloseStatus: true,
      inputType: 'password'
    }
  },
  components: {
    FadeTips,
    PrimaryButton
  },
  methods: {
    login() { // 登录
      this.check()
      if (this.userName.right && this.userPass.right) {
        this.waitting = true
        this.$root.Bus.$emit('maskClick', false) // 不允许毛玻璃效果被点击
        UserLoginApi({
          action: 'login',
          name: this.userName.value,
          password: this.userPass.value
        })
          .then(res => {
            this.userDetails = res
            window.localStorage.setItem('username', this.userName.value)
            let userConflict = parseInt(res.userConflict)
            if (userConflict === 0) {
              this.$root.Bus.$emit('memberLoginSuccess', res) // 触发Bus登录成功事件，用于显示隐藏过期时间和登录按钮
              this.$root.Bus.$emit('maskClick', true) // 触发Bus事件，允许毛玻璃层被点击
              this.$root.Bus.$emit('isLogin', true) // 触发Bus事件，显示NavList的注销按钮
              this.$root.Bus.$emit('memberLoginSuccessBtn', 'LOGIN_OK')
              this.$root.Bus.$emit('memberLoginSuccessOkBtn', 'LOGIN_OK')
              this.$root.Bus.$emit('busLoginOk', 'LOGIN_OK')
              this.$router.push('/')
            } else {
              forceOfflineApi()
                .then(res => {
                  this.$root.Bus.$emit('memberLoginSuccess', this.userDetails) // 触发Bus登录成功事件，用于显示隐藏过期时间和登录按钮
                  this.$root.Bus.$emit('maskClick', true) // 触发Bus事件，允许毛玻璃层被点击
                  this.$root.Bus.$emit('isLogin', true) // 触发Bus事件，显示NavList的注销按钮
                  this.$root.Bus.$emit('memberLoginSuccessOkBtn', 'LOGIN_OK')
                  this.$root.Bus.$emit('memberLoginSuccessBtn', 'LOGIN_OK')
                  this.$root.Bus.$emit('busLoginOk', 'LOGIN_OK')
                  this.$router.push('/')
                })
            }
          })
          .catch(e => {
            this.waitting = false
            this.$root.Bus.$emit('maskClick', true)
            if (e.message) {
              this.tips.msg = e.message
            }
            let code = parseInt(e.code)
            switch (code) {
              case 34:
                this.tips.msg = this.$t('i18n.DO_NOT_REPEAT_LOGIN')
                break
              case 15:
                this.tips.msg = this.$t('i18n.USER_LOGIN_ANOTHER_LJB')
                break
              case 12:
                this.tips.msg = this.$t('i18n.ERROR_CODE_12')
                break
              case 11:
                this.tips.msg = this.$t('i18n.ERROR_CODE_11')
                break
              case 13:
                this.tips.msg = this.$t('i18n.ERROR_CODE_13')
                break
              default:
                this.tips.msg = `${this.$t('i18n.DO_NOT_KONW_ERROR')}(code=${code})`
            }
            this.tips.type = 'error'
            this.tips.show = true
          })
      }
    },
    check() {
      this.userName.value = trimEmpty(this.userName.value, 2)
      this.userPass.right = VaildTools.password(this.userPass.value)
      this.userName.right = Boolean(this.userName.value)
      let isEmpty = Boolean(this.userName.value) // 验证是否为空
      if (isEmpty) {
        // 不为空
        let isRight = VaildTools.verifyUserName(this.userName.value) // 验证是否合法
        if (isRight) {
          // 合法
          this.userName.right = true
        } else {
          // 不合法
          this.userName.right = false
          this.userNameErrorTip = this.$t('i18n.INCORRECT_ACCOUNT_NAME_FORMAT')
        }
      } else {
        // 为空
        this.userName.right = false
        this.userNameErrorTip = this.$t('i18n.MEMBER_ACCOUNT_PLACEHOLDER')
      }
    },
    inputFoucs(item) {
      item.right = true
    },
    clearInputFn: function () {
      this.userName.value = ''
    },
    switchPsdFn: function () {
      this.isCloseStatus = !this.isCloseStatus
      if (this.isCloseStatus) {
        this.inputType = 'password'
      } else {
        this.inputType = 'text'
      }
    }
  },
  mounted() {
    let username = window.localStorage.getItem('username')
    this.userName.value = username
  }
}
</script>

<style lang="scss" scoped>
@import '../../sass/_constant.scss';
@import '../..//sass/_mixin.scss';
.login {
  width: $startContentWidth;
  margin: 0 auto;
  background: white;
  overflow: hidden;
  display: flex;
  flex-direction: column;
  border-radius: $startContentRaduis;
  position: relative;
  .title {
    font-size: 22px;
    color: rgba(58, 56, 72, 1);
    line-height: 30px;
    margin-top: 40px;
    margin-bottom: 50px;
    text-align: center;
  }
  label {
    @include inputWarnTip;
    position: relative;
    display: block;
    width: 312px;
    height: 46px;
    margin: 0 auto;
    margin-bottom: 10px;
    &.error {
      &::before {
        background: #f14466;
      }
    }
    &::before {
      content: '';
      position: absolute;
      bottom: 0;
      left: 4px;
      width: 305px;
      height: 1px;
      background: #eee;
      z-index: 90;
    }
    input {
      position: relative;
      width: 100%;
      height: 100%;
      border: none;
      outline: none;
      padding-left: 10px;
      border: 1px solid transparent;
      border-radius: $startContentRaduis;
      color: rgba(58, 56, 72, 1);
      font-size: 14px;
      &:focus {
        border: 1px solid #009fe8;
        box-shadow: 0px 6px 20px 0px rgba(0, 159, 232, 0.5);
        z-index: 99;
      }
      &::-ms-clear{ display: none; }
      &::-ms-reveal{ display: none; }
    }
    & > .clear {
      position: absolute;
      top: 50%;
      transform: translateY(-50%);
      right: 10px;
      width: 30px;
      height: 30px;
      cursor: pointer;
      z-index: 99;
      & > i {
        position: absolute;
        width: 18px;
        height: 18px;
        transform: translate(-50%, -50%);
        left: 50%;
        top: 50%;
        background: url(../../../assets/images/icon/icon_Input_box_delete_normal.png) no-repeat;
        background-size: cover;
        &:hover {
          background: url(../../../assets/images/icon/icon_Input_box_delete_hover.png) no-repeat;
          background-size: cover;
        }
      }
    }
    & > .switch-psd {
      position: absolute;
      top: 50%;
      transform: translateY(-50%);
      right: 10px;
      width: 30px;
      height: 30px;
      cursor: pointer;
      z-index: 99;
      & > i {
        position: absolute;
        width: 18px;
        height: 18px;
        left: 50%;
        top: 50%;
        transform: translate(-50%, -50%);
        &.close {
          background: url(../../../assets/images/icon/icon_Input_box_close_normal.png) no-repeat;
          background-size: cover;
          &:hover {
            background: url(../../../assets/images/icon/icon_Input_box_close_hover.png) no-repeat;
            background-size: cover;
          }
        }
        &.open {
          background: url(../../../assets/images/icon/icon_Input_box_open_normal.png) no-repeat;
          background-size: cover;
          &:hover {
            background: url(../../../assets/images/icon/icon_Input_box_open_hover.png) no-repeat;
            background-size: cover;
          }
        }
      }
    }
  }

  .login-btn {
    width: 312px;
    margin: 0 auto;
    margin-bottom: 20px;
    margin-top: 40px;
  }
  .forget-pass-btn {
    @include startBtn-outline();
    display: block;
    text-align: center;
    background: transparent;
    font-size: 14px;
    color: rgba(131, 137, 161, 1);
    line-height: 46px;
    text-decoration: none;
    border: 1px solid transparent;
    &:hover,
    &:focus {
      border: 1px solid #009fe8;
      line-height: 44px;
      color: rgba(0, 159, 232, 1);
    }
    margin-bottom: 30px;
  }
}
</style>
