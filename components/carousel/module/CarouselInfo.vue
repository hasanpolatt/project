<template>
  <div class="carousel-info is-flex is-flex-direction-column">
    <nuxt-link
      :to="urlOf({ id: item.id, url, chain: item.chain })"
      :title="item.name"
      :class="[
        'has-text-weight-bold carousel-info-name',
        { 'carousel-info-collection': isCollection },
      ]">
      <span class="is-ellipsis">{{ item.name || '--' }}</span>
      <span v-if="isCollection" class="carousel-info-arrow">----></span>
    </nuxt-link>
    <CollectionDetailsPopover v-if="item?.collectionId" :nft="item">
      <template #trigger>
        <nuxt-link
          v-if="!isCollection && item.collectionId"
          :to="
            urlOf({
              id: item.collectionId,
              url: 'collection',
              chain: item.chain,
            })
          "
          class="is-size-7 carousel-info-collection-name">
          {{ item.collectionName || '--' }}
        </nuxt-link>
      </template>
    </CollectionDetailsPopover>

    <div
      v-if="!isCollection"
      class="carousel-meta is-flex"
      :class="[
        showPrice
          ? 'is-justify-content-space-between'
          : 'is-justify-content-end',
      ]">
      <CommonTokenMoney
        v-if="showPrice"
        :custom-token-id="getTokenId(item.chain)"
        :value="item.price" />
      <p class="is-size-7 chain-name">{{ chainName }}</p>
    </div>
  </div>
</template>

<script lang="ts" setup>
import CommonTokenMoney from '@/components/shared/CommonTokenMoney.vue'
import type { CarouselNFT } from '@/components/base/types'
import { getKusamaAssetId } from '@/utils/api/bsx/query'

import { useCarouselUrl } from '../utils/useCarousel'
import { NAMES } from '@/libs/static/src/names'

const CollectionDetailsPopover = defineAsyncComponent(
  () =>
    import('@/components/collectionDetailsPopover/CollectionDetailsPopover.vue')
)

const props = defineProps<{
  item: CarouselNFT
}>()
const { urlPrefix } = usePrefix()
const { urlOf } = useCarouselUrl()
const url = inject('itemUrl', 'gallery') as string
const isCollection = inject<boolean>('isCollection', false)
const chainName = computed(() => {
  return NAMES[props.item.chain || urlPrefix.value]
})

const showPrice = computed((): boolean => {
  return Number(props.item.price) > 0 && !isCollection
})
const getTokenId = (chain?: string) => {
  return chain && getKusamaAssetId(chain)
}
</script>
