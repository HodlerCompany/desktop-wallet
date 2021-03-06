<template>
  <form
    class="TransactionFormSecondSignature"
    @submit.prevent
  >
    <template v-if="!currentWallet.secondPublicKey">
      <div class="mb-5">
        {{ $t('TRANSACTION.FORM.SECOND_SIGNATURE.INSTRUCTIONS', { address: currentWallet.address }) }}
      </div>

      <Collapse
        :is-open="!isPassphraseStep"
        :animation-duration="{ enter: 0, leave: 0 }"
      >
        <PassphraseWords :passphrase-words="passphraseWords" />

        <button
          type="button"
          class="blue-button mt-5"
          @click="toggleStep"
        >
          {{ $t('COMMON.NEXT') }}
        </button>
      </Collapse>

      <Collapse
        :is-open="isPassphraseStep"
      >
        <PassphraseVerification
          ref="passphraseVerification"
          :passphrase="passphraseWords"
          :word-positions="wordPositions"
          class="mb-20"
          @verified="onVerification"
        />

        <InputFee
          v-if="session_network.apiVersion === 2"
          ref="fee"
          :currency="session_network.token"
          :transaction-type="$options.transactionType"
          @input="onFee"
        />

        <div
          v-if="currentWallet.isLedger"
          class="mt-10"
        >
          {{ $t('TRANSACTION.LEDGER_SIGN_NOTICE') }}
        </div>
        <InputPassword
          v-else-if="currentWallet.passphrase"
          ref="password"
          v-model="$v.form.walletPassword.$model"
          :label="$t('TRANSACTION.PASSWORD')"
          :is-required="true"
        />
        <PassphraseInput
          v-else
          ref="passphrase"
          v-model="$v.form.passphrase.$model"
          :address="currentWallet.address"
          :pub-key-hash="session_network.version"
          class="mt-5"
        />

        <button
          type="button"
          class="blue-button mt-5 mr-4"
          @click="toggleStep"
        >
          {{ $t('COMMON.BACK') }}
        </button>

        <button
          :disabled="$v.form.$invalid || !isPassphraseVerified"
          type="button"
          class="blue-button mt-5"
          @click="onSubmit"
        >
          {{ $t('COMMON.NEXT') }}
        </button>
      </Collapse>

      <ModalLoader
        ref="modalLoader"
        :message="$t('ENCRYPTION.DECRYPTING')"
        :visible="showEncryptLoader"
      />
      <ModalLoader
        :message="$t('TRANSACTION.LEDGER_SIGN_WAIT')"
        :visible="showLedgerLoader"
      />

      <Portal
        v-if="!isPassphraseStep"
        to="transaction-footer"
      >
        <footer class="ModalWindow__container__footer--warning flex flex-row">
          <div class="flex w-80">
            {{ $t('WALLET_SECOND_SIGNATURE.INSTRUCTIONS') }}
          </div>
          <div class="flex flex-row justify-around ml-8">
            <ButtonReload
              :is-refreshing="isGenerating"
              :title="$t('WALLET_SECOND_SIGNATURE.NEW')"
              class="bg-theme-modal-footer-button mr-2"
              text-class="text-theme-modal-footer-button-text mt-1"
              @click="generateNewPassphrase"
            />

            <ButtonClipboard
              :value="secondPassphrase"
              class="flex py-2 px-4 rounded bg-theme-modal-footer-button text-theme-modal-footer-button-text"
            />
          </div>
        </footer>
      </Portal>
    </template>
    <template v-else>
      {{ $t('WALLET_SECOND_SIGNATURE.ALREADY_REGISTERED') }}
    </template>
  </form>
</template>

<script>
import { required } from 'vuelidate/lib/validators'
import { TRANSACTION_TYPES, V1 } from '@config'
import { ButtonClipboard, ButtonReload } from '@/components/Button'
import { Collapse } from '@/components/Collapse'
import { InputFee, InputPassword } from '@/components/Input'
import { ModalLoader } from '@/components/Modal'
import { PassphraseInput, PassphraseVerification, PassphraseWords } from '@/components/Passphrase'
import TransactionService from '@/services/transaction'
import WalletService from '@/services/wallet'

export default {
  name: 'TransactionFormSecondSignature',

  transactionType: TRANSACTION_TYPES.SECOND_SIGNATURE,

  components: {
    ButtonClipboard,
    ButtonReload,
    Collapse,
    InputFee,
    InputPassword,
    ModalLoader,
    PassphraseInput,
    PassphraseVerification,
    PassphraseWords
  },

  data: () => ({
    isGenerating: false,
    isPassphraseStep: false,
    isPassphraseVerified: false,
    secondPassphrase: '',
    form: {
      fee: 0,
      passphrase: '',
      walletPassword: ''
    },
    showEncryptLoader: false,
    showLedgerLoader: false,
    bip38Worker: null
  }),

  computed: {
    wordPositions () {
      return [3, 6, 9]
    },

    passphraseWords () {
      return this.secondPassphrase.split(' ')
    },

    currentWallet () {
      return this.wallet_fromRoute
    }
  },

  watch: {
    isPassphraseStep () {
      this.$refs.passphraseVerification.focusFirst()
    }
  },

  created () {
    this.secondPassphrase = WalletService.generateSecondPassphrase(this.session_profile.bip39Language)
  },

  beforeDestroy () {
    this.bip38Worker.send('quit')
  },

  mounted () {
    if (this.bip38Worker) {
      this.bip38Worker.send('quit')
    }
    this.bip38Worker = this.$bgWorker.bip38()
    this.bip38Worker.on('message', message => {
      if (message.decodedWif === null) {
        this.$error(this.$t('ENCRYPTION.FAILED_DECRYPT'))
        this.showEncryptLoader = false
      } else if (message.decodedWif) {
        this.form.passphrase = null
        this.form.wif = message.decodedWif
        this.showEncryptLoader = false
        this.submit()
      }
    })
  },

  methods: {
    toggleStep () {
      this.isPassphraseStep = !this.isPassphraseStep
    },

    generateNewPassphrase () {
      this.isGenerating = true
      setTimeout(() => {
        this.secondPassphrase = WalletService.generateSecondPassphrase(this.session_profile.bip39Language)
        this.isGenerating = false
      }, 300)
    },

    onFee (fee) {
      this.$set(this.form, 'fee', fee)
    },

    onSubmit () {
      if (this.form.walletPassword && this.form.walletPassword.length) {
        this.showEncryptLoader = true
        this.bip38Worker.send({
          bip38key: this.currentWallet.passphrase,
          password: this.form.walletPassword,
          wif: this.session_network.wif
        })
      } else {
        this.submit()
      }
    },

    async submit () {
      // v1 compatibility
      if (this.session_network.apiVersion === 1) {
        this.form.fee = V1.fees[this.$options.transactionType]
      }
      // Ensure that fee has value, even when the user has not interacted
      if (!this.form.fee) {
        this.form.fee = this.$refs.fee.fee
      }

      const transactionData = {
        passphrase: this.form.passphrase,
        secondPassphrase: this.secondPassphrase,
        fee: parseInt(this.currency_unitToSub(this.form.fee)),
        wif: this.form.wif
      }

      let success = true
      let transaction
      if (!this.currentWallet.isLedger) {
        transaction = await this.$client.buildSecondSignatureRegistration(transactionData, this.$refs.fee && this.$refs.fee.isAdvancedFee)
      } else {
        success = false
        this.showLedgerLoader = true
        try {
          const transactionObject = await this.$client.buildSecondSignatureRegistration(transactionData, this.$refs.fee && this.$refs.fee.isAdvancedFee, true)
          transaction = await TransactionService.ledgerSign(this.currentWallet, transactionObject, this)
          success = true
        } catch (error) {
          this.$error(`${this.$t('TRANSACTION.LEDGER_SIGN_FAILED')}: ${error.message}`)
        }
        this.showLedgerLoader = false
      }

      if (success) {
        this.emitNext(transaction)
        this.reset()
      }
    },

    onVerification () {
      this.isPassphraseVerified = true
    },

    reset () {
      this.isPassphraseStep = false
      this.isPassphraseVerified = false
      if (!this.currentWallet.passphrase && !this.currentWallet.isLedger) {
        this.$set(this.form, 'passphrase', '')
        this.$refs.passphrase.reset()
      } else if (!this.currentWallet.isLedger) {
        this.$set(this.form, 'walletPassword', '')
        this.$refs.password.reset()
      }
      this.$v.$reset()
    },

    emitNext (transaction) {
      this.$emit('next', transaction)
    }
  },

  validations: {
    form: {
      fee: {
        required,
        isValid () {
          if (this.$refs.fee) {
            return !this.$refs.fee.$v.$invalid
          }
          return this.session_network.apiVersion === 1 // Return true if it's v1, since it has a static fee
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
      }
    }
  }
}
</script>

<style scoped>
.TransactionFormSecondSignature {
  min-width: 25em;
}

.TransactionFormSecondSignature /deep/ .Collapse__handler {
  display: none
}
</style>
