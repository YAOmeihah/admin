<script setup lang="ts">
import { ref, computed, onMounted } from 'vue'
import { useI18n } from 'vue-i18n'
import { adminAPI } from '@/api/admin'
import { Card, CardContent, CardDescription, CardHeader, CardTitle } from '@/components/ui/card'
import { Button } from '@/components/ui/button'
import { Input } from '@/components/ui/input'
import { Label } from '@/components/ui/label'
import { Textarea } from '@/components/ui/textarea'
import { Select, SelectContent, SelectItem, SelectTrigger, SelectValue } from '@/components/ui/select'
import { notifySuccess, notifyError } from '@/utils/notify'
import { getImageUrl } from '@/utils/image'
import { Save, Loader2, Upload, Plus, Trash2, ArrowUp, ArrowDown } from 'lucide-vue-next'

const { t } = useI18n()

const supportedLanguages = ['zh-CN', 'zh-TW', 'en-US'] as const
type SupportedLanguage = (typeof supportedLanguages)[number]
type LocalizedText = Record<SupportedLanguage, string>

const emptyLocalized = (): LocalizedText => ({ 'zh-CN': '', 'zh-TW': '', 'en-US': '' })

interface MenuItem {
  key: string
  enabled: boolean
  order: number
  label: LocalizedText
  action: { type: string; value: string }
}

interface HelpItem {
  key: string
  enabled: boolean
  order: number
  summary: LocalizedText
  title: LocalizedText
  content: LocalizedText
  show_support_link: boolean
}

const menuActionTypes = ['builtin', 'url', 'command'] as const
const menuItemsMaxCount = 20
const helpItemsMaxCount = 12

const createMenuItem = (): MenuItem => ({
  key: '',
  enabled: true,
  order: 0,
  label: emptyLocalized(),
  action: { type: 'builtin', value: '' },
})

const createHelpItem = (): HelpItem => ({
  key: '',
  enabled: true,
  order: 0,
  summary: emptyLocalized(),
  title: emptyLocalized(),
  content: emptyLocalized(),
  show_support_link: false,
})

const currentLang = ref<SupportedLanguage>('zh-CN')
const loading = ref(false)
const saving = ref(false)
const uploadingCover = ref(false)

const coverFileInput = ref<HTMLInputElement>()

const languages = computed(() => [
  { code: 'zh-CN' as SupportedLanguage, name: t('admin.common.lang.zhCN') },
  { code: 'zh-TW' as SupportedLanguage, name: t('admin.common.lang.zhTW') },
  { code: 'en-US' as SupportedLanguage, name: t('admin.common.lang.enUS') },
])

const form = ref({
  enabled: false,
  default_locale: 'zh-CN',
  basic: {
    display_name: '',
    description: emptyLocalized(),
    support_url: '',
    cover_url: '',
  },
  welcome: {
    enabled: false,
    message: emptyLocalized(),
  },
  help: {
    enabled: true,
    title: emptyLocalized(),
    intro: emptyLocalized(),
    center_hint: emptyLocalized(),
    support_hint: emptyLocalized(),
    items: [] as HelpItem[],
  },
  menu: {
    items: [] as MenuItem[],
  },
})

const parseLocalized = (raw: unknown): LocalizedText => {
  const result = emptyLocalized()
  if (raw && typeof raw === 'object' && !Array.isArray(raw)) {
    const obj = raw as Record<string, unknown>
    for (const lang of supportedLanguages) {
      if (typeof obj[lang] === 'string') {
        result[lang] = obj[lang] as string
      }
    }
  }
  return result
}

const fetchConfig = async () => {
  loading.value = true
  try {
    const res = await adminAPI.getTelegramBotSettings()
    const data = res.data?.data as Record<string, unknown> | undefined
    if (data) {
      form.value.enabled = (data.enabled as boolean) ?? false
      form.value.default_locale = (data.default_locale as string) ?? 'zh-CN'

      const basic = data.basic as Record<string, unknown> | undefined
      if (basic) {
        form.value.basic.display_name = (basic.display_name as string) ?? ''
        form.value.basic.description = parseLocalized(basic.description)
        form.value.basic.support_url = (basic.support_url as string) ?? ''
        form.value.basic.cover_url = (basic.cover_url as string) ?? ''
      }

      const welcome = data.welcome as Record<string, unknown> | undefined
      if (welcome) {
        form.value.welcome.enabled = (welcome.enabled as boolean) ?? false
        form.value.welcome.message = parseLocalized(welcome.message)
      }

      const help = data.help as Record<string, unknown> | undefined
      if (help) {
        form.value.help.enabled = (help.enabled as boolean) ?? true
        form.value.help.title = parseLocalized(help.title)
        form.value.help.intro = parseLocalized(help.intro)
        form.value.help.center_hint = parseLocalized(help.center_hint)
        form.value.help.support_hint = parseLocalized(help.support_hint)
        if (Array.isArray(help.items)) {
          form.value.help.items = (help.items as unknown[]).map(parseHelpItem)
        }
      }

      const menu = data.menu as Record<string, unknown> | undefined
      if (menu && Array.isArray(menu.items)) {
        form.value.menu.items = (menu.items as unknown[]).map(parseMenuItem)
      }
    }
  } catch {
    notifyError(t('telegramBot.settings.loadFailed'))
  } finally {
    loading.value = false
  }
}

const handleSave = async () => {
  saving.value = true
  try {
    await adminAPI.updateTelegramBotSettings({
      ...form.value,
    })
    notifySuccess(t('telegramBot.settings.saveSuccess'))
  } catch {
    notifyError(t('telegramBot.settings.saveFailed'))
  } finally {
    saving.value = false
  }
}

const handleUploadCover = async (event: Event) => {
  const file = (event.target as HTMLInputElement).files?.[0]
  if (!file) return

  uploadingCover.value = true
  try {
    const formData = new FormData()
    formData.append('file', file)
    const res = await adminAPI.upload(formData, 'telegram')
    const url = (res.data.data as Record<string, string>)?.url || ''
    form.value.basic.cover_url = url
  } catch {
    notifyError(t('telegramBot.settings.uploadFailed'))
  } finally {
    uploadingCover.value = false
    coverFileInput.value && (coverFileInput.value.value = '')
  }
}

const addMenuItem = () => {
  if (form.value.menu.items.length >= menuItemsMaxCount) {
    notifyError(t('telegramBot.settings.menuMaxHint', { max: menuItemsMaxCount }))
    return
  }
  form.value.menu.items.push(createMenuItem())
}

const addHelpItem = () => {
  if (form.value.help.items.length >= helpItemsMaxCount) {
    notifyError(t('telegramBot.settings.helpMaxHint', { max: helpItemsMaxCount }))
    return
  }
  form.value.help.items.push(createHelpItem())
}

const removeHelpItem = (index: number) => {
  form.value.help.items.splice(index, 1)
}

const moveHelpItem = (index: number, dir: 'up' | 'down') => {
  const items = form.value.help.items
  const target = dir === 'up' ? index - 1 : index + 1
  if (target < 0 || target >= items.length) return
  ;[items[index], items[target]] = [items[target]!, items[index]!]
}

const removeMenuItem = (index: number) => {
  form.value.menu.items.splice(index, 1)
}

const moveMenuItem = (index: number, dir: 'up' | 'down') => {
  const items = form.value.menu.items
  const target = dir === 'up' ? index - 1 : index + 1
  if (target < 0 || target >= items.length) return
  ;[items[index], items[target]] = [items[target]!, items[index]!]
}

const parseMenuItem = (raw: unknown): MenuItem => {
  const item = createMenuItem()
  if (!raw || typeof raw !== 'object' || Array.isArray(raw)) return item
  const obj = raw as Record<string, unknown>
  if (typeof obj.key === 'string') item.key = obj.key
  if (typeof obj.enabled === 'boolean') item.enabled = obj.enabled
  if (typeof obj.order === 'number') item.order = obj.order
  item.label = parseLocalized(obj.label)
  if (obj.action && typeof obj.action === 'object' && !Array.isArray(obj.action)) {
    const action = obj.action as Record<string, unknown>
    if (typeof action.type === 'string') item.action.type = action.type
    if (typeof action.value === 'string') item.action.value = action.value
  }
  return item
}

const parseHelpItem = (raw: unknown): HelpItem => {
  const item = createHelpItem()
  if (!raw || typeof raw !== 'object' || Array.isArray(raw)) return item
  const obj = raw as Record<string, unknown>
  if (typeof obj.key === 'string') item.key = obj.key
  if (typeof obj.enabled === 'boolean') item.enabled = obj.enabled
  if (typeof obj.order === 'number') item.order = obj.order
  if (typeof obj.show_support_link === 'boolean') item.show_support_link = obj.show_support_link
  item.summary = parseLocalized(obj.summary)
  item.title = parseLocalized(obj.title)
  item.content = parseLocalized(obj.content)
  return item
}

onMounted(() => {
  fetchConfig()
})
</script>

<template>
  <div class="space-y-6">
    <div class="flex flex-wrap items-center justify-between gap-4">
      <div>
        <h2 class="text-2xl font-bold tracking-tight">{{ t('telegramBot.settings.title') }}</h2>
        <p class="text-muted-foreground">{{ t('telegramBot.settings.subtitle') }}</p>
      </div>
      <div class="flex flex-wrap items-center gap-3">
        <div class="flex rounded-lg border border-border bg-card p-1">
          <button
            v-for="lang in languages"
            :key="lang.code"
            class="rounded-md px-3 py-1.5 text-xs font-medium transition-colors"
            :class="currentLang === lang.code ? 'bg-secondary text-foreground' : 'text-muted-foreground hover:text-foreground'"
            @click="currentLang = lang.code"
          >
            {{ lang.name }}
          </button>
        </div>
        <Button :disabled="saving || loading" @click="handleSave">
          <Loader2 v-if="saving" class="h-4 w-4 mr-2 animate-spin" />
          <Save v-else class="h-4 w-4 mr-2" />
          {{ t('telegramBot.settings.save') }}
        </Button>
      </div>
    </div>

    <!-- 顶层设置 -->
    <Card>
      <CardHeader>
        <CardTitle>{{ t('telegramBot.settings.globalTitle') }}</CardTitle>
        <CardDescription>{{ t('telegramBot.settings.globalDesc') }}</CardDescription>
      </CardHeader>
      <CardContent class="space-y-4">
        <div class="grid grid-cols-1 md:grid-cols-2 gap-4">
          <div class="flex items-center gap-2">
            <input id="bot-enabled" v-model="form.enabled" type="checkbox" class="h-4 w-4 accent-primary" />
            <Label for="bot-enabled">{{ t('telegramBot.settings.enabled') }}</Label>
          </div>
          <div class="space-y-2">
            <Label>{{ t('telegramBot.settings.defaultLocale') }}</Label>
            <Select v-model="form.default_locale">
              <SelectTrigger>
                <SelectValue />
              </SelectTrigger>
              <SelectContent>
                <SelectItem value="zh-CN">简体中文</SelectItem>
                <SelectItem value="zh-TW">繁體中文</SelectItem>
                <SelectItem value="en-US">English</SelectItem>
              </SelectContent>
            </Select>
          </div>
        </div>
      </CardContent>
    </Card>

    <!-- 基本信息 -->
    <Card>
      <CardHeader>
        <div class="flex items-center justify-between">
          <div>
            <CardTitle>{{ t('telegramBot.settings.basicInfo') }}</CardTitle>
            <CardDescription>{{ t('telegramBot.settings.basicInfoDesc') }}</CardDescription>
          </div>
          <span class="rounded bg-muted px-2 py-1 text-xs text-muted-foreground">{{ currentLang }}</span>
        </div>
      </CardHeader>
      <CardContent class="space-y-4">
        <div class="space-y-2">
          <Label>{{ t('telegramBot.settings.displayName') }}</Label>
          <Input v-model="form.basic.display_name" :placeholder="t('telegramBot.settings.displayNamePlaceholder')" />
        </div>
        <div class="space-y-2">
          <Label>{{ t('telegramBot.settings.description') }}</Label>
          <Textarea v-model="form.basic.description[currentLang]" :placeholder="t('telegramBot.settings.descriptionPlaceholder')" rows="2" />
        </div>
        <div class="space-y-2">
          <Label>{{ t('telegramBot.settings.supportUrl') }}</Label>
          <Input v-model="form.basic.support_url" :placeholder="t('telegramBot.settings.supportUrlPlaceholder')" />
        </div>
        <div class="space-y-2">
          <Label>{{ t('telegramBot.settings.coverUrl') }}</Label>
          <div
            class="cursor-pointer rounded-lg border border-dashed border-border p-4 text-center hover:border-primary"
            @click="coverFileInput?.click()"
          >
            <input ref="coverFileInput" type="file" class="hidden" accept="image/*" @change="handleUploadCover($event)" />
            <div v-if="form.basic.cover_url" class="space-y-2">
              <img :src="getImageUrl(form.basic.cover_url)" class="mx-auto h-24 rounded-lg object-cover" />
              <div class="text-xs text-muted-foreground">{{ uploadingCover ? t('admin.common.loading') : t('telegramBot.settings.clickToReplace') }}</div>
            </div>
            <div v-else class="flex flex-col items-center gap-1 py-2">
              <Upload class="h-5 w-5 text-muted-foreground" />
              <span class="text-xs text-muted-foreground">{{ t('telegramBot.settings.clickToUpload') }}</span>
            </div>
          </div>
        </div>
      </CardContent>
    </Card>

    <!-- 欢迎设置 -->
    <Card>
      <CardHeader>
        <div class="flex items-center justify-between">
          <div>
            <CardTitle>{{ t('telegramBot.settings.welcomeTitle') }}</CardTitle>
            <CardDescription>{{ t('telegramBot.settings.welcomeDesc') }}</CardDescription>
          </div>
          <span class="rounded bg-muted px-2 py-1 text-xs text-muted-foreground">{{ currentLang }}</span>
        </div>
      </CardHeader>
      <CardContent class="space-y-4">
        <div class="flex items-center gap-2">
          <input id="welcome-enabled" v-model="form.welcome.enabled" type="checkbox" class="h-4 w-4 accent-primary" />
          <Label for="welcome-enabled">{{ t('telegramBot.settings.welcomeEnabled') }}</Label>
        </div>
        <div class="space-y-2">
          <Label>{{ t('telegramBot.settings.welcomeMessage') }}</Label>
          <Textarea v-model="form.welcome.message[currentLang]" :placeholder="t('telegramBot.settings.welcomeMessagePlaceholder')" rows="3" />
        </div>
      </CardContent>
    </Card>

    <!-- 帮助中心 -->
    <Card>
      <CardHeader>
        <div class="flex items-center justify-between">
          <div>
            <CardTitle>{{ t('telegramBot.settings.helpTitle') }}</CardTitle>
            <CardDescription>{{ t('telegramBot.settings.helpDesc') }}</CardDescription>
          </div>
          <div class="flex items-center gap-2">
            <span class="rounded bg-muted px-2 py-1 text-xs text-muted-foreground">{{ currentLang }}</span>
            <Button type="button" size="sm" variant="outline" @click="addHelpItem">
              <Plus class="h-4 w-4 mr-1" />
              {{ t('telegramBot.settings.helpAdd') }}
            </Button>
          </div>
        </div>
      </CardHeader>
      <CardContent class="space-y-4">
        <div class="flex items-center gap-2">
          <input id="help-enabled" v-model="form.help.enabled" type="checkbox" class="h-4 w-4 accent-primary" />
          <Label for="help-enabled">{{ t('telegramBot.settings.helpEnabled') }}</Label>
        </div>
        <div class="space-y-2">
          <Label>{{ t('telegramBot.settings.helpCenterTitle') }}</Label>
          <Input v-model="form.help.title[currentLang]" :placeholder="t('telegramBot.settings.helpCenterTitlePlaceholder')" />
        </div>
        <div class="space-y-2">
          <Label>{{ t('telegramBot.settings.helpIntro') }}</Label>
          <Textarea v-model="form.help.intro[currentLang]" :placeholder="t('telegramBot.settings.helpIntroPlaceholder')" rows="2" />
        </div>
        <div class="space-y-2">
          <Label>{{ t('telegramBot.settings.helpCenterHint') }}</Label>
          <Textarea v-model="form.help.center_hint[currentLang]" :placeholder="t('telegramBot.settings.helpCenterHintPlaceholder')" rows="2" />
        </div>
        <div class="space-y-2">
          <Label>{{ t('telegramBot.settings.helpSupportHint') }}</Label>
          <Textarea v-model="form.help.support_hint[currentLang]" :placeholder="t('telegramBot.settings.helpSupportHintPlaceholder')" rows="2" />
        </div>
        <div v-if="form.help.items.length === 0" class="rounded-lg border border-dashed border-border p-6 text-center text-sm text-muted-foreground">
          {{ t('telegramBot.settings.helpEmpty') }}
        </div>
        <div
          v-for="(item, index) in form.help.items"
          :key="`help-${index}`"
          class="rounded-lg border border-border p-4 space-y-3"
        >
          <div class="flex flex-wrap items-center gap-2">
            <input
              :id="`help-enabled-${index}`"
              v-model="item.enabled"
              type="checkbox"
              class="h-4 w-4 accent-primary"
            />
            <Input
              v-model="item.key"
              :placeholder="t('telegramBot.settings.helpKeyPlaceholder')"
              class="flex-1 min-w-[120px]"
            />
            <Button
              type="button" size="icon" variant="ghost"
              :disabled="index === 0"
              @click="moveHelpItem(index, 'up')"
            >
              <ArrowUp class="h-4 w-4" />
            </Button>
            <Button
              type="button" size="icon" variant="ghost"
              :disabled="index === form.help.items.length - 1"
              @click="moveHelpItem(index, 'down')"
            >
              <ArrowDown class="h-4 w-4" />
            </Button>
            <Button type="button" size="icon" variant="ghost" @click="removeHelpItem(index)">
              <Trash2 class="h-4 w-4 text-destructive" />
            </Button>
          </div>
          <div class="space-y-1">
            <Label>{{ t('telegramBot.settings.helpSummary') }}</Label>
            <Input v-model="item.summary[currentLang]" :placeholder="t('telegramBot.settings.helpSummaryPlaceholder')" />
          </div>
          <div class="space-y-1">
            <Label>{{ t('telegramBot.settings.helpItemTitle') }}</Label>
            <Input v-model="item.title[currentLang]" :placeholder="t('telegramBot.settings.helpItemTitlePlaceholder')" />
          </div>
          <div class="space-y-1">
            <Label>{{ t('telegramBot.settings.helpContent') }}</Label>
            <Textarea v-model="item.content[currentLang]" :placeholder="t('telegramBot.settings.helpContentPlaceholder')" rows="4" />
          </div>
          <div class="grid grid-cols-1 md:grid-cols-2 gap-3">
            <div class="space-y-1">
              <Label>{{ t('telegramBot.settings.helpOrder') }}</Label>
              <Input v-model.number="item.order" type="number" />
            </div>
            <div class="flex items-center gap-2 pt-6">
              <input :id="`help-support-link-${index}`" v-model="item.show_support_link" type="checkbox" class="h-4 w-4 accent-primary" />
              <Label :for="`help-support-link-${index}`">{{ t('telegramBot.settings.helpShowSupportLink') }}</Label>
            </div>
          </div>
        </div>
      </CardContent>
    </Card>

    <!-- 菜单配置 -->
    <Card>
      <CardHeader>
        <div class="flex items-center justify-between">
          <div>
            <CardTitle>{{ t('telegramBot.settings.menuTitle') }}</CardTitle>
            <CardDescription>{{ t('telegramBot.settings.menuDesc') }}</CardDescription>
          </div>
          <div class="flex items-center gap-2">
            <span class="rounded bg-muted px-2 py-1 text-xs text-muted-foreground">{{ currentLang }}</span>
            <Button type="button" size="sm" variant="outline" @click="addMenuItem">
              <Plus class="h-4 w-4 mr-1" />
              {{ t('telegramBot.settings.menuAdd') }}
            </Button>
          </div>
        </div>
      </CardHeader>
      <CardContent class="space-y-3">
        <div v-if="form.menu.items.length === 0" class="rounded-lg border border-dashed border-border p-6 text-center text-sm text-muted-foreground">
          {{ t('telegramBot.settings.menuEmpty') }}
        </div>
        <div
          v-for="(item, index) in form.menu.items"
          :key="index"
          class="rounded-lg border border-border p-4 space-y-3"
        >
          <!-- 顶栏：enabled + key + 排序 + 删除 -->
          <div class="flex flex-wrap items-center gap-2">
            <input
              :id="`menu-enabled-${index}`"
              v-model="item.enabled"
              type="checkbox"
              class="h-4 w-4 accent-primary"
            />
            <Input
              v-model="item.key"
              :placeholder="t('telegramBot.settings.menuKeyPlaceholder')"
              class="flex-1 min-w-[120px]"
            />
            <Button
              type="button" size="icon" variant="ghost"
              :disabled="index === 0"
              @click="moveMenuItem(index, 'up')"
            >
              <ArrowUp class="h-4 w-4" />
            </Button>
            <Button
              type="button" size="icon" variant="ghost"
              :disabled="index === form.menu.items.length - 1"
              @click="moveMenuItem(index, 'down')"
            >
              <ArrowDown class="h-4 w-4" />
            </Button>
            <Button type="button" size="icon" variant="ghost" @click="removeMenuItem(index)">
              <Trash2 class="h-4 w-4 text-destructive" />
            </Button>
          </div>
          <!-- label (多语言) -->
          <div class="space-y-1">
            <Label>{{ t('telegramBot.settings.menuLabel') }}</Label>
            <Input v-model="item.label[currentLang]" :placeholder="t('telegramBot.settings.menuLabelPlaceholder')" />
          </div>
          <!-- action -->
          <div class="grid grid-cols-1 md:grid-cols-3 gap-3">
            <div class="space-y-1">
              <Label>{{ t('telegramBot.settings.menuActionType') }}</Label>
              <Select v-model="item.action.type">
                <SelectTrigger>
                  <SelectValue />
                </SelectTrigger>
                <SelectContent>
                  <SelectItem v-for="at in menuActionTypes" :key="at" :value="at">
                    {{ t(`telegramBot.settings.menuActionType_${at}`) }}
                  </SelectItem>
                </SelectContent>
              </Select>
            </div>
            <div class="space-y-1">
              <Label>{{ t('telegramBot.settings.menuActionValue') }}</Label>
              <Input v-model="item.action.value" :placeholder="t('telegramBot.settings.menuActionValuePlaceholder')" />
            </div>
            <div class="space-y-1">
              <Label>{{ t('telegramBot.settings.menuOrder') }}</Label>
              <Input v-model.number="item.order" type="number" />
            </div>
          </div>
        </div>
      </CardContent>
    </Card>
  </div>
</template>
