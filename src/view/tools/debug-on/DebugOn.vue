<template>
  <div class="debug-on">
    <h3>{{ $t('i18n.REMOTE_DEBUG') }}</h3>
    <div class="debug-con">
      <div v-text="txt" v-show="!isLoading"></div>
      <div class="loading-wrapper" v-show="isLoading">
        <loading class="wifi-loading"></loading>
        <p class="text">{{ pTxt }}</p>
      </div>
    </div>
  </div>
</template>

<script>
import { sysRemoteDebugControl } from '@/api/system-manage-api'
import Loading from '@/common/components/loading/loading'
export default {
  name: 'DebugOn',
  data: function () {
    return {
      txt: null,
      isLoading: true,
      pTxt: this.$t('i18n.OPENING_REMOTE_DEBUG')
    }
  },
  components: {
    Loading
  },
  methods: {
    closeRemoteDebugFn: function () {
      this.isLoading = true
      sysRemoteDebugControl({action: 'off'})
        .then(res => {
          if (parseInt(res.ret) === 0) {
            this.openRemoteDebugFn()
          } else {
            this.isLoading = false
            this.pTxt = res.msg
          }
        })
        .catch(e => {
          this.isLoading = true
          this.pTxt = this.$t('i18n.REMOTE_DEBUG_FAILED_AND_OTHER')
        })
    },
    openRemoteDebugFn: function () {
      sysRemoteDebugControl({action: 'on'})
        .then(res => {
          if (parseInt(res.ret) === 0) {
            this.isLoading = false
            let obj = {}
            obj.web_id = res.web_id
            obj.ssh_id = res.ssh_id
            this.txt = JSON.stringify(obj)
            this.pTxt = this.$t('i18n.OPENING_REMOTE_DEBUG')
          } else {
            this.isLoading = false
            this.pTxt = res.msg
          }
        })
        .catch(e => {
          this.isLoading = true
          this.pTxt = this.$t('i18n.REMOTE_DEBUG_FAILED_AND_OTHER')
        })
    }
  },
  mounted () {
    this.closeRemoteDebugFn()
  }
}
</script>

<style lang="scss" scoped>
.debug-on {
  position: relative;
  width: 600px;
  height: 200px;
  background:linear-gradient(180deg,rgba(240,243,247,1),rgba(255,255,255,1));
  box-shadow:0px 20px 60px 0px rgba(0,0,0,0.06);
  border-radius:6px;
  margin: 0 auto;
  & > h3 {
    height:76px;
    background-color: #fff;
    border-radius: 6px 6px 0 0;
    font-size: 20px;
    color: rgba(58, 56, 72, 1);
    line-height: 76px;
    text-align: center;
    font-weight: bold;
  }
  & > .debug-con {
    width: 500px;
    height: 100px;
    margin: 0 auto;
    margin-top: 30px;
    font-size: 16px;
    color: rgba(58, 56, 72, 1);
    text-align: left;
  }
  .loading-wrapper {
    text-align: center;
    & > .wifi-loading {
      margin: 0 auto;
    }
    & > p {
      margin-top: 10px;
    }
  }
}
</style>
