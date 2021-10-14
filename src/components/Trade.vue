<template>
  <Select @currencyChange="setCurrency" :currency="currency" :issuer="issuer" :lines="destinationtrustlines" />
  <div class="container">
    <div class="row account">
      <h3>My account:</h3>
      <button class="btn btn-sm btn-secondary btn-rounded switch" @click="signin()"><fa :icon="['fas', 'retweet']" rotation="90" /> Switch account</button>
      <span class="number">
        <span class="dot" :style="{'background-color': online ? '#3BDC96' : '#FF5B5B'}"></span>
        {{ account }}
      </span>
    </div>

    <div v-if="$xapp.getAccountData() === null" class="row signin">
      <h3>{{ $t('xapp.headers.unactivated_account') }}</h3>
      <a @click="signin()" class="btn btn-block btn-primary">
        {{ $t('xapp.button.account_not_found_login_button') }}
        <!-- <fa :icon="['fas', 'arrow-right']" /> -->
      </a>
    </div>

    <div v-else-if="!destination" class="row destination">
      <h3>{{ $t('xapp.headers.select_destination') }}</h3>
      <a @click="selectDestination()" class="btn btn-block btn-primary">
        {{ $t('xapp.button.select_destination') }}
        <!-- <fa :icon="['fas', 'arrow-right']" /> -->
      </a>
    </div>

    <div v-else class="row receive">
      <h3>{{ $t('xapp.headers.select_destination') }}</h3>
      <div @click="selectDestination()" id="destination-selector" :class="{'input-error': destinationError}">
        <a>
          <h4>{{ destinationName || 'Account' }}</h4>
          <h6>{{ destination }}</h6>
        </a>
        <fa :icon="['fas', 'sort-down']"></fa>
      </div>

      <h3>{{ $t('xapp.headers.receive') }}</h3>
      <div class="input">
        <div class="input-label" :class="{'input-error': quantityInputValidator || quantityInputError}">
          <input type="text" inputmode="decimal" :placeholder="$t('xapp.input.quantity')" v-model="quantityInput" @keydown="prevent" />
          <label>{{ $xapp.currencyCodeFormat(currency) }}</label>
        </div>
        <a @click="$emitter.emit('select', true)" class="btn btn-secondary btn-rounded select">{{ $t('xapp.button.select_currency') }}</a>
      </div>
    </div>

    <div v-if="destination && $xapp.getAccountData() !== null && !fetching && (quantity <= 0 || quantity === null)" class="row selectvalue">
      <div class="alert alert-primary">
        <span>{{ $t('xapp.headers.no_input') }}</span>
      </div>
    </div>
    <Spinner v-else-if="fetching" />
    <div class="row offer-list" v-else-if="Object.keys(offers).length > 0">
      <h3>{{ $t('xapp.headers.offers') }}</h3>
      <div class="payment-card" v-for="(items, currency) in offers" :key="currency">
        <div class="listitem" v-if="currency === 'XRP'">
          <div class="labelvalue">
            <label>XRP:</label>
            {{ $xapp.currencyFormat(items.source_amount, 'XRP') }}
          </div>
          <a class="btn btn-success btn-sm" @click="pay(items)">{{ $t('xapp.button.pay_now') }}</a>
        </div>

        <div class="listitem" v-else v-for="(item, issuer) in items" :key="issuer">
          <div class="labelvalue">
            <label>{{ $xapp.currencyCodeFormat(item.source_amount.currency, 4) }}:</label>
            {{ $xapp.currencyFormat(item.source_amount.value, item.source_amount.currency) }}
          </div>
          <a class="btn btn-success btn-sm" @click="pay(item)">{{ $t('xapp.button.pay_now') }}</a>
        </div>
      </div>
    </div>
    <h3 v-else-if="destination">{{ $t('xapp.headers.no_offers') }}</h3>
  </div>
</template>

<script>
import Select from '@/components/Select.vue'
import Spinner from './Spinner.vue'

export default {
  name: 'Trade',
  props: ['data'],
  components: {Select, Spinner},
  data() {
    return {
      quantity: null,
      online: false,
      InputQuantity: null,
      quantityInputError: false,
      destination: null,
      destinationError: false,
      destinationName: null,
      issuer: null,
      currency: 'XRP',
      destinationtrustlines: [],
      fetching: false,
      offers: {},
      index: 0
    }
  },
  computed: {
    quantityInputValidator() {
      if (this.quantity < 0) return true
    },
    account() {
      return this.$xapp.getAccount()
    },
    quantityInput: {
      get() {
        return this.InputQuantity
      },
      set(value) {
        if (this.quantityInputError) this.quantityInputError = false

        value = this.parseValue(value)
        if (value === null) {
          this.closePathFind()
          this.offers = {}
          this.InputQuantity = null
          return (this.quantity = null)
        }
        if (parseFloat(value) === 0) {
          this.closePathFind()
          this.offers = {}
        }
        this.InputQuantity = value.toString()
        value = parseFloat(value)

        if (this.currency === 'XRP') {
          value = Math.trunc(value * 1_000_000)
        }
        if (this.quantity !== value) {
          this.quantity = value
          this.onQuantityChange()
        }
      }
    }
  },
  methods: {
    async onQuantityChange() {
      if (this.quantity <= 0) return
      this.fetching = true
      await this.closePathFind()
      if (typeof this.quantity === 'number') {
        await this.createPathFind()
      }
      this.fetching = false
    },
    parseValue(value) {
      if (value === '' || !value) return null
      if (value === ',' || value === '.') return '0.'
      value = value.replace(/,/g, '.')
      return value
    },
    prevent(e) {
      const input = e.target.value
      if (e.key === 'ArrowLeft') return
      if (e.key === 'ArrowRight') return
      if ((e.key === ',' || e.key === '.') && (input.includes('.') || input.includes(','))) return e.preventDefault()
      if (!/^[0-9]$/i.test(e.key)) {
        switch (e.key) {
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
      if (typeof this.quantity !== 'number' || this.quantity <= 0) {
        this.quantityInputError = true
        inputError = true
      }
      if (inputError) {
        return
      } else {
        this.quantityInputError = false
      }
      this.$emitter.emit('busy', true)

      const payment = {
        TransactionType: 'Payment',
        Account: this.account,
        Destination: this.destination,
        SendMax: path.source_amount,
        Amount: this.currency === 'XRP' ? this.quantity.toString() : {currency: this.currency, value: this.quantity, issuer: this.issuer},
        Paths: path.paths_computed,
        Flags: 131072
      }
      try {
        const res = await this.$xapp.signPayload({
          user_token: this.$xapp.ott,
          txjson: payment
        })
        const txid = res.response.txid
        this.$xapp.openTxViewer(txid, this.account)
        this.quantityInput = null
      } catch (e) {
        if (e.error !== false) {
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
      this.destinationError = false

      try {
        const result = await this.$xapp.destinationSelect()
        this.destination = result.destination.address
        this.destinationName = result.destination.name

        this.closePathFind()
        this.offers = {}
        this.InputQuantity = null
        this.quantity = null
      } catch (e) {
        if (e.error !== false) {
          this.$emitter.emit('modal', {
            type: 'error',
            title: this.$t('xapp.error.modal_title'),
            text: this.$t('xapp.error.signin'),
            buttonText: this.$t('xapp.button.close')
          })
        }
      }

      if (typeof window.ReactNativeWebView === 'undefined') {
        this.destination = 'rJR4MQt2egH9AmibZ8Hu5yTKVuLPv1xumm'
      }
      try {
        const account_lines = await this.$rippled.send({
          command: 'account_lines',
          account: this.destination
        })
        this.destinationtrustlines = account_lines.lines

        if (account_lines.error === 'actNotFound') {
          this.destinationError = true
          this.$emitter.emit('modal', {
            type: 'error',
            title: this.$t('xapp.error.modal_title'),
            text: this.$t(`ledger.request_data_response_ws.${account_lines.error}`),
            buttonText: this.$t('xapp.button.close')
          })
        }
      } catch (e) {
        this.$emitter.emit('modal', {
          type: 'error',
          title: this.$t('xapp.error.modal_title'),
          text: this.$t('xapp.error.account_lines'),
          buttonText: this.$t('xapp.button.close')
        })
      }
      this.$emitter.emit('busy', false)
    },
    setCurrency(obj) {
      if (this.currency === obj.currency && this.issuer === obj.issuer) return
      this.closePathFind()
      this.offers = {}
      this.InputQuantity = null
      this.quantity = null
      if (this.currency === 'XRP' && obj.currency !== 'XRP') this.quantity = this.quantity / 1_000_000
      if (this.currency !== 'XRP' && obj.currency === 'XRP') this.quantity = this.quantity * 1_000_000
      this.currency = obj.currency
      this.issuer = obj.issuer
    },
    async signin() {
      this.$emitter.emit('busy', true)
      try {
        const result = await this.$xapp.signPayload({
          user_token: this.$xapp.ott,
          txjson: {
            TransactionType: 'SignIn'
          }
        })
        const account = result.data.response.account
        if (this.$xapp.getAccount() === account) throw {msg: 'Same account', error: false}

        this.closePathFind()
        this.offers = {}
        this.InputQuantity = null
        this.quantity = null

        await this.$rippled.send({
          command: 'unsubscribe',
          accounts: [this.$xapp.getAccount()]
        })
        await this.setAccountData(account)
        await this.$rippled.send({
          command: 'subscribe',
          accounts: [account]
        })
      } catch (e) {
        if (e.error !== false) {
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
      if (account_info.error === 'actNotFound') {
        this.$xapp.setAccountData(null)
        return this.$xapp.setAccount(account)
      }
      this.$xapp.setAccount(account)

      const account_data = {
        account: this.$xapp.getAccount(),
        account_data: account_info.account_data
      }
      this.$xapp.setAccountData(account_data)
    },
    parsePathFindData(path) {
      path.alternatives.forEach((element) => {
        if (typeof element.source_amount === 'string' || typeof element.source_amount === 'number') {
          this.offers['XRP'] = element
        } else {
          this.offers[element.source_amount.currency] = {
            [element.source_amount.issuer]: element
          }
        }
      })
    },
    onPathFindUpdate(path) {
      if (path.id === this.index) this.parsePathFindData(path)
    },
    async createPathFind() {
      this.offers = {}
      const amount = this.quantity.toString()
      let command = {}
      if (this.currency === 'XRP') {
        command = {
          id: this.index,
          command: 'path_find',
          subcommand: 'create',
          source_account: this.account,
          destination_account: this.account,
          destination_amount: amount
        }
      } else {
        command = {
          id: this.index,
          command: 'path_find',
          subcommand: 'create',
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
        this.parsePathFindData(res.result)
      } catch (e) {
        console.log(e)
        this.$emitter.emit('modal', {
          type: 'error',
          title: this.$t('xapp.error.modal_title'),
          text: this.$t('xapp.error.path_find'),
          buttonText: this.$t('xapp.button.close')
        })
      }
    },
    async closePathFind() {
      this.index++
      await this.$rippled.send({command: 'path_find', subcommand: 'close'})
    }
  },
  async mounted() {
    setInterval(() => {
      this.online = this.$rippled.getState().online
    }, 1000)
    this.$rippled.on('path', (path) => this.onPathFindUpdate(path))
  },
  beforeUnmount() {
    this.closePathFind()
  }
}
</script>

<style>
div.account {
  display: grid;
  grid-template-columns: auto auto;
  padding: 0.75rem 1rem 0.75rem 1rem;
  justify-content: space-between;
  border-bottom: 1px solid var(--var-lightgrey);
}

div.signin {
  margin: 0 0 0.5rem 0;
}
div.signin h3 {
  margin-bottom: 0.4rem;
}

div.destination {
  padding: 0.5rem 1rem 0.5rem 1rem;
  margin: 0 0 0.5rem 0;
}
div.destination h3 {
  margin-bottom: 0.4rem;
}

div.receive {
  padding: 0.75rem 1rem 0.75rem 1rem;
  margin: 0 0 0.5rem 0;
  border-bottom: 1px solid var(--var-lightgrey);
}
div.receive h3 {
  margin-bottom: 0.4rem;
}
h3 {
  margin: 0;
  padding: 0;
}
div.account .btn.switch {
  justify-self: end;
}

.offer-list {
  padding: 0.75rem 1rem 0.75rem 1rem !important;
}
.offer-list h3 {
  margin-bottom: 1rem;
}
.offer-list .payment-card {
  background: var(--var-lightgrey);
  /* background: rgb(241,241,241); */
  background: linear-gradient(90deg, var(--var-lightgrey) 0%, rgba(255, 255, 255, 1) 100%);
  margin-bottom: 0.3rem;
  padding: 0.5rem;
  border-radius: 0.5rem;
}
div.listitem {
  display: grid;
  grid-template-columns: 70% auto;
  line-height: 1rem;
  padding: 0 0.3rem;
}
div.listitem label {
  font-weight: bold;
}
div.listitem div.labelvalue {
  line-height: 1.6rem;
  text-overflow: ellipsis;
  white-space: nowrap;
  overflow: hidden;
}
div.listitem div .btn-sm {
  padding: 5px 8px;
}

span.number {
  font-size: 0.8rem;
  font-weight: 800;
  padding: 0.4rem 0.5rem 0.5rem 0.5rem;
  margin-top: 0.5rem;
  grid-column-start: 1;
  grid-column-end: 3;
  /* background: var(--var-lightgrey); */
  border: 1px solid var(--var-lightgrey);
  border-radius: 2.5rem;
  text-align: center;
  /* background: var(--var-lightgrey); */
  /* border-radius: 0.3rem; */
  /* border-top: 1px solid var(--var-lightgrey); */
}
span.number span.dot {
  width: 10px;
  height: 10px;
  border-radius: 50%;
  background-color: white;
  display: inline-block;
  vertical-align: middle;
  margin-bottom: 1px;
}

#destination-selector {
  border: 1px solid var(--var-border);
  border-radius: 10px;
  margin: 10px 0;
  padding: 10px 15px;
  display: flex;
  flex-direction: row;
  justify-content: space-between;
  align-items: center;
}
#destination-selector h4 {
  margin: 0;
}
#destination-selector h6 {
  margin: 0;
}
div.receive div.input {
  display: grid;
  /* grid-template-rows: 3rem; */
  grid-template-columns: 55% auto;
  grid-column-gap: 0.5rem;
}
div.receive div.input .btn.select {
  padding: 0.3rem 0.5rem;
  max-height: 1.5rem;
  line-height: 1.5rem;
  align-self: center;
}

.input-label {
  text-align: center;
  border-radius: 10px;
  border: 1px solid var(--var-lightgrey);
  position: relative;
  display: grid;
  grid-template-columns: 80% 20%;
}
.input-label:focus-within {
  border: 1px solid var(--var-primary);
}
.input-label label {
  position: absolute;
  top: 2px;
  right: 0.75rem;
  bottom: 0;
  z-index: 2;
  display: flex;
  justify-content: center;
  align-items: center;
  font-size: 0.9rem;
  font-weight: bold;
}
.input-label input {
  color: var(--var-txt-color);
  background-color: transparent;
  border: 0;
  text-align: center;
  font-family: 'Ubuntu Mono' !important;
  font-size: 1.4rem;
  font-weight: bold;
  padding: 0.6rem 0.8rem 0.8rem 0.8rem;
}
.input-label input::-webkit-input-placeholder {
  /* Chrome/Opera/Safari */
  color: var(--var-secondary);
  font-size: 1.2rem;
  line-height: 1rem;
}
.input-label input:focus {
  outline: 0;
}

/*
#offer-list {
  flex: 1 1 auto;
  overflow: auto;
}


.align-end {
  text-align: end;
}
hr {
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
  border-top: 1px solid black;
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
}
.btn.btn-cancel {
  background-color: var(--var-red);
}
.btn.btn-warning {
  background-color: var(--var-orange);
}
.btn.label {
  padding: 5px 10px !important;
  border-radius: 15px !important;
}
.header {
  margin: 0 !important;
  text-align: center;
  color: var(--var-primary);
}
.header .number {
  color: var(--var-txt-color) !important;
}
h3 {
  width: 100%;
  margin: 0;
  padding: 0;
  font-family: 'proxima-nova', sans-serif;
  text-align: left !important;
  font-weight: 800 !important;
  border: 1px solid blue;
}
.input-error {
  border-color: red !important;
}

select {
  -webkit-appearance: none;
}
select {
  width: 100%;
  color: var(--var-txt-color);
  background-color: var(--var-bg-color);
}
select {
  text-align: center;
  text-align-last: center;
  padding: 10px 5px;
  font-size: 14px;
  border-radius: 10px;
  border: 1px solid var(--var-border);
}
select.arrow {
  background-image: linear-gradient(45deg, transparent 50%, gray 50%), linear-gradient(135deg, gray 50%, transparent 50%);
  background-position: calc(100% - 20px) calc(1em + 2px), calc(100% - 15px) calc(1em + 2px);
  background-size: 5px 5px, 5px 5px;
  background-repeat: no-repeat;
}
select:focus {
  border: 1px solid var(--var-primary);
}
select.arrow:focus {
  background-image: linear-gradient(45deg, gray 50%, transparent 50%), linear-gradient(135deg, transparent 50%, gray 50%);
  background-position: calc(100% - 15px) 1em, calc(100% - 20px) 1em;
  background-size: 5px 5px, 5px 5px;
  background-repeat: no-repeat;
  outline: 0;
}
*/
</style>
