<template>
<div class="quadrant">
  <h1>{{typeFriendly}} Wallet</h1>

  <h2 v-if="userAccount">{{userPublicKey}}</h2>
  <button @click="createAccount" v-else>Create Account</button>

  <ul class="assets" v-if="userAccount">
    <li v-for="(asset, i) in userAccount.balances" :key="i">
      <strong>{{asset.asset_code || 'XLM'}}</strong>: {{asset.balance}}
    </li>
  </ul>

  <div class="actions" v-if="
    assetIssuerPublicKey 
    && userAccount 
    && type === 'you'
  ">
    <button @click="pathPayUsd">Purchase USD with XLM</button>
    <button @click="pathPayEur" v-if="friendPublicKey">Send EUR to friend via USD</button>
  </div>
</div>
</template>

<script>
import { Server, Keypair, Account, TransactionBuilder, BASE_FEE, Networks, Operation, Asset } from 'stellar-sdk'
// import BigNumber from 'bignumber.js'
// import _ from 'lodash-es'

export default {
  props: ['type'],
  data() {
    return {
      userSecret: null,
      userAccount: null,
      friendSecret: localStorage.getItem('pathpayFriend'),
      assetIssuerSecret: localStorage.getItem('pathpayAssetIssuer'),
      server: new Server('https://horizon-testnet.stellar.org'),
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
    assetIssuerPublicKey() {
      if (this.assetIssuerSecret)
        return Keypair.fromSecret(this.assetIssuerSecret).publicKey()
    }
  },
  created() {
    this.userSecret = localStorage.getItem(`pathpay${this.typeFriendly}`)
    this.updateUserAccount()

    this.$parent.$on('assetIssuerSecret', (assetIssuerSecret) => this.assetIssuerSecret = assetIssuerSecret)

    if (this.type === 'friend')
      this.$parent.$on('updateUserAccount', this.updateUserAccount)
    else
      this.$parent.$on('friendSecret', (friendSecret) => this.friendSecret = friendSecret)
  },
  methods: {
    async createAccount() {
      let userKeypair

      if (this.userSecret) {
        userKeypair = Keypair.fromSecret(this.userSecret)
      } else {
        userKeypair = Keypair.random()
        this.userSecret = userKeypair.secret()
        await this.$axios(`https://friendbot.stellar.org?addr=${this.userPublicKey}`)
      }

      localStorage.setItem(`pathpay${this.typeFriendly}`, this.userSecret)
      this.updateUserAccount()

      if (this.type === 'friend')
        this.$parent.$emit('friendSecret', this.userSecret)
    },

    pathPayUsd() {
      return this.server
      .accounts()
      .accountId(this.userPublicKey)
      .call()
      .then(({sequence}) => {
        const usd = new Asset('USD', this.assetIssuerPublicKey)
        const account = new Account(this.userPublicKey, sequence)

        const transaction = new TransactionBuilder(account, {
          fee: BASE_FEE,
          networkPassphrase: Networks.TESTNET
        })
        .addOperation(Operation.changeTrust({
          asset: usd
        }))
        .addOperation(Operation.pathPaymentStrictSend({
          sendAsset: Asset.native(),
          sendAmount: '500',
          destination: this.userPublicKey,
          destAsset: usd,
          destMin: '1', // Will receive roughly 25 USD
        }))
        .setTimeout(0)
        .build()

        transaction.sign(this.userKeypair)
        return this.server.submitTransaction(transaction)
      })
      .then((res) => console.log(res))
      .catch((err) => console.error(err))
      .finally(() => this.updateUserAccount())
      .finally(() => setTimeout(() => {
        this.$parent.$emit('updateMarketMakerAccount')
        this.$parent.$emit('updateBooks')
      }, 1000))
    },

    pathPayEur() {
      return this.server
      .accounts()
      .accountId(this.userPublicKey)
      .call()
      .then(({sequence}) => {
        const usd = new Asset('USD', this.assetIssuerPublicKey)
        const eur = new Asset('EUR', this.assetIssuerPublicKey)
        const account = new Account(this.userPublicKey, sequence)

        const transaction = new TransactionBuilder(account, {
          fee: BASE_FEE,
          networkPassphrase: Networks.TESTNET
        })
        .addOperation(Operation.changeTrust({
          asset: eur,
          source: this.friendPublicKey
        }))
        .addOperation(Operation.pathPaymentStrictReceive({
          sendAsset: usd,
          sendMax: '20',
          destination: this.friendPublicKey,
          destAsset: eur,
          destAmount: '10',
          path: [
            usd,
            Asset.native(),
            eur
          ]
        }))
        .setTimeout(0)
        .build()

        transaction.sign(this.userKeypair, this.friendKeypair)
        return this.server.submitTransaction(transaction)
      })
      .then((res) => console.log(res))
      .catch((err) => console.error(err))
      .finally(() => this.updateUserAccount())
      .finally(() => setTimeout(() => {
        this.$parent.$emit('updateUserAccount')
        this.$parent.$emit('updateMarketMakerAccount')
        this.$parent.$emit('updateBooks')
      }, 1000))
    },

    updateUserAccount() {
      if (!this.userPublicKey)
        return

      return this.server
      .accounts()
      .accountId(this.userPublicKey)
      .call()
      .then((account) => this.userAccount = account)
    },
  }
}
</script>

<style lang="scss" scoped>
.quadrant {
  height: 100%;
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
    margin-bottom: 0;
  }
}
</style>