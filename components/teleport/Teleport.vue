<template>
  <section class="is-flex is-justify-content-center">
    <div class="teleport-container">
      <Loader v-model="isLoading" />
      <p class="is-size-3 has-text-weight-bold">
        {{ $i18n.t('teleport.page') }}
      </p>
      <div class="mb-5">
        <h1 class="has-text-weight-bold">{{ $i18n.t('teleport.from') }}</h1>
        <TeleportTabs
          :tabs="fromTabs"
          :value="fromChain"
          @select="onFromChainChange" />
      </div>

      <div class="mb-5 is-flex is-flex-direction-column">
        <div
          class="is-flex is-justify-content-space-between is-align-items-center input-wrapper">
          <input
            v-model="amount"
            class="transfer-amount is-flex is-align-items-center"
            type="number"
            placeholder="type your amount"
            :min="0" />
          <div
            class="token is-flex is-align-items-center is-justify-content-center">
            <span v-if="ksmValue" class="token-value is-size-7"
              >~{{ ksmValue }} usd</span
            >
            KSM
          </div>
        </div>

        <div
          v-if="myKsmBalance"
          class="is-size-7 is-flex is-justify-content-end is-align-items-center">
          <span class="is-flex is-align-items-center">
            <span class="mr-2">{{ $i18n.t('balance') }}:</span
            ><Money :value="myKsmBalance" hide-unit />KSM
          </span>
          <a class="max-button ml-2" @click="handleMaxClick">{{
            $i18n.t('teleport.max')
          }}</a>
        </div>
      </div>

      <div class="mb-5">
        <h1 class="has-text-weight-bold">{{ $i18n.t('teleport.to') }}</h1>
        <TeleportTabs
          :tabs="fromTabs"
          :value="toChain"
          @select="onToChainChange" />
      </div>

      <div class="mb-5">
        {{ $i18n.t('teleport.receiveValue', [amount || 0, toChain]) }}
        <a
          :href="explorerUrl"
          target="_blank"
          rel="noopener noreferrer"
          class="short-address">
          {{ shortAddress(toAddress) }}
        </a>
      </div>

      <NeoButton
        :label="
          insufficientBalance
            ? $t('teleport.insufficientBalance', [currency])
            : $t('teleport.send')
        "
        size="large"
        class="is-size-6 submit-button"
        :loading="isLoading"
        :disabled="isDisabledButton"
        variant="k-accent"
        @click.native="sendXCM" />
    </div>
  </section>
</template>

<script setup lang="ts">
import { ApiPromise, WsProvider } from '@polkadot/api'
import { web3Enable } from '@polkadot/extension-dapp'
import '@polkadot/api-augment'
import { toDefaultAddress } from '@/utils/account'
import { getAddress } from '@/utils/extension'
import { Chain, chainToPrefixMap } from '@/utils/teleport'
import { notificationTypes, showNotification } from '@/utils/notification'
import useAuth from '@/composables/useAuth'
import Loader from '@/components/shared/Loader.vue'
import * as paraspell from '@paraspell/sdk'
import { calculateExactUsdFromToken } from '@/utils/calculation'
import shortAddress from '@/utils/shortAddress'
import Money from '@/components/shared/format/Money.vue'
import { getChainEndpointByPrefix } from '@/utils/chain'
import { txCb } from '@/utils/transactionExecutor'
import TeleportTabs from './TeleportTabs.vue'
import { NeoButton } from '@kodadot1/brick'
import { getss58AddressByPrefix } from '@/utils/account'
import { getAsssetBalance } from '@/utils/api/bsx/query'
import { blockExplorerOf } from '@/utils/config/chain.config'
import { simpleDivision, subscribeBalance } from '@/utils/balance'
import { useFiatStore } from '@/stores/fiat'

const getKusamaApi = async () =>
  await ApiPromise.create({
    provider: new WsProvider(getChainEndpointByPrefix('kusama') as string),
  })
const getBasiliskApi = async () =>
  await ApiPromise.create({
    provider: new WsProvider(getChainEndpointByPrefix('basilisk') as string),
  })

const { accountId } = useAuth()
const { assets } = usePrefix()
const { $i18n } = useNuxtApp()
const fiatStore = useFiatStore()

const chains = ref([Chain.KUSAMA, Chain.BASILISK])
const fromChain = ref(Chain.KUSAMA) //Selected origin parachain
const toChain = ref(Chain.BASILISK) //Selected destination parachain
const amount = ref() //Required amount to be transfered is stored here
const ksmBalanceOnBasilisk = ref()
const ksmBalanceOnKusama = ref()
const currency = ref('KSM') //Selected currency is stored here
const isLoading = ref(false)
const unsubscribeKusamaBalance = ref()
const resetStatus = () => {
  amount.value = undefined
  isLoading.value = false
}

const fromTabs = [
  {
    label: Chain.KUSAMA,
    value: Chain.KUSAMA,
  },
  {
    label: Chain.BASILISK,
    value: Chain.BASILISK,
  },
]

const myKsmBalance = computed(() => {
  return fromChain.value === Chain.KUSAMA
    ? ksmBalanceOnKusama.value
    : ksmBalanceOnBasilisk.value
})
const explorerUrl = computed(() => {
  return `${blockExplorerOf(chainToPrefixMap[toChain.value])}account/${
    toAddress.value
  }`
})
const getAnotherOption = (val) => {
  return chains.value.find((chain) => chain !== val) || Chain.KUSAMA
}
const onFromChainChange = (val) => {
  fromChain.value = val
  toChain.value = getAnotherOption(val)
}

const onToChainChange = (val) => {
  toChain.value = val
  fromChain.value = getAnotherOption(val)
}
const getAddressByChain = (chain) => {
  return getss58AddressByPrefix(accountId.value, chainToPrefixMap[chain])
}
const fromAddress = computed(() => getAddressByChain(fromChain.value))
const toAddress = computed(() => getAddressByChain(toChain.value))

const ksmValue = computed(() =>
  calculateExactUsdFromToken(
    amount.value,
    fiatStore.getCurrentKSMValue as number
  )
)

const fetchBasiliskBalance = async () => {
  const api = await getBasiliskApi()
  getAsssetBalance(api, getAddressByChain(fromChain.value), '1').then(
    (data) => {
      ksmBalanceOnBasilisk.value = data
    }
  )
}

const fetchKusamaBalance = async () => {
  const api = await getKusamaApi()
  unsubscribeKusamaBalance.value = await subscribeBalance(
    api,
    getAddressByChain(Chain.KUSAMA),
    (...data) => {
      ksmBalanceOnKusama.value = data[0]
    }
  )
}

const insufficientBalance = computed(
  () => Number(amount.value) > myKsmBalanceWithoutDivision.value
)

const myKsmBalanceWithoutDivision = computed(() =>
  simpleDivision(myKsmBalance.value, 12)
)

const isDisabledButton = computed(() => {
  return !amount.value || amount.value <= 0 || insufficientBalance.value
})

const ksmTokenDecimals = computed(() => assets(5).decimals)

const handleMaxClick = () => {
  amount.value =
    Math.floor((myKsmBalanceWithoutDivision.value || 0) * 10 ** 4) / 10 ** 4
}
onMounted(() => {
  fetchBasiliskBalance()
  fetchKusamaBalance()
})

onBeforeUnmount(() => {
  unsubscribeKusamaBalance.value && unsubscribeKusamaBalance.value()
})

//Used to create XCM transfer
const sendXCM = async () => {
  if (!amount.value || amount.value < 0) {
    return
  }
  await web3Enable('Kodadot')
  let isFirstStatus = true
  isLoading.value = true
  const amountValue = 10 ** ksmTokenDecimals.value * amount.value
  const transactionHandler = txCb(
    (blockHash) => {
      showNotification(
        `Transaction finalized at blockHash ${blockHash}`,
        notificationTypes.success
      )
      resetStatus()
    },
    (dispatchError) => {
      showNotification(dispatchError.toString(), notificationTypes.warn)
      isLoading.value = false
    },
    ({ txHash }) => {
      if (isFirstStatus) {
        showNotification(
          `Transaction hash is ${txHash.toHex()}`,
          notificationTypes.info
        )
        isFirstStatus = false
      }
    }
  )

  const errorHandler = () => {
    showNotification('Cancelled', notificationTypes.warn)
    isLoading.value = false
  }
  const injector = await getAddress(toDefaultAddress(fromAddress.value))
  if (fromChain.value === Chain.KUSAMA) {
    const apiKusama = await getKusamaApi()
    const promise = paraspell.xcmPallet.transferRelayToPara(
      apiKusama,
      Chain.BASILISK,
      amountValue,
      toAddress.value
    )

    promise
      .signAndSend(
        fromAddress.value,
        { signer: injector.signer },
        transactionHandler
      )
      .catch(errorHandler)
  } else if (fromChain.value === Chain.BASILISK) {
    const apiBasilisk = await getBasiliskApi()

    const promise = paraspell.xcmPallet.send(
      apiBasilisk,
      Chain.BASILISK,
      currency.value,
      1,
      amountValue,
      toAddress.value
    )

    promise
      .signAndSend(
        fromAddress.value,
        { signer: injector.signer },
        transactionHandler
      )
      .catch(errorHandler)
  }
}
</script>
<style lang="scss" scoped>
@import '@/styles/abstracts/variables.scss';

.teleport-container {
  max-width: 30rem;

  .submit-button {
    width: 100%;
    height: 3.5rem;
  }

  .short-address,
  .max-button {
    @include ktheme() {
      color: theme('k-blue');
    }
  }
}

.input-wrapper {
  @include ktheme() {
    border: 1px solid theme('border-color');
  }

  .token {
    width: 16rem;
    position: relative;

    .token-value {
      position: absolute;
      left: 0;
      transform: translateX(-110%);

      @include ktheme() {
        color: theme('k-grey');
      }
    }
  }

  .transfer-amount {
    border: none;

    @include ktheme() {
      color: theme('text-color');
      border-right: 1px solid theme('border-color');
      background: theme('background-color');
    }

    background: transparent;
    padding: 0 0.5rem;
    height: 54px;
    outline: none;
    width: 100%;

    &::-webkit-outer-spin-button,
    &::-webkit-inner-spin-button {
      -webkit-appearance: none !important;
    }
  }
}
</style>
