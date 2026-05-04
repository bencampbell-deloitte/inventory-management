<template>
  <div class="restocking">
    <div class="page-header">
      <h2>Restocking</h2>
      <p>Allocate budget to demand-driven restock recommendations</p>
    </div>

    <div v-if="loading" class="loading">Loading recommendations...</div>
    <div v-else-if="error" class="error">{{ error }}</div>
    <div v-else>

      <!-- Success banner -->
      <div v-if="orderSuccess" class="success-banner">
        Order submitted successfully! It will appear in the Orders tab under "Submitted Restocking Orders".
        <button class="dismiss-btn" @click="orderSuccess = false">Dismiss</button>
      </div>

      <!-- Budget slider card -->
      <div class="card budget-card">
        <div class="budget-label">Available Budget</div>
        <div class="budget-amount">{{ formatCurrency(budget) }}</div>
        <input
          type="range"
          class="budget-slider"
          :min="0"
          :max="totalCost"
          :step="100"
          v-model.number="budget"
        />
        <div class="budget-sublabel">
          Drag to set your restocking budget · Total needed: {{ formatCurrency(totalCost) }}
        </div>
      </div>

      <!-- Recommendations card -->
      <div class="card">
        <div class="card-header">
          <h3 class="card-title">
            Recommended Items
            <span class="count-badge">{{ selectedItems.length }} selected of {{ recommendations.length }}</span>
          </h3>
        </div>

        <div v-if="recommendations.length === 0" class="empty-state">
          No restocking needed — all forecasted demand is covered by current inventory.
        </div>

        <div v-else class="table-container">
          <table class="restock-table">
            <thead>
              <tr>
                <th class="col-check"></th>
                <th class="col-sku">SKU</th>
                <th class="col-name">Item</th>
                <th class="col-trend">Trend</th>
                <th class="col-num">Current Stock</th>
                <th class="col-num">Forecasted</th>
                <th class="col-num">Qty to Restock</th>
                <th class="col-num">Unit Cost</th>
                <th class="col-num">Total Cost</th>
              </tr>
            </thead>
            <tbody>
              <tr
                v-for="item in recommendations"
                :key="item.sku"
                :class="['restock-row', { 'row-excluded': !isSelected(item.sku) }]"
                @click="toggleItem(item.sku)"
              >
                <td class="col-check">
                  <input
                    type="checkbox"
                    :checked="isSelected(item.sku)"
                    @click.stop="toggleItem(item.sku)"
                  />
                </td>
                <td class="col-sku"><code>{{ item.sku }}</code></td>
                <td class="col-name">{{ item.name }}</td>
                <td class="col-trend">
                  <span :class="['badge', item.trend]">{{ item.trend }}</span>
                </td>
                <td class="col-num">{{ item.quantity_on_hand.toLocaleString() }}</td>
                <td class="col-num">{{ item.forecasted_demand.toLocaleString() }}</td>
                <td class="col-num"><strong>{{ item.qty_needed.toLocaleString() }}</strong></td>
                <td class="col-num">{{ formatCurrency(item.unit_cost) }}</td>
                <td class="col-num"><strong>{{ formatCurrency(item.item_cost) }}</strong></td>
              </tr>
            </tbody>
          </table>
        </div>
      </div>

    </div>

    <!-- Sticky summary bar -->
    <div v-if="!loading && !error && recommendations.length > 0" class="summary-bar">
      <div class="summary-left">
        <strong>{{ selectedItems.length }}</strong> item{{ selectedItems.length !== 1 ? 's' : '' }} selected
        &nbsp;·&nbsp;
        Total: <strong>{{ formatCurrency(selectedTotal) }}</strong>
      </div>
      <div class="summary-right">
        <span :class="['budget-remaining', { 'over-budget': budgetRemaining < 0 }]">
          Budget remaining: <strong>{{ formatCurrency(budgetRemaining) }}</strong>
        </span>
        <button
          class="place-order-btn"
          :disabled="selectedItems.length === 0 || placing"
          @click="placeOrder"
        >
          {{ placing ? 'Placing Order...' : 'Place Order' }}
        </button>
      </div>
    </div>
  </div>
</template>

<script>
import { ref, computed, watch, onMounted } from 'vue'
import { api } from '../api'

export default {
  name: 'Restocking',
  setup() {
    const loading = ref(true)
    const error = ref(null)
    const placing = ref(false)
    const orderSuccess = ref(false)

    const allForecasts = ref([])
    const inventoryItems = ref([])

    const budget = ref(0)
    const manualToggles = ref({})

    const recommendations = computed(() => {
      const trendPriority = { increasing: 0, stable: 1, decreasing: 2 }
      const items = []

      for (const forecast of allForecasts.value) {
        const inv = inventoryItems.value.find(i => i.sku === forecast.item_sku)
        if (!inv) continue

        const qty_needed = Math.max(0, forecast.forecasted_demand - inv.quantity_on_hand)
        if (qty_needed === 0) continue

        items.push({
          sku: forecast.item_sku,
          name: forecast.item_name,
          trend: forecast.trend,
          forecasted_demand: forecast.forecasted_demand,
          quantity_on_hand: inv.quantity_on_hand,
          unit_cost: inv.unit_cost,
          qty_needed,
          item_cost: qty_needed * inv.unit_cost,
          priority: trendPriority[forecast.trend] ?? 1
        })
      }

      return items.sort((a, b) => a.priority - b.priority || b.qty_needed - a.qty_needed)
    })

    const totalCost = computed(() =>
      recommendations.value.reduce((sum, r) => sum + r.item_cost, 0)
    )

    const loadData = async () => {
      try {
        loading.value = true
        error.value = null
        const [forecasts, inventory] = await Promise.all([
          api.getDemandForecasts(),
          api.getInventory({})
        ])
        allForecasts.value = forecasts
        inventoryItems.value = inventory
        // Default to 50% of total cost, rounded to nearest $100
        budget.value = Math.round(totalCost.value * 0.5 / 100) * 100
      } catch (err) {
        error.value = 'Failed to load data: ' + err.message
      } finally {
        loading.value = false
      }
    }

    // Greedy selection: pick highest-priority items that fit within budget
    const autoSelected = computed(() => {
      const selected = new Set()
      let remaining = budget.value
      for (const item of recommendations.value) {
        if (item.item_cost <= remaining) {
          selected.add(item.sku)
          remaining -= item.item_cost
        }
      }
      return selected
    })

    // Clear manual overrides whenever the slider moves so auto-selection refreshes cleanly
    watch(budget, () => {
      manualToggles.value = {}
    })

    const isSelected = (sku) => {
      if (sku in manualToggles.value) return manualToggles.value[sku]
      return autoSelected.value.has(sku)
    }

    const toggleItem = (sku) => {
      manualToggles.value = { ...manualToggles.value, [sku]: !isSelected(sku) }
    }

    const selectedItems = computed(() =>
      recommendations.value.filter(r => isSelected(r.sku))
    )

    const selectedTotal = computed(() =>
      selectedItems.value.reduce((sum, r) => sum + r.item_cost, 0)
    )

    const budgetRemaining = computed(() => budget.value - selectedTotal.value)

    const formatCurrency = (value) =>
      value.toLocaleString('en-US', { style: 'currency', currency: 'USD', maximumFractionDigits: 0 })

    const placeOrder = async () => {
      if (selectedItems.value.length === 0) return
      try {
        placing.value = true
        error.value = null
        const items = selectedItems.value.map(r => ({
          sku: r.sku,
          name: r.name,
          quantity: r.qty_needed,
          unit_price: r.unit_cost
        }))
        await api.createRestockingOrder(items)
        orderSuccess.value = true
        manualToggles.value = {}
      } catch (err) {
        error.value = 'Failed to place order: ' + err.message
      } finally {
        placing.value = false
      }
    }

    onMounted(loadData)

    return {
      loading, error, placing, orderSuccess,
      recommendations, budget, totalCost,
      isSelected, toggleItem,
      selectedItems, selectedTotal, budgetRemaining,
      formatCurrency, placeOrder
    }
  }
}
</script>

<style scoped>
.restocking {
  padding-bottom: 80px;
}

.success-banner {
  display: flex;
  align-items: center;
  justify-content: space-between;
  background: #d1fae5;
  border: 1px solid #6ee7b7;
  color: #065f46;
  padding: 0.875rem 1.25rem;
  border-radius: 8px;
  margin-bottom: 1.25rem;
  font-size: 0.9rem;
  font-weight: 500;
}

.dismiss-btn {
  background: none;
  border: 1px solid #6ee7b7;
  color: #065f46;
  padding: 0.25rem 0.75rem;
  border-radius: 6px;
  cursor: pointer;
  font-size: 0.8rem;
  font-weight: 500;
}

.dismiss-btn:hover {
  background: #a7f3d0;
}

.budget-card {
  margin-bottom: 1.25rem;
}

.budget-label {
  font-size: 0.75rem;
  font-weight: 700;
  text-transform: uppercase;
  letter-spacing: 0.05em;
  color: #64748b;
  margin-bottom: 0.5rem;
}

.budget-amount {
  font-size: 2.5rem;
  font-weight: 700;
  color: #0f172a;
  letter-spacing: -0.025em;
  margin-bottom: 1rem;
}

.budget-slider {
  width: 100%;
  height: 6px;
  -webkit-appearance: none;
  appearance: none;
  border-radius: 3px;
  background: #e2e8f0;
  outline: none;
  cursor: pointer;
  margin-bottom: 0.5rem;
}

.budget-slider::-webkit-slider-thumb {
  -webkit-appearance: none;
  width: 20px;
  height: 20px;
  border-radius: 50%;
  background: #2563eb;
  cursor: pointer;
  box-shadow: 0 1px 3px rgba(0, 0, 0, 0.2);
}

.budget-slider::-moz-range-thumb {
  width: 20px;
  height: 20px;
  border-radius: 50%;
  background: #2563eb;
  cursor: pointer;
  border: none;
}

.budget-sublabel {
  font-size: 0.8rem;
  color: #94a3b8;
}

.count-badge {
  display: inline-block;
  margin-left: 0.75rem;
  padding: 0.15rem 0.6rem;
  background: #eff6ff;
  color: #2563eb;
  border-radius: 99px;
  font-size: 0.75rem;
  font-weight: 600;
}

.restock-table {
  width: 100%;
  border-collapse: collapse;
}

.col-check { width: 40px; }
.col-sku   { width: 110px; }
.col-name  { min-width: 180px; }
.col-trend { width: 120px; }
.col-num   { width: 120px; text-align: right; }

.restock-row {
  cursor: pointer;
  transition: background 0.15s;
}

.restock-row:hover {
  background: #f8fafc;
}

.row-excluded {
  opacity: 0.5;
}

.empty-state {
  padding: 2.5rem;
  text-align: center;
  color: #64748b;
  font-size: 0.9rem;
}

code {
  font-family: 'Cascadia Code', 'Fira Code', Consolas, monospace;
  font-size: 0.8rem;
  background: #f1f5f9;
  padding: 0.1rem 0.35rem;
  border-radius: 4px;
  color: #334155;
}

.summary-bar {
  position: fixed;
  bottom: 0;
  left: 0;
  right: 0;
  background: white;
  border-top: 1px solid #e2e8f0;
  box-shadow: 0 -4px 12px rgba(0, 0, 0, 0.06);
  padding: 1rem 2rem;
  display: flex;
  align-items: center;
  justify-content: space-between;
  z-index: 50;
  font-size: 0.9rem;
  color: #334155;
}

.summary-left {
  display: flex;
  align-items: center;
}

.summary-right {
  display: flex;
  align-items: center;
  gap: 1.25rem;
}

.budget-remaining {
  font-size: 0.875rem;
  color: #475569;
}

.budget-remaining.over-budget {
  color: #dc2626;
}

.place-order-btn {
  background: #2563eb;
  color: white;
  border: none;
  padding: 0.625rem 1.5rem;
  border-radius: 8px;
  font-size: 0.9rem;
  font-weight: 600;
  cursor: pointer;
  transition: background 0.2s;
}

.place-order-btn:hover:not(:disabled) {
  background: #1d4ed8;
}

.place-order-btn:disabled {
  background: #94a3b8;
  cursor: not-allowed;
}
</style>
