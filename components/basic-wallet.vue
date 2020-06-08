<template>
<div class="quadrant">
  <h1>{{typeFriendly}} Wallet</h1>

  <h2 v-if="userAccount">{{userPublicKey}}</h2>
  <button @click="createAccount" v-else>{{createAccountLoading ? '...' : 'Create Account'}}</button>

  <ul class="assets" v-if="userAccount">
    <li v-for="(asset, i) in userAccount.balances" :key="i">
      <strong>{{asset.asset_code || 'XLM'}}</strong>: {{asset.balance}}
    </li>
    <li>
      <strong>Sequence</strong>: {{userAccount.sequence - userAccountOgSequence}}
    </li>
  </ul>

  <div class="actions" v-if="
    feeChannelPublicKey
    && userAccount
    && friendPublicKey
    && type === 'bob'
  ">
    <button @click="feeBumpPay()">{{ feeBumpPayLoading ? '...' : 'Send 10 XLM to Friend via Fee Bump Transaction' }}</button>
  </div>

  <div class="actions" v-if="
    feeChannelPublicKey
    && userAccount
    && friendPublicKey
    && type === 'alice'
  ">
    <button @click="plainPayBoth">{{ plainPayLoading ? '...' : 'Send 10 XLM to Friend from Bob and Alice via Plain Transaction' }}</button>
    <button @click="feeBumpPayBoth">{{ feeBumpPayLoading ? '...' : 'Send 10 XLM to Friend from Bob and Alice via Fee Bump Transaction' }}</button>
  </div>

  <pre class="error" v-if="error">{{JSON.stringify(error, null, 2)}}</pre>
</div>
</template>

<script>
import { Server, Keypair, Account, TransactionBuilder, BASE_FEE, Networks, Operation, Asset } from 'stellar-sdk'

export default {
  props: ['type'],
  data() {
    return {
      server: new Server('https://horizon-testnet.stellar.org'),

      userSecret: null,
      userAccount: null,
      userAccountOgSequence: null,

      bobSecret: localStorage.getItem('feebumpBob'),
      aliceSecret: localStorage.getItem('feebumpAlice'),
      friendSecret: localStorage.getItem('feebumpFriend'),

      feeChannelSecret: localStorage.getItem('feebumpFeeChannel'),

      createAccountLoading: false,
      feeBumpPayLoading: false,
      plainPayLoading: false,

      error: null
    }
  },
  computed: {
    typeFriendly() {
      return this.type[0].toUpperCase() +  this.type.substr(1, this.type.length)
    },

    userKeypair() {
      if (this.userSecret)
        return Keypair.fromSecret(this.userSecret)
    },
    userPublicKey() {
      if (this.userKeypair)
        return this.userKeypair.publicKey()
    },

    bobKeypair() {
      if (this.bobSecret)
        return Keypair.fromSecret(this.bobSecret)
    },
    bobPublicKey() {
      if (this.bobKeypair)
        return this.bobKeypair.publicKey()
    },
    aliceKeypair() {
      if (this.aliceSecret)
        return Keypair.fromSecret(this.aliceSecret)
    },
    alicePublicKey() {
      if (this.aliceKeypair)
        return this.aliceKeypair.publicKey()
    },
    friendKeypair() {
      if (this.friendSecret)
        return Keypair.fromSecret(this.friendSecret)
    },
    friendPublicKey() {
      if (this.friendKeypair)
        return this.friendKeypair.publicKey()
    },

    feeChannelKeypair() {
      if (this.feeChannelSecret)
        return Keypair.fromSecret(this.feeChannelSecret)
    },
    feeChannelPublicKey() {
      if (this.feeChannelKeypair)
        return this.feeChannelKeypair.publicKey()
    },
  },
  created() {
    this.userSecret = localStorage.getItem(`feebump${this.typeFriendly}`)
    this.updateUserAccount()

    this.$parent.$on('updateUserAccount', this.updateUserAccount)
    this.$parent.$on('feeChannelSecret', (feeChannelSecret) => this.feeChannelSecret = feeChannelSecret)
    this.$parent.$on('setWalletSecret', ({type, secret}) => this[`${type}Secret`] = secret)
  },
  methods: {
    async createAccount() {
      let userKeypair

      this.createAccountLoading = true

      if (this.userSecret) {
        userKeypair = Keypair.fromSecret(this.userSecret)
      } else {
        userKeypair = Keypair.random()
        this.userSecret = userKeypair.secret()
        await this.$axios(`https://friendbot.stellar.org?addr=${this.userPublicKey}`)
      }

      localStorage.setItem(`feebump${this.typeFriendly}`, this.userSecret)
      this.$parent.$emit('setWalletSecret', {type: this.type, secret: this.userSecret})
      await this.updateUserAccount()

      this.createAccountLoading = true
    },
    updateUserAccount() {
      if (!this.userPublicKey)
        return

      return this.server
      .loadAccount(this.userPublicKey)
      .then((account) => {
        if (this.userAccountOgSequence === null)
          this.userAccountOgSequence = account.sequence

        this.userAccount = account
      })
    },

    feeBumpPay(publicKey, keypair) {
      this.feeBumpPayLoading = true

      return this.server
      .loadAccount(publicKey || this.userPublicKey)
      .then((account) => {
        const innerTx = new TransactionBuilder(account, {
          fee: BASE_FEE,
          networkPassphrase: Networks.TESTNET,
          v1: true
        })
        .addOperation(Operation.payment({
          destination: this.friendPublicKey,
          asset: Asset.native(),
          amount: '10'
        }))
        .setTimeout(0)
        .build()

        innerTx.sign(keypair || this.userKeypair)

        const feeBumpTxn = new TransactionBuilder.buildFeeBumpTransaction(
          this.feeChannelKeypair,
          BASE_FEE,
          innerTx,
          Networks.TESTNET
        )

        feeBumpTxn.sign(this.feeChannelKeypair)

        return this.server.submitTransaction(feeBumpTxn)
      })
      .then((res) => console.log(res))
      .catch((err) => {
        if (
          publicKey
          && keypair
        ) throw err

        else
          console.error(err)
      })
      .finally(() => this.updateUserAccount())
      .finally(() => setTimeout(() => {
        this.$parent.$emit('updateUserAccount')
        this.$parent.$emit('updateFeeChannelAccount')

        if (
          !publicKey
          && !keypair
        ) this.feeBumpPayLoading = false
      }, 1000))
    },
    async feeBumpPayBoth() {
      this.error = null
      this.feeBumpPayLoading = true

      try {
        const bobPromise = this.feeBumpPay(this.bobPublicKey, this.bobKeypair)
        const alicePromise = this.feeBumpPay(this.alicePublicKey, this.aliceKeypair)

        const bobTask = await bobPromise
        const aliceTask = await alicePromise
      }

      catch(err) {
        console.error(err)
        this.error = err.response.data
      }

      finally {
        this.feeBumpPayLoading = false
      }
    },

    plainPay(publicKey, keypair) {
      return this.server
      .loadAccount(this.feeChannelPublicKey)
      .then((account) => {
        const transaction = new TransactionBuilder(account, {
          fee: BASE_FEE,
          networkPassphrase: Networks.TESTNET
        })
        .addOperation(Operation.payment({
          destination: this.friendPublicKey,
          asset: Asset.native(),
          amount: '10',
          source: publicKey
        }))
        .setTimeout(0)
        .build()

        transaction.sign(this.feeChannelKeypair, keypair)

        return this.server.submitTransaction(transaction)
      })
      .then((res) => console.log(res))
      .finally(() => this.updateUserAccount())
      .finally(() => setTimeout(() => {
        this.$parent.$emit('updateUserAccount')
        this.$parent.$emit('updateFeeChannelAccount')
      }, 1000))
    },
    async plainPayBoth() {
      this.error = null
      this.plainPayLoading = true

      try {
        const bobPromise = this.plainPay(this.bobPublicKey, this.bobKeypair)
        const alicePromise = this.plainPay(this.alicePublicKey, this.aliceKeypair)

        const bobTask = await bobPromise
        const aliceTask = await alicePromise
      }

      catch(err) {
        console.error(err)
        this.error = err.response.data
      }

      finally {
        this.plainPayLoading = false
      }
    },
  }
}
</script>

<style lang="scss" scoped>
.quadrant {
  flex-shrink: 0;
}
h1 {
  font-size: 24px;
  font-weight: 600;
  margin-bottom: 5px;
}
h2,
button {
  margin-bottom: 20px;
}
button {
  font-size: 14px;
}

.assets {
  margin-bottom: 20px;

  li {
    margin-bottom: 10px;

    &:last-of-type {
      margin-bottom: 0;
    }

    strong {
      font-weight: 600;
    }
  }
}
.actions {

  button {
    margin-bottom: 10px;

    &:last-of-type {
      margin-bottom: 0;
    }
  }
}
.error {
  font-size: 12px;
  color: red;
  background-color: whitesmoke;
  overflow: scroll;
  width: calc(100% - 20px);
  padding: 10px;
  margin-top: 20px;
}
</style>