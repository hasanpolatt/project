<template>
  <section>
    <h1>Partnership</h1>
    <br />
    We will have some partnership here announced, soon.<br />
    <a href="https://discord.gg/u6ymnbz4PR"
      >If you want to know it sooner, join our Discord</a
    >
  </section>
</template>

<script lang="ts">
import { Component, Vue } from 'nuxt-property-decorator'

@Component<Partnership>({
  head() {
    const title = 'Cares about environmental impact'
    const metaData = {
      title,
      type: 'article',
      url: `${this.$config.public.baseUrl}/partnership`,
      image: `${this.$config.public.baseUrl}/k_card.png`,
    }
    return {
      title,
      meta: [...this.$seoMeta(metaData)],
    }
  },
})
export default class Partnership extends Vue {}
</script>
