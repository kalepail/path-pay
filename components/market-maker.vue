<template>
<div class="quadrant">
  <h1>Market Maker</h1>

  <h2 v-if="marketMakerAccount">{{marketMakerPublicKey}}</h2>
  <button @click="createAccounts" v-else>{{createAccountsLoading ? '...' : 'Create Account'}}</button>

  <ul class="assets" v-if="marketMakerAccount">
    <li v-for="(asset, i) in marketMakerAccount.balances" :key="i">
      <span class="total"><strong>{{asset.asset_code || 'XLM'}}</strong>: {{asset.balance}}</span>
      <span class="liability">Buying: {{asset.buying_liabilities}}</span>
      <span class="liability">Selling: {{asset.selling_liabilities}}</span>
    </li>
  </ul>

  <div class="actions" v-if="marketMakerAccount">
    <button @click="makeMarket('USD')">{{makeMarketUSDLoading ? '...' : 'Make USD market'}} </button>
    <button @click="makeMarket('EUR')">{{makeMarketEURLoading ? '...' : 'Make EUR market'}} </button>
  </div>
</div>
</template>

<script>
import { Server, Keypair, Account, TransactionBuilder, BASE_FEE, Networks, Operation, Asset } from 'stellar-sdk'
import BigNumber from 'bignumber.js'
import _ from 'lodash-es'

export default {
  data() {
    return {
      marketMakerAccount: null,
      marketMakerSecret: localStorage.getItem('pathpayMarketMaker'),
      assetIssuerSecret: localStorage.getItem('pathpayAssetIssuer'),
      server: new Server('https://horizon-testnet.stellar.org'),
      createAccountsLoading: false,
      makeMarketUSDLoading: false,
      makeMarketEURLoading: false,
    }
  },
  computed: {
    marketMakerKeypair() {
      if (this.marketMakerSecret)
        return Keypair.fromSecret(this.marketMakerSecret)
    },
    marketMakerPublicKey() {
      if (this.marketMakerKeypair)
        return this.marketMakerKeypair.publicKey()
    },
    assetIssuerKeypair() {
      if (this.assetIssuerSecret)
        return Keypair.fromSecret(this.assetIssuerSecret)
    },
    assetIssuerPublicKey() {
      if (this.assetIssuerKeypair)
        return this.assetIssuerKeypair.publicKey()
    }
  },
  created() {
    this.updateMarketMakerAccount()
    this.$parent.$on('updateMarketMakerAccount', this.updateMarketMakerAccount)
  },
  watch: {
    assetIssuerSecret() {
      this.$parent.$emit('assetIssuerSecret', this.assetIssuerSecret)
    }
  },
  methods: {
    async createAccounts() {
      let marketMakerKeypair
      let assetIssuerKeypair

      this.createAccountsLoading = true

      if (this.marketMakerSecret) {
        marketMakerKeypair = Keypair.fromSecret(this.marketMakerSecret)
      } else {
        marketMakerKeypair = Keypair.random()
        this.marketMakerSecret = marketMakerKeypair.secret()
        await this.$axios(`https://friendbot.stellar.org?addr=${this.marketMakerPublicKey}`)
      }

      if (this.assetIssuerSecret) {
        assetIssuerKeypair = Keypair.fromSecret(this.assetIssuerSecret)
      } else {
        assetIssuerKeypair = Keypair.random()
        this.assetIssuerSecret = assetIssuerKeypair.secret()
        await this.$axios(`https://friendbot.stellar.org?addr=${this.assetIssuerPublicKey}`)
      }

      return this.server
      .accounts()
      .accountId(this.assetIssuerPublicKey)
      .call()
      .then(({sequence}) => {
        const usd = new Asset('USD', this.assetIssuerPublicKey)
        const eur = new Asset('EUR', this.assetIssuerPublicKey)
        const account = new Account(this.assetIssuerPublicKey, sequence)

        const transaction = new TransactionBuilder(account, {
          fee: BASE_FEE,
          networkPassphrase: Networks.TESTNET
        })
        .addOperation(Operation.changeTrust({
          asset: usd,
          source: this.marketMakerPublicKey
        }))
        .addOperation(Operation.changeTrust({
          asset: eur,
          source: this.marketMakerPublicKey
        }))
        .addOperation(Operation.payment({
          destination: this.marketMakerPublicKey,
          asset: usd,
          amount: '10000'
        }))
        .addOperation(Operation.payment({
          destination: this.marketMakerPublicKey,
          asset: eur,
          amount: '10000'
        }))
        .setTimeout(0)
        .build()

        transaction.sign(this.marketMakerKeypair, this.assetIssuerKeypair)
        return this.server.submitTransaction(transaction)
      })
      .then((res) => {
        console.log(res)
        localStorage.setItem('pathpayMarketMaker', this.marketMakerSecret)
        localStorage.setItem('pathpayAssetIssuer', this.assetIssuerSecret)
      })
      .catch((err) => console.error(err))
      .finally(() => this.updateMarketMakerAccount())
      .finally(() => this.createAccountsLoading = false)
    },

    makeMarket(asset_code) {
      this[`makeMarket${asset_code}Loading`] = true

      let base_asset_price = new BigNumber('0.0500000')

      if (asset_code === 'EUR')
        base_asset_price = new BigNumber('0.0400000')

      const asset = new Asset(asset_code, this.assetIssuerPublicKey)

      return this.server
      .accounts()
      .accountId(this.assetIssuerPublicKey)
      .call()
      .then(({sequence}) => {
        const account = new Account(this.assetIssuerPublicKey, sequence)

        let transaction = new TransactionBuilder(account, {
          fee: BASE_FEE,
          networkPassphrase: Networks.TESTNET
        })

        _.each(_.range(9), (i) => {
          const fraction = new BigNumber(i).plus(1).times(0.0000001)

          transaction = transaction.addOperation(Operation.manageSellOffer({ // Buy USD/EUR, Sell XLM, Red, Bid
            buying: asset,
            selling: Asset.native(),
            amount: '200',
            price: base_asset_price.plus(fraction).toFixed(7),
            offerId: 0,
            source: this.marketMakerPublicKey
          }))
          transaction = transaction.addOperation(Operation.manageBuyOffer({ // Sell USD/EUR, Buy XLM, Green, Ask
            selling: asset,
            buying: Asset.native(),
            buyAmount: '200',
            price: base_asset_price.minus(fraction).toFixed(7),
            offerId: 0,
            source: this.marketMakerPublicKey
          }))
        })

        transaction = transaction
        .setTimeout(0)
        .build()

        transaction.sign(this.marketMakerKeypair, this.assetIssuerKeypair)
        return this.server.submitTransaction(transaction)
      })
      .then((res) => console.log(res))
      .catch((err) => console.error(err))
      .finally(() => this.updateMarketMakerAccount())
      .finally(() => setTimeout(() => {
        this.$parent.$emit('updateBooks')
        this[`makeMarket${asset_code}Loading`] = false
      }, 1000))
    },

    updateMarketMakerAccount() {
      if (!this.marketMakerPublicKey)
        return

      return this.server
      .accounts()
      .accountId(this.marketMakerPublicKey)
      .call()
      .then((account) => this.marketMakerAccount = account)
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
    display: flex;
    flex-direction: column;
    margin-bottom: 10px;

    &:last-of-type {
      margin-bottom: 0;
    }

    strong {
      font-weight: 600;
    }
    span {
      line-height: 1.2;

      &.total {

      }
      &.liability {
        font-size: 14px;
        text-indent: 10px;
      }
    }
  }
}
.actions {

  button {
    margin-bottom: 0;
  }
}
</style>