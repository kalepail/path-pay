<template>
<div class="quadrant">
  <h1>Fee Channel</h1>

  <h2 v-if="marketMakerAccount">{{marketMakerPublicKey}}</h2>
  <button @click="createAccounts" v-else>{{createAccountsLoading ? '...' : 'Create Account'}}</button>

  <ul class="assets" v-if="marketMakerAccount">
    <li v-for="(asset, i) in marketMakerAccount.balances" :key="i">
      <span class="total"><strong>{{asset.asset_code || 'XLM'}}</strong>: {{asset.balance}}</span>
      <span class="liability">Fees: {{fees}}</span>
    </li>
    <li>
      <span><strong>Sequence</strong>: {{marketMakerAccount.sequence - ogSequence}}</span>
    </li>
  </ul>
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
      ogSequence: null,
      marketMakerSecret: localStorage.getItem('feebumpMarketMaker'),
      server: new Server('https://horizon-testnet.stellar.org'),
      createAccountsLoading: false,
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
    fees() {
      const xlmBalance = _.find(this.marketMakerAccount.balances, {asset_type: 'native'}).balance
      return new BigNumber(10000).minus(xlmBalance).toFixed(7)
    }
  },
  created() {
    this.updateMarketMakerAccount()
    this.$parent.$on('updateMarketMakerAccount', this.updateMarketMakerAccount)
  },
  watch: {

  },
  methods: {
    async createAccounts() {
      let marketMakerKeypair

      this.createAccountsLoading = true

      if (this.marketMakerSecret) {
        marketMakerKeypair = Keypair.fromSecret(this.marketMakerSecret)
      } else {
        marketMakerKeypair = Keypair.random()
        this.marketMakerSecret = marketMakerKeypair.secret()
        await this.$axios(`https://friendbot.stellar.org?addr=${this.marketMakerPublicKey}`)
      }

      localStorage.setItem('feebumpMarketMaker', this.marketMakerSecret)
      this.$parent.$emit('marketMakerSecret', this.marketMakerSecret)

      await this.updateMarketMakerAccount()

      this.createAccountsLoading = false
    },

    updateMarketMakerAccount() {
      if (!this.marketMakerPublicKey)
        return

      return this.server
      .loadAccount(this.marketMakerPublicKey)
      .then((account) => {
        if (this.ogSequence === null)
          this.ogSequence = account.sequence

        this.marketMakerAccount = account
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