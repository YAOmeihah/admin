<script setup lang="ts">
import { computed, onMounted, ref } from 'vue'
import { useI18n } from 'vue-i18n'
import { useDebounceFn } from '@vueuse/core'
import { adminAPI } from '@/api/admin'
import type { AdminProduct, AdminProductSKU, AdminCardSecret, AdminCardSecretBatch } from '@/api/types'
import { Upload, PackagePlus } from 'lucide-vue-next'
import IdCell from '@/components/IdCell.vue'
import { Button } from '@/components/ui/button'
import { Input } from '@/components/ui/input'
import { Table, TableBody, TableCell, TableHead, TableHeader, TableRow } from '@/components/ui/table'
import TableSkeleton from '@/components/TableSkeleton.vue'
import { Select, SelectContent, SelectItem, SelectTrigger, SelectValue } from '@/components/ui/select'
import { formatDate, getLocalizedText } from '@/utils/format'
import { confirmAction } from '@/utils/confirm'
import CardSecretEditModal from './components/CardSecretEditModal.vue'
import CardSecretBatchCreateModal from './components/CardSecretBatchCreateModal.vue'

const { t } = useI18n()
const adminPath = import.meta.env.VITE_ADMIN_PATH || ''

const productKeyword = ref('')
const productOptions = ref<AdminProduct[]>([])
const productOptionsLoading = ref(false)
const selectedProductValue = ref('__all__')
const productInfo = ref<AdminProduct | null>(null)
const skuFilterValue = ref('__all__')

const stats = ref<Record<string, unknown> | null>(null)
const statsLoading = ref(false)

const batches = ref<AdminCardSecretBatch[]>([])
const batchesLoading = ref(false)
const batchPagination = ref({
  page: 1,
  page_size: 20,
  total: 0,
  total_page: 1,
})
const batchJumpPage = ref('')

const cardSecrets = ref<AdminCardSecret[]>([])
const cardSecretsLoading = ref(false)
const cardSecretStatus = ref('__all__')
const cardSecretPagination = ref({
  page: 1,
  page_size: 20,
  total: 0,
  total_page: 1,
})
const cardSecretJumpPage = ref('')

const selectedSecretIds = ref<number[]>([])
const batchStatusTarget = ref<'available' | 'reserved' | 'used'>('available')
const batchActionLoading = ref(false)
const batchActionError = ref('')
const batchActionSuccess = ref('')

const showEditModal = ref(false)
const editingCardSecret = ref<AdminCardSecret | null>(null)
const importSectionRef = ref<HTMLElement | null>(null)

const scrollToImport = () => {
  importSectionRef.value?.scrollIntoView({ behavior: 'smooth', block: 'start' })
}

const normalizeFilterValue = (value: string) => (value === '__all__' ? '' : value)

const parseProductId = () => {
  if (selectedProductValue.value === '__all__') return null
  const parsed = Number(selectedProductValue.value)
  if (!Number.isFinite(parsed) || parsed <= 0) return null
  return Math.floor(parsed)
}

const parseSkuId = () => {
  if (skuFilterValue.value === '__all__') return 0
  const parsed = Number(skuFilterValue.value)
  if (!Number.isFinite(parsed) || parsed <= 0) return 0
  return Math.floor(parsed)
}

const formatSkuSpecValues = (specValues: Record<string, string> | null | undefined) => {
  if (!specValues || typeof specValues !== 'object' || Array.isArray(specValues)) return ''
  return Object.entries(specValues as Record<string, string>)
    .map(([key, value]) => {
      const keyText = String(key || '').trim()
      const valueText = Array.isArray(value)
        ? value.map((entry) => String(entry || '').trim()).filter(Boolean).join(', ')
        : String(value ?? '').trim()
      if (!valueText) return ''
      if (!keyText) return valueText
      return `${keyText}:${valueText}`
    })
    .filter(Boolean)
    .join(' / ')
}

const buildSkuLabel = (sku: AdminProductSKU | null | undefined) => {
  const skuCode = String(sku?.sku_code || '').trim()
  const specText = formatSkuSpecValues(sku?.spec_values)
  if (skuCode && specText) return `${skuCode} · ${specText}`
  if (skuCode) return skuCode
  if (specText) return specText
  if (sku?.id) return `#${sku.id}`
  return '-'
}

const buildProductLabel = (product: AdminProduct | null | undefined) => {
  const id = Number(product?.id || 0)
  const name = getLocalizedText(product?.title || {})
  if (id > 0 && name) return `#${id} ${name}`
  if (id > 0) return `#${id}`
  return name || '-'
}

const currentProductId = computed(() => parseProductId())
const currentSkuId = computed(() => parseSkuId())

const productHint = computed(() => {
  if (!currentProductId.value) return t('admin.cardSecrets.productHintEmpty')
  return t('admin.cardSecrets.productHintCurrent', { id: currentProductId.value })
})

const productInfoName = computed(() => {
  if (productInfo.value) return getLocalizedText(productInfo.value.title)
  const option = productOptions.value.find((item: AdminProduct) => Number(item?.id || 0) === currentProductId.value)
  if (!option) return ''
  return getLocalizedText(option.title || {})
})

const availableSkus = computed(() => {
  const rows = Array.isArray(productInfo.value?.skus) ? productInfo.value.skus : []
  return rows
    .filter((sku: AdminProductSKU) => Boolean(sku?.is_active))
    .map((sku: AdminProductSKU) => ({
      ...sku,
      id: Number(sku.id),
      label: buildSkuLabel(sku),
    }))
    .filter((sku: AdminProductSKU & { label: string }) => Number.isFinite(sku.id) && sku.id > 0)
})

const skuFilterDisabled = computed(() => !currentProductId.value || availableSkus.value.length === 0)

const allCurrentPageSelected = computed(() => {
  if (cardSecrets.value.length === 0) return false
  return cardSecrets.value.every((item: AdminCardSecret) => selectedSecretIds.value.includes(Number(item?.id || 0)))
})

const hasSelectedSecrets = computed(() => selectedSecretIds.value.length > 0)

const syncSkuSelection = () => {
  if (!currentProductId.value || availableSkus.value.length === 0) {
    skuFilterValue.value = '__all__'
    return
  }
  if (availableSkus.value.length === 1) {
    skuFilterValue.value = String(availableSkus.value[0]!.id)
    return
  }
  const matched = availableSkus.value.some((sku) => sku.id === currentSkuId.value)
  if (!matched) {
    skuFilterValue.value = '__all__'
  }
}

const resolveSkuLabelById = (skuID: number) => {
  if (!skuID) return '-'
  const target = availableSkus.value.find((sku) => sku.id === skuID)
  if (!target) return `#${skuID}`
  return target.label
}

const resolveProductName = (productId: number) => {
  if (!productId) return ''
  if (productInfo.value && Number(productInfo.value.id || 0) === productId) {
    return getLocalizedText(productInfo.value.title)
  }
  const option = productOptions.value.find((item: AdminProduct) => Number(item?.id || 0) === productId)
  if (!option) return ''
  return getLocalizedText(option.title || {})
}

const productLink = (productId: number) => `${adminPath}/products?product_id=${productId}`
const orderLink = (orderId: number) => `${adminPath}/orders?order_id=${orderId}`

const clearBatchActionMessages = () => {
  batchActionError.value = ''
  batchActionSuccess.value = ''
}

const normalizeSelectedSecretIDs = () => {
  return Array.from(
    new Set(
      selectedSecretIds.value
        .map((item) => Number(item))
        .filter((item) => Number.isFinite(item) && item > 0)
        .map((item) => Math.floor(item))
    )
  )
}

const toggleSelectAllSecrets = () => {
  if (allCurrentPageSelected.value) {
    selectedSecretIds.value = []
    return
  }
  selectedSecretIds.value = cardSecrets.value
    .map((item: AdminCardSecret) => Number(item?.id || 0))
    .filter((item: number) => Number.isFinite(item) && item > 0)
}

const loadProductOptions = async () => {
  productOptionsLoading.value = true
  try {
    const keyword = String(productKeyword.value || '').trim()
    const rows: AdminProduct[] = []
    let page = 1
    let totalPage = 1
    do {
      const response = await adminAPI.getProducts({
        page,
        page_size: 100,
        search: keyword || undefined,
        fulfillment_type: 'auto',
      })
      const list = Array.isArray(response.data.data) ? response.data.data : []
      rows.push(...list.filter((item: AdminProduct) => String(item?.fulfillment_type || '').trim() === 'auto'))
      totalPage = Number(response.data?.pagination?.total_page || 1)
      page += 1
    } while (page <= totalPage && page <= 20)

    const dedup = new Map<number, AdminProduct>()
    rows.forEach((item: AdminProduct) => {
      const id = Number(item?.id || 0)
      if (!Number.isFinite(id) || id <= 0) return
      if (!dedup.has(id)) dedup.set(id, item)
    })

    const options = Array.from(dedup.values())
    if (
      currentProductId.value &&
      !options.some((item: AdminProduct) => Number(item?.id || 0) === currentProductId.value)
    ) {
      if (productInfo.value && Number(productInfo.value.id || 0) === currentProductId.value) {
        options.unshift(productInfo.value)
      } else {
        options.unshift({
          id: currentProductId.value,
          title: {
            'zh-CN': `#${currentProductId.value}`,
            'zh-TW': `#${currentProductId.value}`,
            'en-US': `#${currentProductId.value}`,
          },
          fulfillment_type: 'auto',
        } as unknown as AdminProduct)
      }
    }

    productOptions.value = options
  } catch {
    productOptions.value = []
  } finally {
    productOptionsLoading.value = false
  }
}

const loadProductInfo = async () => {
  const productId = parseProductId()
  if (!productId) {
    productInfo.value = null
    skuFilterValue.value = '__all__'
    return
  }
  try {
    const response = await adminAPI.getProduct(productId)
    productInfo.value = response.data.data
    if (!productOptions.value.some((item: AdminProduct) => Number(item?.id || 0) === productId)) {
      productOptions.value.unshift(response.data.data)
    }
    syncSkuSelection()
  } catch {
    productInfo.value = null
    skuFilterValue.value = '__all__'
  }
}

const refreshStats = async () => {
  const productId = parseProductId()
  if (!productId) {
    stats.value = null
    return
  }
  statsLoading.value = true
  try {
    const response = await adminAPI.getCardSecretStats({
      product_id: productId,
      sku_id: currentSkuId.value || undefined,
    })
    stats.value = response.data.data
  } catch {
    stats.value = null
  } finally {
    statsLoading.value = false
  }
}

const fetchBatches = async (page = 1) => {
  const productId = parseProductId()
  if (!productId) {
    batches.value = []
    batchPagination.value = {
      page: 1,
      page_size: 20,
      total: 0,
      total_page: 1,
    }
    return
  }
  batchesLoading.value = true
  try {
    const response = await adminAPI.getCardSecretBatches({
      product_id: productId,
      sku_id: currentSkuId.value || undefined,
      page,
      page_size: batchPagination.value.page_size,
    })
    batches.value = response.data.data || []
    batchPagination.value = response.data.pagination || batchPagination.value
  } catch {
    batches.value = []
  } finally {
    batchesLoading.value = false
  }
}

const fetchCardSecrets = async (page = 1) => {
  const productId = parseProductId()
  cardSecretsLoading.value = true
  try {
    const params: Record<string, unknown> = {
      status: normalizeFilterValue(cardSecretStatus.value) || undefined,
      page,
      page_size: cardSecretPagination.value.page_size,
    }
    if (productId) {
      params.product_id = productId
      params.sku_id = currentSkuId.value || undefined
    }
    const response = await adminAPI.getCardSecrets(params)
    cardSecrets.value = response.data.data || []
    cardSecretPagination.value = response.data.pagination || cardSecretPagination.value
    selectedSecretIds.value = []
  } catch {
    cardSecrets.value = []
    selectedSecretIds.value = []
  } finally {
    cardSecretsLoading.value = false
  }
}

const refreshAll = async () => {
  const productId = parseProductId()
  clearBatchActionMessages()
  if (productId) {
    await loadProductInfo()
    await Promise.all([refreshStats(), fetchBatches(1), fetchCardSecrets(1)])
    return
  }

  productInfo.value = null
  stats.value = null
  batches.value = []
  batchPagination.value = {
    page: 1,
    page_size: 20,
    total: 0,
    total_page: 1,
  }
  skuFilterValue.value = '__all__'
  await fetchCardSecrets(1)
}

const refreshAfterBatchMutations = async () => {
  await fetchCardSecrets(cardSecretPagination.value.page)
  if (currentProductId.value) {
    await Promise.all([refreshStats(), fetchBatches(batchPagination.value.page)])
  }
}

const handleSearchProducts = async () => {
  await loadProductOptions()
}
const debouncedSearchProducts = useDebounceFn(handleSearchProducts, 300)

const handleProductSelectionChange = async () => {
  skuFilterValue.value = '__all__'
  await refreshAll()
}

const handleSkuSelectionChange = async () => {
  await refreshAll()
}

const changeBatchPage = (page: number) => {
  if (page < 1 || page > batchPagination.value.total_page) return
  fetchBatches(page)
}

const refreshBatches = () => {
  fetchBatches(batchPagination.value.page)
}

const refreshCardSecrets = () => {
  clearBatchActionMessages()
  fetchCardSecrets(1)
}

const changeSecretPage = (page: number) => {
  if (page < 1 || page > cardSecretPagination.value.total_page) return
  fetchCardSecrets(page)
}

const jumpBatchPage = () => {
  if (!batchJumpPage.value) return
  const raw = Number(batchJumpPage.value)
  if (Number.isNaN(raw)) return
  const target = Math.min(Math.max(Math.floor(raw), 1), batchPagination.value.total_page)
  if (target === batchPagination.value.page) return
  changeBatchPage(target)
}

const jumpCardSecretPage = () => {
  if (!cardSecretJumpPage.value) return
  const raw = Number(cardSecretJumpPage.value)
  if (Number.isNaN(raw)) return
  const target = Math.min(Math.max(Math.floor(raw), 1), cardSecretPagination.value.total_page)
  if (target === cardSecretPagination.value.page) return
  changeSecretPage(target)
}

const applyBatchStatus = async () => {
  clearBatchActionMessages()
  const ids = normalizeSelectedSecretIDs()
  if (ids.length === 0) {
    batchActionError.value = t('admin.cardSecrets.errors.selectRequired')
    return
  }

  const statusLabel = cardSecretStatusLabel(batchStatusTarget.value)
  const confirmed = await confirmAction({
    description: t('admin.cardSecrets.batch.confirmStatus', {
      count: ids.length,
      status: statusLabel,
    }),
  })
  if (!confirmed) return

  batchActionLoading.value = true
  try {
    const response = await adminAPI.batchUpdateCardSecretStatus({
      ids,
      status: batchStatusTarget.value,
    })
    const affected = Number(response?.data?.data?.affected || ids.length)
    batchActionSuccess.value = t('admin.cardSecrets.success.batchStatusUpdated', { count: affected })
    await refreshAfterBatchMutations()
  } catch (error: any) {
    batchActionError.value = error?.message || t('admin.cardSecrets.errors.batchStatusFailed')
  } finally {
    batchActionLoading.value = false
  }
}

const deleteSelectedSecrets = async () => {
  clearBatchActionMessages()
  const ids = normalizeSelectedSecretIDs()
  if (ids.length === 0) {
    batchActionError.value = t('admin.cardSecrets.errors.selectRequired')
    return
  }

  const confirmed = await confirmAction({
    description: t('admin.cardSecrets.batch.confirmDelete', { count: ids.length }),
    confirmText: t('admin.common.delete'),
    variant: 'destructive',
  })
  if (!confirmed) return

  batchActionLoading.value = true
  try {
    const response = await adminAPI.batchDeleteCardSecrets({ ids })
    const affected = Number(response?.data?.data?.affected || ids.length)
    batchActionSuccess.value = t('admin.cardSecrets.success.batchDeleted', { count: affected })
    await refreshAfterBatchMutations()
  } catch (error: any) {
    batchActionError.value = error?.message || t('admin.cardSecrets.errors.batchDeleteFailed')
  } finally {
    batchActionLoading.value = false
  }
}

const exportSelectedSecrets = async (format: 'txt' | 'csv') => {
  clearBatchActionMessages()
  const ids = normalizeSelectedSecretIDs()
  if (ids.length === 0) {
    batchActionError.value = t('admin.cardSecrets.errors.selectRequired')
    return
  }

  batchActionLoading.value = true
  try {
    const response = await adminAPI.exportCardSecrets({
      ids,
      format,
    })

    const contentDisposition = String(response?.headers?.['content-disposition'] || '')
    const filenameMatch = contentDisposition.match(/filename=\"?([^\";]+)\"?/i)
    const fallbackName = `card-secrets-${new Date().toISOString().replace(/[:.]/g, '-')}.${format}`
    const filename = filenameMatch?.[1] || fallbackName

    const contentType =
      format === 'csv' ? 'text/csv;charset=utf-8' : 'text/plain;charset=utf-8'
    const blob = new Blob([response.data], { type: contentType })
    const url = URL.createObjectURL(blob)
    const link = document.createElement('a')
    link.href = url
    link.download = filename
    document.body.appendChild(link)
    link.click()
    link.remove()
    URL.revokeObjectURL(url)

    batchActionSuccess.value = t('admin.cardSecrets.success.batchExported', {
      count: ids.length,
      format: format.toUpperCase(),
    })
  } catch (error: any) {
    batchActionError.value = error?.message || t('admin.cardSecrets.errors.batchExportFailed')
  } finally {
    batchActionLoading.value = false
  }
}

const openEditSecret = (secret: AdminCardSecret) => {
  editingCardSecret.value = secret
  showEditModal.value = true
}

const handleEditSuccess = async () => {
  await refreshAfterBatchMutations()
}

const handleBatchCreateSuccess = async () => {
  await refreshAll()
}

const cardSecretStatusLabel = (status: string) => {
  const map: Record<string, string> = {
    available: t('admin.cardSecrets.status.available'),
    reserved: t('admin.cardSecrets.status.reserved'),
    used: t('admin.cardSecrets.status.used'),
  }
  return map[status] || status
}

const cardSecretStatusClass = (status: string) => {
  switch (status) {
    case 'available':
      return 'text-emerald-700 border-emerald-200 bg-emerald-50'
    case 'reserved':
      return 'text-amber-700 border-amber-200 bg-amber-50'
    case 'used':
      return 'text-muted-foreground border-border bg-muted/30'
    default:
      return 'text-muted-foreground border-border bg-muted/30'
  }
}

const batchSkuLabel = (batch: AdminCardSecretBatch) => {
  return resolveSkuLabelById(Number(batch?.sku_id || 0))
}

const secretSkuLabel = (secret: AdminCardSecret) => {
  return resolveSkuLabelById(Number(secret?.sku_id || 0))
}

onMounted(async () => {
  await loadProductOptions()
  await fetchCardSecrets(1)
})
</script>

<template>
  <div class="space-y-6">
    <div class="flex flex-col gap-3 sm:flex-row sm:items-center sm:justify-between">
      <h1 class="text-2xl font-semibold">{{ t('admin.cardSecrets.title') }}</h1>
      <Button v-if="currentProductId" class="w-full sm:w-auto" @click="scrollToImport">
        <Upload class="mr-2 h-4 w-4" />
        {{ t('admin.cardSecrets.importAction') }}
      </Button>
    </div>

    <!-- Guide: shown when no product is selected -->
    <div v-if="!currentProductId" class="rounded-xl border-2 border-dashed border-primary/30 bg-primary/5 p-8">
      <div class="mx-auto max-w-lg text-center space-y-4">
        <div class="mx-auto flex h-16 w-16 items-center justify-center rounded-full bg-primary/10">
          <PackagePlus class="h-8 w-8 text-primary" />
        </div>
        <h2 class="text-xl font-semibold text-foreground">{{ t('admin.cardSecrets.guide.title') }}</h2>
        <p class="text-sm text-muted-foreground">{{ t('admin.cardSecrets.guide.description') }}</p>
        <div class="flex flex-col gap-4 pt-2 text-left sm:flex-row sm:items-start sm:justify-center sm:gap-6">
          <div class="flex items-start gap-3">
            <div class="flex h-7 w-7 shrink-0 items-center justify-center rounded-full bg-primary text-xs font-bold text-primary-foreground">1</div>
            <div>
              <p class="text-sm font-medium text-foreground">{{ t('admin.cardSecrets.guide.step1Title') }}</p>
              <p class="text-xs text-muted-foreground">{{ t('admin.cardSecrets.guide.step1Desc') }}</p>
            </div>
          </div>
          <div class="flex items-start gap-3">
            <div class="flex h-7 w-7 shrink-0 items-center justify-center rounded-full bg-primary text-xs font-bold text-primary-foreground">2</div>
            <div>
              <p class="text-sm font-medium text-foreground">{{ t('admin.cardSecrets.guide.step2Title') }}</p>
              <p class="text-xs text-muted-foreground">{{ t('admin.cardSecrets.guide.step2Desc') }}</p>
            </div>
          </div>
        </div>
      </div>
    </div>

    <div class="rounded-xl border border-border bg-card p-4 shadow-sm">
      <div class="grid grid-cols-1 md:grid-cols-12 gap-3">
        <div class="flex flex-col gap-2 md:col-span-4 sm:flex-row sm:items-center">
          <Input
            v-model="productKeyword"
            :placeholder="t('admin.cardSecrets.productSearchPlaceholder')"
            @update:modelValue="debouncedSearchProducts"
            @keyup.enter="handleSearchProducts"
          />
          <Button
            size="sm"
            variant="outline"
            class="h-9 w-full shrink-0 sm:w-auto"
            :disabled="productOptionsLoading"
            @click="handleSearchProducts"
          >
            {{ productOptionsLoading ? t('admin.common.loading') : t('admin.cardSecrets.searchProducts') }}
          </Button>
        </div>

        <div class="md:col-span-3">
          <Select v-model="selectedProductValue" @update:modelValue="handleProductSelectionChange">
            <SelectTrigger class="h-10">
              <SelectValue :placeholder="t('admin.cardSecrets.productSelectPlaceholder')" />
            </SelectTrigger>
            <SelectContent>
              <SelectItem value="__all__">{{ t('admin.cardSecrets.productAll') }}</SelectItem>
              <SelectItem v-for="product in productOptions" :key="product.id" :value="String(product.id)">
                {{ buildProductLabel(product) }}
              </SelectItem>
            </SelectContent>
          </Select>
        </div>

        <div class="md:col-span-3">
          <Select v-model="skuFilterValue" :disabled="skuFilterDisabled" @update:modelValue="handleSkuSelectionChange">
            <SelectTrigger class="h-10">
              <SelectValue :placeholder="t('admin.cardSecrets.skuPlaceholder')" />
            </SelectTrigger>
            <SelectContent>
              <SelectItem value="__all__">{{ t('admin.cardSecrets.skuAll') }}</SelectItem>
              <SelectItem v-for="sku in availableSkus" :key="sku.id" :value="String(sku.id)">
                {{ sku.label }}
              </SelectItem>
            </SelectContent>
          </Select>
        </div>

        <div class="md:col-span-2">
          <Button class="w-full" variant="outline" @click="refreshAll">{{ t('admin.common.refresh') }}</Button>
        </div>
      </div>

      <div class="mt-3 space-y-1 text-xs text-muted-foreground">
        <p>{{ productHint }}</p>
        <p v-if="productInfoName">
          {{ t('admin.cardSecrets.productNameLabel') }}：
          <a
            v-if="currentProductId"
            :href="productLink(currentProductId)"
            target="_blank"
            rel="noopener"
            class="text-primary underline-offset-4 hover:underline"
          >
            {{ productInfoName }}
          </a>
          <span v-else>{{ productInfoName }}</span>
        </p>
        <p v-if="currentProductId">
          {{ t('admin.cardSecrets.skuLabel') }}：{{ currentSkuId ? resolveSkuLabelById(currentSkuId) : t('admin.cardSecrets.skuAll') }}
        </p>
      </div>
    </div>

    <div ref="importSectionRef">
      <CardSecretBatchCreateModal
        :model-value="!!currentProductId"
        :product-id="currentProductId || 0"
        :sku-id="currentSkuId"
        @success="handleBatchCreateSuccess"
      />
    </div>

    <div class="grid grid-cols-1 lg:grid-cols-3 gap-6">
      <div class="rounded-xl border border-border bg-card p-6">
        <h2 class="text-lg font-semibold text-foreground mb-4">{{ t('admin.cardSecrets.statsTitle') }}</h2>
        <div v-if="statsLoading" class="text-sm text-muted-foreground">{{ t('admin.common.loading') }}</div>
        <div v-else-if="!stats" class="text-sm text-muted-foreground">{{ t('admin.cardSecrets.selectProductTip') }}</div>
        <div v-else class="space-y-3 text-sm">
          <div class="flex justify-between text-muted-foreground"><span>{{ t('admin.cardSecrets.stats.total') }}</span><span class="font-mono text-foreground">{{ stats.total }}</span></div>
          <div class="flex justify-between text-muted-foreground"><span>{{ t('admin.cardSecrets.stats.available') }}</span><span class="font-mono text-foreground">{{ stats.available }}</span></div>
          <div class="flex justify-between text-muted-foreground"><span>{{ t('admin.cardSecrets.stats.reserved') }}</span><span class="font-mono text-foreground">{{ stats.reserved }}</span></div>
          <div class="flex justify-between text-muted-foreground"><span>{{ t('admin.cardSecrets.stats.used') }}</span><span class="font-mono text-foreground">{{ stats.used }}</span></div>
        </div>
      </div>

      <div class="rounded-xl border border-border bg-card p-6 lg:col-span-2">
        <div class="mb-4 flex flex-col gap-3 sm:flex-row sm:items-center sm:justify-between">
          <h2 class="text-lg font-semibold text-foreground">{{ t('admin.cardSecrets.batchesTitle') }}</h2>
          <Button class="w-full sm:w-auto" size="sm" variant="outline" @click="refreshBatches">{{ t('admin.common.refresh') }}</Button>
        </div>

        <div class="overflow-x-auto">
          <Table class="min-w-[920px]">
          <TableHeader class="border-b border-border bg-muted/40 text-xs uppercase text-muted-foreground">
            <TableRow>
              <TableHead class="min-w-[220px] px-4 py-3">{{ t('admin.cardSecrets.table.batchNo') }}</TableHead>
              <TableHead class="min-w-[140px] px-4 py-3">{{ t('admin.cardSecrets.table.source') }}</TableHead>
              <TableHead class="min-w-[220px] px-4 py-3">{{ t('admin.cardSecrets.table.sku') }}</TableHead>
              <TableHead class="min-w-[120px] px-4 py-3">{{ t('admin.cardSecrets.table.count') }}</TableHead>
              <TableHead class="min-w-[220px] px-4 py-3">{{ t('admin.cardSecrets.table.note') }}</TableHead>
              <TableHead class="min-w-[180px] px-4 py-3">{{ t('admin.cardSecrets.table.createdAt') }}</TableHead>
            </TableRow>
          </TableHeader>
          <TableBody class="divide-y divide-border">
            <TableRow v-if="batchesLoading">
              <TableCell :colspan="6" class="p-0">
                <TableSkeleton :columns="6" :rows="5" />
              </TableCell>
            </TableRow>
            <TableRow v-else-if="batches.length === 0">
              <TableCell colspan="6" class="px-4 py-6 text-center text-muted-foreground">{{ t('admin.cardSecrets.emptyBatches') }}</TableCell>
            </TableRow>
            <TableRow v-for="batch in batches" :key="batch.id" class="hover:bg-muted/30">
              <TableCell class="min-w-[220px] px-4 py-3 font-medium text-foreground"><div class="break-all">{{ batch.batch_no || '-' }}</div></TableCell>
              <TableCell class="min-w-[140px] px-4 py-3 text-xs text-muted-foreground">{{ batch.source }}</TableCell>
              <TableCell class="min-w-[220px] px-4 py-3 text-xs text-muted-foreground"><div class="break-words">{{ batchSkuLabel(batch) }}</div></TableCell>
              <TableCell class="min-w-[120px] px-4 py-3 text-xs text-muted-foreground">{{ batch.total_count }}</TableCell>
              <TableCell class="min-w-[220px] px-4 py-3 text-xs text-muted-foreground"><div class="break-words">{{ batch.note || '-' }}</div></TableCell>
              <TableCell class="min-w-[180px] px-4 py-3 text-xs text-muted-foreground">{{ formatDate(batch.created_at) }}</TableCell>
            </TableRow>
          </TableBody>
          </Table>
        </div>

        <div
          v-if="batchPagination.total_page > 1"
          class="flex flex-wrap items-center justify-between gap-3 border-t border-border px-4 py-4"
        >
          <div class="flex items-center gap-3">
            <span class="text-xs text-muted-foreground">
              {{ t('admin.common.pageInfo', { total: batchPagination.total, page: batchPagination.page, totalPage: batchPagination.total_page }) }}
            </span>
          </div>
          <div class="flex flex-wrap items-center gap-2">
            <div class="flex items-center gap-2">
              <Input
                v-model="batchJumpPage"
                type="number"
                min="1"
                :max="batchPagination.total_page"
                class="h-8 w-20"
                :placeholder="t('admin.common.jumpPlaceholder')"
              />
              <Button variant="outline" size="sm" class="h-8" @click="jumpBatchPage">
                {{ t('admin.common.jumpTo') }}
              </Button>
            </div>
            <div class="flex items-center gap-2">
              <Button
                variant="outline"
                size="sm"
                class="h-8"
                :disabled="batchPagination.page <= 1"
                @click="changeBatchPage(batchPagination.page - 1)"
              >
                {{ t('admin.common.prevPage') }}
              </Button>
              <Button
                variant="outline"
                size="sm"
                class="h-8"
                :disabled="batchPagination.page >= batchPagination.total_page"
                @click="changeBatchPage(batchPagination.page + 1)"
              >
                {{ t('admin.common.nextPage') }}
              </Button>
            </div>
          </div>
        </div>
      </div>
    </div>

    <div class="rounded-xl border border-border bg-card p-4 md:p-6">
      <div class="flex flex-col gap-3 mb-4">
        <div class="flex flex-col md:flex-row md:items-center md:justify-between gap-3">
          <h2 class="text-lg font-semibold text-foreground">{{ t('admin.cardSecrets.listTitle') }}</h2>
          <div class="flex flex-wrap items-center gap-3">
            <Select v-model="cardSecretStatus" @update:modelValue="refreshCardSecrets">
              <SelectTrigger class="h-9 w-full md:w-[180px] text-xs">
                <SelectValue :placeholder="t('admin.cardSecrets.statusAll')" />
              </SelectTrigger>
              <SelectContent>
                <SelectItem value="__all__">{{ t('admin.cardSecrets.statusAll') }}</SelectItem>
                <SelectItem value="available">{{ t('admin.cardSecrets.status.available') }}</SelectItem>
                <SelectItem value="reserved">{{ t('admin.cardSecrets.status.reserved') }}</SelectItem>
                <SelectItem value="used">{{ t('admin.cardSecrets.status.used') }}</SelectItem>
              </SelectContent>
            </Select>
            <Button size="sm" variant="outline" @click="refreshCardSecrets">{{ t('admin.common.refresh') }}</Button>
          </div>
        </div>

        <div v-if="hasSelectedSecrets" class="rounded-lg border border-border bg-muted/20 p-3 space-y-3">
          <div class="flex flex-col lg:flex-row lg:items-center lg:justify-between gap-3">
            <span class="text-xs text-muted-foreground">{{ t('admin.cardSecrets.batch.selectedCount', { count: selectedSecretIds.length }) }}</span>
            <div class="flex flex-wrap items-center gap-2">
              <Select v-model="batchStatusTarget">
                <SelectTrigger class="h-8 w-full md:w-[160px] text-xs">
                  <SelectValue :placeholder="t('admin.cardSecrets.batch.statusPlaceholder')" />
                </SelectTrigger>
                <SelectContent>
                  <SelectItem value="available">{{ t('admin.cardSecrets.status.available') }}</SelectItem>
                  <SelectItem value="reserved">{{ t('admin.cardSecrets.status.reserved') }}</SelectItem>
                  <SelectItem value="used">{{ t('admin.cardSecrets.status.used') }}</SelectItem>
                </SelectContent>
              </Select>

              <Button size="sm" variant="outline" :disabled="batchActionLoading" @click="applyBatchStatus">
                {{ t('admin.cardSecrets.batch.applyStatus') }}
              </Button>
              <Button
                size="sm"
                variant="outline"
                class="border-destructive/40 text-destructive hover:bg-destructive/10"
                :disabled="batchActionLoading"
                @click="deleteSelectedSecrets"
              >
                {{ t('admin.cardSecrets.batch.deleteSelected') }}
              </Button>
              <Button size="sm" variant="outline" :disabled="batchActionLoading" @click="exportSelectedSecrets('txt')">
                {{ t('admin.cardSecrets.batch.exportTxt') }}
              </Button>
              <Button size="sm" variant="outline" :disabled="batchActionLoading" @click="exportSelectedSecrets('csv')">
                {{ t('admin.cardSecrets.batch.exportCsv') }}
              </Button>
            </div>
          </div>

          <div v-if="batchActionError" class="rounded-lg border border-destructive/30 bg-destructive/10 p-3 text-sm text-destructive">
            {{ batchActionError }}
          </div>
          <div v-if="batchActionSuccess" class="rounded-lg border border-emerald-200 bg-emerald-50 p-3 text-sm text-emerald-700">
            {{ batchActionSuccess }}
          </div>
        </div>
      </div>

      <div class="overflow-x-auto">
        <Table class="min-w-[1320px]">
        <TableHeader class="border-b border-border bg-muted/40 text-xs uppercase text-muted-foreground">
          <TableRow>
            <TableHead class="px-4 py-3">
              <input type="checkbox" class="h-4 w-4 accent-primary" :checked="allCurrentPageSelected" @change="toggleSelectAllSecrets" />
            </TableHead>
            <TableHead class="px-4 py-3">{{ t('admin.cardSecrets.listTable.id') }}</TableHead>
            <TableHead class="min-w-[260px] px-4 py-3">{{ t('admin.cardSecrets.listTable.secret') }}</TableHead>
            <TableHead class="min-w-[260px] px-4 py-3">{{ t('admin.cardSecrets.listTable.product') }}</TableHead>
            <TableHead class="min-w-[180px] px-4 py-3">{{ t('admin.cardSecrets.listTable.sku') }}</TableHead>
            <TableHead class="min-w-[120px] px-4 py-3">{{ t('admin.cardSecrets.listTable.status') }}</TableHead>
            <TableHead class="min-w-[160px] px-4 py-3">{{ t('admin.cardSecrets.listTable.orderId') }}</TableHead>
            <TableHead class="min-w-[140px] px-4 py-3">{{ t('admin.cardSecrets.listTable.batchId') }}</TableHead>
            <TableHead class="min-w-[180px] px-4 py-3">{{ t('admin.cardSecrets.listTable.createdAt') }}</TableHead>
            <TableHead class="min-w-[140px] px-4 py-3 text-right">{{ t('admin.cardSecrets.listTable.action') }}</TableHead>
          </TableRow>
        </TableHeader>
        <TableBody class="divide-y divide-border">
          <TableRow v-if="cardSecretsLoading">
            <TableCell :colspan="10" class="p-0">
              <TableSkeleton :columns="10" :rows="5" />
            </TableCell>
          </TableRow>
          <TableRow v-else-if="cardSecrets.length === 0">
            <TableCell colspan="10" class="px-4 py-6 text-center text-muted-foreground">{{ t('admin.cardSecrets.emptyList') }}</TableCell>
          </TableRow>
          <TableRow v-for="secret in cardSecrets" :key="secret.id" class="hover:bg-muted/30">
            <TableCell class="px-4 py-3">
              <input type="checkbox" :value="secret.id" v-model="selectedSecretIds" class="h-4 w-4 accent-primary" />
            </TableCell>
            <TableCell class="px-4 py-3">
              <IdCell :value="secret.id" />
            </TableCell>
            <TableCell class="min-w-[260px] px-4 py-3 text-xs font-mono text-muted-foreground break-all">{{ secret.secret }}</TableCell>
            <TableCell class="min-w-[260px] px-4 py-3 text-xs text-muted-foreground">
              <a
                v-if="secret.product_id"
                :href="productLink(secret.product_id)"
                target="_blank"
                rel="noopener"
                class="text-primary underline-offset-4 hover:underline"
              >
                <span class="break-words">#{{ secret.product_id }} {{ resolveProductName(secret.product_id) }}</span>
              </a>
              <span v-else class="text-muted-foreground">-</span>
            </TableCell>
            <TableCell class="min-w-[180px] px-4 py-3 text-xs text-muted-foreground"><div class="break-words">{{ secretSkuLabel(secret) }}</div></TableCell>
            <TableCell class="min-w-[120px] px-4 py-3 text-xs">
              <span class="inline-flex rounded-full border px-2.5 py-1 text-xs" :class="cardSecretStatusClass(secret.status)">
                {{ cardSecretStatusLabel(secret.status) }}
              </span>
            </TableCell>
            <TableCell class="min-w-[160px] px-4 py-3 text-xs">
              <div class="flex flex-col gap-1">
                <a
                  v-if="secret.order_id"
                  :href="orderLink(secret.order_id)"
                  target="_blank"
                  rel="noopener"
                  class="text-primary underline-offset-4 hover:underline"
                >
                  #{{ secret.order_id }}
                </a>
                <span v-else class="text-muted-foreground">-</span>
                <span v-if="secret.status === 'used'" class="text-[11px] text-muted-foreground">
                  {{ t('admin.cardSecrets.listTable.usedOrderHint') }}
                </span>
              </div>
            </TableCell>
            <TableCell class="min-w-[140px] px-4 py-3 text-xs text-muted-foreground">
              <span v-if="secret.batch_id">#{{ secret.batch_id }}</span>
              <span v-else>-</span>
            </TableCell>
            <TableCell class="min-w-[180px] px-4 py-3 text-xs text-muted-foreground">{{ formatDate(secret.created_at) }}</TableCell>
            <TableCell class="min-w-[140px] px-4 py-3 text-right">
              <Button size="sm" variant="outline" @click="openEditSecret(secret)">{{ t('admin.cardSecrets.actions.edit') }}</Button>
            </TableCell>
          </TableRow>
        </TableBody>
        </Table>
      </div>

      <div
        v-if="cardSecretPagination.total_page > 1"
        class="flex flex-wrap items-center justify-between gap-3 border-t border-border px-4 py-4"
      >
        <div class="flex items-center gap-3">
          <span class="text-xs text-muted-foreground">
            {{ t('admin.common.pageInfo', { total: cardSecretPagination.total, page: cardSecretPagination.page, totalPage: cardSecretPagination.total_page }) }}
          </span>
        </div>
        <div class="flex flex-wrap items-center gap-2">
          <div class="flex items-center gap-2">
            <Input
              v-model="cardSecretJumpPage"
              type="number"
              min="1"
              :max="cardSecretPagination.total_page"
              class="h-8 w-20"
              :placeholder="t('admin.common.jumpPlaceholder')"
            />
            <Button variant="outline" size="sm" class="h-8" @click="jumpCardSecretPage">
              {{ t('admin.common.jumpTo') }}
            </Button>
          </div>
          <div class="flex items-center gap-2">
            <Button
              variant="outline"
              size="sm"
              class="h-8"
              :disabled="cardSecretPagination.page <= 1"
              @click="changeSecretPage(cardSecretPagination.page - 1)"
            >
              {{ t('admin.common.prevPage') }}
            </Button>
            <Button
              variant="outline"
              size="sm"
              class="h-8"
              :disabled="cardSecretPagination.page >= cardSecretPagination.total_page"
              @click="changeSecretPage(cardSecretPagination.page + 1)"
            >
              {{ t('admin.common.nextPage') }}
            </Button>
          </div>
        </div>
      </div>
    </div>

    <CardSecretEditModal
      v-model="showEditModal"
      :card-secret="editingCardSecret"
      @success="handleEditSuccess"
    />
  </div>
</template>
