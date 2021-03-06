<template>
  <v-dialog v-model="visible" scrollable @keydown.esc="visible = false" max-width="480">
    <div class="main-container">
      <dialog-titlebar :title="title" v-on:close="visible = false" />

      <div class="dialog-contents">
        <div
          class="top-dialog-text"
        >Don't loose these keys! Print or save them to a secure USB or hard disk</div>

        <v-select
          :items="accountsUI"
          item-text="name"
          v-model="selectedSource"
          label="Account"
          return-object
          max-height="600"
        ></v-select>

        <div class="operations-item" v-for="key in Array.from(summaryMap.keys())" :key="key">
          <div class="item-name">{{key}}:</div>
          <div class="item-value">{{summaryMap.get(key)}}</div>
        </div>

        <div class="buttons-area">
          <v-tooltip open-delay="200" bottom>
            <template v-slot:activator="{ on }">
              <v-btn color="primary" rounded v-on="on" @click="buttonClick('save-keys')">Save Keys...</v-btn>
            </template>
            <span>Save the keys to a file on a USB or disk</span>
          </v-tooltip>
          <v-tooltip open-delay="200" bottom>
            <template v-slot:activator="{ on }">
              <v-btn
                color="primary"
                rounded
                v-on="on"
                @click="buttonClick('print-keys')"
              >Print Keys...</v-btn>
            </template>
            <span>Print the keys for safety</span>
          </v-tooltip>
        </div>
      </div>
      <toast-component :absolute="true" location="save-keys-dialog" :bottom="false" :top="true" />
    </div>
  </v-dialog>
</template>

<script>
import Helper from '../../js/helper.js'
import { DialogTitleBar } from 'stellarkit-js-ui'
import ToastComponent from '../ToastComponent.vue'
import StellarUtils from '../../js/StellarUtils.js'
import StellarAccounts from '../../js/StellarAccounts.js'
import StellarCommonMixin from '../StellarCommonMixin.js'
const FileSaver = require('file-saver')
import $ from 'jquery'

export default {
  props: ['ping', 'publicKey'],
  mixins: [StellarCommonMixin],
  components: {
    'dialog-titlebar': DialogTitleBar,
    'toast-component': ToastComponent
  },
  data() {
    return {
      visible: false,
      title: 'Save the Keys',
      summaryMap: [],
      selectedSource: null
    }
  },
  watch: {
    selectedSource: function() {
      this.updateSummary()
    },
    ping: function() {
      this.visible = true
      this.setup()
    }
  },
  mounted() {
    this.setup()
  },
  methods: {
    setup() {
      // find account with the publicKey passed in
      if (this.publicKey) {
        this.selectedSource = StellarAccounts.accountWithPublicKey(
          this.publicKey
        )
      }

      if (!this.selectedSource && this.accountsUI.length > 0) {
        this.selectedSource = this.accountsUI[0]
      }

      this.updateSummary()
    },
    updateSummary() {
      this.summaryMap = new Map()

      if (this.selectedSource) {
        this.summaryMap.set('Public', this.selectedSource.publicKey)
        this.summaryMap.set('Secret', this.selectedSource.secret)
      } else {
        this.summaryMap.set('Public', '--')
        this.summaryMap.set('Secret', '--')
      }
    },
    printFile() {
      if (this.selectedSource) {
        // insert an iframe into the DOM
        // print iframe window.  popups could be blocked, so frame is safter than opening a new window
        const iframe = $('<iframe></iframe>')[0]

        iframe.setAttribute('id', 'printkeys')
        iframe.setAttribute('name', 'printkeys')

        document.body.appendChild(iframe)

        const frameWindow = window.frames['printkeys']
        frameWindow.document.head.innerHTML =
          '<style>body{margin-top: 100px; font-size: 12px; text-align: center; font-family: Arial, Helvetica, sans-serif;}</style>'
        frameWindow.document.body.innerHTML =
          '<div>' + this.keyString(true) + '</div>'

        frameWindow.print()

        document.body.removeChild(iframe)
      } else {
        Helper.DebugLog('Nothing to print?')
      }
    },
    keyString(html = false) {
      let result = ''
      if (this.selectedSource) {
        const isMainnet = !StellarUtils.isTestnet()

        const network = isMainnet ? '(public net)' : '(test net)'
        let newLines = '\n\n'
        if (html) {
          newLines = '<br><br>'
        }
        result += newLines
        result += 'Stellar account keys: ' + network
        result += newLines
        result += 'Account name: ' + this.selectedSource.name
        result += newLines
        result += 'Public: ' + this.selectedSource.publicKey
        result += newLines
        result += 'Secret: ' + this.selectedSource.secret
      }

      return result
    },
    saveFile() {
      if (this.selectedSource) {
        const jsonString = this.keyString()
        const blob = new Blob([jsonString], {
          type: 'text/plain;charset=utf-8'
        })
        FileSaver.saveAs(blob, 'stellar-account-keys.txt')
      } else {
        Helper.debugLog('Nothing to save?')
      }
    },
    buttonClick(id) {
      switch (id) {
        case 'save-keys':
          this.saveFile()
          break
        case 'print-keys':
          this.printFile()
          break
        default:
          break
      }
    },
    displayToast(message, error = false) {
      Helper.toast(message, error, 'save-keys-dialog')
    }
  }
}
</script>

<style lang='scss' scoped>
@import '../../scss/styles.scss';

.main-container {
  @include standard-dialog-contents();

  .dialog-contents {
    @include inner-dialog-contents();

    .top-dialog-text {
      margin-bottom: 10px;
    }
    .operations-item {
      display: flex;
      font-size: 0.65em;

      .item-name {
        text-align: right;
        padding-right: 5px;
        font-weight: bold;
        flex: 1 1 10%;
      }

      .item-value {
        text-align: left;
        flex: 2 2 90%;
        padding-left: 5px;
        white-space: nowrap;
        overflow: hidden;
        text-overflow: ellipsis;
        font-family: monospace;
        width: 0;
      }
    }

    .buttons-area {
      width: 100%;
      display: flex;
      justify-content: center;
      align-items: center;
      margin-top: 20px;
    }
  }
}
</style>
