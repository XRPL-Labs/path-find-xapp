<template>
    <meta name="viewport" content="width=device-width, initial-scale=1.0, maximum-scale=1.0, user-scalable=0">
    <div id="view" :style="$xapp.themeStyles">
        <SpinnerModal v-if="busy"/>
        <Trade v-if="ready" :data="data"/>
        <Modal />
        <Alert />
        <div v-if="error" class="column h-100">
            <div id="failed-start" class="column">
                <fa :icon="['fas', 'exclamation-circle']" />
                <p>{{ error }}</p>
                <a @click="if($t('xapp.error.get_ott_data') === error) getTokenData(); else if($t('xapp.error.subscribe_to_account') === error) subscribe();" class="btn btn-primary">{{ $t("xapp.button.try_again") }}</a>
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
        this.$emitter.on('busy', boolean => {
            this.busy = boolean
        })
    },
    async mounted() {
        try {
            await this.getTokenData()
            await this.subscribe()
        } catch(e) {
            this.busy = false
        }
    },
    methods: {
        async setAccountData () {
            const account_info = await this.$rippled.send({
                command: 'account_info',
                account: this.$xapp.getAccount()
            })
            if(account_info.error === 'actNotFound') return this.$xapp.setAccountData(null)

            const account_lines = await this.$rippled.send({
                command: 'account_lines',
                account: this.$xapp.getAccount()
            })
            // if (account_lines.lines < 1) alert('no trustlines')
            const account_objects = await this.$rippled.send({
                command: 'account_objects',
                account: this.$xapp.getAccount()
            })

            const account_data = {
                account: this.$xapp.getAccount(),
                account_data: account_info.account_data,
                objects: account_objects.account_objects,
                lines: account_lines.lines
            }
            this.$xapp.setAccountData(account_data)
        },
        async getTokenData () {
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
                    this.busy = false
                    this.error = this.$t('xapp.error.get_ott_data')
                    throw e
                }
            }
        },
        async subscribe() {
            this.busy = true
            try {
                const url = this.getWebSocketUrl(this.data.nodetype)
                const ws = await this.$rippled.connect(url, { NoUserAgent: true, MaxConnectTryCount: 5 })
                await this.setAccountData()
                await ws.send({
                    command: 'subscribe',
                    accounts: [this.data.account]
                })

                ws.on('transaction', tx => {
                    this.setAccountData()
                    this.$xapp.onTransaction(tx)
                })
                this.busy = false
                this.ready = true
                this.error = false
            } catch(e) {
                this.busy = false
                console.log(e)
                this.error = this.$t('xapp.error.subscribe_to_account')
                throw e
            }
        },
        getWebSocketUrl(nodetype) {
            switch (nodetype) {
                case "MAINNET":
                    return 'wss://xrplcluster.com'
                case "TESTNET":
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
    padding: 10px 5px;
    border-radius: 10px;
    width: calc(100% - 30px);
    margin: 0 10px;
    cursor: pointer;
    font-weight: 700;
    text-align: center;
    color: var(--var-white);
}
.btn.btn-primary {
    background-color: var(--var-primary);
}
.btn.disabled {
    opacity: 0.5;
}
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
#app {
    font-family: proxima-nova, sans-serif;
    -webkit-font-smoothing: antialiased;
    -moz-osx-font-smoothing: grayscale;
    height: 100%;
    overflow-y: auto;
}

html, body {
    height: 100%;
    width: 100%;
    margin: 0;
    overscroll-behavior-y: none;
    position: fixed;
    overflow: hidden;
}

.LIGHT {
    background-color: rgb(255, 255, 255);
    color: white
}

.DARK {
    background-color: rgb(0, 0, 0);
    color: white
}

.MOONLIGHT {
    background-color: #181A21;
    color: white
}

.ROYAL {
    background-color: #030B36;
    color: white
}

#view {
    height: 100%;
    padding: 0 8px;
    overflow: hidden;
    background-color: var(--var-bg-color);
    color: var(--var-txt-color);
}


.swal2-container.swal2-backdrop-show,
.swal2-container.swal2-noanimation {
    background: rgba(255,255,255,.4) !important;
}
.swal2-popup {
    box-shadow: 2px 2px 11px rgba(0,0,0,0.3) !important;
    border-radius: 10px !important;
}
.row {
    display: flex;
    flex-direction: row;
    width: 100%;
    margin: 5px 0;
    align-items: center;
}
.column {
    display: flex;
    flex-direction: column;
    width: 100%;
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