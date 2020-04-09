<template>
<main>
  <section>
    <basicWallet v-for="type in types" :type="type" :key="type"/>
  </section>

  <section>
    <marketMaker />
    <orderBook />
  </section>
</main>
</template>

<script>
import basicWallet from '~/components/basic-wallet'
import marketMaker from '~/components/market-maker'
import orderBook from '~/components/order-book'

export default {
  data() {
    return {
      types: ['you', 'friend']
    }
  },
  components: {
    basicWallet,
    marketMaker,
    orderBook
  }
}
</script>

<style lang="scss">
main {
  width: 100vw;
  height: 100vh;
  display: flex;
  align-items: stretch;
}
section {
  width: 50vw;
  display: flex;
  flex-direction: column;
}

.quadrant {
  padding: 10px;
  border: 1px solid black;
  display: flex;
  flex-direction: column;
  overflow: hidden;
  align-items: flex-start;
}
</style>