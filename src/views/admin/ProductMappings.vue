<script setup lang="ts">
import { computed, onMounted, reactive, ref, watch } from 'vue'
import { useI18n } from 'vue-i18n'
import { adminAPI } from '@/api/admin'
import type { AdminProductMapping, AdminSiteConnection, AdminCategory, AdminProduct, AdminProductSKU } from '@/api/types'
import { Button } from '@/components/ui/button'
import { Input } from '@/components/ui/input'
import { Dialog, DialogScrollContent, DialogHeader, DialogTitle } from '@/components/ui/dialog'
import { Select, SelectContent, SelectItem, SelectTrigger, SelectValue } from '@/components/ui/select'
import { confirmAction } from '@/utils/confirm'
import { notifyError, notifySuccess } from '@/utils/notify'
import { getLocalizedText } from '@/utils/format'
import { buildAdminCategoryPath, createAdminCategoryChildCountMap, createAdminCategoryMap, flattenAdminCategories, isAdminProductCategorySelectable } from '@/utils/category'
import TableSkeleton from '@/components/TableSkeleton.vue'

const { t, locale } = useI18n()
const loading = ref(true)
const mappings = ref<(AdminProductMapping & { product?: AdminProduct })[]>([])
const connections = ref<AdminSiteConnection[]>([])
const categories = ref<AdminCategory[]>([])
const categoryMap = computed(() => createAdminCategoryMap(categories.value))
const categoryChildCountMap = computed(() => createAdminCategoryChildCountMap(categories.value))
const categoryOptions = computed(() => flattenAdminCategories(categories.value).map((item) => ({
  ...item,
  selectable: isAdminProductCategorySelectable(item.category, categoryChildCountMap.value),
})))
const pagination = reactive({ page: 1, page_size: 20, total: 0, total_page: 1 })
const jumpPage = ref('')
const filters = reactive({ connection_id: '__all__' })
const syncingId = ref<number | null>(null)

// Expand detail
const expandedMappingId = ref<number | null>(null)
const detailLoading = ref(false)

interface SkuMapping {
  local_sku_id: number
  upstream_sku_id: number
  upstream_price: number
  upstream_stock: number
  upstream_is_active: boolean
}

interface MappingDetail {
  mapping: AdminProductMapping
  sku_mappings: SkuMapping[]
}

interface UpstreamSku {
  id: number
  sku_code?: string
  spec_values?: Record<string, string>
  price_amount: number | string
  stock_status: string
  is_active: boolean
}

interface UpstreamProduct {
  id: number
  title: Record<string, string>
  price_amount: number | string
  currency?: string
  is_active: boolean
  skus?: UpstreamSku[]
}

const detailData = ref<MappingDetail | null>(null)

// Import dialog
const showImportModal = ref(false)
const importConnectionId = ref('')
const importCategoryId = ref('__none__')
const upstreamProducts = ref<UpstreamProduct[]>([])
const loadingUpstream = ref(false)
const selectedProductIds = ref<Set<number>>(new Set())
const importExpandedIds = ref<Set<number>>(new Set())
const importing = ref(false)

const normalizeFilterValue = (value: string) => (value === '__all__' ? '' : value)

// --- List helpers ---

const getLocalProductTitle = (mapping: AdminProductMapping & { product?: AdminProduct }) => {
  if (!mapping.product) return `#${mapping.local_product_id}`
  return getLocalizedText(mapping.product.title)
}

const getLocalPriceRange = (mapping: AdminProductMapping & { product?: AdminProduct }) => {
  const p = mapping.product
  if (!p) return '-'
  if (!p.skus || p.skus.length === 0) return p.price_amount || '-'
  const skus = p.skus as AdminProductSKU[]
  const prices = skus.map((s) => parseFloat(String(s.price_amount))).filter((v) => !isNaN(v) && v > 0)
  if (prices.length === 0) return p.price_amount || '-'
  const min = Math.min(...prices)
  const max = Math.max(...prices)
  return min === max ? `${min}` : `${min} ~ ${max}`
}

const getLocalSkuCount = (mapping: AdminProductMapping & { product?: AdminProduct }) => mapping.product?.skus?.length || 0

const formatTime = (raw?: string) => {
  if (!raw) return '-'
  const d = new Date(raw)
  return Number.isNaN(d.getTime()) ? '-' : d.toLocaleString()
}

const formatSpecValues = (specValues: Record<string, string> | undefined | null) => {
  if (!specValues || typeof specValues !== 'object') return '-'
  const entries = Object.entries(specValues)
  if (entries.length === 0) return '-'
  return entries.map(([k, v]) => `${k}: ${getLocalizedText(v)}`).join(' / ')
}

const getConnectionName = (connectionId: number) => {
  const conn = connections.value.find((c) => c.id === connectionId)
  return conn?.name || `#${connectionId}`
}

const getCategoryOptionLabel = (category: AdminCategory) => {
  return buildAdminCategoryPath(category, categoryMap.value, (item) => getLocalizedText(item.name))
}

const getCategoryOptionName = (category: AdminCategory) => {
  return getLocalizedText(category.name)
}

const getCategoryLeafTip = () => {
  if (locale.value === 'en-US') {
    return 'Root categories with child categories cannot receive products directly. Prefer a leaf category.'
  }
  if (locale.value === 'zh-TW') {
    return '已有二級分類的一級分類不能直接掛商品，請優先選擇末級分類'
  }
  return '已有二级分类的一级分类不能直接挂商品，请优先选择末级分类'
}

// --- Detail expand ---

const toggleMappingExpand = async (mapping: AdminProductMapping & { product?: AdminProduct }) => {
  if (expandedMappingId.value === mapping.id) {
    expandedMappingId.value = null
    detailData.value = null
    return
  }
  expandedMappingId.value = mapping.id
  detailLoading.value = true
  detailData.value = null
  try {
    const res = await adminAPI.getProductMapping(mapping.id)
    detailData.value = res.data.data as unknown as MappingDetail
  } catch {
    detailData.value = null
  } finally {
    detailLoading.value = false
  }
}

// Build a lookup: local_sku_id -> sku_mapping
const skuMappingByLocalId = computed(() => {
  const map: Record<number, SkuMapping> = {}
  if (detailData.value?.sku_mappings) {
    for (const sm of detailData.value.sku_mappings) {
      map[sm.local_sku_id] = sm
    }
  }
  return map
})

// --- Fetch ---

const fetchConnections = async () => {
  try {
    const res = await adminAPI.getSiteConnections({ page_size: 100 })
    connections.value = res.data.data || []
  } catch { connections.value = [] }
}

const fetchCategories = async () => {
  try {
    const res = await adminAPI.getCategories()
    categories.value = res.data.data || []
  } catch { categories.value = [] }
}

const fetchMappings = async (page = 1) => {
  loading.value = true
  expandedMappingId.value = null
  detailData.value = null
  try {
    const connId = normalizeFilterValue(filters.connection_id)
    const res = await adminAPI.getProductMappings({
      page, page_size: pagination.page_size,
      connection_id: connId || undefined,
    })
    mappings.value = (res.data.data as (AdminProductMapping & { product?: AdminProduct })[]) || []
    const p = res.data.pagination
    if (p) { pagination.page = p.page; pagination.page_size = p.page_size; pagination.total = p.total; pagination.total_page = p.total_page }
  } catch { mappings.value = [] } finally { loading.value = false }
}

const changePage = (page: number) => { if (page >= 1 && page <= pagination.total_page) fetchMappings(page) }

const jumpToPage = () => {
  if (!jumpPage.value) return
  const raw = Number(jumpPage.value)
  if (Number.isNaN(raw)) return
  const target = Math.min(Math.max(Math.floor(raw), 1), pagination.total_page)
  if (target !== pagination.page) changePage(target)
}

const handleFilterChange = () => fetchMappings(1)

// --- Actions ---

const handleSync = async (mapping: AdminProductMapping) => {
  syncingId.value = mapping.id
  try {
    await adminAPI.syncProductMapping(mapping.id)
    notifySuccess(t('productMappings.sync.success'))
    fetchMappings(pagination.page)
  } catch (err: any) {
    notifyError(t('productMappings.sync.failed') + ': ' + (err?.response?.data?.message || err?.message || ''))
  } finally { syncingId.value = null }
}

const handleToggleStatus = async (mapping: AdminProductMapping) => {
  try {
    await adminAPI.updateProductMappingStatus(mapping.id, { is_active: !mapping.is_active })
    fetchMappings(pagination.page)
    notifySuccess()
  } catch (err: any) { notifyError(err?.response?.data?.message || err?.message) }
}

const handleDelete = async (mapping: AdminProductMapping) => {
  const confirmed = await confirmAction({
    description: t('productMappings.delete.confirm', { id: mapping.id }),
    confirmText: t('admin.common.delete'),
    variant: 'destructive',
  })
  if (!confirmed) return
  try {
    await adminAPI.deleteProductMapping(mapping.id)
    fetchMappings(pagination.page)
    notifySuccess()
  } catch (err: any) { notifyError(err?.response?.data?.message || err?.message) }
}

// --- Import dialog ---

const allSelected = computed(() => {
  if (upstreamProducts.value.length === 0) return false
  return upstreamProducts.value.every((p) => selectedProductIds.value.has(p.id))
})

const toggleSelectAll = () => {
  selectedProductIds.value = allSelected.value
    ? new Set()
    : new Set(upstreamProducts.value.map((p) => p.id))
}

const toggleProduct = (id: number) => {
  const next = new Set(selectedProductIds.value)
  next.has(id) ? next.delete(id) : next.add(id)
  selectedProductIds.value = next
}

const toggleImportExpand = (id: number) => {
  const next = new Set(importExpandedIds.value)
  next.has(id) ? next.delete(id) : next.add(id)
  importExpandedIds.value = next
}

const getSkuPriceRange = (product: UpstreamProduct) => {
  if (!product.skus || product.skus.length === 0) return product.price_amount || '-'
  const prices = product.skus.map((s) => parseFloat(String(s.price_amount))).filter((p: number) => !isNaN(p) && p > 0)
  if (prices.length === 0) return product.price_amount || '-'
  const min = Math.min(...prices)
  const max = Math.max(...prices)
  return min === max ? `${min}` : `${min} ~ ${max}`
}

const getSkuStockSummary = (product: UpstreamProduct) => {
  if (!product.skus || product.skus.length === 0) return '-'
  const total = product.skus.length
  const inStock = product.skus.filter((s) => s.stock_status === 'in_stock').length
  if (inStock === total) return t('productMappings.import.stockAllInStock')
  if (inStock === 0) return t('productMappings.import.stockAllOutOfStock')
  return t('productMappings.import.stockPartial', { inStock, total })
}

const getSkuStockClass = (product: UpstreamProduct) => {
  if (!product.skus || product.skus.length === 0) return 'text-muted-foreground'
  const total = product.skus.length
  const inStock = product.skus.filter((s) => s.stock_status === 'in_stock').length
  if (inStock === total) return 'text-emerald-600'
  if (inStock === 0) return 'text-red-500'
  return 'text-amber-600'
}

const openImportModal = () => {
  importConnectionId.value = ''
  importCategoryId.value = '__none__'
  upstreamProducts.value = []
  selectedProductIds.value = new Set()
  importExpandedIds.value = new Set()
  showImportModal.value = true
}

const closeImportModal = () => { showImportModal.value = false }

const fetchUpstreamProducts = async (connectionId: string) => {
  if (!connectionId) { upstreamProducts.value = []; return }
  loadingUpstream.value = true
  try {
    const res = await adminAPI.getUpstreamProducts({ connection_id: connectionId, page_size: 200 })
    const data = res.data.data as UpstreamProduct[] | { items?: UpstreamProduct[] } | null
    upstreamProducts.value = (Array.isArray(data) ? data : (data as { items?: UpstreamProduct[] })?.items) || []
  } catch { upstreamProducts.value = [] } finally { loadingUpstream.value = false }
}

watch(importConnectionId, (value) => {
  selectedProductIds.value = new Set()
  importExpandedIds.value = new Set()
  fetchUpstreamProducts(value)
})

const handleBatchImport = async () => {
  const ids = Array.from(selectedProductIds.value)
  if (ids.length === 0) return
  importing.value = true
  try {
    const categoryId = importCategoryId.value !== '__none__' ? Number(importCategoryId.value) : 0
    let results: { upstream_product_id: number; success: boolean; error?: string }[] = []
    let successCount = 0
    try {
      const res = await adminAPI.batchImportUpstreamProducts({
        connection_id: Number(importConnectionId.value),
        upstream_product_ids: ids,
        category_id: categoryId || undefined,
      })
      const result = res.data.data as { results?: typeof results; success_count?: number } | null
      results = result?.results || []
      successCount = result?.success_count || 0
    } catch (batchErr: any) {
      if (batchErr?.response?.status === 404) {
        for (const id of ids) {
          try {
            await adminAPI.importUpstreamProduct({ connection_id: Number(importConnectionId.value), upstream_product_id: id, category_id: categoryId || undefined })
            results.push({ upstream_product_id: id, success: true }); successCount++
          } catch (singleErr: any) {
            results.push({ upstream_product_id: id, success: false, error: singleErr?.response?.data?.message || singleErr?.message })
          }
        }
      } else { throw batchErr }
    }
    if (successCount === ids.length) {
      notifySuccess(t('productMappings.import.batchSuccess', { count: successCount }))
      closeImportModal()
    } else {
      const failed = results.filter((r) => !r.success)
      const failedDetails = failed.map((r) => {
        const prod = upstreamProducts.value.find((p) => p.id === r.upstream_product_id)
        const name = prod ? getLocalizedText(prod.title) : `#${r.upstream_product_id}`
        return `${name}: ${r.error || t('productMappings.import.unknownError')}`
      }).join('\n')
      if (successCount > 0) notifySuccess(t('productMappings.import.batchPartial', { success: successCount, total: ids.length }))
      notifyError(failedDetails)
      const successIds = new Set(results.filter((r) => r.success).map((r) => r.upstream_product_id))
      selectedProductIds.value = new Set([...selectedProductIds.value].filter(id => !successIds.has(id)))
    }
    fetchMappings(1)
  } catch (err: any) { notifyError(err?.response?.data?.message || err?.message) } finally { importing.value = false }
}

onMounted(() => { fetchConnections(); fetchCategories(); fetchMappings() })
</script>

<template>
  <div class="space-y-6">
    <div class="flex flex-col gap-3 sm:flex-row sm:items-center sm:justify-between">
      <h1 class="text-2xl font-semibold">{{ t('productMappings.title') }}</h1>
      <Button class="w-full sm:w-auto" @click="openImportModal">{{ t('productMappings.importButton') }}</Button>
    </div>

    <!-- Filter -->
    <div class="rounded-xl border border-border bg-card p-4 shadow-sm">
      <div class="flex flex-wrap items-center gap-3">
        <Select v-model="filters.connection_id" @update:modelValue="handleFilterChange">
          <SelectTrigger class="h-9 w-full sm:w-[220px]">
            <SelectValue :placeholder="t('productMappings.filter.connectionPlaceholder')" />
          </SelectTrigger>
          <SelectContent>
            <SelectItem value="__all__">{{ t('productMappings.filter.allConnections') }}</SelectItem>
            <SelectItem v-for="conn in connections" :key="conn.id" :value="String(conn.id)">{{ conn.name }}</SelectItem>
          </SelectContent>
        </Select>
      </div>
    </div>

    <!-- Mapping list -->
    <div class="space-y-3">
      <div v-if="loading" class="rounded-xl border border-border bg-card overflow-hidden">
        <TableSkeleton :columns="5" :rows="5" />
      </div>
      <div v-else-if="mappings.length === 0" class="rounded-xl border border-border bg-card px-6 py-12 text-center text-muted-foreground">
        {{ t('productMappings.empty') }}
      </div>

      <div
        v-for="mapping in mappings"
        :key="mapping.id"
        class="rounded-xl border border-border bg-card overflow-hidden transition-shadow"
        :class="expandedMappingId === mapping.id ? 'shadow-md ring-1 ring-primary/20' : 'shadow-sm'"
      >
        <!-- Collapsed row -->
        <div
          class="flex flex-col gap-3 px-5 py-4 cursor-pointer transition-colors hover:bg-muted/20 sm:flex-row sm:items-center sm:gap-4"
          @click="toggleMappingExpand(mapping)"
        >
          <!-- Expand arrow -->
          <svg
            class="h-4 w-4 shrink-0 text-muted-foreground transition-transform duration-200"
            :class="expandedMappingId === mapping.id ? 'rotate-90' : ''"
            xmlns="http://www.w3.org/2000/svg" viewBox="0 0 20 20" fill="currentColor"
          >
            <path fill-rule="evenodd" d="M7.21 14.77a.75.75 0 01.02-1.06L11.168 10 7.23 6.29a.75.75 0 111.04-1.08l4.5 4.25a.75.75 0 010 1.08l-4.5 4.25a.75.75 0 01-1.06-.02z" clip-rule="evenodd" />
          </svg>

          <!-- Product info -->
          <div class="min-w-0 flex-1">
            <div class="flex flex-wrap items-center gap-2">
              <span class="break-words text-sm font-medium text-foreground sm:truncate">{{ getLocalProductTitle(mapping) }}</span>
              <span class="shrink-0 text-[10px] font-mono text-muted-foreground">#{{ mapping.local_product_id }}</span>
              <span
                class="shrink-0 inline-flex rounded-full border px-2 py-0.5 text-[10px]"
                :class="mapping.is_active ? 'text-emerald-700 border-emerald-200 bg-emerald-50' : 'text-muted-foreground border-border bg-muted/30'"
              >
                {{ mapping.is_active ? t('productMappings.status.active') : t('productMappings.status.inactive') }}
              </span>
            </div>
            <div class="mt-1 flex flex-wrap items-center gap-x-4 gap-y-1 text-xs text-muted-foreground">
              <span>{{ t('productMappings.columns.connection') }}: <span class="text-foreground">{{ getConnectionName(mapping.connection_id) }}</span></span>
              <span>{{ t('productMappings.detail.upstreamId') }}: <span class="font-mono text-foreground">{{ mapping.upstream_product_id }}</span></span>
              <span>{{ t('productMappings.detail.localPrice') }}: <span class="font-mono text-foreground">{{ getLocalPriceRange(mapping) }}</span></span>
              <span>SKU: <span class="text-foreground">{{ getLocalSkuCount(mapping) }}</span></span>
              <span>{{ t('productMappings.columns.lastSynced') }}: {{ formatTime((mapping.last_synced_at ?? mapping.last_sync_at) as string | undefined) }}</span>
            </div>
          </div>

          <!-- Actions -->
          <div class="flex w-full flex-col gap-2 sm:w-auto sm:flex-row sm:items-center sm:gap-2 shrink-0" @click.stop>
            <Button size="sm" variant="outline" class="w-full sm:w-auto" :disabled="syncingId === mapping.id" @click="handleSync(mapping)">
              {{ syncingId === mapping.id ? t('productMappings.actions.syncing') : t('productMappings.actions.sync') }}
            </Button>
            <Button size="sm" variant="outline" class="w-full sm:w-auto" @click="handleToggleStatus(mapping)">
              {{ mapping.is_active ? t('productMappings.actions.disable') : t('productMappings.actions.enable') }}
            </Button>
            <Button size="sm" variant="destructive" class="w-full sm:w-auto" @click="handleDelete(mapping)">{{ t('admin.common.delete') }}</Button>
          </div>
        </div>

        <!-- Expanded detail -->
        <div v-if="expandedMappingId === mapping.id" class="border-t border-border bg-muted/10">
          <div v-if="detailLoading" class="px-6 py-8 text-center text-sm text-muted-foreground">
            {{ t('admin.common.loading') }}
          </div>
          <div v-else-if="!detailData" class="px-6 py-8 text-center text-sm text-muted-foreground">
            {{ t('productMappings.detail.loadFailed') }}
          </div>
          <div v-else class="px-5 py-4">
            <!-- SKU comparison table -->
            <h4 class="mb-3 text-xs font-semibold uppercase tracking-wider text-muted-foreground">
              {{ t('productMappings.detail.skuComparison') }}
            </h4>
            <div class="overflow-x-auto rounded-lg border border-border">
              <table class="min-w-[860px] w-full text-xs">
                <thead>
                  <tr class="bg-muted/50 text-muted-foreground">
                    <th class="min-w-[160px] px-3 py-2.5 text-left font-medium">{{ t('productMappings.detail.skuCode') }}</th>
                    <th class="min-w-[220px] px-3 py-2.5 text-left font-medium">{{ t('productMappings.import.skuSpec') }}</th>
                    <th class="min-w-[120px] px-3 py-2.5 text-right font-medium">{{ t('productMappings.detail.localPrice') }}</th>
                    <th class="min-w-[120px] px-3 py-2.5 text-right font-medium">{{ t('productMappings.detail.upstreamPrice') }}</th>
                    <th class="min-w-[120px] px-3 py-2.5 text-center font-medium">{{ t('productMappings.detail.priceDiff') }}</th>
                    <th class="min-w-[120px] px-3 py-2.5 text-center font-medium">{{ t('productMappings.detail.upstreamStock') }}</th>
                    <th class="min-w-[120px] px-3 py-2.5 text-center font-medium">{{ t('productMappings.detail.upstreamActive') }}</th>
                  </tr>
                </thead>
                <tbody class="divide-y divide-border">
                  <tr v-if="!mapping.product?.skus?.length" >
                    <td colspan="7" class="px-3 py-6 text-center text-muted-foreground">{{ t('productMappings.detail.noSkus') }}</td>
                  </tr>
                  <tr
                    v-for="sku in (mapping.product?.skus as AdminProductSKU[] | undefined) || []"
                    :key="sku.id"
                    class="hover:bg-muted/20"
                  >
                    <td class="min-w-[160px] px-3 py-2.5 font-mono text-muted-foreground break-all">{{ sku.sku_code }}</td>
                    <td class="min-w-[220px] px-3 py-2.5 text-foreground break-words">{{ formatSpecValues(sku.spec_values) }}</td>
                    <td class="min-w-[120px] px-3 py-2.5 text-right font-mono text-foreground">{{ sku.price_amount }}</td>
                    <td class="min-w-[120px] px-3 py-2.5 text-right font-mono">
                      <template v-if="skuMappingByLocalId[sku.id]">
                        <span class="text-foreground">{{ skuMappingByLocalId[sku.id]?.upstream_price }}</span>
                      </template>
                      <span v-else class="text-muted-foreground">-</span>
                    </td>
                    <td class="px-3 py-2.5 text-center">
                      <template v-if="skuMappingByLocalId[sku.id]">
                        <span
                          v-if="Number(sku.price_amount) !== Number(skuMappingByLocalId[sku.id]?.upstream_price)"
                          class="inline-flex rounded-full bg-amber-50 px-1.5 py-0.5 text-[10px] font-medium text-amber-700 border border-amber-200"
                        >
                          {{ (Number(sku.price_amount) - Number(skuMappingByLocalId[sku.id]?.upstream_price)).toFixed(2) }}
                        </span>
                        <span v-else class="text-emerald-600">-</span>
                      </template>
                      <span v-else class="text-muted-foreground">-</span>
                    </td>
                    <td class="px-3 py-2.5 text-center">
                      <template v-if="skuMappingByLocalId[sku.id]">
                        <span
                          class="inline-flex rounded-full px-1.5 py-0.5 text-[10px]"
                          :class="(skuMappingByLocalId[sku.id]?.upstream_stock ?? 0) > 0
                            ? 'text-emerald-700 bg-emerald-50'
                            : 'text-red-600 bg-red-50'"
                        >
                          {{ (skuMappingByLocalId[sku.id]?.upstream_stock ?? 0) > 0 ? t('productMappings.import.inStock') : t('productMappings.import.outOfStock') }}
                        </span>
                      </template>
                      <span v-else class="text-muted-foreground">-</span>
                    </td>
                    <td class="px-3 py-2.5 text-center">
                      <template v-if="skuMappingByLocalId[sku.id]">
                        <span
                          class="inline-block h-2 w-2 rounded-full"
                          :class="skuMappingByLocalId[sku.id]?.upstream_is_active ? 'bg-emerald-500' : 'bg-gray-300'"
                        />
                      </template>
                      <span v-else class="text-muted-foreground">-</span>
                    </td>
                  </tr>
                </tbody>
              </table>
            </div>

            <!-- Mapping metadata -->
            <div class="mt-4 flex flex-wrap gap-x-6 gap-y-1 text-xs text-muted-foreground">
              <span>{{ t('productMappings.detail.mappingId') }}: <span class="font-mono text-foreground">{{ mapping.id }}</span></span>
              <span>{{ t('productMappings.detail.createdAt') }}: {{ formatTime(mapping.created_at) }}</span>
              <span>{{ t('productMappings.detail.updatedAt') }}: {{ formatTime(mapping.updated_at) }}</span>
              <span v-if="mapping.product?.fulfillment_type">
                {{ t('productMappings.detail.fulfillmentType') }}:
                <span class="text-foreground">{{ mapping.product.fulfillment_type }}</span>
              </span>
            </div>
          </div>
        </div>
      </div>

      <!-- Pagination -->
      <div v-if="pagination.total_page > 1" class="flex flex-wrap items-center justify-between gap-3 rounded-xl border border-border bg-card px-6 py-4">
        <span class="text-xs text-muted-foreground">
          {{ t('admin.common.pageInfo', { total: pagination.total, page: pagination.page, totalPage: pagination.total_page }) }}
        </span>
        <div class="flex flex-wrap items-center gap-2">
          <Input v-model="jumpPage" type="number" min="1" :max="pagination.total_page" class="h-8 w-20" :placeholder="t('admin.common.jumpPlaceholder')" />
          <Button variant="outline" size="sm" class="h-8" @click="jumpToPage">{{ t('admin.common.jumpTo') }}</Button>
          <Button variant="outline" size="sm" class="h-8" :disabled="pagination.page <= 1" @click="changePage(pagination.page - 1)">{{ t('admin.common.prevPage') }}</Button>
          <Button variant="outline" size="sm" class="h-8" :disabled="pagination.page >= pagination.total_page" @click="changePage(pagination.page + 1)">{{ t('admin.common.nextPage') }}</Button>
        </div>
      </div>
    </div>

    <!-- Import Dialog -->
    <Dialog v-model:open="showImportModal" @update:open="(value: boolean) => { if (!value) closeImportModal() }">
      <DialogScrollContent class="w-[calc(100vw-1rem)] max-w-5xl max-h-[85vh] p-4 sm:p-6" @interact-outside="(e: Event) => e.preventDefault()">
        <DialogHeader>
          <DialogTitle>{{ t('productMappings.importTitle') }}</DialogTitle>
        </DialogHeader>
        <div class="space-y-5">
          <div class="flex flex-col gap-4 lg:flex-row lg:items-end">
            <div class="min-w-[200px] flex-1">
              <label class="mb-1.5 block text-xs font-medium text-muted-foreground">{{ t('productMappings.import.selectConnection') }}</label>
              <Select v-model="importConnectionId">
                <SelectTrigger class="h-9 w-full"><SelectValue :placeholder="t('productMappings.import.selectConnectionPlaceholder')" /></SelectTrigger>
                <SelectContent>
                  <SelectItem v-for="conn in connections" :key="conn.id" :value="String(conn.id)">{{ conn.name }}</SelectItem>
                </SelectContent>
              </Select>
            </div>
            <div class="min-w-[200px] flex-1">
              <label class="mb-1.5 block text-xs font-medium text-muted-foreground">{{ t('productMappings.import.category') }}</label>
              <Select v-model="importCategoryId">
                <SelectTrigger class="h-9 w-full"><SelectValue :placeholder="t('productMappings.import.categoryPlaceholder')" /></SelectTrigger>
                <SelectContent>
                  <SelectItem value="__none__">{{ t('productMappings.import.noCategory') }}</SelectItem>
                  <SelectItem
                    v-for="item in categoryOptions"
                    :key="item.category.id"
                    :value="String(item.category.id)"
                    :disabled="!item.selectable"
                    :class="item.depth > 0 ? 'pl-6' : ''"
                  >
                    {{ item.depth > 0 ? getCategoryOptionName(item.category) : getCategoryOptionLabel(item.category) }}
                  </SelectItem>
                </SelectContent>
              </Select>
              <p class="mt-1 text-xs text-muted-foreground">{{ getCategoryLeafTip() }}</p>
            </div>
          </div>

          <div class="rounded-lg border border-border overflow-hidden">
            <div v-if="!importConnectionId" class="px-6 py-12 text-center text-sm text-muted-foreground">{{ t('productMappings.import.selectConnectionFirst') }}</div>
            <div v-else-if="loadingUpstream" class="px-6 py-12 text-center text-sm text-muted-foreground">{{ t('productMappings.import.upstreamProductLoading') }}</div>
            <div v-else-if="upstreamProducts.length === 0" class="px-6 py-12 text-center text-sm text-muted-foreground">{{ t('productMappings.import.noUpstreamProducts') }}</div>
            <div v-else>
              <div class="flex items-center gap-3 border-b border-border bg-muted/40 px-4 py-2.5">
                <input type="checkbox" :checked="allSelected" :indeterminate="selectedProductIds.size > 0 && !allSelected" class="h-4 w-4 rounded border-border accent-primary cursor-pointer" @change="toggleSelectAll" />
                <span class="text-xs font-medium text-muted-foreground">{{ t('productMappings.import.selectAll') }} ({{ upstreamProducts.length }})</span>
              </div>
              <div class="divide-y divide-border max-h-[50vh] overflow-y-auto">
                <div v-for="product in upstreamProducts" :key="product.id" class="transition-colors" :class="selectedProductIds.has(product.id) ? 'bg-primary/5' : ''">
                  <div class="flex items-center gap-3 px-4 py-3 cursor-pointer hover:bg-muted/20" @click="toggleProduct(product.id)">
                    <input type="checkbox" :checked="selectedProductIds.has(product.id)" class="h-4 w-4 shrink-0 rounded border-border accent-primary cursor-pointer" @click.stop="toggleProduct(product.id)" />
                    <div class="min-w-0 flex-1">
                      <div class="flex flex-wrap items-center gap-2">
                        <span class="break-words text-sm font-medium text-foreground sm:truncate">{{ getLocalizedText(product.title) }}</span>
                        <span class="shrink-0 text-[10px] font-mono text-muted-foreground">#{{ product.id }}</span>
                        <span class="shrink-0 inline-flex rounded-full border px-2 py-0.5 text-[10px]" :class="product.is_active ? 'text-emerald-700 border-emerald-200 bg-emerald-50' : 'text-muted-foreground border-border bg-muted/30'">
                          {{ product.is_active ? t('productMappings.status.active') : t('productMappings.status.inactive') }}
                        </span>
                      </div>
                      <div class="mt-1 flex flex-wrap items-center gap-4 text-xs text-muted-foreground">
                        <span>{{ t('productMappings.import.colPrice') }}: <span class="font-mono text-foreground">{{ getSkuPriceRange(product) }}</span><span v-if="product.currency" class="ml-0.5">{{ product.currency }}</span></span>
                        <span>SKU: <span class="text-foreground">{{ product.skus?.length || 0 }}</span></span>
                        <span :class="getSkuStockClass(product)">{{ getSkuStockSummary(product) }}</span>
                      </div>
                    </div>
                    <button v-if="product.skus && product.skus.length > 0" class="shrink-0 rounded p-1 text-muted-foreground hover:bg-muted hover:text-foreground transition-colors" @click.stop="toggleImportExpand(product.id)" :title="t('productMappings.import.toggleSkuDetails')">
                      <svg class="h-4 w-4 transition-transform duration-200" :class="importExpandedIds.has(product.id) ? 'rotate-180' : ''" xmlns="http://www.w3.org/2000/svg" viewBox="0 0 20 20" fill="currentColor">
                        <path fill-rule="evenodd" d="M5.23 7.21a.75.75 0 011.06.02L10 11.168l3.71-3.938a.75.75 0 111.08 1.04l-4.25 4.5a.75.75 0 01-1.08 0l-4.25-4.5a.75.75 0 01.02-1.06z" clip-rule="evenodd" />
                      </svg>
                    </button>
                  </div>
                  <div v-if="importExpandedIds.has(product.id) && product.skus && product.skus.length > 0" class="border-t border-border/50 bg-muted/20 px-4 py-2">
                    <div class="ml-7 overflow-x-auto">
                      <div class="grid min-w-[640px] grid-cols-[1fr_auto_auto_auto] gap-x-4 text-xs">
                        <div class="font-medium text-muted-foreground py-1.5 border-b border-border/30">{{ t('productMappings.import.skuSpec') }}</div>
                        <div class="font-medium text-muted-foreground py-1.5 border-b border-border/30 text-right">{{ t('productMappings.import.skuPrice') }}</div>
                        <div class="font-medium text-muted-foreground py-1.5 border-b border-border/30 text-center">{{ t('productMappings.import.skuStock') }}</div>
                        <div class="font-medium text-muted-foreground py-1.5 border-b border-border/30 text-center">{{ t('productMappings.import.skuActive') }}</div>
                        <template v-for="sku in product.skus" :key="sku.id">
                          <div class="py-1.5 text-foreground">
                            <span v-if="sku.sku_code" class="mr-2 break-all font-mono text-muted-foreground">{{ sku.sku_code }}</span>
                            <span>{{ formatSpecValues(sku.spec_values) }}</span>
                          </div>
                          <div class="py-1.5 text-right font-mono text-foreground">{{ sku.price_amount }}<span v-if="product.currency" class="text-muted-foreground ml-0.5">{{ product.currency }}</span></div>
                          <div class="py-1.5 text-center">
                            <span class="inline-flex rounded-full px-1.5 py-0.5 text-[10px]" :class="sku.stock_status === 'in_stock' ? 'text-emerald-700 bg-emerald-50' : 'text-red-600 bg-red-50'">
                              {{ sku.stock_status === 'in_stock' ? t('productMappings.import.inStock') : t('productMappings.import.outOfStock') }}
                            </span>
                          </div>
                          <div class="py-1.5 text-center"><span class="inline-block h-2 w-2 rounded-full" :class="sku.is_active ? 'bg-emerald-500' : 'bg-gray-300'" /></div>
                        </template>
                      </div>
                    </div>
                  </div>
                </div>
              </div>
            </div>
          </div>

          <div class="flex flex-col gap-3 border-t border-border pt-4 sm:flex-row sm:items-center sm:justify-between">
            <span v-if="selectedProductIds.size > 0" class="text-sm text-muted-foreground">{{ t('productMappings.import.selectedCount', { count: selectedProductIds.size }) }}</span>
            <span v-else />
            <div class="flex flex-col-reverse gap-3 sm:flex-row">
              <Button variant="outline" class="w-full sm:w-auto" @click="closeImportModal">{{ t('admin.common.cancel') }}</Button>
              <Button class="w-full sm:w-auto" :disabled="selectedProductIds.size === 0 || !importConnectionId || importing" @click="handleBatchImport">
                {{ importing ? t('productMappings.import.importing') : t('productMappings.import.submitBatch', { count: selectedProductIds.size }) }}
              </Button>
            </div>
          </div>
        </div>
      </DialogScrollContent>
    </Dialog>
  </div>
</template>
