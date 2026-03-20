<script setup lang="ts">
import { reactive, ref, watch } from 'vue'
import { useI18n } from 'vue-i18n'
import { adminAPI } from '@/api/admin'
import { Input } from '@/components/ui/input'
import { Textarea } from '@/components/ui/textarea'
import { notifyError, notifySuccess } from '@/utils/notify'

const { t } = useI18n()

const supportedLanguages = ['zh-CN', 'zh-TW', 'en-US'] as const
type SupportedLanguage = (typeof supportedLanguages)[number]

type NotificationLocalizedTemplate = { title: string; body: string }
type NotificationSceneTemplate = Record<SupportedLanguage, NotificationLocalizedTemplate>

interface NotificationData {
  default_locale: string
  dedupe_ttl_seconds: number
  channels: {
    email: {
      enabled: boolean
      recipients_text: string
    }
    telegram: {
      enabled: boolean
      recipients_text: string
    }
  }
  scenes: {
    wallet_recharge_success: boolean
    order_paid_success: boolean
    manual_fulfillment_pending: boolean
    exception_alert: boolean
  }
  templates: {
    wallet_recharge_success: NotificationSceneTemplate
    order_paid_success: NotificationSceneTemplate
    manual_fulfillment_pending: NotificationSceneTemplate
    exception_alert: NotificationSceneTemplate
  }
}

const props = defineProps<{
  data: NotificationData
  currentLang: SupportedLanguage
}>()

const emit = defineEmits<{
  saved: []
}>()

const submitting = ref(false)

const createNotificationLocalizedTemplate = (): NotificationLocalizedTemplate => ({ title: '', body: '' })
const createNotificationSceneTemplate = (): NotificationSceneTemplate => ({
  'zh-CN': createNotificationLocalizedTemplate(),
  'zh-TW': createNotificationLocalizedTemplate(),
  'en-US': createNotificationLocalizedTemplate(),
})

const deepCloneTemplate = (src: NotificationSceneTemplate): NotificationSceneTemplate => {
  const result = createNotificationSceneTemplate()
  ;(['zh-CN', 'zh-TW', 'en-US'] as const).forEach((lang) => {
    result[lang].title = src[lang]?.title || ''
    result[lang].body = src[lang]?.body || ''
  })
  return result
}

const form = reactive({
  default_locale: 'zh-CN',
  dedupe_ttl_seconds: 300,
  channels: {
    email: {
      enabled: false,
      recipients_text: '',
    },
    telegram: {
      enabled: false,
      recipients_text: '',
    },
  },
  scenes: {
    wallet_recharge_success: true,
    order_paid_success: true,
    manual_fulfillment_pending: true,
    exception_alert: true,
  },
  templates: {
    wallet_recharge_success: createNotificationSceneTemplate(),
    order_paid_success: createNotificationSceneTemplate(),
    manual_fulfillment_pending: createNotificationSceneTemplate(),
    exception_alert: createNotificationSceneTemplate(),
  },
})

const syncFromProps = () => {
  form.default_locale = props.data.default_locale
  form.dedupe_ttl_seconds = props.data.dedupe_ttl_seconds
  form.channels.email.enabled = props.data.channels.email.enabled
  form.channels.email.recipients_text = props.data.channels.email.recipients_text
  form.channels.telegram.enabled = props.data.channels.telegram.enabled
  form.channels.telegram.recipients_text = props.data.channels.telegram.recipients_text
  Object.assign(form.scenes, props.data.scenes)
  form.templates.wallet_recharge_success = deepCloneTemplate(props.data.templates.wallet_recharge_success)
  form.templates.order_paid_success = deepCloneTemplate(props.data.templates.order_paid_success)
  form.templates.manual_fulfillment_pending = deepCloneTemplate(props.data.templates.manual_fulfillment_pending)
  form.templates.exception_alert = deepCloneTemplate(props.data.templates.exception_alert)
}

syncFromProps()

watch(() => props.data, () => {
  syncFromProps()
}, { deep: true })

const splitRecipients = (raw: string) => {
  return raw
    .split(/\r?\n|,/)
    .map((item) => item.trim())
    .filter((item) => item !== '')
}

const notifyErrorIfNeeded = (err: unknown, fallback: string) => {
  const known = err as Error & { __notified?: boolean }
  if (known?.__notified) return
  notifyError(known?.message || fallback)
}

const save = async () => {
  submitting.value = true
  try {
    const payload = {
      default_locale: form.default_locale,
      dedupe_ttl_seconds: Number(form.dedupe_ttl_seconds),
      channels: {
        email: {
          enabled: form.channels.email.enabled,
          recipients: splitRecipients(form.channels.email.recipients_text),
        },
        telegram: {
          enabled: form.channels.telegram.enabled,
          recipients: splitRecipients(form.channels.telegram.recipients_text),
        },
      },
      scenes: {
        wallet_recharge_success: form.scenes.wallet_recharge_success,
        order_paid_success: form.scenes.order_paid_success,
        manual_fulfillment_pending: form.scenes.manual_fulfillment_pending,
        exception_alert: form.scenes.exception_alert,
      },
      templates: form.templates,
    }
    await adminAPI.updateNotificationCenterSettings(payload)
    notifySuccess(t('admin.settings.alerts.saveSuccess'))
    emit('saved')
  } catch (err) {
    notifyErrorIfNeeded(err, t('admin.settings.alerts.saveFailed'))
  } finally {
    submitting.value = false
  }
}

defineExpose({ save, submitting })
</script>

<template>
  <div class="space-y-6">
    <div class="rounded-xl border border-border bg-card">
      <div class="border-b border-border bg-muted/40 px-6 py-4">
        <h2 class="text-lg font-semibold">{{ t('admin.settings.notification.title') }}</h2>
        <p class="mt-1 text-xs text-muted-foreground">{{ t('admin.settings.notification.subtitle') }}</p>
      </div>

      <div class="space-y-6 p-6">
        <div class="grid grid-cols-1 gap-6 md:grid-cols-2">
          <div class="space-y-2">
            <label class="text-xs font-medium text-muted-foreground">{{ t('admin.settings.notification.defaultLocale') }}</label>
            <select v-model="form.default_locale" class="h-10 w-full rounded-md border border-input bg-background px-3 text-sm">
              <option value="zh-CN">zh-CN</option>
              <option value="zh-TW">zh-TW</option>
              <option value="en-US">en-US</option>
            </select>
          </div>
          <div class="space-y-2">
            <label class="text-xs font-medium text-muted-foreground">{{ t('admin.settings.notification.dedupeTTLSeconds') }}</label>
            <Input v-model.number="form.dedupe_ttl_seconds" type="number" min="30" max="86400" />
          </div>
        </div>

        <div class="grid grid-cols-1 gap-6 md:grid-cols-2">
          <div class="rounded-xl border border-border">
            <div class="border-b border-border bg-muted/30 px-4 py-3">
              <h3 class="text-sm font-semibold">{{ t('admin.settings.notification.channels.email.title') }}</h3>
            </div>
            <div class="space-y-3 p-4">
              <label class="flex items-center gap-2 text-sm">
                <input v-model="form.channels.email.enabled" type="checkbox" class="h-4 w-4 accent-primary" />
                {{ t('admin.settings.notification.channels.email.enabled') }}
              </label>
              <div class="space-y-2">
                <label class="text-xs font-medium text-muted-foreground">{{ t('admin.settings.notification.channels.email.recipients') }}</label>
                <Textarea
                  v-model="form.channels.email.recipients_text"
                  rows="5"
                  :placeholder="t('admin.settings.notification.channels.email.recipientsPlaceholder')"
                />
              </div>
            </div>
          </div>

          <div class="rounded-xl border border-border">
            <div class="border-b border-border bg-muted/30 px-4 py-3">
              <h3 class="text-sm font-semibold">{{ t('admin.settings.notification.channels.telegram.title') }}</h3>
            </div>
            <div class="space-y-3 p-4">
              <label class="flex items-center gap-2 text-sm">
                <input v-model="form.channels.telegram.enabled" type="checkbox" class="h-4 w-4 accent-primary" />
                {{ t('admin.settings.notification.channels.telegram.enabled') }}
              </label>
              <div class="space-y-2">
                <label class="text-xs font-medium text-muted-foreground">{{ t('admin.settings.notification.channels.telegram.recipients') }}</label>
                <Textarea
                  v-model="form.channels.telegram.recipients_text"
                  rows="5"
                  :placeholder="t('admin.settings.notification.channels.telegram.recipientsPlaceholder')"
                />
              </div>
            </div>
          </div>
        </div>

        <div class="rounded-xl border border-border bg-muted/20 p-4">
          <h3 class="text-sm font-semibold">{{ t('admin.settings.notification.scenes.title') }}</h3>
          <div class="mt-3 grid grid-cols-1 gap-3 md:grid-cols-2">
            <label class="flex items-center gap-2 text-sm">
              <input v-model="form.scenes.wallet_recharge_success" type="checkbox" class="h-4 w-4 accent-primary" />
              {{ t('admin.settings.notification.scenes.walletRechargeSuccess') }}
            </label>
            <label class="flex items-center gap-2 text-sm">
              <input v-model="form.scenes.order_paid_success" type="checkbox" class="h-4 w-4 accent-primary" />
              {{ t('admin.settings.notification.scenes.orderPaidSuccess') }}
            </label>
            <label class="flex items-center gap-2 text-sm">
              <input v-model="form.scenes.manual_fulfillment_pending" type="checkbox" class="h-4 w-4 accent-primary" />
              {{ t('admin.settings.notification.scenes.manualFulfillmentPending') }}
            </label>
            <label class="flex items-center gap-2 text-sm">
              <input v-model="form.scenes.exception_alert" type="checkbox" class="h-4 w-4 accent-primary" />
              {{ t('admin.settings.notification.scenes.exceptionAlert') }}
            </label>
          </div>
          <p class="mt-3 text-xs text-muted-foreground">{{ t('admin.settings.notification.scenes.exceptionThresholdHint') }}</p>
        </div>

        <div class="rounded-xl border border-border">
          <div class="flex flex-col gap-2 border-b border-border bg-muted/30 px-4 py-3 sm:flex-row sm:items-center sm:justify-between">
            <h3 class="text-sm font-semibold">{{ t('admin.settings.notification.templates.title') }}</h3>
            <span class="w-fit rounded bg-muted px-2 py-1 text-xs text-muted-foreground">{{ currentLang }}</span>
          </div>
          <div class="space-y-4 p-4">
            <div class="rounded-lg border border-border bg-muted/10 p-4">
              <h4 class="text-sm font-medium">{{ t('admin.settings.notification.scenes.walletRechargeSuccess') }}</h4>
              <div class="mt-3 space-y-2">
                <Input v-model="form.templates.wallet_recharge_success[currentLang].title" :placeholder="t('admin.settings.notification.templates.titlePlaceholder')" />
                <Textarea v-model="form.templates.wallet_recharge_success[currentLang].body" rows="4" :placeholder="t('admin.settings.notification.templates.bodyPlaceholder')" />
              </div>
            </div>

            <div class="rounded-lg border border-border bg-muted/10 p-4">
              <h4 class="text-sm font-medium">{{ t('admin.settings.notification.scenes.orderPaidSuccess') }}</h4>
              <div class="mt-3 space-y-2">
                <Input v-model="form.templates.order_paid_success[currentLang].title" :placeholder="t('admin.settings.notification.templates.titlePlaceholder')" />
                <Textarea v-model="form.templates.order_paid_success[currentLang].body" rows="4" :placeholder="t('admin.settings.notification.templates.bodyPlaceholder')" />
              </div>
            </div>

            <div class="rounded-lg border border-border bg-muted/10 p-4">
              <h4 class="text-sm font-medium">{{ t('admin.settings.notification.scenes.manualFulfillmentPending') }}</h4>
              <div class="mt-3 space-y-2">
                <Input v-model="form.templates.manual_fulfillment_pending[currentLang].title" :placeholder="t('admin.settings.notification.templates.titlePlaceholder')" />
                <Textarea v-model="form.templates.manual_fulfillment_pending[currentLang].body" rows="4" :placeholder="t('admin.settings.notification.templates.bodyPlaceholder')" />
              </div>
            </div>

            <div class="rounded-lg border border-border bg-muted/10 p-4">
              <h4 class="text-sm font-medium">{{ t('admin.settings.notification.scenes.exceptionAlert') }}</h4>
              <div class="mt-3 space-y-2">
                <Input v-model="form.templates.exception_alert[currentLang].title" :placeholder="t('admin.settings.notification.templates.titlePlaceholder')" />
                <Textarea v-model="form.templates.exception_alert[currentLang].body" rows="4" :placeholder="t('admin.settings.notification.templates.bodyPlaceholder')" />
              </div>
              <p class="mt-2 text-xs text-muted-foreground">{{ t('admin.settings.notification.templates.variableHint') }}</p>
            </div>
          </div>
        </div>
      </div>
    </div>
  </div>
</template>
