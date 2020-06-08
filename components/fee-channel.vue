<template>
<div class="quadrant">
  <h1>Fee Channel</h1>

  <h2 v-if="feeChannelAccount">{{feeChannelPublicKey}}</h2>
  <button @click="createAccounts" v-else>{{createAccountsLoading ? '...' : 'Create Account'}}</button>

  <ul class="assets" v-if="feeChannelAccount">
    <li v-for="(asset, i) in feeChannelAccount.balances" :key="i">
      <span class="total"><strong>{{asset.asset_code || 'XLM'}}</strong>: {{asset.balance}}</span>
      <span class="liability">Fees: {{fees}}</span>
    </li>
    <li>
      <span><strong>Sequence</strong>: {{feeChannelAccount.sequence - ogSequence}}</span>
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
      feeChannelAccount: null,
      ogSequence: null,
      feeChannelSecret: localStorage.getItem('feebumpFeeChannel'),
      server: new Server('https://horizon-testnet.stellar.org'),
      createAccountsLoading: false,
    }
  },
  computed: {
    feeChannelKeypair() {
      if (this.feeChannelSecret)
        return Keypair.fromSecret(this.feeChannelSecret)
    },
    feeChannelPublicKey() {
      if (this.feeChannelKeypair)
        return this.feeChannelKeypair.publicKey()
    },
    fees() {
      const xlmBalance = _.find(this.feeChannelAccount.balances, {asset_type: 'native'}).balance
      return new BigNumber(10000).minus(xlmBalance).toFixed(7)
    }
  },
  created() {
    this.updateFeeChannelAccount()
    this.$parent.$on('updateFeeChannelAccount', this.updateFeeChannelAccount)
  },
  watch: {

  },
  methods: {
    async createAccounts() {
      let feeChannelKeypair

      this.createAccountsLoading = true

      if (this.feeChannelSecret) {
        feeChannelKeypair = Keypair.fromSecret(this.feeChannelSecret)
      } else {
        feeChannelKeypair = Keypair.random()
        this.feeChannelSecret = feeChannelKeypair.secret()
        await this.$axios(`https://friendbot.stellar.org?addr=${this.feeChannelPublicKey}`)
      }

      localStorage.setItem('feebumpFeeChannel', this.feeChannelSecret)
      this.$parent.$emit('feeChannelSecret', this.feeChannelSecret)

      await this.updateFeeChannelAccount()

      this.createAccountsLoading = false
    },

    updateFeeChannelAccount() {
      if (!this.feeChannelPublicKey)
        return

      return this.server
      .loadAccount(this.feeChannelPublicKey)
      .then((account) => {
        if (this.ogSequence === null)
          this.ogSequence = account.sequence

        this.feeChannelAccount = account
      })
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