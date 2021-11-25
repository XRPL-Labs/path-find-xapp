<template>
  <meta name="viewport" content="width=device-width, initial-scale=1.0, maximum-scale=1.0, user-scalable=0" />
  <div id="view" :style="$xapp.themeStyles">
    <SpinnerModal v-if="busy" />
    <Trade v-if="ready" :data="data" />
    <Modal />
    <Alert />
    <div v-if="error" class="column h-100">
      <div id="failed-start" class="column">
        <fa :icon="['fas', 'exclamation-circle']" />
        <p>{{ error }}</p>
        <a v-if="$t('xapp.error.get_ott_data') === error" @click="getTokenData()" class="btn btn-primary">{{ $t('xapp.button.try_again') }}</a>
        <a v-else-if="$t('xapp.error.version') === error" @click="close()" class="btn btn-primary">{{ $t('xapp.button.close') }}</a>
        <a v-else-if="$t('xapp.error.subscribe_to_account') === error" @click="subscribe()" class="btn btn-primary">{{ $t('xapp.button.try_again') }}</a>
      </div>
    </div>
  </div>
</template>

<script>
import Trade from '@/components/Trade.vue'
import Spinner from '@/components/Spinner.vue'
import SpinnerModal from '@/components/SpinnerModal.vue'
import Alert from '@/components/Alert.vue'
import Modal from '@/components/Modal.vue'

export default {
  name: 'App',
  components: {
    Trade,
    Spinner,
    SpinnerModal,
    Alert,
    Modal,
    Alert
  },
  data() {
    return {
      data: {},
      busy: true,
      ready: false,
      error: ''
    }
  },
  created() {
    this.$emitter.on('busy', (boolean) => {
      this.busy = boolean
    })
  },
  async mounted() {
    try {
      await this.getTokenData()
      await this.subscribe()
    } catch (e) {
      this.busy = false
      this.error = e
    }
  },
  methods: {
    close() {
      try {
        this.$xapp.closeXapp()
      } catch (e) {}
    },
    versionCheck(v1, v2) {
      var v1parts = v1.split('.')
      var v2parts = v2.split('.')

      // First, validate both numbers are true version numbers
      function validateParts(parts) {
        for (var i = 0; i < parts.length; ++i) {
          if (!/^\d+$/.test(parts[i])) {
            return false
          }
        }
        return true
      }
      if (!validateParts(v1parts) || !validateParts(v2parts)) {
        return NaN
      }

      for (var i = 0; i < v1parts.length; ++i) {
        if (v2parts.length === i) {
          return 1
        }

        if (v1parts[i] === v2parts[i]) {
          continue
        }
        if (v1parts[i] > v2parts[i]) {
          return 1
        }
        return -1
      }

      if (v1parts.length != v2parts.length) {
        return -1
      }

      return 0
    },
    async setAccountData() {
      const account_info = await this.$rippled.send({
        command: 'account_info',
        account: this.$xapp.getAccount()
      })
      if (account_info.error === 'actNotFound') return this.$xapp.setAccountData(null)

      const account_data = {
        account: this.$xapp.getAccount(),
        account_data: account_info.account_data
      }
      this.$xapp.setAccountData(account_data)
    },
    async getTokenData() {
      this.busy = true
      // todo DELETE MEEE ASAP ONLY FOR TESTING ON LOCALHOST
      if (typeof window.ReactNativeWebView === 'undefined') {
          this.data = {
              account: 'rJR4MQt2egH9AmibZ8Hu5yTKVuLPv1xumm',
              nodetype: 'MAINNET',
              // account: 'rMtfWxk9ZLr5mHrRzJMnaE5x1fqN3oPdJ7',
              // nodetype: 'TESTNET'
          }
          this.$xapp.setAccount(this.data.account)
      } else {
          try {
              this.data = await this.$xapp.getTokenData()
              this.$xapp.setAccount(this.data.account)
          } catch(e) {
              console.log(e)
              throw this.$t('xapp.error.get_ott_data')
          }
          if (this.versionCheck(this.data.version, '2.2.5') < 0) {
              throw this.$t('xapp.error.version')
          }
      }
    },
    async subscribe() {
      this.busy = true
      try {
        const url = this.getWebSocketUrl(this.data.nodetype)
        await this.$rippled.connect(url, {NoUserAgent: true, MaxConnectTryCount: 5})
        this.setAccountData()
        this.$rippled.send({
          command: 'subscribe',
          accounts: [this.data.account]
        })

        this.$rippled.on('transaction', (tx) => {
          this.setAccountData()
          this.$xapp.onTransaction(tx)
        })

        this.busy = false
        this.ready = true
        this.error = false
      } catch (e) {
        console.log(e)
        throw this.$t('xapp.error.subscribe_to_account')
      }
    },
    getWebSocketUrl(nodetype) {
      switch (nodetype) {
        case 'MAINNET':
          return 'wss://xrplcluster.com'
        case 'TESTNET':
          return 'wss://testnet.xrpl-labs.com'
      }
      return 'wss://xrplcluster.com'
    }
  }
}
</script>

<style>
@import url('https://use.typekit.net/iqo4nny.css');
@import url('https://fonts.googleapis.com/css?family=Ubuntu+Mono');

.number {
  font-family: 'Ubuntu Mono' !important;
}

/* :root {
    --var-bg-color: #030B36;
    --var-txt-color: #ffffff;
} */

#failed-start {
  align-items: center;
  margin: auto;
}
.btn {
  padding: 0.75rem 1rem;
  border-radius: 0.75rem;
  /* margin: 0 10px; */
  cursor: pointer;
  font-weight: 700;
  text-align: center;
  color: var(--var-white);
  outline: none;
  border-color: transparent;
}
.btn.btn-sm {
  /* padding: 3px 8px; */
  padding: 0.3rem 0.75rem;
  font-size: 0.8rem;
}
.btn.btn-rounded {
  border-radius: 20rem;
}
.btn.btn-block {
  display: block;
  width: auto;
}
.btn.btn-secondary {
  background-color: var(--var-secondary);
}
.btn.btn-primary {
  background-color: var(--var-primary);
}
.btn.btn-success {
  background-color: var(--var-green);
}
.btn.disabled {
  opacity: 0.5;
}
.text-center {
  text-align: center;
}
/*

.btn-0-margin {
  width: calc(100% - 10px);
  margin: 0 !important;
}
.h-100 {
  height: 100%;
}
.margin-0 {
  margin: 0 !important;
}
*/
#app {
  font-family: proxima-nova, sans-serif;
  -webkit-font-smoothing: antialiased;
  -moz-osx-font-smoothing: grayscale;
  height: 100%;
  overflow-y: auto;
}

html,
body {
  height: 100%;
  width: 100%;
  margin: 0;
  overscroll-behavior-y: none;
  position: fixed;
  overflow: hidden;
}

.LIGHT {
  background-color: rgb(255, 255, 255);
  color: white;
}

.DARK {
  background-color: rgb(0, 0, 0);
  color: white;
}

.MOONLIGHT {
  background-color: #181a21;
  color: white;
}

.ROYAL {
  background-color: #030b36;
  color: white;
}

#view {
  height: 100%;
  padding: 0;
  /* padding: 0 15px; */
  overflow: hidden;
  background-color: var(--var-bg-color);
  color: var(--var-txt-color);
}

/* .swal2-container.swal2-backdrop-show,
.swal2-container.swal2-noanimation {
  background: rgba(255, 255, 255, 0.4) !important;
}
.swal2-popup {
  box-shadow: 2px 2px 11px rgba(0, 0, 0, 0.3) !important;
  border-radius: 10px !important;
} */

.row {
  /* padding: 0 1rem; */
  /* display: flex; */
  /* flex-direction: row; */
  width: 95%;
  margin: 0 10rem;
  align-items: center;
}
.column {
  display: flex;
  flex-direction: column;
  width: 100%;
}
.alert {
  position: relative;
  padding: 0.75rem 1.25rem;
  margin-bottom: 1rem;
  border: 1px solid transparent;
  border-radius: 0.5rem;
  text-align: center;
}
.alert-primary {
  color: #004085;
  background-color: #cce5ff;
  border-color: #b8daff;
}
fieldset {
  display: flex;
  align-items: center;
  margin-inline-start: 0;
  margin-inline-end: 0;
  padding-block-start: 0;
  padding-inline-start: 0;
  padding-inline-end: 0;
  padding-block-end: 0;
  min-inline-size: min-content;
  border-width: 0;
  border-style: none;
  border-color: rgba(255, 255, 255, 0);
  border-image: none;
}
</style>
