<template>
    <Select @currencyChange="setCurrency" :currency="currency" :issuer="issuer" :lines="destinationtrustlines"/>
    <div class="container">

        <div class="row header" style="margin-bottom: 0;">
            <h6 class="number">
                <span class="dot" :style="{ 'background-color': online ? 'green' : 'red' }"></span>
                {{ account }}
            </h6>
            <h3 @click="signin()" style="margin-left: auto;" >
                <fa :icon="['fas', 'sign-in-alt']"/>
            </h3>
        </div>
        
        <hr>
        <div v-if="!destination" class="column">
            <h3>{{ $t('xapp.headers.select_destination') }}</h3>
            <a @click="selectDestination()" class="btn btn-primary btn-0-margin">
                {{ $t('xapp.button.select_destination') }}
                <fa :icon="['fas', 'arrow-right']" />
            </a>
        </div>
        <div v-else class="column">
            <h3>{{ $t('xapp.headers.select_destination') }}</h3>
            <a @click="selectDestination()" id="destination-selector">
                <h4>Account name</h4>
                <h6>{{ destination }}</h6>
            </a>
            <h3>{{ $t('xapp.headers.receive') }}</h3>
            <div class="row push">
                <div class="input-label" :class="{ 'input-error': quantityInputValidator || quantityInputError }">
                    <input type="text" inputmode="decimal" :placeholder="$t('xapp.trade.quantity')" v-model="quantityInput" @keydown="prevent">
                    <label>{{ $xapp.currencyCodeFormat(currency) }}</label>
                </div>
                <a @click="$emitter.emit('select', true)" class="btn btn-primary label btn-small">{{ $t('xapp.button.select_currency') }}</a>
            </div>
            <!-- <div class="row">
                <a v-if="$xapp.getAccountData() !== null" @click="order()" class="btn btn-primary btn-0-margin" :class="{ disabled: !funds && funds <= 0 }">{{ $t('xapp.trade.confirm_order') }}</a>
                <a v-else @click="signin()" class="btn btn-warning btn-0-margin">{{ $t('xapp.button.account_not_found_login_button') }}</a>
            </div> -->
        </div>

        <hr class="divide">

        <Spinner v-if="fetching"/>
        <div class="column" v-else-if="pathFindData">
            <h3>{{ $t('xapp.headers.offers') }}</h3>
            <div class="row payment-card push" v-for="item in pathFindData.alternatives">
                <div class="number" v-if="typeof item.source_amount === 'string' || typeof item.source_amount === 'number'">
                    <label>XRP:</label>
                    {{ $xapp.currencyFormat(item.source_amount, 'XRP') }}
                </div>
                <div v-else class="number">
                    <label>{{ $xapp.currencyCodeFormat(item.source_amount.currency, 4) }}:</label>
                    {{ $xapp.currencyFormat(item.source_amount.value, item.source_amount.currency) }}
                </div>
                <a class="btn btn-success btn-0-margin label btn-small" @click="pay(item)">Pay Now</a>
            </div>
        </div>
    </div>
</template>

<script>
import Select from '@/components/Select.vue'
import Spinner from './Spinner.vue'

export default {
    name: 'Trade',
    props: ['data'],
    components: { Select, Spinner },
    data() {
        return {
            quantity: null,
            online: false,
            InputQuantity: null,
            sliderValue: null,
            quantityInputError: false,
            destination: null,
            issuer: null,
            currency: 'XRP',
            destinationtrustlines: [],
            fetching: false,
            pathFindData: null
        }
    },
    watch: {
        quantity: async function(newValue, oldValue) {
            // this.fetching = true
            // await this.closePathFind()
            // if(newValue > 0 && typeof newValue === 'number') {
            //     await this.createPathFind()
            // }
            // this.fetching = false
        }
    },
    computed: {
        quantityInputValidator() {
            if(this.quantity < 0) return true
            return this.quantity > this.funds
        },
        account() {
            return this.$xapp.getAccount()
        },
        funds() {
            return Infinity
        },
        quantityInput: {
            get() {
                return this.InputQuantity
            },
            set(value) {
                if(this.quantityInputError) this.quantityInputError = false

                value = this.parseValue(value)
                if(value === null) {
                    this.sliderValue = 0
                    this.InputQuantity = null
                    return this.quantity = null
                }
                this.InputQuantity = value.toString()
                value = parseFloat(value)

                if(this.currency === 'XRP') {
                    value = Math.trunc(value * 1_000_000)
                }
                this.quantity = value
                this.onQuantityChange()
            }
        },
    },
    methods: {
        async onQuantityChange() {
            this.fetching = true
            await this.closePathFind()
            if(this.quantity > 0 && typeof this.quantity === 'number') {
                await this.createPathFind()
            }
            this.fetching = false
        },
        parseValue(value) {
            if(value === '') return null
            if(value === ',' || value === '.') return '0.'
            value = value.replace(/,/g, '.')
            return value
        },
        prevent(e) {
            const input = e.target.value
            if(e.key === 'ArrowLeft') return
            if(e.key === 'ArrowRight') return
            if( (e.key === ',' || e.key === '.') && (input.includes('.') || input.includes(',')) ) return e.preventDefault()
            if(!/^[0-9]$/i.test(e.key)) {
                switch(e.key) {
                    case 'Home':
                    case 'End':
                    case 'Shift':
                    case 'Control':
                    case 'Escape':
                    case 'Alt':
                    case 'Meta':
                    case 'Tab':
                    case 'Backspace':
                    case 'Delete':
                    case 'Enter':
                    case '.':
                    case ',':
                    case 'ArrowLeft':
                    case 'ArrowRight':
                        break
                    default:
                        console.log(`Prevent key: ${e.key}`)
                        return e.preventDefault()
                }
            }
        },
        async pay(path) {
            let inputError = false
            if(typeof this.quantity !== 'number' || this.quantity <= 0) {
                this.quantityInputError = true
                inputError = true
            }
            if(inputError) {
                return
            } else {
                this.quantityInputError = false   
            }
            this.$emitter.emit('busy', true)
            console.log(path)

            const payment = {
                TransactionType: "Payment",
                Account: this.account,
                Destination: this.destination,
                SendMax: path.source_amount,
                Amount: this.currency === 'XRP' ? this.quantity.toString() : { currency: this.currency, value: this.quantity, issuer: this.issuer },
                Paths: path.paths_computed
            }
            try {
                await this.$xapp.signPayload({
                    user_token: this.$xapp.ott,
                    txjson: payment
                })
            } catch(e) {
                if(e.error !== false) {
                    this.$emitter.emit('modal', {
                        type: 'error',
                        title: this.$t('xapp.error.modal_title'),
                        text: this.$t('xapp.error.sign_offer_create'),
                        buttonText: this.$t('xapp.button.close')
                    })
                }
            }
            this.$emitter.emit('busy', false)
        },
        async order() {
            if(!this.funds || this.funds <= 0) {
                return this.$emitter.emit('modal', {
                    type: 'info',
                    title: this.$t('xapp.error.modal_title'),
                    text: this.$t('xapp.info.no_funds'),
                    buttonText: this.$t('xapp.button.close')
                })
            }

            let inputError = false
            if(typeof this.quantity !== 'number' || this.quantity <= 0) {
                this.quantityInputError = true
                inputError = true
            }

            if(inputError) {
                return
            } else {
                this.quantityInputError = false   
            }

            this.$emitter.emit('busy', true)
            try {
                await this.$xapp.signPayload({
                    user_token: this.$xapp.ott,
                    txjson: OfferCreate
                })
            } catch(e) {
                if(e.error !== false) {
                    this.$emitter.emit('modal', {
                        type: 'error',
                        title: this.$t('xapp.error.modal_title'),
                        text: this.$t('xapp.error.sign_offer_create'),
                        buttonText: this.$t('xapp.button.close')
                    })
                }
            }
            this.$emitter.emit('busy', false)
        },
        async selectDestination() {
            this.$emitter.emit('busy', true)
            try {
                const result = await this.$xapp.signPayload({
                    user_token: this.$xapp.ott,
                    txjson: {
                        TransactionType: "SignIn"
                    }
                })
                const account = result.data.response.account
                this.destination = account
                // await this.setAccountData(account)
            } catch(e) {
                if(e.error !== false) {
                    this.$emitter.emit('modal', {
                        type: 'error',
                        title: this.$t('xapp.error.modal_title'),
                        text: this.$t('xapp.error.signin'),
                        buttonText: this.$t('xapp.button.close')
                    })
                }
            }
            try {
                const account_lines = await this.$rippled.send({
                    command: 'account_lines',
                    account: this.destination
                })
                this.destinationtrustlines = account_lines.lines
            } catch(e) {
                this.$emitter.emit('modal', {
                    type: 'error',
                    title: this.$t('xapp.error.modal_title'),
                    text: 'Error getting destination account lines (HC)',
                    buttonText: this.$t('xapp.button.close')
                })
            }
            this.$emitter.emit('busy', false)
        },
        setCurrency(obj) {
            if(this.currency === 'XRP' && obj.currency !== 'XRP') this.quantity = this.quantity / 1_000_000
            if(this.currency !== 'XRP' && obj.currency === 'XRP') this.quantity = this.quantity * 1_000_000
            this.currency = obj.currency
            this.issuer = obj.issuer
        },
        async signin() {
            this.$emitter.emit('busy', true)
            try {
                const result = await this.$xapp.signPayload({
                    user_token: this.$xapp.ott,
                    txjson: {
                        TransactionType: "SignIn"
                    }
                })
                const account = result.data.response.account
                if(this.$xapp.getAccount() === account) throw { msg: 'Same account', error: false }
                await this.$rippled.send({
                    command: 'unsubscribe',
                    accounts: [this.$xapp.getAccount()]
                })
                await this.setAccountData(account)
                await this.$rippled.send({
                    command: 'subscribe',
                    accounts: [account]
                })
            } catch(e) {
                if(e.error !== false) {
                    this.$emitter.emit('modal', {
                        type: 'error',
                        title: this.$t('xapp.error.modal_title'),
                        text: this.$t('xapp.error.signin'),
                        buttonText: this.$t('xapp.button.close')
                    })
                }
            }
            this.$emitter.emit('busy', false)
        },
        async setAccountData(account) {
            const account_info = await this.$rippled.send({
                command: 'account_info',
                account: account
            })
            if(account_info.error === 'actNotFound') {
                this.$xapp.setAccountData(null)
                return this.$xapp.setAccount(account)
            }
            this.$xapp.setAccount(account)

            const account_lines = await this.$rippled.send({
                command: 'account_lines',
                account: account
            })
            const account_objects = await this.$rippled.send({
                command: 'account_objects',
                account: account
            })

            const account_data = {
                account: this.$xapp.getAccount(),
                account_data: account_info.account_data,
                objects: account_objects.account_objects,
                lines: account_lines.lines
            }
            this.$xapp.setAccountData(account_data)
        },
        onPathFindUpdate(path) {
            this.pathFindData = path
        },
        async createPathFind() {
            const amount = this.quantity.toString()
            let command = {}
            if(this.currency === 'XRP') {
                command = {
                    command: 'path_find',
                    subcommand: 'create',
                    source_account: this.account,
                    destination_account: this.account,
                    destination_amount: amount
                }
            } else {
                command = {
                    command: "path_find",
                    subcommand: "create",
                    source_account: this.account,
                    destination_account: this.account,
                    destination_amount: {
                        value: amount,
                        currency: this.currency,
                        issuer: this.issuer
                    }
                }
            }

            try {
                const res = await this.$rippled.send(command)
                this.pathFindData = res
            } catch(e) {
                alert(JSON.stringify(e))
            }
        },
        async closePathFind() {
            await this.$rippled.send({ command: "path_find", subcommand: "close" })
        }
    },
    async mounted() {
        if (typeof window.ReactNativeWebView === 'undefined') {
            this.destination = 'rJR4MQt2egH9AmibZ8Hu5yTKVuLPv1xumm'
            try {
                const account_lines = await this.$rippled.send({
                    command: 'account_lines',
                    account: this.destination
                })
                this.destinationtrustlines = account_lines.lines
            } catch(e) {
                this.$emitter.emit('modal', {
                    type: 'error',
                    title: this.$t('xapp.error.modal_title'),
                    text: 'Error getting destination account lines (HC)',
                    buttonText: this.$t('xapp.button.close')
                })
            }
        }
        setInterval(() => {
            this.online = this.$rippled.getState().online
        }, 1000)
        this.$rippled.on('path', path => this.onPathFindUpdate(path))
    },
    beforeUnmount() {
        this.closePathFind()
    }
}
</script>

<style>
#destination-selector {
    border: 1px solid var(--var-border);
    border-radius: 10px;
    margin: 10px 0;
    padding: 10px 15px;
}
#destination-selector h4 {
    margin: 0;
}
#destination-selector h6 {
    margin: 0;
}
.payment-card {
    border-bottom: solid 1px black;
    width: 90%;
    height: 100%;
    /* margin: 10px 0 !important; */
    text-align: center;
    padding-bottom: 16px;
    padding-top: 8px;
}
span.dot {
    width: 10px;
    height: 10px;
    border-radius: 50%;
    background-color: white;
    display: inline-block;
}
h3 {
    margin: 10px 0;
    font-size: 16px;
}
h6 {
    margin: 0;
}
.align-end {
    text-align: end;
}
hr {
    /* display: block;
    unicode-bidi: isolate;
    margin-block-start: 0.5em;
    margin-block-end: 0.5em;
    margin-inline-start: auto;
    margin-inline-end: auto;
    overflow: hidden;
    border-style: solid;
    border-width: 1px; */
    margin: 0;
    border: 0;
    border-top: 1px solid var(--var-border);
    width: 100%;
}
hr.divide {
    border: 1px solid black;
    width: 110%;
}
.container {
    display: flex;
    flex-direction: column;
    width: 100%;
    height: 100%;
    align-items: center;
}
.row.push {
    justify-content: space-between;
}
.btn.btn-secondary {
    background-color: var(--var-secondary) !important;
    color: grey !important;
}
.btn.btn-small {
    max-width: fit-content !important;
    padding-left: 10px;
}
.btn.btn-success {
    background-color: var(--var-green);
    /* background-color: green; */
}
.btn.btn-cancel {
    background-color: var(--var-red);
    /* background-color: red; */
}
.btn.btn-warning {
    background-color: var(--var-orange);
}
.btn.label {
    padding: 5px 10px !important;
    border-radius: 15px !important;
}
.header {
    margin: 10px;
    text-align: center;
    color: var(--var-primary);
}
.input-error {
    border-color: red !important;
}
.input-label {
    width: 100%;
    text-align: center;
    padding: 10px 5px;
    font-size: 14px;
    border-radius: 10px;
    border: 1px solid var(--var-border);
    position: relative;
}
.input-label:focus-within {
    border: 1px solid var(--var-primary);
}
.input-label label {
    position: absolute;
    top: 0;
    right: 8px;
    bottom: 0;
    z-index: 2;
    display: flex;
    justify-content: center;
    align-items: center;
    /* background-color: var(--var-bg-color); */
}
.input-label input {
    color: var(--var-txt-color);
    background-color: var(--var-bg-color);
    border: 0;
    text-align-last: center;
    font-family: 'Ubuntu Mono' !important;
    width: calc(100% - 8px);
}
.input-label input:focus {
    outline: 0;
}
select {
    -webkit-appearance: none;
}
/* input[inputmode=decimal] */
select {
    width: 100%;
    color: var(--var-txt-color);
    background-color: var(--var-bg-color);
}
/* input[inputmode=decimal] */
select {
    text-align: center;
    text-align-last: center;
    padding: 10px 5px;
    font-size: 14px;
    border-radius: 10px;
    border: 1px solid var(--var-border);
}
select.arrow {
    background-image:
        linear-gradient(45deg, transparent 50%, gray 50%),
        linear-gradient(135deg, gray 50%, transparent 50%);
    background-position:
        calc(100% - 20px) calc(1em + 2px),
        calc(100% - 15px) calc(1em + 2px);
    background-size:
        5px 5px,
        5px 5px;
    background-repeat: no-repeat;
}
select:focus {
    border: 1px solid var(--var-primary);
}
select.arrow:focus {
  background-image:
    linear-gradient(45deg, gray 50%, transparent 50%),
    linear-gradient(135deg, transparent 50%, gray 50%);
  background-position:
    calc(100% - 15px) 1em,
    calc(100% - 20px) 1em;
  background-size:
    5px 5px,
    5px 5px;
  background-repeat: no-repeat;
  outline: 0;
}
</style>