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
      <strong>Sequence</strong>: {{userAccount.sequence - ogSequence}}
    </li>
  </ul>

  <div class="actions" v-if="
    marketMakerSecret
    && userAccount
    && type === 'you'
  ">
    <button @click="feeBumpPay" v-if="friendPublicKey">{{ feeBumpPayLoading ? '...' : 'Send 10 XLM to Friend via Fee Bump Transaction' }}</button>
  </div>
</div>
</template>

<script>
import { Server, Keypair, Account, TransactionBuilder, BASE_FEE, Networks, Operation, Asset } from 'stellar-sdk'

export default {
  props: ['type'],
  data() {
    return {
      userSecret: null,
      userAccount: null,
      ogSequence: null,
      friendSecret: localStorage.getItem('feebumpFriend'),
      marketMakerSecret: localStorage.getItem('feebumpMarketMaker'),
      server: new Server('https://horizon-testnet.stellar.org'),
      createAccountLoading: false,
      feeBumpPayLoading: false,
    }
  },
  computed: {
    typeFriendly() {
      return this.type === 'you' ? 'Your' : 'Friend'
    },
    userKeypair() {
      if (this.userSecret)
        return Keypair.fromSecret(this.userSecret)
    },
    userPublicKey() {
      if (this.userKeypair)
        return this.userKeypair.publicKey()
    },
    friendKeypair() {
      if (this.friendSecret)
        return Keypair.fromSecret(this.friendSecret)
    },
    friendPublicKey() {
      if (this.friendKeypair)
        return this.friendKeypair.publicKey()
    },
    marketMakerPublicKey() {
      if (this.marketMakerSecret)
        return Keypair.fromSecret(this.marketMakerSecret).publicKey()
    },
    marketMakerKeypair() {
      if (this.marketMakerSecret)
        return Keypair.fromSecret(this.marketMakerSecret)
    },
  },
  created() {
    this.userSecret = localStorage.getItem(`feebump${this.typeFriendly}`)
    this.updateUserAccount()

    this.$parent.$on('marketMakerSecret', (marketMakerSecret) => this.marketMakerSecret = marketMakerSecret)

    if (this.type === 'friend')
      this.$parent.$on('updateUserAccount', this.updateUserAccount)
    else
      this.$parent.$on('friendSecret', (friendSecret) => this.friendSecret = friendSecret)
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
      this.updateUserAccount()

      if (this.type === 'friend')
        this.$parent.$emit('friendSecret', this.userSecret)

      this.createAccountLoading = true
    },

    feeBumpPay() {
      this.feeBumpPayLoading = true

      this.server
      .loadAccount(this.userPublicKey)
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

        innerTx.sign(this.userKeypair)

        const feeBumpTxn = new TransactionBuilder.buildFeeBumpTransaction(
          this.marketMakerKeypair,
          BASE_FEE,
          innerTx,
          Networks.TESTNET
        )

        feeBumpTxn.sign(this.marketMakerKeypair)

        return this.server.submitTransaction(feeBumpTxn)
      })
      .then((res) => console.log(res))
      .catch((err) => console.error(err))
      .finally(() => this.updateUserAccount())
      .finally(() => setTimeout(() => {
        this.$parent.$emit('updateUserAccount')
        this.$parent.$emit('updateMarketMakerAccount')
        this.feeBumpPayLoading = false
      }, 1000))
    },

    updateUserAccount() {
      if (!this.userPublicKey)
        return

      return this.server
      .loadAccount(this.userPublicKey)
      .then((account) => {
        if (this.ogSequence === null)
          this.ogSequence = account.sequence

        this.userAccount = account
      })
    },
  }
}
</script>

<style lang="scss" scoped>
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
    margin-bottom: 0;
  }
}
</style>