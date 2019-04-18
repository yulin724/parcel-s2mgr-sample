<template>
<div>
  <div class="mui--text-center">
    <span class="dialogTitle">Add New Device</span>
  </div>
  <div>
    <strong>{{msg}}</strong>
    <p>TODO: 60秒倒计时认证需要</p>
  </div>
  <div v-if="dskInputVisible">
    <input v-model.text="firstDigitalOfDSK"
           type="text">
    <button class="pure-button button-success"
            @click="abort">Confirm</button>
  </div>
  <br/>
  <br/>
  <button class="pure-button button-success"
          @click="startInclude">Start Include</button>
  <button class="pure-button button-error"
          @click="abort">Abort</button>
</div>

</template>

<script>
import bus from '../bus'
import api from '../api'

export default {
  name: 'ClassicInclusionPage',
  props: [
    'uid'
  ],
  data() {
    return {
      msg: '',
      dskInputVisible: false,
      firstDigitalOfDSK: '12345'
    }
  },
  methods: {
    startInclude() {
      // 调用API
      api.ZW.StartInclude(this.uid, {
        success: (payload) => {
          console.log('success')
          this.msg = 'Succeed.'
        },
        failure: (payload) => {
          console.log('failure')
          this.msg = 'Failed.'
        },
        notified: (event, params) => {
          if (event === 'ZW.ModeChanged' && params.mode === 'including') { //
            this.msg = '处于添加模式中'
          } else if (event === 'ZW.ModeChanged' && params.mode === 'discovered') {
            this.msg = '发现新设备'
          } else if (event === 'ZW.ModeChanged' && params.mode === 'querying') {
            this.msg = '正在查询设备信息'
          } else if (event === 'ZW.ModeChanged' && params.mode === 'device_dsk_enter') {
            // input_pin10_on_joining_device
            this.msg = '请在被添加的设备上输入: ' + params.pin10
          } else if (event === 'ZW.ModeChanged' && params.mode === 'received_full_dsk') {
            this.msg = '确定要添加 ' + params.dsk + ' 设备吗?'
          } else if (event === 'ZW.ModeChanged' && params.mode === 'received_parted_dsk') {
            this.msg = '请在被添加设备DSK前5个数字, DSK的其它部分为: ' + params.dsk
            // 需要显示输入对话框
            this.dskInputVisible = true
          } else if (event === 'ZW.DeviceIncluded') {
            // 重要!!!
            this.msg = '成功添加设备，节点号 ' + params.nodeid
          } else if (event === 'ZW.ModeChanged' && params.mode === 'abort') {
            this.msg = '添加模式已经停止'
          } else {
            this.msg = event + ' ' + JSON.stringify(params)
          }
        }
      })
    },
    abort() {
      api.ZW.AbortLastOp(this.uid, {
        success: (payload) => {
        },
        failure: (payload) => {
        },
        notified: (event, params) => {
        }
      })
    }
  },
  mounted() {}
}

</script>

<style lang="css" scoped="">
</style>
