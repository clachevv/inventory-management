<template>
  <div class="restocking">
    <div class="page-header">
      <h2>Restocking Planner</h2>
      <p>Allocate budget across demand-prioritized items</p>
    </div>

    <div v-if="loading" class="loading">Loading...</div>
    <div v-else-if="error" class="error">{{ error }}</div>
    <div v-else>
      <div class="card budget-card">
        <div class="card-header">
          <h3 class="card-title">Available Budget</h3>
          <span class="budget-display">{{ currencySymbol }}{{ budget.toLocaleString() }}</span>
        </div>
        <div class="slider-wrapper">
          <input
            type="range"
            class="budget-slider"
            :min="0"
            :max="budgetMax"
            :step="100"
            v-model.number="budget"
          />
          <div class="slider-labels">
            <span>{{ currencySymbol }}0</span>
            <span>{{ currencySymbol }}{{ budgetMax.toLocaleString() }}</span>
          </div>
        </div>
        <div class="budget-summary">
          {{ selectedItems.length }} items selected
          &middot;
          Total cost: {{ currencySymbol }}{{ totalCost.toLocaleString() }}
          &middot;
          Remaining: {{ currencySymbol }}{{ remaining.toLocaleString() }}
        </div>
      </div>

      <div class="card">
        <div class="card-header">
          <h3 class="card-title">Recommended Items ({{ recommendations.length }})</h3>
        </div>

        <div v-if="recommendations.length === 0" class="empty-state">
          No items match demand forecast data.
        </div>
        <div v-else>
          <div class="table-container">
            <table class="restock-table">
              <thead>
                <tr>
                  <th>Item</th>
                  <th>SKU</th>
                  <th>Category</th>
                  <th>Warehouse</th>
                  <th>Trend</th>
                  <th>Priority</th>
                  <th>On Hand</th>
                  <th>Reorder Point</th>
                  <th>Suggested Qty</th>
                  <th>Unit Cost</th>
                  <th>Allocated Qty</th>
                  <th>Total Cost</th>
                </tr>
              </thead>
              <tbody>
                <tr
                  v-for="item in recommendations"
                  :key="item.sku"
                  :class="{ 'row-selected': getAllocatedQty(item.sku) > 0, 'row-partial': isPartial(item.sku) }"
                >
                  <td><strong>{{ item.name }}</strong></td>
                  <td class="mono">{{ item.sku }}</td>
                  <td>{{ item.category }}</td>
                  <td>{{ item.warehouse }}</td>
                  <td>
                    <span :class="['badge', item.trend]">{{ item.trend }}</span>
                  </td>
                  <td>
                    <span :class="['badge', getPriorityClass(item.trend)]">{{ getLeadTimeLabel(item.trend) }}</span>
                  </td>
                  <td>{{ item.quantity_on_hand }}</td>
                  <td>{{ item.reorder_point }}</td>
                  <td><strong>{{ item.suggested_quantity }}</strong></td>
                  <td>{{ currencySymbol }}{{ item.unit_cost.toLocaleString() }}</td>
                  <td>
                    <span v-if="getAllocatedQty(item.sku) > 0" :class="['qty-chip', { 'qty-partial': isPartial(item.sku) }]">
                      {{ getAllocatedQty(item.sku) }}
                    </span>
                    <span v-else class="qty-zero">—</span>
                  </td>
                  <td>
                    <span v-if="getAllocatedQty(item.sku) > 0">
                      {{ currencySymbol }}{{ (getAllocatedQty(item.sku) * item.unit_cost).toLocaleString() }}
                    </span>
                    <span v-else class="qty-zero">—</span>
                  </td>
                </tr>
              </tbody>
            </table>
          </div>

          <div class="order-section">
            <div v-if="successMessage" class="success-message">{{ successMessage }}</div>
            <div v-if="submitError" class="error">{{ submitError }}</div>
            <button
              class="place-order-btn"
              :disabled="selectedItems.length === 0 || submitting"
              @click="placeOrder"
            >
              {{ submitting ? 'Submitting...' : 'Place Order' }}
            </button>
          </div>
        </div>
      </div>
    </div>
  </div>
</template>

<script>
import { ref, computed, onMounted } from 'vue'
import { api } from '../api'
import { useI18n } from '../composables/useI18n'

export default {
  name: 'Restocking',
  setup() {
    const { currentCurrency } = useI18n()

    const currencySymbol = computed(() => currentCurrency.value === 'JPY' ? '¥' : '$')

    const inventoryItems = ref([])
    const demandForecasts = ref([])
    const loading = ref(true)
    const error = ref(null)
    const submitting = ref(false)
    const successMessage = ref('')
    const submitError = ref(null)

    const budget = ref(0)

    // Join inventory + demand forecasts, sort by ROI priority
    const recommendations = computed(() => {
      const forecastMap = new Map()
      demandForecasts.value.forEach(f => forecastMap.set(f.item_sku, f))

      const joined = inventoryItems.value
        .filter(item => forecastMap.has(item.sku))
        .map(item => {
          const forecast = forecastMap.get(item.sku)
          const suggested_quantity = Math.max(item.reorder_point - item.quantity_on_hand, 0) + forecast.forecasted_demand
          return {
            ...item,
            trend: forecast.trend,
            forecasted_demand: forecast.forecasted_demand,
            suggested_quantity
          }
        })

      const trendOrder = { increasing: 0, stable: 1, decreasing: 2 }
      joined.sort((a, b) => {
        const trendDiff = (trendOrder[a.trend] ?? 3) - (trendOrder[b.trend] ?? 3)
        if (trendDiff !== 0) return trendDiff
        return a.unit_cost - b.unit_cost
      })

      return joined
    })

    // Total cost if we order everything at suggested quantities
    const rawBudgetMax = computed(() =>
      recommendations.value.reduce((sum, item) => sum + item.suggested_quantity * item.unit_cost, 0)
    )

    // Round up to nearest $1000 for a clean slider max
    const budgetMax = computed(() => Math.ceil(rawBudgetMax.value / 1000) * 1000)

    // Greedy budget allocation
    const selectedItems = computed(() => {
      let remaining = budget.value
      const result = []

      for (const item of recommendations.value) {
        if (remaining <= 0) break
        const fullCost = item.suggested_quantity * item.unit_cost
        if (fullCost <= remaining) {
          result.push({ ...item, quantity: item.suggested_quantity })
          remaining -= fullCost
        } else {
          const partialQty = Math.floor(remaining / item.unit_cost)
          if (partialQty > 0) {
            result.push({ ...item, quantity: partialQty })
            remaining -= partialQty * item.unit_cost
          }
        }
      }

      return result
    })

    const totalCost = computed(() =>
      selectedItems.value.reduce((sum, item) => sum + item.quantity * item.unit_cost, 0)
    )

    const remaining = computed(() => budget.value - totalCost.value)

    const getAllocatedQty = (sku) => {
      const found = selectedItems.value.find(i => i.sku === sku)
      return found ? found.quantity : 0
    }

    const isPartial = (sku) => {
      const found = selectedItems.value.find(i => i.sku === sku)
      if (!found) return false
      const rec = recommendations.value.find(i => i.sku === sku)
      return rec && found.quantity < rec.suggested_quantity
    }

    const getLeadTimeLabel = (trend) => {
      if (trend === 'increasing') return 'High priority'
      if (trend === 'stable') return 'Medium priority'
      return 'Low priority'
    }

    const getPriorityClass = (trend) => {
      if (trend === 'increasing') return 'high'
      if (trend === 'stable') return 'medium'
      return 'low'
    }

    const loadData = async () => {
      loading.value = true
      error.value = null
      try {
        const [inventory, forecasts] = await Promise.all([
          api.getInventory(),
          api.getDemandForecasts()
        ])
        inventoryItems.value = inventory
        demandForecasts.value = forecasts
        // Set initial budget to 50% of max (after computed is ready, use nextTick-free approach)
      } catch (err) {
        error.value = 'Failed to load restocking data'
        console.error(err)
      } finally {
        loading.value = false
      }
    }

    // Watch budgetMax to set initial budget once data is loaded
    const initBudget = computed(() => {
      const max = budgetMax.value
      if (max > 0 && budget.value === 0) {
        return Math.round(max * 0.5 / 100) * 100
      }
      return budget.value
    })

    // We can't directly set budget from a computed, so we track and update on load
    const placeOrder = async () => {
      if (selectedItems.value.length === 0) return
      submitting.value = true
      submitError.value = null
      successMessage.value = ''
      try {
        const payload = selectedItems.value.map(item => ({
          sku: item.sku,
          name: item.name,
          quantity: item.quantity,
          unit_cost: item.unit_cost,
          warehouse: item.warehouse,
          category: item.category,
          trend: item.trend
        }))
        const result = await api.createRestockingOrder(payload)
        successMessage.value = `Order ${result.order_number} submitted successfully`
        // Reset budget to zero to clear selection
        budget.value = 0
      } catch (err) {
        submitError.value = 'Failed to submit order. Please try again.'
        console.error(err)
      } finally {
        submitting.value = false
      }
    }

    onMounted(async () => {
      await loadData()
      // Set initial budget to 50% of max after data loads
      if (budgetMax.value > 0) {
        budget.value = Math.round(budgetMax.value * 0.5 / 100) * 100
      }
    })

    return {
      currencySymbol,
      loading,
      error,
      submitting,
      successMessage,
      submitError,
      budget,
      budgetMax,
      recommendations,
      selectedItems,
      totalCost,
      remaining,
      getAllocatedQty,
      isPartial,
      getLeadTimeLabel,
      getPriorityClass,
      placeOrder
    }
  }
}
</script>

<style scoped>
.restocking {
  padding-bottom: 2rem;
}

.budget-card .card-header {
  align-items: center;
}

.budget-display {
  font-size: 2rem;
  font-weight: 700;
  color: #2563eb;
  letter-spacing: -0.025em;
}

.slider-wrapper {
  padding: 0.5rem 0 0.25rem;
}

.budget-slider {
  width: 100%;
  height: 6px;
  -webkit-appearance: none;
  appearance: none;
  background: #e2e8f0;
  border-radius: 3px;
  outline: none;
  cursor: pointer;
}

.budget-slider::-webkit-slider-thumb {
  -webkit-appearance: none;
  appearance: none;
  width: 20px;
  height: 20px;
  border-radius: 50%;
  background: #2563eb;
  cursor: pointer;
  border: 2px solid white;
  box-shadow: 0 1px 4px rgba(37, 99, 235, 0.4);
}

.budget-slider::-moz-range-thumb {
  width: 20px;
  height: 20px;
  border-radius: 50%;
  background: #2563eb;
  cursor: pointer;
  border: 2px solid white;
  box-shadow: 0 1px 4px rgba(37, 99, 235, 0.4);
}

.slider-labels {
  display: flex;
  justify-content: space-between;
  margin-top: 0.375rem;
  font-size: 0.75rem;
  color: #94a3b8;
}

.budget-summary {
  margin-top: 0.75rem;
  font-size: 0.875rem;
  color: #64748b;
  font-weight: 500;
}

.restock-table {
  table-layout: auto;
  width: 100%;
}

.mono {
  font-family: 'Menlo', 'Monaco', 'Courier New', monospace;
  font-size: 0.813rem;
}

.row-selected {
  background: #f0f9ff;
}

.row-partial {
  background: #fffbeb;
}

.qty-chip {
  display: inline-block;
  padding: 0.188rem 0.625rem;
  border-radius: 20px;
  background: #dbeafe;
  color: #1e40af;
  font-weight: 700;
  font-size: 0.813rem;
}

.qty-chip.qty-partial {
  background: #fef3c7;
  color: #92400e;
}

.qty-zero {
  color: #cbd5e1;
}

.order-section {
  padding: 1.25rem 0 0.25rem;
  display: flex;
  flex-direction: column;
  gap: 0.75rem;
}

.place-order-btn {
  width: 100%;
  padding: 0.875rem 1.5rem;
  background: #2563eb;
  color: white;
  border: none;
  border-radius: 8px;
  font-size: 1rem;
  font-weight: 600;
  cursor: pointer;
  transition: background 0.2s ease;
}

.place-order-btn:hover:not(:disabled) {
  background: #1d4ed8;
}

.place-order-btn:disabled {
  background: #94a3b8;
  cursor: not-allowed;
}

.success-message {
  background: #d1fae5;
  border: 1px solid #6ee7b7;
  color: #065f46;
  padding: 0.875rem 1rem;
  border-radius: 8px;
  font-size: 0.938rem;
  font-weight: 500;
}

.empty-state {
  text-align: center;
  padding: 3rem;
  color: #64748b;
  font-size: 0.938rem;
}
</style>
