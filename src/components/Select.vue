<template>
    <div v-show="active" @click.self="active = false; start = true; issuerSelect = false" class="overlay">
        <div id="select-element" v-if="start">
            <div class="exchange-form column">
                <div class="currency-change column" @click="start = false">
                    <h6>Change currency</h6>
                    <a class="btn btn-primary label">{{ $xapp.currencyCodeFormat(currency) }}</a>
                    {{ getIssuerName(issuer) }}
                </div>
            </div>
        </div>
        
        <div id="select-element" v-else>
            <div class="column">
                <div class="header list-header">
                    {{ 'issuer select' }}
                    <fa @click="start = true" :icon="['fas', 'arrow-right']" />
                </div>
                <ul>
                    <li @click="setCurrency('XRP')">XRP</li>
                    <li @click="setCurrency(currency)" v-for="(item, currency, index) in currencyList" :key="index">{{ $xapp.currencyCodeFormat(currency, 16) }}</li>
                </ul>
            </div>
        </div>

        <div v-if="issuerSelect" id="select-element">
            <div class="column">
                <div class="header list-header">
                    <fa @click="start = false; issuerSelect = false" :icon="['fas', 'arrow-left']" />
                    {{ $t('xapp.trade.issuer') }}
                    <fa @click="start = true; issuerSelect = false" :icon="['fas', 'arrow-right']" />
                </div>
                <ul>
                    <li @click="setIssuer({ currency: item.currency, issuer: item.account })" v-for="(item, key, index) in issuers" :key="index">{{ getIssuerName(key) }}</li>
                </ul>
            </div>
        </div>
    </div>
</template>

<script>
import svgImg from '@/components/svg.vue'
import axios from 'redaxios'

export default {
    emits: ['close'],
    components: { svgImg },
    props: ['currency', 'issuer', 'lines'],
    emits: ['currencyChange'],
    data() {
        return {
            active: false,
            header: 'Im a header',
            start: true,
            issuerSelect: false,
            target: '',
            selectedCurrency: '',
            curatedAssets: {},
            tokens: []
        }
    },
    computed: {
        issuers() {
            const obj = this.currencyObject[this.selectedCurrency]
            return obj
        },
        currencyList() {
            var list = this.currencyObject
            return list
        },
        currencyObject() {
            const lines = this.accountTrustlines
            return lines
        },
        curatedCurrencies() {
            const obj = {}

            for(const exchange in this.curatedAssets.details) {
                // If exchange is on short list continue else continue
                if(!this.curatedAssets.details[exchange].shortlist) continue
                for(const currency in this.curatedAssets.details[exchange].currencies) {
                    const details = this.curatedAssets.details[exchange].currencies[currency]
                    if(!details.shortlist) continue
                    
                    const issuer = details.issuer

                    if(typeof obj[currency] === 'undefined') {
                        obj[currency] = {
                           [issuer]: details
                        }
                    } else {
                        // if(obj.currency.hasOwnProperty(issuer)) continue
                        obj[currency][issuer] = details
                    }
                }
            }
            return obj
        },
        accountTrustlines() {
            const array = this.lines
            const obj = {}

            if(Array.isArray(array) && array.length > 0) {
                array.forEach(line => {
                    if(typeof obj[line.currency] === 'undefined') {
                        obj[line.currency] = {
                           [line.account]: line
                        }
                    } else {
                        obj[line.currency][line.account] = line
                    }
                })
            }
            return obj
        }
    },
    methods: {
        getIssuerName(issuer) {
            if(issuer === null) return null
            for(const exchange in this.curatedAssets.details) {
                if(!this.curatedAssets.details[exchange].shortlist) continue
                for(const currency in this.curatedAssets.details[exchange].currencies) {
                    if (this.curatedAssets.details[exchange].currencies[currency].issuer === issuer) return this.curatedAssets.details[exchange].name
                }
            }
            for(const token of this.tokens) {
                if(token === issuer) return this.token[token].data.username
            }
            return 'Unknown'
        },
        openIssuerSelect(line) {
            this.issuerSelect = true
            this.selectedCurrency = line
        },
        setIssuer(object) {
            this.issuerSelect = false
            this.start = true
            this.$emit('currencyChange', object)
        },
        setCurrency(currency) {
            if(currency === 'XRP') {
                this.start = true
                this.$emit('currencyChange', { currency: 'XRP', issuer: null })
                return 
            }
            this.openIssuerSelect(currency)
        }
    },
    async created() {
        this.$emitter.on('select', bool => {
            this.active = bool
        })

        try {
            this.curatedAssets = await this.$xapp.getCuratedAssets()
        } catch(e) {
            this.$emitter.emit('modal', {
                type: 'error',
                title: this.$t('xapp.error.modal_title'),
                text: this.$t('xapp.error.get_curated_assets'),
                buttonText: this.$t('xapp.button.close')
            })
        }

        try {
            const res = await fetch('https://tokens.xumm.community/api/v1/tokens')
            // const res = await axios.get('https://tokens.xumm.community/api/v1/tokens')
            this.tokens = res.tokens
        } catch(e) {
            // alert('error with nixer API')
            // alert(e)
        }
    }
}
</script>

<style scoped>
.overlay {
    position: fixed;
    width: 100%;
    height: 100%;
    top: 0;
    left: 0;
    right: 0;
    bottom: 0;
    background-color: rgba(0,0,0,0.5);
    z-index: 99;
}
#select-element {
    position: absolute;
    top: 50%;
    left: 50%;
    transform: translate(-50%, -50%);
    background-color: var(--var-bg-color);
    width: 160px;
    height: 160px;
    padding: 10px;
    border-radius: 15px;
    display: flex;
    flex-direction: row;
    align-items: center;
}
.list-header {
    border-bottom: 1px solid var(--var-txt-color);
}
.currency-label {
    border: 1px solid var(--var-txt-color);
    width: 50px;
    color: var(--var-txt-color);
    align-self: center;
}
ul {
  list-style-type: none;
  margin: 0;
  padding: 0;
  font-size: 15px;
  height: 100px;
  overflow: auto;
  width: 90%;
  align-self: center;
  /* color: var(--var-txt-color); */
}
li {
    border-bottom: 1px solid var(--var-border);
    padding: 4px 0;
}
</style>