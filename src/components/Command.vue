<template>
  <div :style="outerStyle" class="ddd-sdk-outer">
    <div ref="logDiv" :style="printStyle" class="ddd-sdk-print">
      {{ commandLog }}
    </div>
    <div ref="commandDiv" :style="commandDivStyle" class="ddd-sdk-command">
      <label>></label>
      <textarea
          v-model="currentCommand"
          :style="inputStyle"
          spellcheck="false"
          @input="commandChangeHandler"
          @keydown.delete="commandChangeHandler(true)"
          @keydown.enter.native.prevent="enterHandler"
          @keydown.tab.native.prevent="tabHandler"
          @keydown.up.native.prevent="prefHistory"
          @keydown.down.native.prevent="nextHistory"
          @keydown.right="completeCommand"
      />
      <label :style="inputStyle" class="ddd-sdk-command-placeholder" @click.stop>{{ placeholder }}</label>
    </div>
  </div>
</template>

<script>
import {COMMAND_STYLE} from "../scripts/enums/command";
class SimpleApi {
  execute(command) {
    return new Promise((resolve, reject) => {
    })
  }
  remindCommand(command) {
    console.warn("Remind command")
    return new Promise((resolve, reject) => {
    })
  }
}
const restApi = new SimpleApi()
const {ERROR, SUCCESS, WARN, MESSAGE, INFO} = COMMAND_STYLE

//监听鼠标右键
document.oncontextmenu = function (event) {
  return false
}
//禁用Ctrl+S
document.onkeydown = function (event) {
  if (event.ctrlKey && window.event.keyCode === 83) {
    return false
  }
}

export default {
  name: "Command",
  props: {
    width: {
      type: String,
      default: '100%'
    },
    height: {
      type: String,
      default: '100%'
    },
    fontSize: {
      type: String,
      default: '16px'
    }
  },
  data() {
    return {
      outerStyle: {
        width: this.width,
        height: this.height,
        fontSize: this.fontSize,
        lineHeight: 'calc(' + this.fontSize + ' + 4px)'
      },
      printStyle: {
        maxHeight: 'calc(100% - ' + this.fontSize + ' - 8px)'
      },
      inputStyle: {
        fontSize: this.fontSize,
        height: 'calc(' + this.fontSize + ' + 4px)',
        paddingLeft: this.fontSize
      },
      commandDivStyle: {
        height: 'calc(' + this.fontSize + ' + 4px)',
      },
      commandLog: '',
      currentCommand: '',
      history: [''],
      historyIndex: 0,
      remindQuery: true,
      remindList: [],
      remindIndex: -1,
      placeholder: ''
    }
  },
  methods: {
    appendLog(content, type = INFO) {
      if (content instanceof Array) {
        content = content.join('\n')
      } else if (content instanceof Object) {
        let newContent = []
        Object.keys(content).forEach(key => newContent.push(`${key}: ${content[key]}`))
        content = newContent.join('\n')
      }
      const dom = window.document.createElement('span')
      dom.innerHTML = content
      dom.style.color = type === ERROR ? 'lightcoral'
          : type === SUCCESS ? 'lightseagreen'
              : type === MESSAGE ? 'lightskyblue'
                  : type === WARN ? 'lightsalmon'
                      : type === INFO ? 'white' : 'white'
      if (this.$refs.logDiv.children.length > 0) {
        this.$refs.logDiv.innerHTML += '\n'
      }
      this.$refs.logDiv.appendChild(dom)
      this.$nextTick(() => {
        this.$refs.logDiv.scrollTop = this.$refs.logDiv.offsetHeight
      })
    },
    enterHandler() {
      if (this.commandLog) {
        this.appendLog('\n')
      }
      this.appendLog('>' + this.currentCommand)
      if (/^\s*$/.test(this.currentCommand)) { //空指令
        return
      }
      this.history[this.historyIndex] = this.currentCommand
      if (this.history.length > 50) {
        this.history.shift()
      }
      this.history.push('')
      this.$nextTick(this.nextHistory)
      this.executeCommandHandler(this.currentCommand).then((res) => {
        let result = res.data
        if (result instanceof Array) {
          result = result[0]
        }
        this.appendLog(result, SUCCESS)
      }).catch(err => {
        const msg = err.response ? err.response.data.message : err.message
        this.appendLog(msg, ERROR)
      })
    },
    tabHandler() {
      if (this.remindQuery) {
        this.queryRemindHandler().then(res => {
          this.remindList = res.data[0]
          this.remindQuery = false
          this.$nextTick(() => {
            this.nextRemind()
          })
        }).catch(err => {
          this.appendLog(err.message, ERROR)
          this.remindList = []
          this.$nextTick(() => {
            this.nextRemind()
          })
        })
      } else {
        this.nextRemind()
      }
    },
    commandChangeHandler(force = false) {
      if (force || !this.placeholder.startsWith(this.currentCommand)) {
        this.remindIndex = -1
        this.remindQuery = true
      }
      this.showRemind()
    },
    queryRemindHandler() {
      return restApi.remindCommand(this.currentCommand)
    },
    nextRemind() {
      if (this.remindList.length > 0) {
        this.remindIndex = (this.remindIndex + 1) % this.remindList.length
      } else {
        this.remindIndex = -1
      }
      this.showRemind()
    },
    prefHistory() {
      if (this.historyIndex > 0) {
        this.historyIndex--
        this.$nextTick(() => {
          this.currentCommand = this.history[this.historyIndex]
          this.commandChangeHandler()
        })
      }
    },
    nextHistory() {
      if (this.history.length - 1 > this.historyIndex) {
        this.historyIndex++
        this.$nextTick(() => {
          this.currentCommand = this.history[this.historyIndex]
          this.commandChangeHandler()
        })
      }
    },
    completeCommand() {
      if (this.placeholder.startsWith(this.currentCommand)) {
        this.currentCommand = this.placeholder
        this.commandChangeHandler(true)
      }
    },
    showRemind() {
      if (this.remindList.length > 0 && this.remindIndex >= 0) {
        this.placeholder = this.remindList[this.remindIndex]
      } else {
        this.placeholder = ''
      }
    },
    executeCommandHandler() {
      return restApi.execute(this.currentCommand)
    }
  }
}
</script>

<style scoped>
.ddd-sdk-outer {
  overflow: hidden;
  font-family: 'Consolas', sans-serif;
  display: flex;
  flex-direction: column;
  color: #fff;
  background-color: #000;
}

.ddd-sdk-outer .ddd-sdk-print,
.ddd-sdk-outer .ddd-sdk-command {
  position: relative;
  width: 100%;
  padding: 0 8px 0;
}

.ddd-sdk-outer .ddd-sdk-print {
  white-space: pre-wrap;
  overflow-y: auto;
  overflow-wrap: break-word;
}

.ddd-sdk-outer .ddd-sdk-command label {
  position: absolute;
}

.ddd-sdk-outer .ddd-sdk-command input,
.ddd-sdk-outer .ddd-sdk-command textarea {
  z-index: 2;
  font-family: 'Consolas', sans-serif;
  display: inline-block;
  line-height: 100%;
  color: #fff;
  background: none;
  border: 0;
  position: absolute;
  width: 100%;
  left: 0;
  bottom: 0;
}

.ddd-sdk-outer .ddd-sdk-command .ddd-sdk-command-placeholder {
  white-space: pre-wrap;
  color: rgba(255, 255, 255, 0.5);
  pointer-events: none;
  position: absolute;
  background: none;
  left: 0;
}
</style>