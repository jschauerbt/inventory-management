<template>
  <div class="restocking">
    <div class="page-header">
      <h2>Restocking</h2>
      <p>Set your budget and place a restocking order based on demand forecasts.</p>
    </div>

    <div v-if="loading" class="loading">Loading...</div>
    <div v-else-if="error" class="error">{{ error }}</div>
    <div v-else>

      <!-- Budget Card -->
      <div class="card" style="margin-bottom: 1.5rem;">
        <div class="card-header">
          <h3 class="card-title">Restocking Budget</h3>
        </div>
        <div class="card-body">
          <input
            type="range"
            :min="sliderMin"
            :max="sliderMax"
            step="1"
            v-model.number="budget"
            class="budget-slider"
          />
          <div class="budget-stats">
            <div class="budget-stat">
              <div class="budget-stat-label">Budget</div>
              <div class="budget-stat-value">{{ formatCurrencyValue(budget) }}</div>
            </div>
            <div class="budget-stat">
              <div class="budget-stat-label">Allocated</div>
              <div
                class="budget-stat-value"
                :style="{ color: totalCost > 0 ? '#d97706' : '#0f172a' }"
              >{{ formatCurrencyValue(totalCost) }}</div>
            </div>
            <div class="budget-stat">
              <div class="budget-stat-label">Remaining</div>
              <div
                class="budget-stat-value"
                :style="{ color: remainingBudget > 0 ? '#059669' : remainingBudget < 0 ? '#dc2626' : '#0f172a' }"
              >{{ formatCurrencyValue(remainingBudget) }}</div>
            </div>
          </div>
        </div>
      </div>

      <!-- Recommendations Card -->
      <div class="card">
        <div class="card-header">
          <h3 class="card-title">Recommended Items</h3>
          <p class="card-subtitle">Sorted by forecasted demand (highest first). Items are included if their full forecasted quantity fits within budget.</p>
        </div>

        <div class="table-container">
          <table>
            <thead>
              <tr>
                <th>Item Name</th>
                <th>SKU</th>
                <th>Trend</th>
                <th>Forecasted Qty</th>
                <th>Unit Cost</th>
                <th>Subtotal</th>
              </tr>
            </thead>
            <tbody>
              <tr v-if="recommendations.length === 0">
                <td colspan="6" class="empty-state">
                  No items fit within the current budget. Increase the budget to see recommendations.
                </td>
              </tr>
              <tr v-for="item in recommendations" :key="item.item_sku">
                <td>{{ item.item_name }}</td>
                <td><strong>{{ item.item_sku }}</strong></td>
                <td>
                  <span :class="['badge', getTrend(item.item_sku)]">
                    {{ getTrend(item.item_sku) }}
                  </span>
                </td>
                <td>{{ item.quantity.toLocaleString() }}</td>
                <td>{{ formatCurrencyValue(item.unit_cost) }}</td>
                <td>{{ formatCurrencyValue(item.subtotal) }}</td>
              </tr>
              <tr v-if="recommendations.length > 0" class="total-row">
                <td colspan="5"><strong>Total</strong></td>
                <td><strong>{{ formatCurrencyValue(totalCost) }}</strong></td>
              </tr>
            </tbody>
          </table>
        </div>

        <div class="order-actions">
          <button
            class="btn-primary"
            :disabled="placing || !recommendations.length || orderPlaced"
            @click="placeOrder"
          >
            {{ placing ? 'Placing...' : 'Place Order' }}
          </button>

          <div v-if="orderPlaced" class="success-banner">
            <span>Order placed successfully. Check the Orders tab to see your restocking order.</span>
            <button class="btn-reset" @click="orderPlaced = false">Place Another Order</button>
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

    const loading = ref(true)
    const error = ref(null)
    const forecasts = ref([])
    const budget = ref(0)
    const placing = ref(false)
    const orderPlaced = ref(false)

    // Currency symbol derived from locale
    const currencySymbol = computed(() => {
      return currentCurrency.value === 'JPY' ? '¥' : '$'
    })

    const formatCurrencyValue = (value) => {
      if (currentCurrency.value === 'JPY') {
        return currencySymbol.value + Math.round(value).toLocaleString('en-US')
      }
      return currencySymbol.value + value.toLocaleString('en-US', {
        minimumFractionDigits: 2,
        maximumFractionDigits: 2
      })
    }

    const sliderMin = computed(() => {
      if (!forecasts.value.length) return 0
      return Math.min(...forecasts.value.map(f => f.unit_cost))
    })

    const sliderMax = computed(() => {
      if (!forecasts.value.length) return 0
      return Math.round(
        forecasts.value.reduce((sum, f) => sum + f.forecasted_demand * f.unit_cost, 0)
      )
    })

    const recommendations = computed(() => {
      const sorted = [...forecasts.value].sort((a, b) => b.forecasted_demand - a.forecasted_demand)
      const result = []
      let remaining = budget.value
      for (const item of sorted) {
        const cost = Math.round(item.forecasted_demand * item.unit_cost * 100) / 100
        if (cost <= remaining) {
          result.push({
            item_sku: item.item_sku,
            item_name: item.item_name,
            quantity: item.forecasted_demand,
            unit_cost: item.unit_cost,
            subtotal: cost
          })
          remaining -= cost
        }
      }
      return result
    })

    const totalCost = computed(() =>
      Math.round(recommendations.value.reduce((s, i) => s + i.subtotal, 0) * 100) / 100
    )

    const remainingBudget = computed(() =>
      Math.round((budget.value - totalCost.value) * 100) / 100
    )

    // Look up trend from the raw forecasts ref by SKU
    const getTrend = (sku) => {
      const forecast = forecasts.value.find(f => f.item_sku === sku)
      return forecast ? forecast.trend : 'stable'
    }

    const loadForecasts = async () => {
      try {
        loading.value = true
        error.value = null
        forecasts.value = await api.getDemandForecasts()
        // Set budget to midpoint of range after data loads
        budget.value = Math.round((sliderMin.value + sliderMax.value) / 2)
      } catch (err) {
        error.value = 'Failed to load demand forecasts: ' + err.message
      } finally {
        loading.value = false
      }
    }

    const placeOrder = async () => {
      if (!recommendations.value.length || placing.value || orderPlaced.value) return
      try {
        placing.value = true
        error.value = null
        await api.createRestockingOrder(recommendations.value)
        orderPlaced.value = true
      } catch (err) {
        error.value = 'Failed to place order: ' + err.message
      } finally {
        placing.value = false
      }
    }

    onMounted(loadForecasts)

    return {
      loading,
      error,
      forecasts,
      budget,
      placing,
      orderPlaced,
      sliderMin,
      sliderMax,
      recommendations,
      totalCost,
      remainingBudget,
      formatCurrencyValue,
      getTrend,
      placeOrder
    }
  }
}
</script>

<style scoped>
.restocking {
  /* page uses global .main-content padding */
}

.card-body {
  padding-top: 0.5rem;
}

.card-subtitle {
  font-size: 0.813rem;
  color: #64748b;
  font-weight: 400;
  margin-top: 0.25rem;
}

.budget-slider {
  width: 100%;
  height: 6px;
  accent-color: #2563eb;
  cursor: pointer;
  margin: 1rem 0;
}

.budget-stats {
  display: grid;
  grid-template-columns: repeat(3, 1fr);
  gap: 1rem;
  margin-top: 1rem;
}

.budget-stat {
  text-align: center;
  padding: 1rem;
  background: #f8fafc;
  border-radius: 8px;
  border: 1px solid #e2e8f0;
}

.budget-stat-label {
  font-size: 0.75rem;
  font-weight: 600;
  text-transform: uppercase;
  letter-spacing: 0.05em;
  color: #64748b;
  margin-bottom: 0.25rem;
}

.budget-stat-value {
  font-size: 1.5rem;
  font-weight: 700;
  color: #0f172a;
  transition: color 0.2s ease;
}

.empty-state {
  text-align: center;
  color: #64748b;
  padding: 2rem 0.75rem;
  font-style: italic;
}

.total-row td {
  font-weight: 700;
  border-top: 2px solid #e2e8f0;
}

.order-actions {
  padding: 1.25rem 0 0.25rem;
  display: flex;
  flex-direction: column;
  gap: 0;
}

.btn-primary {
  background: #2563eb;
  color: white;
  border: none;
  border-radius: 6px;
  padding: 0.625rem 1.5rem;
  font-size: 0.875rem;
  font-weight: 600;
  cursor: pointer;
  align-self: flex-start;
  transition: background 0.2s ease;
}

.btn-primary:hover:not(:disabled) {
  background: #1d4ed8;
}

.btn-primary:disabled {
  opacity: 0.5;
  cursor: not-allowed;
}

.success-banner {
  display: flex;
  align-items: center;
  justify-content: space-between;
  background: #d1fae5;
  border: 1px solid #6ee7b7;
  border-radius: 8px;
  padding: 1rem 1.25rem;
  margin-top: 1rem;
  color: #065f46;
  font-weight: 500;
  gap: 1rem;
}

.btn-reset {
  background: transparent;
  border: 1px solid #059669;
  color: #065f46;
  border-radius: 6px;
  padding: 0.375rem 0.875rem;
  font-size: 0.813rem;
  font-weight: 600;
  cursor: pointer;
  white-space: nowrap;
  transition: background 0.2s ease;
  flex-shrink: 0;
}

.btn-reset:hover {
  background: #a7f3d0;
}
</style>
