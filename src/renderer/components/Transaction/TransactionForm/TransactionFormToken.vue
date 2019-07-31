<template>
  <form
    class="flex flex-col"
    @submit.prevent
  >
    <ListDivided
      v-if="senderLabel"
      :is-floating-label="true"
    >
      <ListDividedItem :label="$t('TRANSACTION.SENDER')">
        {{ senderLabel }}
        <span
          v-if="senderLabel !== currentWallet.address"
          class="text-sm text-theme-page-text-light"
        >
          {{ currentWallet.address }}
        </span>
      </ListDividedItem>
    </ListDivided>

    <InputSelect
      v-model="$v.qae.type.$model"
      :items="qae1types"
      :label="$t('TRANSACTION.QAETYPE')"
      name="qaetype"
      class="flex-1"
    />

    <InputSelect
      v-if="qae.type === 'SEND'"
      v-model="$v.qae.tokenID.$model"
      :items="qae1tokenIDs"
      :label="$t('QAE.TOKENID_LABEL')"
      name="tokenID"
      class="flex-1"
    />

    <InputText
      v-if="qae.type === 'GENESIS'"
      ref="tokenName"
      v-model="$v.qae.tokenName.$model"
      :label="$t('QAE.NAME_LABEL')"
      :xhelper-text="$t('QAE.NAME_HELPER')"
      name="tokenName"
    />

    <InputText
      v-if="qae.type === 'GENESIS'"
      ref="tokenSymbol"
      v-model="$v.qae.tokenSymbol.$model"
      :label="$t('QAE.SYMBOL_LABEL')"
      :xhelper-text="$t('QAE.SYMBOL_HELPER')"
      name="tokenSymbol"
    />

    <InputText
      v-if="qae.type === 'GENESIS' || qae.type === 'SEND'"
      ref="tokenAmount"
      v-model="$v.qae.tokenAmount.$model"
      :label="$t('QAE.AMOUNT_LABEL')"
      :xhelper-text="$t('QAE.AMOUNT_HELPER')"
      name="tokenAmount"
      class="mb-2"
    />

    <InputText
      v-if="qae.type === 'GENESIS'"
      ref="tokenDecimals"
      v-model="$v.qae.tokenDecimals.$model"
      :label="$t('QAE.DECIMALS_LABEL')"
      :xhelper-text="$t('QAE.DECIMALS_HELPER')"
      name="tokenDecimals"
      class="mb-2"
    />

    <InputText
      v-if="qae.type === 'GENESIS'"
      ref="tokenURI"
      v-model="$v.qae.tokenURI.$model"
      :label="$t('QAE.URI_LABEL')"
      :xhelper-text="$t('QAE.URI_HELPER')"
      name="tokenURI"
      class="mb-2"
    />

    <InputText
      ref="tokenNote"
      v-model="$v.qae.tokenNote.$model"
      :label="$t('QAE.NOTE_LABEL')"
      :xhelper-text="$t('QAE.NOTE_HELPER')"
      name="tokenNote"
      class="mb-2"
    />

    <InputAddress
      v-if="qae.type === 'SEND'"
      ref="recipient"
      v-model="$v.form.recipientId.$model"
      :label="$t('TRANSACTION.RECIPIENT')"
      :pub-key-hash="walletNetwork.version"
      :show-suggestions="true"
      :is-disabled="!currentWallet"
      name="recipientId"
      class="mb-2"
    />

    <div
      v-if="currentWallet && currentWallet.isLedger"
      class="mt-3"
    >
      {{ $t('TRANSACTION.LEDGER_SIGN_NOTICE') }}
    </div>
    <InputPassword
      v-else-if="currentWallet && currentWallet.passphrase"
      ref="password"
      v-model="$v.form.walletPassword.$model"
      class="mt-4"
      :label="$t('TRANSACTION.PASSWORD')"
      :is-required="true"
    />
    <PassphraseInput
      v-else
      ref="passphrase"
      v-model="$v.form.passphrase.$model"
      class="mt-4"
      :address="currentWallet && currentWallet.address"
      :pub-key-hash="walletNetwork.version"
      :is-disabled="!currentWallet"
    />

    <PassphraseInput
      v-if="currentWallet && currentWallet.secondPublicKey"
      ref="secondPassphrase"
      v-model="$v.form.secondPassphrase.$model"
      :label="$t('TRANSACTION.SECOND_PASSPHRASE')"
      :pub-key-hash="walletNetwork.version"
      :public-key="currentWallet.secondPublicKey"
      class="mt-5"
    />

    <div class="self-start">
      <button
        :disabled="$v.form.$invalid"
        class="blue-button mt-10"
        @click="onSubmit"
      >
        {{ $t('COMMON.NEXT') }}
      </button>
    </div>

    <span
      class="mt-4 text-sm text-theme-page-text-light"
    >
      {{ form.vendorField }}
    </span>

    <ModalConfirmation
      v-if="showConfirmSendAll"
      :question="$t('TRANSACTION.CONFIRM_SEND_ALL')"
      :title="$t('TRANSACTION.CONFIRM_SEND_ALL_TITLE')"
      :note="$t('TRANSACTION.CONFIRM_SEND_ALL_NOTE')"
      container-classes="SendAllConfirmation"
      portal-target="loading"
      @close="emitCancelSendAll"
      @cancel="emitCancelSendAll"
      @continue="enableSendAll"
    />
    <ModalLoader
      :message="$t('ENCRYPTION.DECRYPTING')"
      :visible="showEncryptLoader"
    />
    <ModalLoader
      :message="$t('TRANSACTION.LEDGER_SIGN_WAIT')"
      :visible="showLedgerLoader"
    />
  </form>
</template>

<script>
import { required } from 'vuelidate/lib/validators'
import { QAE1, TRANSACTION_TYPES, V1 } from '@config'
import { InputAddress, InputPassword, InputText, InputSelect } from '@/components/Input'
import { ModalConfirmation, ModalLoader } from '@/components/Modal'
import { PassphraseInput } from '@/components/Passphrase'
import TransactionService from '@/services/transaction'
import onSubmit from './mixin-on-submit'
import QreditSltService from '@/services/qreditslt'
import { ListDivided, ListDividedItem } from '@/components/ListDivided'

export default {
  name: 'TransactionFormToken',

  transactionType: TRANSACTION_TYPES.TOKEN,

  components: {
    InputAddress,
    InputPassword,
    InputSelect,
    InputText,
    ListDivided,
    ListDividedItem,
    ModalConfirmation,
    ModalLoader,
    PassphraseInput
  },

  mixins: [onSubmit],

  props: {
    schema: {
      type: Object,
      required: false,
      default: () => {}
    }
  },

  data: vm => ({
    form: {
      amount: '',
      fee: 0,
      passphrase: '',
      walletPassword: '',
      recipientId: '',
      vendorField: ''
    },
    qae: {
      type: '',
      tokenID: '',
      tokenURI: '',
      tokenNote: '',
      tokenName: '',
      tokenSymbol: '',
      tokenAmount: '',
      tokenDecimals: 8
    },
    tokens: [],
    isSendAllActive: false,
    showEncryptLoader: false,
    showLedgerLoader: false,
    previousAmount: '',
    wallet: null,
    showConfirmSendAll: false
  }),

  computed: {
    tokenNameLabel () {
      return 'df'
    },

    qae1tokenIDs () {
      return this.tokens.reduce((all, token) => {
        all[token.tokenIdHex] = token.symbol + ' - ' + this.currency_decimals(token.tokenBalance, token.tokenDecimals) + ' : ' + token.tokenIdHex

        return all
      }, {})
    },
    qae1types () {
      return QAE1.types.reduce((all, type) => {
        all[type] = this.$t(`QAE1_TYPES.${type}`)

        return all
      }, {})
    },
    alternativeCurrency () {
      return this.$store.getters['session/currency']
    },
    // Customize the message to display the minimum amount as subunit
    amountTooLowError () {
      const { fractionDigits } = this.walletNetwork
      const minimumAmount = Math.pow(10, -fractionDigits)
      const amount = this.currency_simpleFormatCrypto(minimumAmount.toFixed(fractionDigits))
      return this.$t('INPUT_CURRENCY.ERROR.LESS_THAN_MINIMUM', { amount })
    },
    notEnoughBalanceError () {
      if (!this.currentWallet) {
        return ''
      }

      const balance = this.formatter_networkCurrency(this.currentWallet.balance)
      return this.$t('TRANSACTION_FORM.ERROR.NOT_ENOUGH_BALANCE', { balance })
    },
    minimumAmount () {
      return this.currency_subToUnit(1)
    },
    maximumAvailableAmount () {
      if (!this.currentWallet) {
        return 0
      }
      return this.currency_subToUnit(this.currentWallet.balance).minus(this.form.fee)
    },
    senderLabel () {
      return this.currentWallet ? this.wallet_formatAddress(this.currentWallet.address) : null
    },
    senderWallet () {
      return this.wallet
    },
    walletNetwork () {
      const sessionNetwork = this.session_network
      if (!this.currentWallet || !this.currentWallet.id) {
        return sessionNetwork
      }

      const profile = this.$store.getters['profile/byId'](this.currentWallet.profileId)
      if (!profile.id) {
        return sessionNetwork
      }
      return this.$store.getters['network/byId'](profile.networkId) || sessionNetwork
    },
    currentWallet: {
      get () {
        return this.senderWallet || this.wallet_fromRoute
      },
      set (wallet) {
        this.wallet = wallet
      }
    }
  },
  watch: {
    'qae': {
      handler: function () {
        this.qaejson()
      },
      deep: true
    },
    wallet () {
      this.ensureAvailableAmount()
      this.$v.form.recipientId.$touch()
      this.pullTokens()
    }
  },

  mounted () {
    // Note: we set this here and not in the data property so validation is triggered properly when fields get pre-populated
    if (this.schema) {
      this.$set(this.form, 'amount', this.schema.amount || '')
      this.$set(this.form, 'recipientId', this.schema.address || '')
    }
    if (this.currentWallet && this.currentWallet.id) {
      this.$set(this, 'wallet', this.currentWallet || null)
      this.$v.wallet.$touch()
    }

    // Set default fees with v1 compatibility
    if (this.walletNetwork.apiVersion === 1) {
      this.form.fee = V1.fees[this.$options.transactionType] / 1e8
    } else {
      this.form.fee = this.$refs.fee
    }
  },

  methods: {
    qaejson () {
      var jsontemplate
      if (this.qae.type === 'GENESIS') {
        jsontemplate = { 'qae1': { 'tp': this.qae.type, 'na': this.qae.tokenName, 'sy': this.qae.tokenSymbol, 'de': this.qae.tokenDecimals.toString(), 'qt': this.qae.tokenAmount, 'du': this.qae.tokenURI, 'no': this.qae.tokenNote } }
      } else if (this.qae.type === '...') {

      } else {
        var rawquantity = this.currency_unitToSub(this.qae.tokenAmount, { fractionDigits: this.qae.tokenDecimals })
        jsontemplate = { 'qae1': { 'tp': this.qae.type, 'id': this.qae.tokenID, 'qt': rawquantity, 'no': this.qae.tokenNote } }
      }
      this.form.vendorField = JSON.stringify(jsontemplate, null, 1)
    },

    pullTokens () {
      QreditSltService.getAllWalletTokens(this.currentWallet.address).then((result) => {
        this.tokens = result
      })
    },

    emitNext (transaction) {
      this.$emit('next', {
        transaction,
        wallet: this.senderWallet
      })
    },

    onFee (fee) {
      this.$set(this.form, 'fee', fee)
      this.ensureAvailableAmount()
    },
    setSendAll (isActive, setPreviousAmount = true) {
      if (isActive) {
        this.confirmSendAll()
        this.previousAmount = this.form['amount']
      }
      if (!isActive) {
        if (setPreviousAmount && !this.previousAmount && this.previousAmount.length) {
          this.$set(this.form, 'amount', this.previousAmount)
        }
        this.previousAmount = ''
        this.isSendAllActive = isActive
        this.ensureAvailableAmount()
      }
    },

    canSendAll () {
      return this.maximumAvailableAmount > 0
    },

    ensureAvailableAmount () {
      if (this.isSendAllActive && this.canSendAll()) {
        this.$set(this.form, 'amount', this.maximumAvailableAmount)
      }
    },

    async submit () {
      const transactionData = {
        amount: this.currency_unitToSub(this.form.amount),
        recipientId: this.form.recipientId,
        vendorField: this.form.vendorField,
        passphrase: this.form.passphrase,
        fee: this.currency_unitToSub(this.form.fee),
        wif: this.form.wif,
        networkWif: this.walletNetwork.wif
      }
      if (this.currentWallet.secondPublicKey) {
        transactionData.secondPassphrase = this.form.secondPassphrase
      }

      let success = true
      let transaction
      if (!this.currentWallet || !this.currentWallet.isLedger) {
        transaction = await this.$client.buildTransfer(transactionData, this.$refs.fee && this.$refs.fee.isAdvancedFee)
      } else {
        success = false
        this.showLedgerLoader = true
        try {
          const transactionObject = await this.$client.buildTransfer(transactionData, this.$refs.fee && this.$refs.fee.isAdvancedFee, true)
          transaction = await TransactionService.ledgerSign(this.currentWallet, transactionObject, this)
          success = true
        } catch (error) {
          this.$error(`${this.$t('TRANSACTION.LEDGER_SIGN_FAILED')}: ${error.message}`)
        }
        this.showLedgerLoader = false
      }

      if (success) {
        this.emitNext(transaction)
      }
    },

    enableSendAll () {
      this.isSendAllActive = true
      this.ensureAvailableAmount()
      this.showConfirmSendAll = false
    },

    confirmSendAll () {
      this.showConfirmSendAll = true
    },

    emitCancelSendAll () {
      this.showConfirmSendAll = false
      this.isSendAllActive = false
    }
  },

  validations: {
    wallet: {},
    qae: {
      tokenURI: {
        isValid (value) {
          return true
        }
      },
      tokenNote: {
        isValid (value) {
          return true
        }
      },
      tokenAmount: {
        required,
        isValid (value) {
          if (this.$refs.type) {
            return !this.$refs.type.$v.$invalid
          }
          return false
        }
      },
      tokenID: {
        required,
        isValid (value) {
          if (this.$refs.type) {
            return !this.$refs.type.$v.$invalid
          }
          return false
        }
      },
      tokenDecimals: {
        required,
        isValid (value) {
          if (this.$refs.type) {
            return !this.$refs.type.$v.$invalid
          }
          return false
        }
      },
      tokenName: {
        required,
        isValid (value) {
          if (this.$refs.type) {
            return !this.$refs.type.$v.$invalid
          }
          return false
        }
      },
      tokenSymbol: {
        required,
        isValid (value) {
          if (this.$refs.type) {
            return !this.$refs.type.$v.$invalid
          }
          return false
        }
      },
      type: {
        required,
        isValid (value) {
          if (this.$refs.type) {
            return !this.$refs.type.$v.$invalid
          }
          return false
        }
      }
    },
    form: {
      recipientId: {
        required,
        isValid (value) {
          if (this.$refs.recipient) {
            return !this.$refs.recipient.$v.$invalid
          }
          return false
        }
      },
      amount: {
        required,
        isValid (value) {
          if (this.$refs.amount) {
            return !this.$refs.amount.$v.$invalid
          }
          return false
        }
      },
      fee: {
        required,
        isValid () {
          if (this.$refs.fee) {
            return !this.$refs.fee.$v.$invalid
          }
          return this.walletNetwork.apiVersion === 1 // Return true if it's v1, since it has a static fee
        }
      },
      passphrase: {
        isValid (value) {
          if (this.currentWallet.isLedger || this.currentWallet.passphrase) {
            return true
          }

          if (this.$refs.passphrase) {
            return !this.$refs.passphrase.$v.$invalid
          }

          return false
        }
      },
      walletPassword: {
        isValid (value) {
          if (this.currentWallet.isLedger || !this.currentWallet.passphrase) {
            return true
          }

          if (!this.form.walletPassword || !this.form.walletPassword.length) {
            return false
          }

          if (this.$refs.password) {
            return !this.$refs.password.$v.$invalid
          }

          return false
        }
      },
      secondPassphrase: {
        isValid (value) {
          if (!this.currentWallet.secondPublicKey) {
            return true
          }

          if (this.$refs.secondPassphrase) {
            return !this.$refs.secondPassphrase.$v.$invalid
          }
          return false
        }
      }
    }
  }
}
</script>

<style>
.SendAllConfirmation .ModalConfirmation__container {
  min-width: calc(var(--contact-identicon-xl) + 74px * 2);
  max-width: calc(var(--contact-identicon-xl) + 74px * 2 + 50px)
}
</style>