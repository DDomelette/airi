<script setup lang="ts">
import type { SpeechProviderWithExtraOptions } from '@xsai-ext/providers/utils'
import type { UnVolcengineOptions } from 'unspeech'

import {
  SpeechPlayground,
  SpeechProviderSettings,
} from '@proj-airi/stage-ui/components'
import { useSpeechStore } from '@proj-airi/stage-ui/stores/modules/speech'
import { useProvidersStore } from '@proj-airi/stage-ui/stores/providers'
import { FieldInput, FieldRange } from '@proj-airi/ui'
import { storeToRefs } from 'pinia'
import { computed, onMounted, ref, watch } from 'vue'
import { useI18n } from 'vue-i18n'

const providerId = 'volcengine'
const defaultModel = 'v1'

const speed = ref<number>(1.0)

const speechStore = useSpeechStore()
const providersStore = useProvidersStore()
const { providers } = storeToRefs(providersStore)
const { t } = useI18n()

const appId = computed({
  get: () => (providers.value[providerId]?.app as any)?.appId as string | undefined || '',
  set: (value) => {
    if (!providers.value[providerId])
      providers.value[providerId] = {}

    providers.value[providerId].app = {
      appId: value,
    }
  },
})

const apiKeyConfigured = computed(() => !!providers.value[providerId]?.apiKey)

const availableVoices = computed(() => {
  return speechStore.availableVoices[providerId] || []
})

async function handleGenerateSpeech(input: string, voiceId: string, _useSSML: boolean) {
  const provider = await providersStore.getProviderInstance(providerId) as SpeechProviderWithExtraOptions<string, UnVolcengineOptions>
  if (!provider) {
    throw new Error('Failed to initialize speech provider')
  }

  const providerConfig = providersStore.getProviderConfig(providerId)

  const model = providerConfig.model as string | undefined || defaultModel

  return await speechStore.speech(
    provider,
    model,
    input,
    voiceId,
    {
      ...providerConfig,
    },
  )
}

onMounted(async () => {
  const providerConfig = providersStore.getProviderConfig(providerId)
  const providerMetadata = providersStore.getProviderMetadata(providerId)
  if (await providerMetadata.validators.validateProviderConfig(providerConfig)) {
    await speechStore.loadVoicesForProvider(providerId)
  }
  else {
    console.error('Failed to validate provider config', providerConfig)
  }
})

watch(speed, async () => {
  const providerConfig = providersStore.getProviderConfig(providerId)
  providerConfig.speed = speed.value
})

watch([providers, appId], async () => {
  const providerConfig = providersStore.getProviderConfig(providerId)
  const providerMetadata = providersStore.getProviderMetadata(providerId)
  if (await providerMetadata.validators.validateProviderConfig(providerConfig)) {
    await speechStore.loadVoicesForProvider(providerId)
  }
  else {
    console.error('Failed to validate provider config', providerConfig)
  }
}, {
  immediate: true,
})
</script>

<template>
  <SpeechProviderSettings
    :provider-id="providerId"
    :default-model="defaultModel"
  >
    <template #basic-settings>
      <div flex="~ col gap-4">
        <FieldInput
          v-model="appId"
          :label="t('settings.pages.providers.provider.volcengine.fields.field.appId.label')"
          :description="t('settings.pages.providers.provider.volcengine.fields.field.appId.description')"
          required
        />
      </div>
    </template>

    <template #voice-settings>
      <FieldRange
        v-model="speed"
        :label="t('settings.pages.providers.provider.common.fields.field.speed.label')"
        :description="t('settings.pages.providers.provider.common.fields.field.speed.description')"
        :min="0.5"
        :max="2.0" :step="0.01"
      />
    </template>

    <template #playground>
      <SpeechPlayground
        :available-voices="availableVoices"
        :generate-speech="handleGenerateSpeech"
        :api-key-configured="apiKeyConfigured"
        default-text="你好！这是一段火山引擎语音合成测试。"
      />
    </template>
  </SpeechProviderSettings>
</template>

<route lang="yaml">
  meta:
    layout: settings
    stageTransition:
      name: slide
  </route>
