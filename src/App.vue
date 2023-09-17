<script setup lang="ts">
import { onMounted, reactive, ref } from 'vue'
import type { DefineComponent } from 'vue';
import { DArrowLeft, DArrowRight, Promotion, CircleCheck } from '@element-plus/icons-vue'
import { ElMessage } from "element-plus"
import axios from "axios"
import useClipboard from 'vue-clipboard3'
import CryptoJS from 'crypto-js'

const e2eConfig: E2eConfig = reactive({
  username: '',
  password: '',
  server: '',
  secret: '',
  connected: false
})
const textarea = ref('')

class ForwardMsg {
  content: string
  timestamp: string
  color?: string
  size?: string
  type?: string
  hollow: boolean
  icon?: DefineComponent
  constructor(content: string, timestamp: string, color?: string, size?: string, type?: string, icon?: DefineComponent) {
    this.content = content;
    this.timestamp = timestamp;
    this.size = size;
    this.type = type;
    this.color = color;
    this.hollow = true;
    this.icon = icon;
  }
}
const messgaes: ForwardMsg[] = reactive([]);

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

  const se = localStorage.getItem('server')
  if (se) {
    e2eConfig.server = se
  }
  const s = localStorage.getItem('secret')
  if (s) {
    e2eConfig.secret = s
  }
}
const copy2Clipboard = (text: string) => {
  const to = useClipboard();
  to.toClipboard(text);
  ElMessage('Copied')
}
interface MsgReceiver {
  receive(msg: string): void
}
const receiver: MsgReceiver = {
  receive(msg: string) {
    const date = new Date();
    const padding = (i:number) => i < 10 ? '0' + i : i;
    const datetimeStr = `${date.getFullYear()}-${date.getMonth()+1}-${date.getDate()} ${padding(date.getHours())}:${padding(date.getMinutes())}:${padding(date.getSeconds())}`;
    messgaes.unshift(new ForwardMsg(msg, datetimeStr, '#0bbd87'))
  }
}
class E2eClient {
  config: E2eConfig
  wsClient?: WebSocket
  r: MsgReceiver

  cipher: Cipher

  constructor(conf: E2eConfig, r: MsgReceiver) {
    this.config = conf;
    this.r = r;
    this.cipher = new Cipher(conf.secret);
  }
  connect() {
    let url = this.config.server + "/login";
    if (url.indexOf("http") === -1) {
      url = "http://" + url;
    }
    axios.postForm(url, {
      username: this.config.username,
      password: this.config.password
    }).then(r => {
      if (r.status == 200) {
        this.start(r.data)
      }
    });
  }
  start(path: string) {
    let url;
    if (this.config.server.indexOf("http") === -1) {
      url = "ws://" + this.config.server + "/ws/" + path
    } else {
      url = this.config.server.replace('http', "ws") + "/ws/" + path;
    }
    this.wsClient = new WebSocket(url)
    this.wsClient.binaryType = 'arraybuffer'
    this.wsClient.onmessage = (e) => this.dispatch(e.data)
    this.cipher = new Cipher(this.config.secret);

    this.config.connected = true;

    localStorage.setItem("server", this.config.server)
    localStorage.setItem("username", this.config.username)
    localStorage.setItem("password", this.config.password)
    localStorage.setItem("secret", this.config.secret)
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
    console.log("Send: " + msg)
    this.sendWebsocketMsg(4, msg)
  }

  dispatch(msg: any) {
    if (msg instanceof ArrayBuffer) {
      // binary frame
      const arr = new Uint8Array(msg);
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
    const content = this.cipher.encrypt(msg);
    const m = new Uint8Array(1 + content.length);
    m.fill(type, 0, 1);
    m.set(content, 1);
    return m;
  }
  decodeMsg(data: Uint8Array) {
    return this.cipher.decrypt(data.slice(1, data.length));
  }
}
interface E2eConfig {
  username: string
  password: string
  server: string
  secret: string
  connected: boolean
}

class Cipher {
  secret: string
  textEncoder: TextEncoder
  textDecoder: TextDecoder

  constructor(secret: string) {
    this.secret = secret;
    this.textDecoder = new TextDecoder();
    this.textEncoder = new TextEncoder();
  }
  encrypt(t: string) {
    const secret = CryptoJS.enc.Utf8.parse(this.secret);
    const text = CryptoJS.enc.Utf8.parse(t);
    const s = CryptoJS.AES.encrypt(text, secret, {
      iv: secret,
      mode: CryptoJS.mode.CFB,
      padding: CryptoJS.pad.AnsiX923
    }).toString();
    console.log("encrypt to: " + s);
    return this.textEncoder.encode(s);
  }
  decrypt(arr: Uint8Array) {
    const s = this.textDecoder.decode(arr);
    const secret = CryptoJS.enc.Utf8.parse(this.secret);
    console.log("decrypt from: " + s);
    return CryptoJS.AES.decrypt(s, secret, {
      iv: secret,
      mode: CryptoJS.mode.CFB,
      padding: CryptoJS.pad.AnsiX923
    }).toString(CryptoJS.enc.Utf8);
  }
}
let e2eClient: E2eClient = new E2eClient(e2eConfig, receiver);

const connectToServer = () => {
  if (e2eConfig.password && e2eConfig.username && e2eConfig.secret) {
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
          <el-input v-model="e2eConfig.secret" placeholder="Please input secret" />
          <el-button type="primary" @click="connectToServer">Connect</el-button>
          <el-icon v-show="e2eConfig.connected">
            <CircleCheck color="#0bbd87" />
          </el-icon>
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
          <el-button type="primary" style="width: 100%;" @click="e2eClient.sendChat(textarea); textarea = ''">
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
  max-height: 640px;
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
