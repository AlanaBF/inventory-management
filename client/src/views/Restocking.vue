<template>
  <div class="restocking">
    <div class="page-header">
      <h2>Restocking</h2>
      <p>Set your available budget and submit a restocking order based on demand forecasts.</p>
    </div>

    <div v-if="loading" class="loading">Loading...</div>
    <div v-else-if="error" class="error">{{ error }}</div>
    <div v-else>
      <div class="card">
        <div class="card-header">
          <h3 class="card-title">Available budget</h3>
        </div>
        <div class="budget-readout">{{ formatBudget(budget) }}</div>
        <input
          type="range"
          class="budget-slider"
          min="1000"
          max="100000"
          step="500"
          v-model.number="budget"
        />
        <div class="budget-range-labels">
          <span>Min $1,000</span>
          <span>Max $100,000</span>
        </div>
      </div>

      <div class="card">
        <div class="card-header">
          <h3 class="card-title">Recommended items ({{ suggestions.length }})</h3>
        </div>
        <div class="table-container">
          <table class="restock-table">
            <thead>
              <tr>
                <th class="col-item">Item</th>
                <th class="col-trend">Trend</th>
                <th class="col-num">Forecasted Demand</th>
                <th class="col-cost">Unit Cost</th>
                <th class="col-qty">Quantity</th>
                <th class="col-total">Line Total</th>
              </tr>
            </thead>
            <tbody>
              <tr v-for="row in suggestions" :key="row.item_sku">
                <td class="col-item">
                  <div class="item-name">{{ row.item_name }}</div>
                  <div class="item-sku">{{ row.item_sku }}</div>
                </td>
                <td class="col-trend">
                  <span :class="['badge', row.trend]">{{ row.trend }}</span>
                </td>
                <td class="col-num num">{{ row.forecasted_demand }}</td>
                <td class="col-cost num">{{ formatCurrency(row.unit_cost) }}</td>
                <td class="col-qty">
                  <input
                    type="number"
                    min="0"
                    class="qty-input"
                    :value="effectiveQty(row.item_sku)"
                    @input="setOverride(row.item_sku, $event.target.value)"
                  />
                </td>
                <td class="col-total num"><strong>{{ formatCurrency(lineTotal(row)) }}</strong></td>
              </tr>
              <tr v-if="suggestions.length === 0">
                <td colspan="6" class="empty">No items match the current forecast data.</td>
              </tr>
            </tbody>
          </table>
        </div>
        <div class="order-summary">
          <div class="summary-row">
            <span class="summary-label">Order total</span>
            <span :class="['summary-value', { 'over-budget': overBudget }]">{{ formatCurrency(grandTotal) }}</span>
          </div>
          <div class="summary-row">
            <span class="summary-label">Budget</span>
            <span class="summary-value">{{ formatBudget(budget) }}</span>
          </div>
          <div v-if="overBudget" class="over-budget-note">
            Over budget by {{ formatCurrency(grandTotal - budget) }}
          </div>
        </div>
      </div>

      <div class="action-bar">
        <button class="place-order-btn" :disabled="!canSubmit" @click="placeOrder">
          {{ submitting ? 'Submitting...' : 'Place Order' }}
        </button>
        <div v-if="submitMessage" :class="['submit-message', submitMessage.type]">
          {{ submitMessage.text }}
        </div>
      </div>
    </div>
  </div>
</template>

<script>
import { ref, computed, onMounted } from 'vue'
import { api } from '../api'

export default {
  name: 'Restocking',
  setup() {
    const budget = ref(25000)
    const forecasts = ref([])
    const inventory = ref([])
    const overrides = ref({})
    const loading = ref(true)
    const error = ref(null)
    const submitting = ref(false)
    const submitMessage = ref(null)

    const loadData = async () => {
      try {
        loading.value = true
        error.value = null
        const [f, i] = await Promise.all([
          api.getDemandForecasts(),
          api.getInventory()
        ])
        forecasts.value = f
        inventory.value = i
      } catch (err) {
        error.value = 'Failed to load restocking data: ' + err.message
      } finally {
        loading.value = false
      }
    }
    onMounted(loadData)

    const inventoryBySku = computed(() => {
      const map = {}
      for (const inv of inventory.value) map[inv.sku] = inv
      return map
    })

    const suggestions = computed(() => {
      const trendRank = { increasing: 0, stable: 1, decreasing: 2 }
      const sorted = [...forecasts.value].sort((a, b) => {
        const t = (trendRank[a.trend] ?? 99) - (trendRank[b.trend] ?? 99)
        if (t !== 0) return t
        return (b.forecasted_demand - b.current_demand) - (a.forecasted_demand - a.current_demand)
      })
      let remaining = budget.value
      const rows = []
      for (const f of sorted) {
        const inv = inventoryBySku.value[f.item_sku]
        if (!inv) continue
        // "need" = units to raise stock up to forecasted demand; clamp to >= 0 so already-stocked items don't auto-suggest
        const need = Math.max(0, f.forecasted_demand - inv.quantity_on_hand)
        const affordable = inv.unit_cost > 0 ? Math.floor(remaining / inv.unit_cost) : 0
        const qty = Math.min(need, Math.max(0, affordable))
        remaining -= qty * inv.unit_cost
        rows.push({
          item_sku: f.item_sku,
          item_name: f.item_name,
          trend: f.trend,
          forecasted_demand: f.forecasted_demand,
          unit_cost: inv.unit_cost,
          suggested_qty: qty
        })
      }
      return rows
    })

    const effectiveQty = (sku) => {
      const o = overrides.value[sku]
      if (o !== undefined && o !== null && o !== '') return Number(o)
      const s = suggestions.value.find(r => r.item_sku === sku)
      return s ? s.suggested_qty : 0
    }

    const setOverride = (sku, value) => {
      const n = Number(value)
      overrides.value = { ...overrides.value, [sku]: isNaN(n) || n < 0 ? 0 : n }
    }

    const lineTotal = (row) => effectiveQty(row.item_sku) * row.unit_cost

    const grandTotal = computed(() =>
      suggestions.value.reduce((sum, r) => sum + lineTotal(r), 0)
    )

    const overBudget = computed(() => grandTotal.value > budget.value)

    const canSubmit = computed(() =>
      !submitting.value && grandTotal.value > 0 && !overBudget.value
    )

    const formatCurrency = (v) =>
      '$' + Number(v).toLocaleString('en-US', { minimumFractionDigits: 2, maximumFractionDigits: 2 })

    const formatBudget = (v) =>
      '$' + Number(v).toLocaleString('en-US')

    const placeOrder = async () => {
      submitting.value = true
      submitMessage.value = null
      try {
        const items = suggestions.value
          .map(r => ({
            sku: r.item_sku,
            name: r.item_name,
            quantity: effectiveQty(r.item_sku),
            unit_price: r.unit_cost
          }))
          .filter(i => i.quantity > 0)
        if (items.length === 0) {
          submitMessage.value = { type: 'error', text: 'No items selected for restocking.' }
          return
        }
        const order = await api.createOrder({
          items,
          customer: 'Internal Restock',
          lead_time_days: 7
        })
        submitMessage.value = {
          type: 'success',
          text: `Order ${order.order_number} placed — see Orders tab.`
        }
        overrides.value = {}
      } catch (err) {
        const detail = err.response?.data?.detail || err.message || String(err)
        submitMessage.value = { type: 'error', text: 'Failed to place order: ' + detail }
      } finally {
        submitting.value = false
      }
    }

    return {
      budget,
      loading,
      error,
      submitting,
      submitMessage,
      suggestions,
      grandTotal,
      overBudget,
      canSubmit,
      effectiveQty,
      setOverride,
      lineTotal,
      formatCurrency,
      formatBudget,
      placeOrder
    }
  }
}
</script>

<style scoped>
.budget-readout {
  font-size: 2.5rem;
  font-weight: 700;
  color: #0f172a;
  letter-spacing: -0.025em;
  margin-bottom: 0.75rem;
}

.budget-slider {
  -webkit-appearance: none;
  appearance: none;
  width: 100%;
  height: 6px;
  border-radius: 999px;
  background: #e2e8f0;
  outline: none;
  margin: 0.25rem 0 0.5rem;
}

.budget-slider::-webkit-slider-thumb {
  -webkit-appearance: none;
  appearance: none;
  width: 20px;
  height: 20px;
  border-radius: 50%;
  background: #2563eb;
  cursor: pointer;
  border: 2px solid #ffffff;
  box-shadow: 0 1px 3px rgba(15, 23, 42, 0.2);
}

.budget-slider::-moz-range-thumb {
  width: 20px;
  height: 20px;
  border-radius: 50%;
  background: #2563eb;
  cursor: pointer;
  border: 2px solid #ffffff;
  box-shadow: 0 1px 3px rgba(15, 23, 42, 0.2);
}

.budget-range-labels {
  display: flex;
  justify-content: space-between;
  font-size: 0.813rem;
  color: #64748b;
}

.restock-table {
  table-layout: fixed;
  width: 100%;
}

.col-item { width: 240px; }
.col-trend { width: 120px; }
.col-num { width: 140px; }
.col-cost { width: 120px; }
.col-qty { width: 110px; }
.col-total { width: 130px; }

.num { text-align: right; font-variant-numeric: tabular-nums; }

.item-name {
  font-size: 0.875rem;
  font-weight: 600;
  color: #0f172a;
}

.item-sku {
  font-size: 0.75rem;
  color: #64748b;
  margin-top: 0.125rem;
  font-family: ui-monospace, SFMono-Regular, Menlo, monospace;
}

.qty-input {
  width: 80px;
  padding: 0.375rem 0.5rem;
  border: 1px solid #cbd5e1;
  border-radius: 6px;
  font-size: 0.875rem;
  color: #0f172a;
  background: #ffffff;
  font-variant-numeric: tabular-nums;
  text-align: right;
}

.qty-input:focus {
  outline: none;
  border-color: #2563eb;
  box-shadow: 0 0 0 3px rgba(37, 99, 235, 0.15);
}

.empty {
  text-align: center;
  color: #64748b;
  padding: 1.5rem;
  font-style: italic;
}

.order-summary {
  margin-top: 1rem;
  padding-top: 0.875rem;
  border-top: 1px solid #e2e8f0;
}

.summary-row {
  display: flex;
  justify-content: space-between;
  align-items: center;
  padding: 0.25rem 0;
}

.summary-label {
  font-size: 0.875rem;
  color: #64748b;
  font-weight: 500;
}

.summary-value {
  font-size: 1.125rem;
  font-weight: 700;
  color: #0f172a;
  font-variant-numeric: tabular-nums;
}

.summary-value.over-budget {
  color: #dc2626;
}

.over-budget-note {
  margin-top: 0.5rem;
  font-size: 0.813rem;
  color: #dc2626;
  font-weight: 500;
  text-align: right;
}

.action-bar {
  display: flex;
  align-items: center;
  gap: 1rem;
  margin-top: 0.5rem;
}

.place-order-btn {
  background: #2563eb;
  color: #ffffff;
  padding: 0.75rem 1.5rem;
  font-size: 0.938rem;
  font-weight: 600;
  border: none;
  border-radius: 8px;
  cursor: pointer;
  transition: background 0.15s ease;
}

.place-order-btn:hover:not(:disabled) {
  background: #1d4ed8;
}

.place-order-btn:disabled {
  background: #cbd5e1;
  cursor: not-allowed;
}

.submit-message {
  font-size: 0.875rem;
  font-weight: 500;
}

.submit-message.success {
  color: #059669;
}

.submit-message.error {
  color: #dc2626;
}
</style>
