<template>
  <div class="restocking">
    <div class="page-header">
      <h2>{{ t('restocking.title') }}</h2>
      <p>{{ t('restocking.description') }}</p>
    </div>

    <div v-if="loading" class="loading">{{ t('common.loading') }}</div>
    <div v-else-if="error" class="error">{{ error }}</div>
    <div v-else>
      <div class="card">
        <div class="card-header">
          <h3 class="card-title">{{ t('restocking.budgetLabel') }}</h3>
        </div>
        <div class="budget-control">
          <input
            type="range"
            min="0"
            :max="maxBudget"
            step="100"
            v-model.number="budget"
            class="budget-slider"
          />
          <span class="budget-value">{{ currencySymbol }}{{ budget.toLocaleString() }}</span>
        </div>
      </div>

      <div class="stats-grid">
        <div class="stat-card info">
          <div class="stat-label">{{ t('restocking.selectedItems') }}</div>
          <div class="stat-value">{{ totalSelectedCount }}</div>
        </div>
        <div class="stat-card">
          <div class="stat-label">{{ t('restocking.selectedCost') }}</div>
          <div class="stat-value">{{ currencySymbol }}{{ totalSelectedCost.toLocaleString(undefined, { minimumFractionDigits: 2, maximumFractionDigits: 2 }) }}</div>
        </div>
        <div class="stat-card" :class="remainingBudget < 0 ? 'danger' : 'success'">
          <div class="stat-label">{{ t('restocking.remainingBudget') }}</div>
          <div class="stat-value">{{ currencySymbol }}{{ remainingBudget.toLocaleString(undefined, { minimumFractionDigits: 2, maximumFractionDigits: 2 }) }}</div>
        </div>
      </div>

      <div v-if="isOverBudget" class="error">
        {{ t('restocking.overBudget', { amount: currencySymbol + overBudgetAmount.toLocaleString(undefined, { minimumFractionDigits: 2, maximumFractionDigits: 2 }) }) }}
      </div>

      <div class="card">
        <div class="card-header">
          <h3 class="card-title">{{ t('restocking.recommendationsTitle') }} ({{ rows.length }})</h3>
          <button
            class="place-order-btn"
            :disabled="placeOrderDisabled"
            @click="placeOrder"
          >
            {{ t('restocking.placeOrder') }}
          </button>
        </div>

        <div v-if="submitError" class="error">{{ submitError }}</div>
        <div v-if="successMessage" class="success-message">{{ successMessage }}</div>

        <div v-if="rows.length === 0" class="loading">{{ t('restocking.noData') }}</div>
        <div v-else class="table-container">
          <table>
            <thead>
              <tr>
                <th>{{ t('restocking.table.include') }}</th>
                <th>{{ t('restocking.table.sku') }}</th>
                <th>{{ t('restocking.table.itemName') }}</th>
                <th>{{ t('restocking.table.category') }}</th>
                <th>{{ t('restocking.table.warehouse') }}</th>
                <th>{{ t('restocking.table.currentStock') }}</th>
                <th>{{ t('restocking.table.reorderPoint') }}</th>
                <th>{{ t('restocking.table.trend') }}</th>
                <th>{{ t('restocking.table.forecastedDemand') }}</th>
                <th>{{ t('restocking.table.unitCost') }}</th>
                <th>{{ t('restocking.table.quantity') }}</th>
                <th>{{ t('restocking.table.lineTotal') }}</th>
              </tr>
            </thead>
            <tbody>
              <tr v-for="row in rows" :key="row.sku">
                <td><input type="checkbox" v-model="row.checked" /></td>
                <td><strong>{{ row.sku }}</strong></td>
                <td>{{ translateProductName(row.name) }}</td>
                <td>{{ translateCategory(row.category) }}</td>
                <td>{{ translateWarehouse(row.warehouse) }}</td>
                <td>{{ row.quantity_on_hand }}</td>
                <td>{{ row.reorder_point }}</td>
                <td>
                  <span :class="['badge', row.trend]">{{ t(`trends.${row.trend}`) }}</span>
                </td>
                <td>{{ row.forecasted_demand }}</td>
                <td>{{ currencySymbol }}{{ row.unit_cost.toFixed(2) }}</td>
                <td>
                  <input
                    type="number"
                    min="0"
                    v-model.number="row.quantity"
                    class="qty-input"
                  />
                </td>
                <td><strong>{{ currencySymbol }}{{ (row.quantity * row.unit_cost).toLocaleString(undefined, { minimumFractionDigits: 2, maximumFractionDigits: 2 }) }}</strong></td>
              </tr>
            </tbody>
          </table>
        </div>
      </div>
    </div>
  </div>
</template>

<script>
import { ref, onMounted, computed } from 'vue'
import { api } from '../api'
import { useI18n } from '../composables/useI18n'

const TREND_RANK = { increasing: 0, stable: 1, decreasing: 2 }

export default {
  name: 'Restocking',
  setup() {
    const { t, currentCurrency, translateProductName, translateWarehouse } = useI18n()

    const currencySymbol = computed(() => {
      return currentCurrency.value === 'JPY' ? '¥' : '$'
    })

    const loading = ref(true)
    const error = ref(null)
    const inventoryItems = ref([])
    const demandForecasts = ref([])

    const budget = ref(0)
    const rows = ref([])

    const submitting = ref(false)
    const submitError = ref(null)
    const successMessage = ref(null)

    // Join demand forecasts with matching inventory items
    const joinedItems = computed(() => {
      const joined = []
      demandForecasts.value.forEach(forecast => {
        const invItem = inventoryItems.value.find(item => item.sku === forecast.item_sku)
        if (!invItem) return // Skip forecasts with no matching inventory item

        const understocked = invItem.quantity_on_hand <= invItem.reorder_point
        const suggestedQuantity = Math.max(
          forecast.forecasted_demand - invItem.quantity_on_hand,
          invItem.reorder_point - invItem.quantity_on_hand,
          0
        )

        joined.push({
          sku: invItem.sku,
          name: invItem.name,
          category: invItem.category,
          warehouse: invItem.warehouse,
          quantity_on_hand: invItem.quantity_on_hand,
          reorder_point: invItem.reorder_point,
          unit_cost: invItem.unit_cost,
          forecasted_demand: forecast.forecasted_demand,
          trend: forecast.trend,
          trendRank: TREND_RANK[forecast.trend] ?? 1,
          understocked,
          suggestedQuantity
        })
      })
      return joined
    })

    // Understocked first, then by trend (increasing before stable before decreasing),
    // then by how far below reorder point (most urgent first)
    const sortedItems = computed(() => {
      return joinedItems.value.slice().sort((a, b) => {
        if (a.understocked !== b.understocked) return a.understocked ? -1 : 1
        if (a.trendRank !== b.trendRank) return a.trendRank - b.trendRank
        return (a.quantity_on_hand - a.reorder_point) - (b.quantity_on_hand - b.reorder_point)
      })
    })

    // Dynamic slider max: total cost to fully close every positive quantity gap,
    // rounded up to a clean multiple of 1000
    const maxBudget = computed(() => {
      const totalGapCost = joinedItems.value.reduce((sum, item) => {
        return item.suggestedQuantity > 0 ? sum + item.suggestedQuantity * item.unit_cost : sum
      }, 0)
      if (totalGapCost <= 0) return 1000
      return Math.ceil(totalGapCost / 1000) * 1000
    })

    // Build the editable row state and greedily pre-check rows in priority order
    // within the current budget. Only called on initial load - user edits (checkbox
    // toggles, quantity changes) are never overwritten by this after that.
    const buildRows = () => {
      let runningTotal = 0
      rows.value = sortedItems.value.map(item => {
        let checked = false
        if (item.suggestedQuantity > 0) {
          const cost = item.suggestedQuantity * item.unit_cost
          if (runningTotal + cost <= budget.value) {
            checked = true
            runningTotal += cost
          }
        }
        return {
          sku: item.sku,
          name: item.name,
          category: item.category,
          warehouse: item.warehouse,
          quantity_on_hand: item.quantity_on_hand,
          reorder_point: item.reorder_point,
          unit_cost: item.unit_cost,
          forecasted_demand: item.forecasted_demand,
          trend: item.trend,
          suggestedQuantity: item.suggestedQuantity,
          checked,
          quantity: item.suggestedQuantity
        }
      })
    }

    const loadData = async () => {
      try {
        loading.value = true
        error.value = null
        const [inventoryData, demandData] = await Promise.all([
          api.getInventory(),
          api.getDemandForecasts()
        ])
        inventoryItems.value = inventoryData
        demandForecasts.value = demandData

        budget.value = Math.round(maxBudget.value / 2)
        buildRows()
      } catch (err) {
        error.value = 'Failed to load restocking data: ' + err.message
      } finally {
        loading.value = false
      }
    }

    const totalSelectedCount = computed(() => rows.value.filter(r => r.checked).length)

    const totalSelectedCost = computed(() => {
      return rows.value
        .filter(r => r.checked)
        .reduce((sum, r) => sum + r.quantity * r.unit_cost, 0)
    })

    const remainingBudget = computed(() => budget.value - totalSelectedCost.value)
    const isOverBudget = computed(() => totalSelectedCost.value > budget.value)
    const overBudgetAmount = computed(() => Math.max(totalSelectedCost.value - budget.value, 0))

    const placeOrderDisabled = computed(() => {
      return totalSelectedCount.value === 0 || isOverBudget.value || submitting.value
    })

    const translateCategory = (category) => {
      const categoryMap = {
        'Circuit Boards': t('categories.circuitBoards'),
        'Sensors': t('categories.sensors'),
        'Actuators': t('categories.actuators'),
        'Controllers': t('categories.controllers'),
        'Power Supplies': t('categories.powerSupplies')
      }
      return categoryMap[category] || category
    }

    const placeOrder = async () => {
      submitError.value = null
      successMessage.value = null
      submitting.value = true
      try {
        const items = rows.value
          .filter(r => r.checked && r.quantity > 0)
          .map(r => ({
            sku: r.sku,
            name: r.name,
            quantity: r.quantity,
            unit_price: r.unit_cost
          }))

        const order = await api.createRestockOrder(items, budget.value)
        successMessage.value = t('restocking.successMessage', {
          orderNumber: order.order_number,
          leadTime: order.lead_time_days
        })

        // Reset selection state so the user can place another order
        rows.value.forEach(r => { r.checked = false })
      } catch (err) {
        submitError.value = t('restocking.errorMessage')
        console.error(err)
      } finally {
        submitting.value = false
      }
    }

    onMounted(loadData)

    return {
      t,
      loading,
      error,
      rows,
      budget,
      maxBudget,
      totalSelectedCount,
      totalSelectedCost,
      remainingBudget,
      isOverBudget,
      overBudgetAmount,
      placeOrderDisabled,
      submitError,
      successMessage,
      placeOrder,
      currencySymbol,
      translateProductName,
      translateWarehouse,
      translateCategory
    }
  }
}
</script>

<style scoped>
.page-header {
  margin-bottom: 1.5rem;
}

.page-header p {
  color: #64748b;
  font-size: 0.875rem;
}

.budget-control {
  display: flex;
  align-items: center;
  gap: 1.25rem;
  padding: 0.5rem 0;
}

.budget-slider {
  flex: 1;
  max-width: 480px;
  accent-color: #2563eb;
}

.budget-value {
  font-size: 1.125rem;
  font-weight: 700;
  color: #0f172a;
  min-width: 100px;
}

.place-order-btn {
  padding: 0.625rem 1.25rem;
  background: #2563eb;
  color: white;
  border: none;
  border-radius: 8px;
  font-weight: 600;
  font-size: 0.875rem;
  cursor: pointer;
  transition: all 0.2s ease;
  white-space: nowrap;
}

.place-order-btn:hover:not(:disabled) {
  background: #1d4ed8;
}

.place-order-btn:disabled {
  opacity: 0.5;
  cursor: not-allowed;
}

.qty-input {
  width: 80px;
  padding: 0.375rem 0.5rem;
  border: 1px solid #cbd5e1;
  border-radius: 6px;
  font-size: 0.875rem;
  color: #0f172a;
}

.qty-input:focus {
  outline: none;
  border-color: #2563eb;
  box-shadow: 0 0 0 3px rgba(37, 99, 235, 0.1);
}

.success-message {
  background: #d1fae5;
  border: 1px solid #a7f3d0;
  color: #065f46;
  padding: 1rem;
  border-radius: 8px;
  margin: 1rem 0;
  font-size: 0.938rem;
  font-weight: 600;
}
</style>
