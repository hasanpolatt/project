<template>
  <div>
    <NeoCollapse :disabled="disabled">
      <div class="is-flex">
        {{ $t('massmint.uploadDesc') }}
        <NeoIcon
          v-if="showCheckmark"
          icon="circle-check"
          size="small"
          variant="success"
          class="ml-3" />
        <div v-else class="icon-placeholder ml-3" />
      </div>
      <template #content>
        <DragDrop
          accept=".txt,.json,.csv"
          :loading="loading"
          @fileSelected="onFileSelected">
          <template #title>
            <span class="mb-4 has-text-left">{{
              $t('massmint.uploadDescriptionTip')
            }}</span>
            <span class="has-text-left"
              ><b>{{ $t('massmint.supportedFormats') }}</b>
              {{ acceptedMediaFormatsString }}</span
            >
          </template>
        </DragDrop>
      </template>
    </NeoCollapse>
  </div>
</template>

<script setup lang="ts">
import { NeoCollapse, NeoIcon } from '@kodadot1/brick'
import DragDrop from '@/components/shared/DragDrop.vue'
const { $consola } = useNuxtApp()

withDefaults(
  defineProps<{
    disabled?: boolean
  }>(),
  {
    disabled: false,
  }
)

const loading = ref(false)
const showCheckmark = ref(false)

const acceptedMediaFormatsString = '.TXT, .CSV, .JSON'

const emit = defineEmits(['fileLoaded'])

const onFileSelected = (file) => {
  showCheckmark.value = false

  // replace true with check for valid file type
  if (true) {
    loading.value = true
    const reader = new FileReader()
    reader.readAsDataURL(file)
    reader.onload = async () => {
      // const filePath = reader.result as string

      // process the file

      //dummy load time

      setTimeout(() => {
        loading.value = false
        showCheckmark.value = true
        emit('fileLoaded')
      }, 2000)
    }

    reader.onerror = () => {
      $consola.error('Error reading zip file.')
    }
  } else {
    $consola.error('Invalid file type.')
  }
}
</script>
<style lang="scss" scoped>
.icon-placeholder {
  width: 1rem;
}
</style>
