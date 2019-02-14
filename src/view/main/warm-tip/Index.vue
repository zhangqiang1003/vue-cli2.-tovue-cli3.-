<template>
  <div id="ljb-main-warm-tip" class="">
    <div class="warm-tip-title">{{ $t('i18n.WARM_TIP') }}</div>
    <div ref="BoxTxt" class="warm-tip-txt" id="warm-tip-txt">
      <div class='box-txt' v-html="txt"></div>
    </div>
  </div>
</template>

<script>
import { warmTipsApi } from '@/api/system-manage-api'
export default {
  name: 'WarmTip',
  data: function () {
    return {
      txt: this.$t('i18n.GETTING_WARM_TIP')
    }
  },
  methods: {
    apiWarmTips: function () {
      // this.txt = window.localStorage.getItem('warmTip')
    },
    apiWarmTipsFn: function () {
      this.$nextTick(() => {
        warmTipsApi()
          .then(res => {
            this.txt = res.content
            let aa = setTimeout(() => {
              clearTimeout(aa)
              this.scrollFn()
            }, 10)
          })
          .catch(e => {
            this.txt = this.$t('i18n.GET_WARM_TIP_FAILED_AND_RESETART')
          })
      })
    },
    scrollFn: function () {
      this.$nextTick(() => {
        if (!this.bar) {
          this.bar = new this.$geminiScrollbar({
            element: this.$refs.BoxTxt
          }).create()
        } else {
          this.bar.update()
        }
      })
    }
  },
  created () {
    // this.apiWarmTips()
    this.apiWarmTipsFn()
  },
  mounted () {
    // this.$nextTick(() => {
    //   new this.$geminiScrollbar({
    //     element: document.getElementById('warm-tip-txt')
    //   }).create()
    // })
  }
}
</script>

<style lang="scss" scoped>
@mixin component-center {
  position: absolute;
  top: 0px;
  left: 50%;
  transform: translateX(-50%);
}
#ljb-main-warm-tip {
  width: 560px;
  height: 536px;
  overflow: auto;
  background: linear-gradient(
    180deg,
    rgba(240, 243, 247, 1),
    rgba(255, 255, 255, 1)
  );
  box-shadow: 0px 20px 60px 0px rgba(0, 0, 0, 0.06);
  border-radius: 6px;
  @include component-center;
  & > .warm-tip-title {
    width: 100%;
    height: 76px;
    color: rgba(58, 56, 72, 1);
    font-size: 20px;
    font-weight: bold;
    line-height: 76px;
    text-align: center;
    background-color: #fff;
    border-radius: 6px 6px 0 0;
  }
  .warm-tip-txt {
    width: 560px;
    height: 460px;
    overflow-y: auto;
  }
  .box-txt {
    padding: 30px 40px 60px!important;
    line-height: 25px;
    letter-spacing: 1px;
  }
}
</style>

<style lang="scss">
  .box-txt {
    & > h3 {
      margin-bottom: 10px;
      font-weight: bold;
    }
    & > p {
      color: #75738B;
      text-indent: 2em;
      line-height: 22px;
    }
  }
</style>
