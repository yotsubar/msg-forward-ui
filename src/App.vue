<script setup lang="ts">
import { onMounted, reactive, ref } from 'vue'
import type { DefineComponent } from 'vue';
import { DArrowLeft, DArrowRight, Promotion, CircleCheck, CopyDocument } from '@element-plus/icons-vue'
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
const messgaes = ref<ForwardMsg[]>([]);

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
  sync(json: string): void
  messgaes(): ForwardMsg[]
}
const receiver: MsgReceiver = {
  receive(msg: string) {
    messgaes.value.unshift(buildMsg(msg))
  },
  sync(json: string) {
    messgaes.value = JSON.parse(json);
  },
  messgaes() {
    return messgaes.value;
  }
}
const buildMsg = (msg: string, ts?: string): ForwardMsg => {
  if (ts) {
    return new ForwardMsg(msg, ts, '#0bbd87');
  }
  const date = new Date();
  const padding = (i: number) => i < 10 ? '0' + i : i;
  const datetimeStr = `${date.getFullYear()}-${date.getMonth() + 1}-${date.getDate()} ${padding(date.getHours())}:${padding(date.getMinutes())}:${padding(date.getSeconds())}`;
  return new ForwardMsg(msg, datetimeStr, '#0bbd87');
}
const msgType = {
  ERROR: 0,
  OK: 1,
  PING: 2,
  PONG: 3,
  MSG: 4,
  ASK_SYNC: 5,
  SYNC_ANSWER: 6,
  NO_SYNC_ANSWER: 7
}
class E2eClient {
  config: E2eConfig
  wsClient?: WebSocket
  r: MsgReceiver

  cipher: Cipher
  onsync: boolean

  constructor(conf: E2eConfig, r: MsgReceiver) {
    this.config = conf;
    this.r = r;
    this.cipher = new Cipher(conf.secret);
    this.onsync = false;
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
    this.wsClient.onopen = () => {
      this.askForSync();
      this.config.connected = true;
    }
    this.wsClient.onclose = () => this.config.connected = false;

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
  askForSync() {
    this.sendWebsocketMsg(msgType.ASK_SYNC);
    this.onsync = true;
  }
  sendChat(msg: string): boolean {
    try {
      this.sendWebsocketMsg(msgType.MSG, msg);
      return true;
    } catch (e) {
      return false;
    }
  }

  dispatch(msg: any) {
    if (msg instanceof ArrayBuffer) {
      // binary frame
      const arr = new Uint8Array(msg);
      if (arr[0] == msgType.PING) {
        this.sendWebsocketMsg(msgType.PONG);
      } else if (arr[0] == msgType.MSG) {
        this.handleChatMessageReceived(arr)
      } else if (arr[0] == msgType.ASK_SYNC) {
        this.sendSyncAnswer();
      } else if (arr[0] == msgType.SYNC_ANSWER) {
        this.handleSyncAnswer(arr)
      } else if (arr[0] == msgType.NO_SYNC_ANSWER) {
        // nothing to sync
        this.onsync = false;
      }
    } else {
      // text frame
      console.log("Receive string:" + msg)
    }
  }
  handleChatMessageReceived(data: Uint8Array) {
    this.r.receive(this.decodeMsg(data));
  }
  sendSyncAnswer() {
    this.sendWebsocketMsg(msgType.SYNC_ANSWER, JSON.stringify(this.r.messgaes()));
  }
  handleSyncAnswer(data: Uint8Array) {
    if (!this.onsync) {
      return;
    }
    this.r.sync(this.decodeMsg(data));
    this.onsync = false;
  }
  sendWebsocketMsg(type: number, msg?: string) {
    if (msg) {
      this.wsClient?.send(this.encodeMsg(type, msg))
    } else {
      const d = new Uint8Array(1);
      d[0] = type
      this.wsClient?.send(d)
    }
  }
  encodeMsg(type: number, msg: string): Uint8Array {
    const content = this.cipher.encrypt(msg);
    const m = new Uint8Array(1 + content.length);
    m.fill(type, 0, 1);
    m.set(content, 1);
    return m;
  }
  decodeMsg(data: Uint8Array): string {
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

  constructor(secret: string) {
    this.secret = secret;
  }
  encrypt(t: string): Uint8Array {
    const secret = CryptoJS.enc.Utf8.parse(this.secret);
    const text = CryptoJS.enc.Utf8.parse(t);
    const words = CryptoJS.AES.encrypt(text, secret, {
      iv: secret,
      mode: CryptoJS.mode.CFB,
      padding: CryptoJS.pad.AnsiX923
    }).ciphertext;
    return this.words2uint8(words);
  }
  decrypt(arr: Uint8Array): string {
    const secret = CryptoJS.enc.Utf8.parse(this.secret);
    const s = CryptoJS.lib.CipherParams.create({
      ciphertext: this.uint8towords(arr)
    })
    const words = CryptoJS.AES.decrypt(s, secret, {
      iv: secret,
      mode: CryptoJS.mode.CFB,
      padding: CryptoJS.pad.AnsiX923
    });
    return CryptoJS.enc.Utf8.stringify(words);
  }
  words2uint8(wordArray: CryptoJS.lib.WordArray): Uint8Array {
    //copied from web
    let len = wordArray.words.length,
      u8_array = new Uint8Array(len << 2),
      offset = 0, word, i
      ;
    for (i = 0; i < len; i++) {
      word = wordArray.words[i];
      u8_array[offset++] = word >> 24;
      u8_array[offset++] = (word >> 16) & 0xff;
      u8_array[offset++] = (word >> 8) & 0xff;
      u8_array[offset++] = word & 0xff;
    }
    return u8_array;
  }
  uint8towords(arr: Uint8Array): CryptoJS.lib.WordArray {
    const len = arr.length;
    const numbers: number[] = [];
    // if thing goes well, arr.at(i) won't be undefined, since arr is from a WordArray, but ts complains it.
    const parse = (n: number | undefined) => n === undefined ? 0 : n;
    for (let i = 0; i < len; i += 4) {
      numbers[i >> 2] = (parse(arr.at(i)) << 24) | (parse(arr.at(i + 1)) << 16) | (parse(arr.at(i + 2)) << 8) | parse(arr.at(i + 3))
    }
    return CryptoJS.lib.WordArray.create(numbers)
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
const sendChat = () => {
  if (e2eClient.sendChat(textarea.value)) {
    textarea.value = '';
  }
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
              :timestamp="msg.timestamp">
              <el-card>{{ msg.content }}</el-card>
              <el-icon @click="copy2Clipboard(msg.content)"  class="copy-icon">
                <CopyDocument />
              </el-icon>
            </el-timeline-item>
          </el-timeline>
        </el-main>
        <el-footer>
          <el-input class="msg-input" v-model="textarea" :rows="3" type="textarea" placeholder="Please input" />
          <el-button type="primary" style="width: 100%;" @click="sendChat" :disabled="!e2eConfig.connected || textarea == ''">
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

.infinite-list .infinite-list-item .copy-icon {
  cursor: pointer;
}

.msg-input {
  margin: 0;
}
</style>
