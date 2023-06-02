<template>
  <div>
    <nuxt-link :to="`/${urlPrefix}/explore/items`" class="menu-item mr-2">
      {{ $t('items') }}
    </nuxt-link>
    <nuxt-link
      :to="`/${urlPrefix}/explore/collectibles`"
      class="menu-item mr-2">
      {{ $t('collections') }}
    </nuxt-link>
    <hr aria-role="menuitem" class="dropdown-divider my-4" />
    <div>
    </div>
  </div>
</template>

<script lang="ts" setup>
const { $router, $store } = useNuxtApp()
const { urlPrefix } = usePrefix()
const { availableChains } = useChain()


const setSelectedChain = (value) => {
  $store.dispatch('setUrlPrefix', value)
  $router.push({ path: `/${value}/explore` })
}
</script>
