<template>
  <div class="restocking">
    <div class="page-header">
      <h2>{{ t('restocking.title') }}</h2>
    </div>

    <!-- Success banner -->
    <div v-if="successMessage" class="success-banner">
      {{ successMessage }}
    </div>

    <div v-if="loading" class="loading">{{ t('common.loading') }}</div>
    <div v-else-if="error" class="error">{{ error }}</div>
    <div v-else>
      <!-- Controls card -->
      <div class="card controls-card">
        <div class="controls-grid">
          <!-- Budget slider -->
          <div class="control-group budget-group">
            <label class="control-label">
              {{ t('restocking.budget') }}
              <span class="budget-value">${{ budget.toLocaleString() }}</span>
            </label>
            <input
              v-model.number="budget"
              type="range"
              min="0"
              max="50000"
              step="500"
              class="budget-slider"
            />
            <div class="slider-range-labels">
              <span>$0</span>
              <span>$50,000</span>
            </div>
          </div>

          <!-- Lead time input -->
          <div class="control-group lead-time-group">
            <label class="control-label">{{ t('restocking.leadTime') }}</label>
            <div class="lead-time-input-wrap">
              <input
                v-model.number="leadTime"
                type="number"
                min="1"
                max="90"
                class="lead-time-input"
              />
              <span class="lead-time-unit">days</span>
            </div>
          </div>
        </div>
      </div>

      <!-- Recommendations table card -->
      <div class="card">
        <div class="card-header">
          <h3 class="card-title">{{ t('restocking.recommendations') }}</h3>
        </div>
        <div class="table-container">
          <table class="restocking-table">
            <thead>
              <tr>
                <th class="col-check"></th>
                <th class="col-name">Item Name</th>
                <th class="col-sku">SKU</th>
                <th class="col-trend">Trend</th>
                <th class="col-demand">Current Demand</th>
                <th class="col-demand">Forecasted Demand</th>
                <th class="col-qty">{{ t('restocking.toOrder') }}</th>
                <th class="col-cost">Unit Cost</th>
                <th class="col-subtotal">Subtotal</th>
              </tr>
            </thead>
            <tbody>
              <tr
                v-for="item in tableItems"
                :key="item.id"
                :class="{ 'row-selected': item.checked, 'row-zero-qty': item.qty_to_order === 0 }"
              >
                <td class="col-check">
                  <input
                    type="checkbox"
                    :checked="item.checked"
                    :disabled="item.qty_to_order === 0"
                    @change="toggleItem(item.id)"
                    class="row-checkbox"
                  />
                </td>
                <td class="col-name"><span class="item-name">{{ item.item_name }}</span></td>
                <td class="col-sku"><code class="sku-code">{{ item.item_sku }}</code></td>
                <td class="col-trend">
                  <span :class="['badge', item.trend]">{{ item.trend }}</span>
                </td>
                <td class="col-demand">{{ item.current_demand.toLocaleString() }}</td>
                <td class="col-demand">{{ item.forecasted_demand.toLocaleString() }}</td>
                <td class="col-qty">
                  <input
                    v-if="item.checked"
                    v-model.number="item.edit_qty"
                    type="number"
                    min="0"
                    class="qty-input"
                    @input="updateQty(item.id, item.edit_qty)"
                  />
                  <span v-else class="qty-display">{{ item.qty_to_order.toLocaleString() }}</span>
                </td>
                <td class="col-cost">${{ item.unit_cost.toLocaleString('en-US', { minimumFractionDigits: 2, maximumFractionDigits: 2 }) }}</td>
                <td class="col-subtotal">
                  <strong :class="{ 'subtotal-active': item.checked }">
                    ${{ computeSubtotal(item).toLocaleString('en-US', { minimumFractionDigits: 2, maximumFractionDigits: 2 }) }}
                  </strong>
                </td>
              </tr>
            </tbody>
          </table>
        </div>
      </div>

      <!-- Summary bar -->
      <div class="summary-bar">
        <div class="summary-stats">
          <div class="summary-stat">
            <span class="summary-label">Selected</span>
            <span class="summary-value">{{ selectedCount }} items</span>
          </div>
          <div class="summary-stat">
            <span class="summary-label">{{ t('restocking.totalCost') }}</span>
            <span class="summary-value">${{ totalCost.toLocaleString('en-US', { minimumFractionDigits: 2, maximumFractionDigits: 2 }) }}</span>
          </div>
          <div class="summary-stat">
            <span class="summary-label">{{ t('restocking.remaining') }}</span>
            <span :class="['summary-value', { 'over-budget': remainingBudget < 0 }]">
              ${{ remainingBudget.toLocaleString('en-US', { minimumFractionDigits: 2, maximumFractionDigits: 2 }) }}
            </span>
          </div>
        </div>
        <button
          class="place-order-btn"
          :disabled="selectedCount === 0 || totalCost === 0 || submitting"
          @click="placeOrder"
        >
          {{ submitting ? 'Placing Order...' : t('restocking.placeOrder') }}
        </button>
      </div>
    </div>
  </div>
</template>

<script>
import { ref, computed, onMounted, watch } from 'vue'
import { api } from '../api'
import { useI18n } from '../composables/useI18n'

export default {
  name: 'Restocking',
  setup() {
    const { t } = useI18n()

    const loading = ref(true)
    const error = ref(null)
    const submitting = ref(false)
    const successMessage = ref(null)
    const successTimer = ref(null)

    // Raw forecast data from API
    const forecasts = ref([])

    // Controls
    const budget = ref(20000)
    const leadTime = ref(7)

    // Mutable table rows: each row extends a forecast item with reactive edit_qty and checked state
    const tableItems = ref([])

    // Trend score for auto-select algorithm
    const trendScore = (trend) => {
      if (trend === 'increasing') return 3
      if (trend === 'stable') return 2
      return 1 // decreasing
    }

    // Build initial table rows from forecasts, applying auto-select for the given budget
    const buildTableItems = (data, currentBudget) => {
      // Compute qty_to_order per item
      const items = data.map(f => ({
        ...f,
        qty_to_order: Math.max(0, f.forecasted_demand - f.current_demand),
        edit_qty: Math.max(0, f.forecasted_demand - f.current_demand),
        checked: false
      }))

      // Sort: trendScore desc, then demand gap desc
      const eligible = items
        .filter(i => i.qty_to_order > 0)
        .slice()
        .sort((a, b) => {
          const scoreDiff = trendScore(b.trend) - trendScore(a.trend)
          if (scoreDiff !== 0) return scoreDiff
          const gapA = a.forecasted_demand - a.current_demand
          const gapB = b.forecasted_demand - b.current_demand
          return gapB - gapA
        })

      // Greedy selection within budget
      const selected = new Set()
      let remaining = currentBudget
      for (const item of eligible) {
        const cost = item.qty_to_order * item.unit_cost
        if (cost <= remaining) {
          selected.add(item.id)
          remaining -= cost
        }
      }

      // Apply checked state
      return items.map(i => ({ ...i, checked: selected.has(i.id) }))
    }

    // Re-run auto-select whenever budget changes, preserving manual qty edits
    watch(budget, (newBudget) => {
      if (forecasts.value.length === 0) return
      const rebuilt = buildTableItems(forecasts.value, newBudget)
      // Merge: keep existing edit_qty values but update checked state from auto-select
      tableItems.value = rebuilt.map(rebuilt_item => {
        const existing = tableItems.value.find(t => t.id === rebuilt_item.id)
        return {
          ...rebuilt_item,
          edit_qty: existing ? existing.edit_qty : rebuilt_item.edit_qty
        }
      })
    })

    const loadForecasts = async () => {
      try {
        loading.value = true
        error.value = null
        const data = await api.getDemandForecasts()
        forecasts.value = data
        tableItems.value = buildTableItems(data, budget.value)
      } catch (err) {
        error.value = t('common.error') + ': ' + err.message
        console.error('Failed to load demand forecasts:', err)
      } finally {
        loading.value = false
      }
    }

    // Toggle a row's checked state manually
    const toggleItem = (id) => {
      const item = tableItems.value.find(i => i.id === id)
      if (item) item.checked = !item.checked
    }

    // Update edit_qty for an item
    const updateQty = (id, qty) => {
      const item = tableItems.value.find(i => i.id === id)
      if (item) {
        item.edit_qty = Math.max(0, qty || 0)
      }
    }

    // Subtotal for a single row (uses edit_qty if checked, else qty_to_order)
    const computeSubtotal = (item) => {
      const qty = item.checked ? (item.edit_qty || 0) : item.qty_to_order
      return qty * item.unit_cost
    }

    // Derived summary values
    const selectedCount = computed(() => tableItems.value.filter(i => i.checked).length)

    const totalCost = computed(() =>
      tableItems.value
        .filter(i => i.checked)
        .reduce((sum, i) => sum + computeSubtotal(i), 0)
    )

    const remainingBudget = computed(() => budget.value - totalCost.value)

    // Place order
    const placeOrder = async () => {
      submitting.value = true
      error.value = null
      try {
        const selectedItems = tableItems.value
          .filter(i => i.checked && (i.edit_qty || 0) > 0)
          .map(i => ({
            sku: i.item_sku,
            name: i.item_name,
            quantity: i.edit_qty || 0,
            unit_cost: i.unit_cost,
            subtotal: computeSubtotal(i)
          }))

        const result = await api.createRestockingOrder({
          lead_time_days: leadTime.value,
          items: selectedItems
        })

        // Show success banner for 4 seconds then reset
        const orderId = result.order_id || result.id || 'RST-' + Date.now()
        successMessage.value = `${t('restocking.orderPlaced')} — Order ID: ${orderId}`

        if (successTimer.value) clearTimeout(successTimer.value)
        successTimer.value = setTimeout(() => {
          successMessage.value = null
          // Reset form to defaults
          budget.value = 20000
          leadTime.value = 7
          tableItems.value = buildTableItems(forecasts.value, 20000)
        }, 4000)
      } catch (err) {
        error.value = 'Failed to place order: ' + err.message
        console.error('Failed to place restocking order:', err)
      } finally {
        submitting.value = false
      }
    }

    onMounted(loadForecasts)

    return {
      t,
      loading,
      error,
      submitting,
      successMessage,
      budget,
      leadTime,
      tableItems,
      selectedCount,
      totalCost,
      remainingBudget,
      toggleItem,
      updateQty,
      computeSubtotal,
      placeOrder
    }
  }
}
</script>

<style scoped>
.restocking {
  /* Page wrapper — main-content provides padding */
}

/* Success banner */
.success-banner {
  background: #d1fae5;
  border: 1px solid #6ee7b7;
  color: #065f46;
  padding: 0.875rem 1.25rem;
  border-radius: 8px;
  font-size: 0.938rem;
  font-weight: 500;
  margin-bottom: 1.25rem;
}

/* Controls card layout */
.controls-card {
  margin-bottom: 1.25rem;
}

.controls-grid {
  display: grid;
  grid-template-columns: 1fr 200px;
  gap: 2rem;
  align-items: start;
}

.control-group {
  display: flex;
  flex-direction: column;
  gap: 0.5rem;
}

.control-label {
  font-size: 0.875rem;
  font-weight: 600;
  color: #475569;
  text-transform: uppercase;
  letter-spacing: 0.05em;
  display: flex;
  align-items: center;
  gap: 0.75rem;
}

.budget-value {
  font-size: 1rem;
  font-weight: 700;
  color: #0f172a;
  text-transform: none;
  letter-spacing: 0;
}

/* Budget slider */
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
  width: 18px;
  height: 18px;
  border-radius: 50%;
  background: #2563eb;
  cursor: pointer;
  border: 2px solid white;
  box-shadow: 0 1px 4px rgba(0, 0, 0, 0.2);
}

.budget-slider::-moz-range-thumb {
  width: 18px;
  height: 18px;
  border-radius: 50%;
  background: #2563eb;
  cursor: pointer;
  border: 2px solid white;
  box-shadow: 0 1px 4px rgba(0, 0, 0, 0.2);
}

.slider-range-labels {
  display: flex;
  justify-content: space-between;
  font-size: 0.75rem;
  color: #94a3b8;
}

/* Lead time input */
.lead-time-input-wrap {
  display: flex;
  align-items: center;
  gap: 0.5rem;
}

.lead-time-input {
  width: 80px;
  padding: 0.5rem 0.625rem;
  border: 1px solid #e2e8f0;
  border-radius: 6px;
  font-size: 0.938rem;
  color: #0f172a;
  background: white;
  outline: none;
  transition: border-color 0.15s;
}

.lead-time-input:focus {
  border-color: #2563eb;
  box-shadow: 0 0 0 2px rgba(37, 99, 235, 0.12);
}

.lead-time-unit {
  font-size: 0.875rem;
  color: #64748b;
}

/* Restocking table */
.restocking-table {
  table-layout: fixed;
  width: 100%;
}

.col-check  { width: 40px; }
.col-name   { width: 200px; }
.col-sku    { width: 140px; }
.col-trend  { width: 110px; }
.col-demand { width: 120px; }
.col-qty    { width: 120px; }
.col-cost   { width: 110px; }
.col-subtotal { width: 130px; }

/* Row states */
.row-selected {
  background: #f0f9ff;
}

.row-zero-qty {
  opacity: 0.45;
}

.row-checkbox {
  width: 16px;
  height: 16px;
  cursor: pointer;
  accent-color: #2563eb;
}

.item-name {
  font-weight: 500;
  color: #0f172a;
}

.sku-code {
  font-family: 'Menlo', 'Monaco', 'Courier New', monospace;
  font-size: 0.813rem;
  color: #475569;
  background: #f1f5f9;
  padding: 0.125rem 0.375rem;
  border-radius: 4px;
}

/* Qty editable input */
.qty-input {
  width: 80px;
  padding: 0.313rem 0.5rem;
  border: 1px solid #e2e8f0;
  border-radius: 5px;
  font-size: 0.875rem;
  color: #0f172a;
  background: white;
  outline: none;
  transition: border-color 0.15s;
}

.qty-input:focus {
  border-color: #2563eb;
  box-shadow: 0 0 0 2px rgba(37, 99, 235, 0.1);
}

.qty-display {
  color: #64748b;
  font-size: 0.875rem;
}

.subtotal-active {
  color: #0f172a;
}

/* Summary bar */
.summary-bar {
  background: white;
  border: 1px solid #e2e8f0;
  border-radius: 10px;
  padding: 1rem 1.5rem;
  display: flex;
  align-items: center;
  justify-content: space-between;
  gap: 2rem;
  margin-top: 0.5rem;
  box-shadow: 0 1px 3px rgba(0, 0, 0, 0.05);
}

.summary-stats {
  display: flex;
  gap: 2.5rem;
}

.summary-stat {
  display: flex;
  flex-direction: column;
  gap: 0.25rem;
}

.summary-label {
  font-size: 0.75rem;
  font-weight: 600;
  color: #64748b;
  text-transform: uppercase;
  letter-spacing: 0.05em;
}

.summary-value {
  font-size: 1.125rem;
  font-weight: 700;
  color: #0f172a;
}

.summary-value.over-budget {
  color: #dc2626;
}

/* Place order button */
.place-order-btn {
  padding: 0.75rem 2rem;
  background: #2563eb;
  color: white;
  border: none;
  border-radius: 8px;
  font-size: 0.938rem;
  font-weight: 600;
  cursor: pointer;
  transition: background 0.15s, box-shadow 0.15s;
  white-space: nowrap;
  flex-shrink: 0;
}

.place-order-btn:hover:not(:disabled) {
  background: #1d4ed8;
  box-shadow: 0 4px 12px rgba(37, 99, 235, 0.3);
}

.place-order-btn:disabled {
  background: #cbd5e1;
  color: #94a3b8;
  cursor: not-allowed;
}
</style>
