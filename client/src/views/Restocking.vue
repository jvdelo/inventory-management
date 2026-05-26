<template>
  <div class="restocking">
    <div class="page-header">
      <h2>Restocking</h2>
      <p>Set a budget and we'll recommend items to reorder based on demand forecasts.</p>
    </div>

    <div class="card budget-card">
      <div class="card-header">
        <h3 class="card-title">Available Budget</h3>
        <div class="budget-display">${{ budget.toLocaleString() }}</div>
      </div>
      <div class="slider-row">
        <span class="slider-bound">${{ MIN_BUDGET.toLocaleString() }}</span>
        <input
          type="range"
          class="budget-slider"
          :min="MIN_BUDGET"
          :max="MAX_BUDGET"
          :step="STEP"
          v-model.number="budget"
        />
        <span class="slider-bound">${{ MAX_BUDGET.toLocaleString() }}</span>
      </div>
    </div>

    <div v-if="loading" class="loading">Calculating recommendations…</div>
    <div v-else-if="error" class="error">{{ error }}</div>
    <div v-else-if="data">
      <div class="stats-grid">
        <div class="stat-card info">
          <div class="stat-label">Recommended Items</div>
          <div class="stat-value">{{ data.item_count }}</div>
        </div>
        <div class="stat-card success">
          <div class="stat-label">Total Cost</div>
          <div class="stat-value">${{ data.total_cost.toLocaleString() }}</div>
        </div>
        <div class="stat-card">
          <div class="stat-label">Remaining Budget</div>
          <div class="stat-value">${{ data.remaining_budget.toLocaleString() }}</div>
        </div>
        <div class="stat-card warning">
          <div class="stat-label">Expected Delivery</div>
          <div class="stat-value lead">{{ data.longest_lead_time_days }} <span class="unit">days</span></div>
        </div>
      </div>

      <div class="card">
        <div class="card-header">
          <h3 class="card-title">Recommended Restock ({{ data.recommendations.length }})</h3>
          <button
            class="place-order-btn"
            :disabled="data.recommendations.length === 0 || submitting"
            @click="placeOrder"
          >
            {{ submitting ? 'Submitting…' : 'Place Order' }}
          </button>
        </div>

        <div v-if="successMessage" class="success-banner">
          {{ successMessage }}
        </div>

        <div v-if="data.recommendations.length === 0" class="empty">
          No items can be ordered with this budget. Try increasing it.
        </div>
        <div v-else class="table-container">
          <table>
            <thead>
              <tr>
                <th>Item</th>
                <th>Category</th>
                <th>Trend</th>
                <th class="num">Demand Gap</th>
                <th class="num">Unit Cost</th>
                <th class="num">Order Qty</th>
                <th class="num">Line Total</th>
                <th class="num">Lead Time</th>
              </tr>
            </thead>
            <tbody>
              <tr v-for="rec in data.recommendations" :key="rec.sku">
                <td>
                  <div class="item-name">{{ rec.name }}</div>
                  <div class="item-sku">{{ rec.sku }}</div>
                </td>
                <td>{{ rec.category }}</td>
                <td>
                  <span :class="['badge', rec.trend]">{{ rec.trend }}</span>
                </td>
                <td class="num">{{ rec.gap }}</td>
                <td class="num">${{ rec.unit_cost.toFixed(2) }}</td>
                <td class="num">
                  <strong>{{ rec.recommended_qty }}</strong>
                  <span v-if="rec.is_partial" class="partial-tag">partial</span>
                </td>
                <td class="num"><strong>${{ rec.line_total.toLocaleString() }}</strong></td>
                <td class="num">{{ rec.lead_time_days }} days</td>
              </tr>
            </tbody>
          </table>
        </div>
      </div>
    </div>
  </div>
</template>

<script>
import { ref, computed, onMounted, watch } from 'vue'
import { api } from '../api'

const MIN_BUDGET = 500
const MAX_BUDGET = 20000
const STEP = 500
const DEFAULT_BUDGET = 5000

export default {
  name: 'Restocking',
  setup() {
    const budget = ref(DEFAULT_BUDGET)
    const data = ref(null)
    const loading = ref(false)
    const error = ref(null)
    const submitting = ref(false)
    const successMessage = ref('')

    // Debounce timer for slider drags — avoid hammering the API on every tick
    let debounceHandle = null

    const loadRecommendations = async () => {
      try {
        loading.value = true
        error.value = null
        data.value = await api.getRestockingRecommendations(budget.value)
      } catch (err) {
        error.value = 'Failed to load recommendations: ' + err.message
      } finally {
        loading.value = false
      }
    }

    watch(budget, () => {
      // Clearing the success banner is intentional — once the user changes the
      // budget, the previously submitted plan is no longer what's on screen.
      successMessage.value = ''
      if (debounceHandle) clearTimeout(debounceHandle)
      debounceHandle = setTimeout(loadRecommendations, 200)
    })

    const placeOrder = async () => {
      if (!data.value || data.value.recommendations.length === 0) return
      try {
        submitting.value = true
        const items = data.value.recommendations.map(r => ({
          sku: r.sku,
          quantity: r.recommended_qty
        }))
        const order = await api.submitRestockingOrder(items)
        successMessage.value =
          `Order ${order.order_number} submitted — $${order.total_value.toLocaleString()} ` +
          `across ${order.items.length} item(s), expected ${order.expected_delivery} ` +
          `(${order.lead_time_days} days). View it on the Orders tab.`
      } catch (err) {
        error.value = 'Failed to submit order: ' + (err.response?.data?.detail || err.message)
      } finally {
        submitting.value = false
      }
    }

    onMounted(loadRecommendations)

    return {
      MIN_BUDGET,
      MAX_BUDGET,
      STEP,
      budget,
      data,
      loading,
      error,
      submitting,
      successMessage,
      placeOrder
    }
  }
}
</script>

<style scoped>
.budget-card {
  padding-bottom: 1.5rem;
}

.budget-display {
  font-size: 1.75rem;
  font-weight: 700;
  color: #2563eb;
  letter-spacing: -0.025em;
}

.slider-row {
  display: flex;
  align-items: center;
  gap: 1rem;
  margin-top: 0.5rem;
}

.slider-bound {
  font-size: 0.813rem;
  color: #64748b;
  font-weight: 500;
  min-width: 80px;
}

.slider-bound:last-child {
  text-align: right;
}

.budget-slider {
  flex: 1;
  -webkit-appearance: none;
  appearance: none;
  height: 6px;
  background: #e2e8f0;
  border-radius: 999px;
  outline: none;
  cursor: pointer;
}

.budget-slider::-webkit-slider-thumb {
  -webkit-appearance: none;
  appearance: none;
  width: 22px;
  height: 22px;
  border-radius: 50%;
  background: #2563eb;
  border: 3px solid white;
  box-shadow: 0 1px 3px rgba(0, 0, 0, 0.2);
  cursor: grab;
}

.budget-slider::-webkit-slider-thumb:active {
  cursor: grabbing;
}

.budget-slider::-moz-range-thumb {
  width: 22px;
  height: 22px;
  border-radius: 50%;
  background: #2563eb;
  border: 3px solid white;
  box-shadow: 0 1px 3px rgba(0, 0, 0, 0.2);
  cursor: grab;
}

.lead .unit {
  font-size: 1rem;
  font-weight: 500;
  color: #64748b;
  margin-left: 0.25rem;
}

.place-order-btn {
  padding: 0.625rem 1.5rem;
  background: #2563eb;
  color: white;
  border: none;
  border-radius: 6px;
  font-weight: 600;
  font-size: 0.938rem;
  cursor: pointer;
  transition: background-color 0.15s ease;
}

.place-order-btn:hover:not(:disabled) {
  background: #1d4ed8;
}

.place-order-btn:disabled {
  background: #cbd5e1;
  cursor: not-allowed;
}

.success-banner {
  background: #ecfdf5;
  border: 1px solid #a7f3d0;
  color: #065f46;
  padding: 0.875rem 1rem;
  border-radius: 8px;
  margin-bottom: 1rem;
  font-size: 0.875rem;
  line-height: 1.5;
}

.empty {
  text-align: center;
  padding: 2.5rem;
  color: #64748b;
  font-size: 0.938rem;
}

.num {
  text-align: right;
  font-variant-numeric: tabular-nums;
}

.item-name {
  font-size: 0.875rem;
  font-weight: 500;
  color: #0f172a;
}

.item-sku {
  font-size: 0.75rem;
  color: #64748b;
  font-family: 'SF Mono', 'Consolas', monospace;
  margin-top: 0.125rem;
}

.partial-tag {
  display: inline-block;
  margin-left: 0.5rem;
  padding: 0.125rem 0.4rem;
  background: #fef3c7;
  color: #92400e;
  border-radius: 4px;
  font-size: 0.7rem;
  font-weight: 600;
  text-transform: uppercase;
  letter-spacing: 0.04em;
}
</style>
