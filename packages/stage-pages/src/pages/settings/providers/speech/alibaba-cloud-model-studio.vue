<script setup lang="ts">
import type { SpeechProviderWithExtraOptions } from '@xsai-ext/providers/utils'
import type { UnAlibabaCloudOptions } from 'unspeech'

import {
  SpeechPlayground,
  SpeechProviderSettings,
} from '@proj-airi/stage-ui/components'
import { useSpeechStore } from '@proj-airi/stage-ui/stores/modules/speech'
import { useProvidersStore } from '@proj-airi/stage-ui/stores/providers'
import { FieldRange } from '@proj-airi/ui'
import { storeToRefs } from 'pinia'
import { computed, onMounted, ref, watch } from 'vue'
import { useI18n } from 'vue-i18n'

const providerId = 'alibaba-cloud-model-studio'
const defaultModel = 'cosyvoice-v1'

const defaultVoiceSettings = {
  speed: 1.0,
}

const pitch = ref<number>(0)
const speed = ref<number>(1.0)
const volume = ref<number>(0)

const speechStore = useSpeechStore()
const providersStore = useProvidersStore()
const { providers } = storeToRefs(providersStore)
const { t } = useI18n()

const apiKeyConfigured = computed(() => !!providers.value[providerId]?.apiKey)

const availableVoices = computed(() => {
  return speechStore.availableVoices[providerId] || []
})

async function handleGenerateSpeech(input: string, voiceId: string, _useSSML: boolean) {
  const provider = await providersStore.getProviderInstance(providerId) as SpeechProviderWithExtraOptions<string, UnAlibabaCloudOptions>
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
      ...defaultVoiceSettings,
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

watch(pitch, async () => {
  const providerConfig = providersStore.getProviderConfig(providerId)
  providerConfig.pitch = pitch.value
})

watch(speed, async () => {
  const providerConfig = providersStore.getProviderConfig(providerId)
  providerConfig.speed = speed.value
})

watch(volume, async () => {
  const providerConfig = providersStore.getProviderConfig(providerId)
  providerConfig.volume = volume.value
})

watch(providers, async () => {
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
    :additional-settings="defaultVoiceSettings"
  >
    <template #voice-settings>
      <div flex="~ col gap-4">
        <FieldRange
          v-model="pitch"
          :label="t('settings.pages.providers.provider.common.fields.field.pitch.label')"
          :description="t('settings.pages.providers.provider.common.fields.field.pitch.description')"
          :min="-100"
          :max="100" :step="1" :format-value="value => `${value}%`"
        />

        <FieldRange
          v-model="speed"
          :label="t('settings.pages.providers.provider.common.fields.field.speed.label')"
          :description="t('settings.pages.providers.provider.common.fields.field.speed.description')"
          :min="0.5"
          :max="2.0" :step="0.01"
        />

        <FieldRange
          v-model="volume"
          :label="t('settings.pages.providers.provider.common.fields.field.volume.label')"
          :description="t('settings.pages.providers.provider.common.fields.field.volume.description')"
          :min="-100"
          :max="100" :step="1" :format-value="value => `${value}%`"
        />
      </div>
    </template>

    <template #playground>
      <SpeechPlayground
        :available-voices="availableVoices"
        :generate-speech="handleGenerateSpeech"
        :api-key-configured="apiKeyConfigured"
        default-text="你好！这是一段阿里云语音合成测试。"
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
