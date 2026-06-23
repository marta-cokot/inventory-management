<script setup>
import { ref, computed, onMounted } from 'vue'
import { api } from '../api'
import { useI18n } from '../composables/useI18n'
import { formatCurrency } from '../utils/currency'

const { t, currentCurrency, translateProductName } = useI18n()

const budget = ref(50000)
const allForecasts = ref([])
const inventoryMap = ref({})
const loading = ref(false)
const error = ref(null)
const placingOrder = ref(false)
const orderPlaced = ref(false)
const placedOrderNumber = ref('')

const loadData = async () => {
  loading.value = true
  error.value = null
  try {
    const [forecasts, inventory] = await Promise.all([
      api.getDemandForecasts(),
      api.getInventory({})
    ])
    allForecasts.value = forecasts
    inventoryMap.value = inventory.reduce((map, item) => {
      map[item.sku] = item
      return map
    }, {})
  } catch (err) {
    error.value = 'Failed to load data: ' + err.message
  } finally {
    loading.value = false
  }
}

onMounted(loadData)

const enrichedItems = computed(() => {
  return allForecasts.value.map(forecast => {
    const inv = inventoryMap.value[forecast.item_sku] || {}
    const unitCost = inv.unit_cost || 0
    const restockQty = forecast.forecasted_demand
    return {
      ...forecast,
      unit_cost: unitCost,
      quantity_on_hand: inv.quantity_on_hand ?? 0,
      reorder_point: inv.reorder_point ?? 0,
      restocking_cost: restockQty * unitCost,
      // Only flag as below reorder point if this SKU actually exists in inventory
      belowReorderPoint: inv.unit_cost !== undefined && (inv.quantity_on_hand ?? 0) <= (inv.reorder_point ?? 0)
    }
  })
})

const maxBudget = computed(() => {
  return Math.ceil(enrichedItems.value.reduce((sum, i) => sum + i.restocking_cost, 0) / 1000) * 1000
})

const sortedItems = computed(() => {
  const trendOrder = { increasing: 0, stable: 1, decreasing: 2 }
  return [...enrichedItems.value].sort((a, b) => {
    if (a.belowReorderPoint !== b.belowReorderPoint) return a.belowReorderPoint ? -1 : 1
    return (trendOrder[a.trend] ?? 1) - (trendOrder[b.trend] ?? 1)
  })
})

const recommendedItems = computed(() => {
  let remaining = budget.value
  return sortedItems.value.map(item => {
    const canAfford = remaining >= item.restocking_cost && item.restocking_cost > 0
    if (canAfford) remaining -= item.restocking_cost
    return { ...item, selected: canAfford }
  })
})

const selectedItems = computed(() => recommendedItems.value.filter(i => i.selected))
const totalCost = computed(() => selectedItems.value.reduce((sum, i) => sum + i.restocking_cost, 0))

const placeOrder = async () => {
  if (!selectedItems.value.length || placingOrder.value || orderPlaced.value) return
  placingOrder.value = true
  error.value = null
  try {
    const items = selectedItems.value.map(i => ({
      sku: i.item_sku,
      name: i.item_name,
      quantity: i.forecasted_demand,
      unit_cost: i.unit_cost
    }))
    const result = await api.createRestockingOrder(items)
    placedOrderNumber.value = result.order_number
    orderPlaced.value = true
  } catch (err) {
    error.value = 'Failed to place order: ' + err.message
  } finally {
    placingOrder.value = false
  }
}
</script>

<template>
  <div class="restocking">
    <div class="page-header">
      <h2>{{ t('restocking.title') }}</h2>
      <p>{{ t('restocking.description') }}</p>
    </div>

    <div v-if="loading" class="loading">{{ t('common.loading') }}</div>
    <div v-else-if="error" class="error">{{ error }}</div>

    <div v-else>
      <div v-if="orderPlaced" class="success-banner">
        {{ t('restocking.orderSuccess', { orderNumber: placedOrderNumber }) }}
      </div>

      <div class="card budget-card">
        <div class="card-header">
          <h3 class="card-title">{{ t('restocking.budgetLabel') }}</h3>
          <span class="budget-display">{{ formatCurrency(budget, currentCurrency) }}</span>
        </div>
        <input
          type="range"
          v-model.number="budget"
          :min="0"
          :max="maxBudget"
          :step="1000"
          class="budget-slider"
        />
        <div class="budget-summary">
          <span>{{ t('restocking.estimatedSpend') }}: <strong>{{ formatCurrency(totalCost, currentCurrency) }}</strong></span>
          <span class="budget-remaining">{{ formatCurrency(budget - totalCost, currentCurrency) }} {{ t('restocking.budgetRemaining') }}</span>
          <span>{{ t('restocking.itemsSelected', { count: selectedItems.length }) }}</span>
        </div>
      </div>

      <div class="card" v-if="recommendedItems.length > 0">
        <div class="card-header">
          <h3 class="card-title">Recommended Items</h3>
        </div>
        <div class="table-container">
          <table class="restocking-table">
            <thead>
              <tr>
                <th>{{ t('restocking.tableHeaders.sku') }}</th>
                <th>{{ t('restocking.tableHeaders.item') }}</th>
                <th>{{ t('restocking.tableHeaders.inStock') }}</th>
                <th>{{ t('restocking.tableHeaders.reorderPoint') }}</th>
                <th>{{ t('restocking.tableHeaders.forecastedNeed') }}</th>
                <th>{{ t('restocking.tableHeaders.unitCost') }}</th>
                <th>{{ t('restocking.tableHeaders.totalCost') }}</th>
                <th>{{ t('restocking.tableHeaders.status') }}</th>
              </tr>
            </thead>
            <tbody>
              <tr
                v-for="item in recommendedItems"
                :key="item.id"
                :class="['item-row', { 'row-selected': item.selected, 'row-over-budget': !item.selected }]"
              >
                <td><strong>{{ item.item_sku }}</strong></td>
                <td>{{ translateProductName(item.item_name) }}</td>
                <td :class="{ 'text-warning': item.belowReorderPoint }">{{ item.quantity_on_hand }}</td>
                <td>{{ item.reorder_point }}</td>
                <td><strong>{{ item.forecasted_demand }}</strong></td>
                <td>{{ formatCurrency(item.unit_cost, currentCurrency) }}</td>
                <td><strong>{{ formatCurrency(item.restocking_cost, currentCurrency) }}</strong></td>
                <td>
                  <span v-if="item.belowReorderPoint" class="badge warning" style="margin-bottom:0.25rem; display:block">
                    {{ t('restocking.statusBadges.belowReorderPoint') }}
                  </span>
                  <span :class="['badge', item.trend]">{{ item.trend }}</span>
                  <span v-if="item.selected" class="badge success" style="margin-top:0.25rem; display:block">
                    {{ t('restocking.statusBadges.selected') }}
                  </span>
                  <span v-else class="badge over-budget" style="margin-top:0.25rem; display:block">
                    {{ t('restocking.statusBadges.overBudget') }}
                  </span>
                </td>
              </tr>
            </tbody>
          </table>
        </div>
      </div>

      <div v-else class="empty-state">{{ t('restocking.noItems') }}</div>

      <div class="actions">
        <button
          class="btn-primary"
          :disabled="!selectedItems.length || placingOrder || orderPlaced"
          @click="placeOrder"
        >
          {{ placingOrder ? t('restocking.placingOrder') : t('restocking.placeOrder') }}
        </button>
      </div>
    </div>
  </div>
</template>

<style scoped>
.budget-card .card-header {
  justify-content: space-between;
  align-items: center;
}

.budget-display {
  font-size: 1.5rem;
  font-weight: 700;
  color: #0f172a;
}

.budget-slider {
  width: 100%;
  margin: 1rem 0;
  accent-color: #2563eb;
  height: 6px;
  cursor: pointer;
}

.budget-summary {
  display: flex;
  gap: 1.5rem;
  flex-wrap: wrap;
  font-size: 0.875rem;
  color: #64748b;
  margin-top: 0.5rem;
}

.budget-summary strong {
  color: #0f172a;
}

.budget-remaining {
  margin-left: auto;
  color: #059669;
  font-weight: 600;
}

.row-selected {
  background: #f0fdf4;
}

.row-selected td {
  color: #0f172a;
}

.row-over-budget {
  opacity: 0.5;
}

.text-warning {
  color: #d97706;
  font-weight: 600;
}

.badge.over-budget {
  background: #f1f5f9;
  color: #64748b;
}

.success-banner {
  background: #d1fae5;
  border: 1px solid #6ee7b7;
  border-radius: 8px;
  padding: 1rem 1.25rem;
  color: #065f46;
  font-weight: 500;
  margin-bottom: 1.25rem;
  font-size: 0.938rem;
}

.actions {
  display: flex;
  justify-content: flex-end;
  margin-top: 1rem;
}

.btn-primary {
  background: #2563eb;
  color: white;
  border: none;
  border-radius: 8px;
  padding: 0.75rem 2rem;
  font-size: 0.938rem;
  font-weight: 600;
  cursor: pointer;
}

.btn-primary:hover {
  background: #1d4ed8;
}

.btn-primary:disabled {
  opacity: 0.4;
  cursor: not-allowed;
}

.empty-state {
  text-align: center;
  padding: 3rem;
  color: #64748b;
}

.restocking-table {
  table-layout: auto;
}
</style>
