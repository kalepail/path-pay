<template>
<div class="quadrant">
  <h1>Order Books</h1>

  <div class="books-container" v-if="assetIssuerPublicKey">
    <div class="books" v-for="(book, i) in books" :key="i">
      <h2>{{book.base.code}} ↔ {{book.counter.code}}</h2>

      <div class="header">
        <span>Market Size</span>
        <span>Price</span>
      </div>

      <div class="book-container" ref="books">
        <ul class="bid book">
          <li class="order" v-for="(order, i) in sort(book.bids, ['price'], ['asc'])" :key="i">
            <span>{{(order.amount * order.price_r.d / order.price_r.n).toFixed(7)}}</span>
            <span>{{(order.price_r.d / order.price_r.n).toFixed(7)}}</span>
          </li>
        </ul>

        <ul class="ask book">
          <li class="order" v-for="(order, i) in sort(book.asks, ['price'], ['asc'])" :key="i">
            <span>{{parseInt(order.amount, 10).toFixed(7)}}</span>
            <span>{{(order.price_r.d / order.price_r.n).toFixed(7)}}</span>
          </li>
        </ul>
      </div>
    </div>
  </div>
</div>
</template>

<script>
import { Server, Keypair, Account, TransactionBuilder, BASE_FEE, Networks, Operation, Asset } from 'stellar-sdk'
import _ from 'lodash-es'

export default {
  data() {
    const randomPublicKey = Keypair.random().publicKey()

    return {
      assetIssuerSecret: localStorage.getItem('pathpayAssetIssuer'),
      server: new Server('https://horizon-testnet.stellar.org'),
      books: [{
        base: new Asset('USD', this.assetIssuerPublicKey || randomPublicKey),
        counter: Asset.native(),
        bids: null,
        asks: null
      }, {
        base: new Asset('EUR', this.assetIssuerPublicKey || randomPublicKey),
        counter: Asset.native(),
        bids: null,
        asks: null
      }],
    }
  },
  computed: {
    assetIssuerPublicKey() {
      if (this.assetIssuerSecret)
        return Keypair.fromSecret(this.assetIssuerSecret).publicKey()
    }
  },
  created() {
    this.updateBooks()

    this.$parent.$on('updateBooks', this.updateBooks)
    this.$parent.$on('assetIssuerSecret', (assetIssuerSecret) => this.assetIssuerSecret = assetIssuerSecret)
  },
  watch: {
    assetIssuerPublicKey() {
      this.updateBooks()
    }
  },
  methods: {
    sort(array, keys, directions) {
      return _.orderBy(array, keys, directions)
    },

    updateBooks() {
      _.each(this.books, (book, i) => {
        if (
          this.assetIssuerPublicKey 
          && book.base.issuer !== this.assetIssuerPublicKey
        ) book.base.issuer = this.assetIssuerPublicKey

        this.server
        .orderbook(book.base, book.counter)
        .call()
        .then((res) => {
          book.bids = res.bids
          book.asks = res.asks
        })
        .then(() => this.$nextTick(this.centerBooks(i)))
        .then(() => setTimeout(() => this.centerBooks(i), 100))
      })
    },
    centerBooks(i) {
      if (!_.has(this.$refs, `books[${i}]`))
        return

      const totalHeight = this.$refs.books[i].querySelector('.bid').offsetHeight + this.$refs.books[i].querySelector('.ask').offsetHeight
      const windowHeight = this.$refs.books[i].offsetHeight

      this.$refs.books[i].scrollTo(0, (totalHeight - windowHeight) / 2)
    }
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
  margin-bottom: 10px;
}

.books-container {
  display: flex;
  justify-content: space-between;
  overflow: hidden;
  width: 100%;
}
.books {
  width: calc(100% / 2 - 5px);
  border: 1px solid black;
  display: flex;
  flex-direction: column;

  h2 {
    padding: 5px;
  }
}

.book-container {
  display: flex;
  flex-direction: column;
  overflow: scroll;
}
.book {
  padding: 5px 0;

  &.bid {
    color: orangered;
    border-bottom: 1px solid black;
  }
  &.ask {
    color: green;
  }
}

.header,
.order {
  
  span {
    width: 150px;
    font-size: 14px;
  }
}
.header {
  display: flex;
  background-color: black;

  span {
    text-align: right;
    color: white;
    padding: 5px 0;
  }
}
.order {
  display: flex;
  font-size: 14px;
  text-align: right;
  padding: 5px 0;
}
</style>