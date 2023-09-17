<script setup lang="ts">
import { onMounted, reactive, ref } from 'vue'
import { MoreFilled, DArrowLeft, DArrowRight, Promotion } from '@element-plus/icons-vue'
import { ElMessage } from "element-plus"
import axios from "axios"
import useClipboard from 'vue-clipboard3'

const e2eConfig: E2eConfig = reactive({
  username: '',
  password: '',
  server: '',
})
const secret = ref('')
const textarea = ref('')
const testMsg = [
  {
    content: 'Custom icon',
    timestamp: '2018-04-12 20:46',
    size: 'large',
    type: 'primary',
    icon: MoreFilled
  },
  {
    content: 'Custom color',
    timestamp: '2018-04-03 20:46',
    color: '#0bbd87'
  },
  {
    content: 'Custom size',
    timestamp: '2018-04-03 20:46',
    size: 'large'
  },
  {
    content: 'Custom hollow',
    timestamp: '2018-04-03 20:46',
    type: 'primary',
    hollow: true
  },
  {
    content: 'Default node',
    timestamp: '2018-04-03 20:46'
  }
]
const messgaes = ref(testMsg.concat(testMsg).concat(testMsg))

onMounted(() => {
  tryLoad()
})
const tryLoad = () => {
  const u = localStorage.getItem('username')
  if (u) {
    e2eConfig.username = u
  }
  const cp = localStorage.getItem('password')
  if (cp) {
    e2eConfig.password = cp
  }

  const s = localStorage.getItem('secret')
  if (s) {
    secret.value = s
  }
}
const copy2Clipboard = (text: string) => {
  const to = useClipboard();
  to.toClipboard(text);
  ElMessage('Copied.')
}
interface MsgReceiver {
  receive(msg: string): void
}
const receiver: MsgReceiver = {
  receive(msg: string) {
    messgaes.value.push({
      content: msg,
      timestamp: new Date().toDateString(),
      color: '#0bbd87'
    })
  }
}
class E2eClient {
  config: E2eConfig
  wsClient?: WebSocket
  r: MsgReceiver
  textEncoder: TextEncoder
  textDecoder: TextDecoder

  constructor(conf: E2eConfig, r: MsgReceiver) {
    this.config = conf;
    this.r = r;
    this.textDecoder = new TextDecoder();
    this.textEncoder = new TextEncoder();
  }
  connect() {
    axios.postForm("http://" + this.config.server + "/login", {
      username: this.config.username,
      password: this.config.password
    }).then(r => {
      if (r.status == 200) {
        this.start(r.data)
      }
    });
  }
  start(path: string) {
    this.wsClient = new WebSocket("ws://" + this.config.server + "/ws/" + path)
    this.wsClient.binaryType = 'arraybuffer'
    this.wsClient.onmessage = (e) => this.dispatch(e.data)
  }
  shutdown() {
    if (this.wsClient) {
      this.wsClient.close()
      this.wsClient.onmessage = null
    }
  }
  reconnect() {
    this.shutdown()
    this.connect()
  }

  sendChat(msg: string) {
    this.sendWebsocketMsg(4, msg)
  }

  dispatch(msg: any) {
    if (msg instanceof ArrayBuffer) {
      // binary frame
      const arr = new Uint8Array(msg);
      console.log("Receive msg,type:" + arr[0])
      if (arr[0] == 2) {
        this.sendWebsocketMsg(3);
      } else if (arr[0] == 4) {
        this.handleChatMessageReceived(arr)
      }
    } else {
      // text frame
      console.log("Receive string:" + msg)
    }
  }
  handleChatMessageReceived(data: Uint8Array) {
    const s = this.decodeMsg(data);
    this.r.receive(s);
  }
  sendWebsocketMsg(type: number, msg?: string) {
    console.log("Sending msg, type:" + type)
    if (msg) {
      this.wsClient?.send(this.encodeMsg(type, msg))
    } else {
      const d = new Uint8Array(1);
      d[0] = type
      this.wsClient?.send(d)
    }
  }
  encodeMsg(type: number, msg: string) {
    const content = this.textEncoder.encode(msg)
    const m = new Uint8Array(1 + content.length);
    m.fill(type, 0, 1);
    m.set(content, 1);
    return m;
  }
  decodeMsg(data: Uint8Array) {
    return this.textDecoder.decode(data.slice(1, data.length));
  }
}
interface E2eConfig {
  username: string
  password: string
  server: string
}

let e2eClient: E2eClient = new E2eClient(e2eConfig, receiver)

const connectToServer = () => {
  if (e2eConfig.password && e2eConfig.username) {
    e2eClient.reconnect()
  }
}

const collapse = ref(sessionStorage.getItem('collapse') === 'true')
const collapseLeft = () => {
  collapse.value = !collapse.value
  sessionStorage.setItem('collapse', '' + collapse.value)
}

</script>

<template>
  <div class="common-layout">
    <el-container>
      <el-aside :width="collapse ? '12px' : '200px'">
        <span class="collapse-span">
          <el-button @click="collapseLeft" style="padding: 8px 1px;">
            <el-icon>
              <DArrowRight v-show="collapse" />
              <DArrowLeft v-show="!collapse" />
            </el-icon>
          </el-button>
        </span>
        <div v-show="!collapse">
          <el-input v-model="e2eConfig.server" placeholder="Please input server" />
          <el-input v-model="e2eConfig.username" placeholder="Please input username" />
          <el-input v-model="e2eConfig.password" type="password" placeholder="Please input password" show-password />
          <el-button type="primary" @click="connectToServer">Connect</el-button>
          <el-input v-model="secret" placeholder="Please input secret" />
        </div>
      </el-aside>
      <el-container>
        <el-main>
          <el-timeline class="infinite-list">
            <el-timeline-item v-for="(msg, index) in messgaes" :key="index" :icon="msg.icon" :type="msg.type"
              class="infinite-list-item " :color="msg.color" :size="msg.size" :hollow="msg.hollow"
              :timestamp="msg.timestamp" @click="copy2Clipboard(msg.content)">
              <el-card>
                {{ msg.content }}
              </el-card>
            </el-timeline-item>
          </el-timeline>
        </el-main>
        <el-footer>

          <el-input class="msg-input" v-model="textarea" :rows="3" type="textarea" placeholder="Please input" />
          <el-button type="primary" @click="e2eClient.sendChat(textarea); textarea = ''">
            <el-icon>
              <Promotion />
            </el-icon>
          </el-button>
        </el-footer>
      </el-container>
    </el-container>
  </div>
</template>
<style scoped>
.collapse-span {
  display: flex;
  flex-direction: column-reverse;
}

.el-timeline {
  padding-left: 2px;
}

.infinite-list {
  max-height: 450px;
  padding: 5px 0;
  margin: 0;
  list-style: none;
}

.infinite-list .infinite-list-item {
  cursor: pointer;
}

.msg-input {
  margin: 0;
}
</style>
