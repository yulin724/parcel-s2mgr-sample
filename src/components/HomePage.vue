<template>
<div style="padding:15px;">
  <!--  -->
  <!--  -->
  <!-- Gateway IP地址 -->
  <div v-if="!mqtt.connected"
       class="pure-form">
    <fieldset>
      <legend>Gateway Setting:</legend>
      <input type="text"
             style="width:100%;"
             v-model="gw.uid"
             placeholder="uid">
      <input type="text"
             style="width:100%;"
             v-model="mqtt.host">
      <button @click="mqttConnect"
              class="pure-button button-success">{{mqtt.connected?'Connected':'Connect'}}</button>
    </fieldset>
    <hr/>
  </div>
  <!--  -->
  <!--  -->
  <!-- 功能界面UI -->
  <div style="padding:5px;">
    <button class="pure-button button-error"
            @click="openDialog('ResetDialog')">Reset</button>
  </div>
  <div style="padding:5px;">
    <button class="pure-button button-warning"
            @click="openDialog('ClassicInclusionDialog')">Abort</button>
  </div>
  <div style="padding:5px;">
    <button class="pure-button button-secondary"
            @click="openDialog('ClassicInclusionDialog')">Classic Include</button>
  </div>
  <div style="padding:5px;">
    <button class="pure-button button-secondary"
            @click="openDialog">Smart Include with QRCode (InDev)</button>
  </div>
  <div style="padding:5px;">
    <button class="pure-button button-secondary"
            @click="openDialog('ExclusionDialog')">Exclude</button>
  </div>
  <div style="padding:5px;">
    <button class="pure-button button-secondary"
            @click="openDialog('RemoveFailedNodeDialog')">Remove Failed Node</button>
  </div>
  <div style="padding:5px;">
    <button class="pure-button button-secondary"
            @click="openDialog('ReplaceFailedNodeDialog')">Replace Failed Node (In Dev)</button>
  </div>
  <div style="padding:5px;">
    <button class="pure-button button-secondary"
            @click="openDialog">Learn Mode (InDev)</button>
  </div>
  <div style="padding:5px;">
    <button class="pure-button button-secondary"
            @click="openDialog('UpdateNetworkDialog')">Update Network</button>
  </div>
  <div style="padding:5px;">
    <button class="pure-button button-secondary"
            @click="openDialog">Backup & Restore(InDev)</button>
  </div>
  <div style="padding:5px;">
    <button class="pure-button button-secondary"
            @click="openDialog">Configuration(InDev)</button>
  </div>
  <div style="padding:5px;">
    <button class="pure-button button-secondary"
            @click="openDialog">Group Assocation(InDev)</button>
  </div>
  <div style="padding:5px;">
    <button class="pure-button button-secondary"
            @click="openDialog">Device OTA (InDev)</button>
  </div>
  <modal name="ResetDialog">
    <div class="dialogLayout">
      <ResetNetworkPage :uid="this.gw.uid" />
    </div>
  </modal>
  <modal name="ClassicInclusionDialog">
    <div class="dialogLayout">
      <ClassicInclusionPage :uid="this.gw.uid" />
    </div>
  </modal>
  <modal name="ExclusionDialog">
    <div class="dialogLayout">
      <ExclusionPage :uid="this.gw.uid" />
    </div>
  </modal>
  <modal name="RemoveFailedNodeDialog">
    <div class="dialogLayout">
      <RemoveFailedNodePage :uid="this.gw.uid" />
    </div>
  </modal>
  <modal name="ReplaceFailedNodeDialog">
    <div class="dialogLayout">
      <ReplaceFailedNodePage :uid="this.gw.uid" />
    </div>
  </modal>
  <modal name="UpdateNetworkDialog">
    <div class="dialogLayout">
      <UpdateNetworkPage :uid="this.gw.uid" />
    </div>
  </modal>
</div>

</template>

<script>
import mqtt from 'mqtt'
import ResetNetworkPage from './ResetNetworkPage'
import ClassicInclusionPage from './ClassicInclusionPage'
import ExclusionPage from './ExclusionPage'
import RemoveFailedNodePage from './RemoveFailedNodePage'
import ReplaceFailedNodePage from './ReplaceFailedNodePage'
import UpdateNetworkPage from './UpdateNetworkPage'

import bus from '../bus'

import api from '../api'
api.init(bus)

var app = {
  debug: console.debug,
  log: console.log
}

export default {
  name: 'HomePage',
  components: {
    ResetNetworkPage,
    ClassicInclusionPage,
    ExclusionPage,
    RemoveFailedNodePage,
    ReplaceFailedNodePage,
    UpdateNetworkPage
  },
  data() {
    return {
      app: {
        debug: false
      },
      gw: {
        uid: 'SWK65YGRHK373TUB111A'
      },
      mqtt: {
        host: '192.168.2.106',
        port: 9001,
        client: null,
        connected: false,
        options: {
          clientId: 's2mgr_sample_app' + new Date().getTime(),
          keepalive: 60 * 60,
          resubscribe: true, // resubscribe after reconnect
          reconnectPeriod: 1000, // reconnect period
          username: 'gw',
          password: 'test'
        },
        reqTimer: {}
      }
    }
  },
  methods: {
    // MQTT 传输层初始化
    mqttCallbackInit() {
      this.mqtt.client.on('connect', () => {
        app.log(' mqtt connect ok')
        this.mqtt.connected = true
        // 调用订阅
        this.mqttSubscribe()
      })
      this.mqtt.client.on('close', () => {
        app.log(' mqtt connect close')
        this.mqtt.connected = false
      })
      this.mqtt.client.on('message', (topic, message) => {
        let json = JSON.parse(message.toString())
        if (this.app.debug) {
          app.debug('收到', topic, message.toString())
        }
        this.injectMessageIntoApp(json)
      })
    },
    mqttConnect() {
      app.log('1. MQTT Connect')
      this.mqtt.options.username = 'app' + this.gw.uid
      this.mqtt.options.password = Buffer.from('app' + this.gw.uid)

      this.mqtt.client = mqtt.connect('ws://' + this.mqtt.host + ':' + this.mqtt.port + '/mqtt', this.mqtt.options)
      this.mqttCallbackInit()
    },
    mqttSubscribe() {
      app.log('2. MQTT Subscribe')
      this.mqtt.client.subscribe('s2://' + this.gw.uid + '/event/output')
      app.log(' 订阅 ' + 's2://' + this.gw.uid + '/event/output')
      this.mqtt.client.subscribe(this.gw.uid + '/' + this.mqtt.options.clientId)
      app.log(' 订阅 app clientid ' + this.mqtt.options.clientId)
    },
    // 将 MQTT 收到的消息 注入到 App 消息层
    injectMessageIntoApp(json) {
      // 由 MQTTClient on message 回调函数进行调用，见 mqttCallbackInit on message
      if (json.type === 'resp') {
        app.log(json.type, json)
        let callid = json.callid
        let event = 'callid_' + callid
        let timerName = 'callid_' + callid + '_timer'
        // 清理 timeout 定时器
        clearTimeout(this.mqtt.reqTimer[timerName])
        // 发送事件
        bus.$emit(event, json)

      } else if (json.type === 'notif') {
        // app.log(json.type, json.name, json.params)
        bus.$emit(json.name, json)
      } else {
        app.log('unknown payload')
      }
    },
    // 为 Request 注册 Response Callback
    registerResponseCallback(callid, reqRespCallback) {
      let event = 'callid_' + callid
      bus.$once(event, payload => {
        reqRespCallback(payload)
      })
      this.mqtt.reqTimer[event + '_timer'] = setTimeout(() => {
        bus.$off(event)
        clearTimeout(this.mqtt.reqTimer[event + '_timer'])
      }, 5000)
    },
    AppSend(json) {
      // 使用MQTTClient进行消息的发送
      let topic = json.topic
      let payload = json.payload
      console.log(payload.type, payload)

      payload.callid = Math.floor(new Date().getTime() / 1000) // javascript的timestamp精度13位
      payload.source = this.gw.uid + '/' + this.mqtt.options.clientId

      // Replace
      // console.debug(topic, JSON.stringify(payload))
      this.mqtt.client.publish(topic, JSON.stringify(payload), (error) => {
        if (error === undefined) {
          if (json.reqRespCallback !== undefined) {
            this.registerResponseCallback(payload.callid, json.reqRespCallback)
          }
        } else {
          alert('MQTT Error')
        }
      })
    },
    openDialog(dialogName) {
      this.$modal.show(dialogName)
      bus.$emit('test', 'hello')
    },
    closeDialog() {
      this.$modal.hide(dialogName)
    }
  },
  mounted() {
    // this.mqttConnect()

    // 定义一个bus event用于接收来自子组件的 信息发送请求
    bus.$on('AppSend', (json) => {
      this.AppSend(json)
    })
  }
}

</script>

<style scoped="">
.dialogLayout {
  padding: 15px;
}
</style>
