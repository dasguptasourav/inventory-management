<template>
  <div class="restocking">
    <div class="page-header">
      <h2>{{ t('restocking.title') }}</h2>
      <p>{{ t('restocking.description') }}</p>
    </div>

    <!-- Success state after order placed -->
    <div v-if="submitted" class="card success-card">
      <div class="success-icon">✓</div>
      <h3>{{ t('restocking.orderPlaced') }}</h3>
      <p>{{ t('restocking.orderNumber') }}: <strong>{{ submitted.order_number }}</strong></p>
      <p>{{ t('restocking.expectedDelivery') }}: <strong>{{ formatDate(submitted.expected_delivery) }}</strong></p>
      <p class="lead-time-note">{{ t('restocking.leadTime', { days: submitted.lead_time_days }) }}</p>
      <button class="btn-secondary" @click="resetForm">{{ t('restocking.placeAnother') }}</button>
    </div>

    <div v-else>
      <!-- Budget Slider -->
      <div class="card budget-card">
        <div class="card-header">
          <h3 class="card-title">{{ t('restocking.budget') }}</h3>
          <span class="budget-display">{{ currencySymbol }}{{ budget.toLocaleString() }}</span>
        </div>
        <div class="slider-container">
          <span class="slider-label">{{ currencySymbol }}0</span>
          <input
            type="range"
            v-model.number="budget"
            min="0"
            max="500000"
            step="1000"
            class="budget-slider"
          />
          <span class="slider-label">{{ currencySymbol }}500,000</span>
        </div>
      </div>

      <div v-if="loading" class="loading">{{ t('common.loading') }}</div>
      <div v-else-if="error" class="error">{{ error }}</div>

      <div v-else class="card">
        <div class="card-header">
          <h3 class="card-title">{{ t('restocking.recommendations') }} ({{ recommendedItems.length }})</h3>
          <div class="budget-summary">
            <span class="selected-count">{{ selectedCount }} {{ t('restocking.itemsSelected') }}</span>
            <span class="budget-used" :class="{ 'over-budget': selectedTotal > budget }">
              {{ currencySymbol }}{{ selectedTotal.toLocaleString() }} / {{ currencySymbol }}{{ budget.toLocaleString() }}
            </span>
          </div>
        </div>

        <div v-if="recommendedItems.length === 0" class="no-data-state">
          {{ t('restocking.noBudget') }}
        </div>

        <div v-else class="table-container">
          <table class="restocking-table">
            <thead>
              <tr>
                <th class="col-check"></th>
                <th>{{ t('restocking.table.name') }}</th>
                <th class="col-sku">{{ t('restocking.table.sku') }}</th>
                <th class="col-trend">{{ t('restocking.table.trend') }}</th>
                <th class="col-num">{{ t('restocking.table.forecastQty') }}</th>
                <th class="col-num">{{ t('restocking.table.unitCost') }}</th>
                <th class="col-num">{{ t('restocking.table.totalCost') }}</th>
              </tr>
            </thead>
            <tbody>
              <tr
                v-for="item in recommendedItems"
                :key="item.item_sku"
                :class="{ 'row-deselected': !itemSelections[item.item_sku] }"
              >
                <td class="col-check">
                  <input
                    type="checkbox"
                    :checked="itemSelections[item.item_sku]"
                    @change="toggleSelection(item.item_sku)"
                  />
                </td>
                <td>{{ item.item_name }}</td>
                <td class="col-sku sku-code">{{ item.item_sku }}</td>
                <td class="col-trend">
                  <span :class="['badge', item.trend]">{{ item.trend }}</span>
                </td>
                <td class="col-num">{{ item.forecasted_demand.toLocaleString() }}</td>
                <td class="col-num">{{ currencySymbol }}{{ item.unit_cost.toFixed(2) }}</td>
                <td class="col-num"><strong>{{ currencySymbol }}{{ item.total_cost.toLocaleString() }}</strong></td>
              </tr>
            </tbody>
          </table>
        </div>

        <div class="card-footer">
          <span v-if="selectedCount === 0" class="footer-hint">{{ t('restocking.selectItems') }}</span>
          <span v-else class="footer-summary">
            {{ selectedCount }} {{ t('restocking.itemsSelected') }} &middot;
            {{ t('restocking.total') }}: <strong>{{ currencySymbol }}{{ selectedTotal.toLocaleString() }}</strong>
          </span>
          <button
            class="btn-primary"
            :disabled="selectedCount === 0 || submitting"
            @click="placeOrder"
          >
            {{ submitting ? t('restocking.placing') : t('restocking.placeOrder') }}
          </button>
        </div>
      </div>
    </div>
  </div>
</template>

<script>
import { ref, computed, watch, onMounted } from 'vue'
import { api } from '../api'
import { useI18n } from '../composables/useI18n'

export default {
  name: 'Restocking',
  setup() {
    const { t, currentCurrency } = useI18n()
    const currencySymbol = computed(() => currentCurrency.value === 'JPY' ? '¥' : '$')

    const loading = ref(true)
    const error = ref(null)
    const submitting = ref(false)
    const submitted = ref(null)

    const budget = ref(50000)
    const allForecasts = ref([])
    const inventoryMap = ref({})
    const itemSelections = ref({})

    // Join demand forecasts with unit_cost from inventory
    const enrichedForecasts = computed(() => {
      return allForecasts.value
        .map(f => ({ ...f, unit_cost: inventoryMap.value[f.item_sku] || 0 }))
        .filter(f => f.unit_cost > 0)
    })

    // Greedy fill: increasing trend first, then stable; skip decreasing
    const recommendedItems = computed(() => {
      const candidates = enrichedForecasts.value
        .filter(f => f.trend !== 'decreasing')
        .sort((a, b) => {
          if (a.trend === 'increasing' && b.trend !== 'increasing') return -1
          if (b.trend === 'increasing' && a.trend !== 'increasing') return 1
          return 0
        })

      let remaining = budget.value
      const result = []
      for (const item of candidates) {
        const cost = item.forecasted_demand * item.unit_cost
        if (remaining >= cost) {
          result.push({ ...item, total_cost: Math.round(cost * 100) / 100 })
          remaining -= cost
        }
      }
      return result
    })

    const initSelections = () => {
      const selections = {}
      recommendedItems.value.forEach(item => {
        // Preserve existing selection if item was already recommended
        selections[item.item_sku] = itemSelections.value[item.item_sku] !== undefined
          ? itemSelections.value[item.item_sku]
          : true
      })
      itemSelections.value = selections
    }

    watch(budget, initSelections)

    const toggleSelection = (sku) => {
      itemSelections.value = { ...itemSelections.value, [sku]: !itemSelections.value[sku] }
    }

    const selectedCount = computed(() =>
      Object.values(itemSelections.value).filter(Boolean).length
    )

    const selectedTotal = computed(() =>
      Math.round(
        recommendedItems.value
          .filter(item => itemSelections.value[item.item_sku])
          .reduce((sum, item) => sum + item.total_cost, 0) * 100
      ) / 100
    )

    const placeOrder = async () => {
      const itemsToOrder = recommendedItems.value.filter(item => itemSelections.value[item.item_sku])
      if (itemsToOrder.length === 0) return

      try {
        submitting.value = true
        error.value = null
        submitted.value = await api.submitRestockingOrder({
          items: itemsToOrder.map(item => ({
            sku: item.item_sku,
            name: item.item_name,
            quantity: item.forecasted_demand,
            unit_price: item.unit_cost
          })),
          total_value: selectedTotal.value
        })
      } catch (err) {
        error.value = 'Failed to place order: ' + err.message
      } finally {
        submitting.value = false
      }
    }

    const resetForm = () => {
      submitted.value = null
      budget.value = 50000
      initSelections()
    }

    const formatDate = (dateString) => {
      const { currentLocale } = useI18n()
      const locale = currentLocale.value === 'ja' ? 'ja-JP' : 'en-US'
      return new Date(dateString).toLocaleDateString(locale, {
        year: 'numeric',
        month: 'long',
        day: 'numeric'
      })
    }

    onMounted(async () => {
      try {
        loading.value = true
        const [forecasts, inventory] = await Promise.all([
          api.getDemandForecasts(),
          api.getInventory()
        ])
        const skuMap = {}
        inventory.forEach(item => { skuMap[item.sku] = item.unit_cost })
        inventoryMap.value = skuMap
        allForecasts.value = forecasts
        initSelections()
      } catch (err) {
        error.value = 'Failed to load data: ' + err.message
      } finally {
        loading.value = false
      }
    })

    return {
      t, currencySymbol,
      loading, error, submitting, submitted,
      budget, recommendedItems, itemSelections,
      selectedCount, selectedTotal,
      toggleSelection, placeOrder, resetForm, formatDate
    }
  }
}
</script>

<style scoped>
.budget-card .card-header {
  align-items: center;
}

.budget-display {
  font-size: 1.75rem;
  font-weight: 700;
  color: #2563eb;
  letter-spacing: -0.025em;
}

.slider-container {
  display: flex;
  align-items: center;
  gap: 1rem;
  padding: 0.5rem 0;
}

.slider-label {
  font-size: 0.813rem;
  color: #64748b;
  white-space: nowrap;
  min-width: 60px;
}

.slider-label:last-child {
  text-align: right;
}

.budget-slider {
  flex: 1;
  height: 6px;
  border-radius: 3px;
  appearance: none;
  background: linear-gradient(to right, #2563eb 0%, #2563eb calc(var(--value, 10%) * 1%), #e2e8f0 calc(var(--value, 10%) * 1%), #e2e8f0 100%);
  cursor: pointer;
  outline: none;
}

.budget-slider::-webkit-slider-thumb {
  appearance: none;
  width: 20px;
  height: 20px;
  border-radius: 50%;
  background: #2563eb;
  border: 2px solid white;
  box-shadow: 0 1px 4px rgba(37, 99, 235, 0.4);
  cursor: pointer;
}

.budget-slider::-moz-range-thumb {
  width: 20px;
  height: 20px;
  border-radius: 50%;
  background: #2563eb;
  border: 2px solid white;
  box-shadow: 0 1px 4px rgba(37, 99, 235, 0.4);
  cursor: pointer;
}

.budget-summary {
  display: flex;
  align-items: center;
  gap: 1rem;
  font-size: 0.875rem;
}

.selected-count {
  color: #64748b;
}

.budget-used {
  font-weight: 600;
  color: #059669;
}

.budget-used.over-budget {
  color: #dc2626;
}

.restocking-table {
  width: 100%;
  border-collapse: collapse;
}

.col-check {
  width: 40px;
  text-align: center;
}

.col-sku {
  width: 120px;
}

.col-trend {
  width: 120px;
}

.col-num {
  width: 130px;
  text-align: right;
}

.sku-code {
  font-family: 'SF Mono', 'Fira Code', monospace;
  font-size: 0.813rem;
  color: #475569;
}

.row-deselected td {
  opacity: 0.45;
}

.no-data-state {
  padding: 2.5rem;
  text-align: center;
  color: #64748b;
  font-size: 0.938rem;
}

.card-footer {
  display: flex;
  justify-content: space-between;
  align-items: center;
  padding-top: 1rem;
  margin-top: 0.75rem;
  border-top: 1px solid #e2e8f0;
}

.footer-hint {
  font-size: 0.875rem;
  color: #94a3b8;
  font-style: italic;
}

.footer-summary {
  font-size: 0.875rem;
  color: #475569;
}

.btn-primary {
  background: #2563eb;
  color: white;
  border: none;
  padding: 0.625rem 1.5rem;
  border-radius: 6px;
  font-size: 0.938rem;
  font-weight: 600;
  cursor: pointer;
  transition: background 0.2s;
}

.btn-primary:hover:not(:disabled) {
  background: #1d4ed8;
}

.btn-primary:disabled {
  background: #94a3b8;
  cursor: not-allowed;
}

.btn-secondary {
  background: white;
  color: #374151;
  border: 1px solid #e2e8f0;
  padding: 0.625rem 1.5rem;
  border-radius: 6px;
  font-size: 0.938rem;
  font-weight: 600;
  cursor: pointer;
  transition: all 0.2s;
  margin-top: 1rem;
}

.btn-secondary:hover {
  background: #f8fafc;
  border-color: #cbd5e1;
}

/* Success state */
.success-card {
  text-align: center;
  padding: 3rem 2rem;
}

.success-icon {
  width: 56px;
  height: 56px;
  background: #d1fae5;
  color: #059669;
  border-radius: 50%;
  display: flex;
  align-items: center;
  justify-content: center;
  font-size: 1.5rem;
  font-weight: 700;
  margin: 0 auto 1.25rem;
}

.success-card h3 {
  font-size: 1.375rem;
  font-weight: 700;
  color: #0f172a;
  margin-bottom: 0.75rem;
}

.success-card p {
  color: #475569;
  font-size: 0.938rem;
  margin-bottom: 0.375rem;
}

.lead-time-note {
  color: #64748b;
  font-size: 0.813rem;
  margin-top: 0.5rem;
}
</style>
