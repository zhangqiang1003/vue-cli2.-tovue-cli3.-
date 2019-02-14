<template>
  <div class="button-wrapper primary" :style="{height: `${height}px`}">
    <button
    type="button"
    :class="{'loading': loading, 'disabled': disabled && isBetaVer}"
    :disabled="disabled"
    @click="clickBtn"
    :style="{height: `${height}px`}">{{ text }}</button>
  </div>
</template>

<script>
export default {
  props: {
    disabled: {
      type: Boolean,
      default: false
    },
    loading: {
      type: Boolean,
      default: false
    },
    text: {
      type: String,
      default: ''
    },
    height: {
      type: String,
      default: '56'
    },
    isBetaVer: {
      type: Boolean,
      default: false
    }
  },
  methods: {
    clickBtn() {
      this.$emit('btnClicked')
    }
  }
}
</script>

<style lang="scss" scoped>
@import '../../sass/_constant';
@import '../../sass/_mixin.scss';

.button-wrapper {
  width: 100%;
  margin-top: 40px;
  button {
    @include startBtn-primaryHover();
    width: 100%;
    &.disabled {
      background: #888;
      &:hover {
        box-shadow: 0px 6px 20px 0px rgba(136, 136, 136, 0.5);
      }
    }
    &.loading {
      &::before {
        position: relative;
        bottom: 2px;
        margin-right: 10px;
        content: '';
        display: inline-block;
        width: 18px;
        height: 18px;
        background: url('../../../assets/images/icon/icon_buttonloading@2x.png');
        background-size: 100%;
        vertical-align: middle;
        animation: identifier 2s linear;
        animation-iteration-count: infinite;
      }
      @keyframes identifier {
        0% {
          transform: rotate(0deg);
        }
        50% {
          transform: rotate(180deg);
        }
        100% {
          transform: rotate(360deg);
        }
      }
    }
  }
}
</style>
