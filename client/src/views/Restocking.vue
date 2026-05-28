<template>
  <div class="restocking">
    <div class="page-header">
      <h2>{{ t('restocking.title') }}</h2>
      <p>{{ t('restocking.description') }}</p>
    </div>

    <div v-if="loading" class="loading">{{ t('common.loading') }}</div>
    <div v-else-if="error" class="error">{{ error }}</div>
    <div v-else>

      <!-- Budget card -->
      <div class="card budget-card">
        <div class="budget-label">{{ t('restocking.budget') }}</div>
        <div class="budget-controls">
          <input
            type="range"
            min="0"
            max="200000"
            step="500"
            v-model.number="budget"
            class="budget-slider"
          />
          <input
            type="number"
            min="0"
            max="200000"
            step="500"
            v-model.number="budget"
            class="budget-input"
          />
        </div>
        <div class="budget-display">{{ formatCurrencyWithDecimals(budget, currentCurrency) }}</div>
        <div class="budget-hint">{{ t('restocking.budgetHint') }}</div>
      </div>

      <!-- Orphan SKU chip -->
      <div v-if="orphanSkuCount > 0" class="orphan-chip">
        {{ t('restocking.orphansSkipped', { n: orphanSkuCount }) }}
      </div>

      <!-- Recommendations card -->
      <div class="card">
        <div class="card-header">
          <h3 class="card-title">{{ t('restocking.recommended') }} ({{ scoredItems.length }})</h3>
        </div>
        <div class="table-container">
          <table class="restock-table">
            <thead>
              <tr>
                <th>{{ t('restocking.table.sku') }}</th>
                <th>{{ t('restocking.table.item') }}</th>
                <th>{{ t('restocking.table.category') }}</th>
                <th class="col-num">{{ t('restocking.table.current') }}</th>
                <th class="col-num">{{ t('restocking.table.forecasted') }}</th>
                <th>{{ t('restocking.table.trend') }}</th>
                <th class="col-num">{{ t('restocking.table.unitCost') }}</th>
                <th class="col-num">{{ t('restocking.table.qty') }}</th>
                <th class="col-num">{{ t('restocking.table.subtotal') }}</th>
                <th class="col-num">{{ t('restocking.table.score') }}</th>
                <th class="col-num">{{ t('restocking.table.leadTime') }}</th>
              </tr>
            </thead>
            <tbody>
              <tr v-if="scoredItems.length === 0">
                <td colspan="11" class="empty-cell">{{ t('restocking.noRecommendations') }}</td>
              </tr>
              <tr
                v-for="row in scoredItems"
                :key="row.sku"
                :class="{ 'row-excluded': !row.included }"
              >
                <td><strong>{{ row.sku }}</strong></td>
                <td>
                  {{ translateProductName(row.name) }}
                  <span v-if="row.capped" class="badge-capped">{{ t('restocking.qtyCapped') }}</span>
                </td>
                <td>{{ row.category }}</td>
                <td class="col-num">{{ row.current_demand }}</td>
                <td class="col-num">{{ row.forecasted_demand }}</td>
                <td>
                  <span :class="['trend-badge', row.trend]">{{ t(`trends.${row.trend}`) }}</span>
                </td>
                <td class="col-num">{{ formatCurrencyWithDecimals(row.unit_cost, currentCurrency, 2) }}</td>
                <td class="col-num">{{ row.qty }}</td>
                <td class="col-num">{{ formatCurrencyWithDecimals(row.subtotal, currentCurrency, 2) }}</td>
                <td class="col-num">{{ row.score.toFixed(2) }}</td>
                <td class="col-num">{{ row.leadTime }}d</td>
              </tr>
            </tbody>
          </table>
        </div>

        <!-- Total / CTA bar -->
        <div class="cta-bar">
          <div class="cta-summary">
            <div class="summary-line">
              <span class="summary-label">{{ t('restocking.totalSpend') }}</span>
              <span class="summary-value" :class="{ 'over-budget': cartTotal > budget }">
                {{ formatCurrencyWithDecimals(cartTotal, currentCurrency, 2) }}
              </span>
              <span class="summary-of">{{ t('restocking.ofBudget') }}</span>
              <span class="summary-value">{{ formatCurrencyWithDecimals(budget, currentCurrency) }}</span>
            </div>
            <div class="summary-line">
              <span class="summary-label">{{ t('restocking.leadTimeDays') }}</span>
              <span class="summary-value">{{ maxLeadTime }} {{ t('restocking.days') }}</span>
              <span v-if="expectedDelivery" class="summary-sub">({{ expectedDelivery }})</span>
            </div>
            <div class="summary-line">
              <span class="summary-label">{{ t('restocking.warehouseLabel') }}</span>
              <span class="summary-value">{{ consolidatedWarehouse }}</span>
            </div>
          </div>
          <div class="cta-actions">
            <div v-if="submitSuccess" class="submit-success">
              {{ t('restocking.submitSuccess', { orderNumber: lastOrderNumber }) }}
            </div>
            <div v-if="submitError" class="submit-error">{{ submitError }}</div>
            <button
              class="btn-primary"
              :disabled="cart.length === 0 || submitting"
              @click="placeOrder"
            >
              {{ submitting ? t('restocking.placing') : t('restocking.placeOrder') }}
            </button>
          </div>
        </div>
      </div>

      <!-- Session submissions card -->
      <div v-if="sessionSubmissions.length > 0" class="card">
        <div class="card-header">
          <h3 class="card-title">{{ t('restocking.sessionSubmissions') }} ({{ sessionSubmissions.length }})</h3>
        </div>
        <div class="table-container">
          <table class="submissions-table">
            <thead>
              <tr>
                <th>{{ t('orders.table.orderNumber') }}</th>
                <th class="col-num">{{ t('restocking.table.qty') }}</th>
                <th>{{ t('orders.table.expectedDelivery') }}</th>
                <th>{{ t('restocking.warehouseLabel') }}</th>
                <th class="col-num">{{ t('orders.table.totalValue') }}</th>
              </tr>
            </thead>
            <tbody>
              <tr v-for="sub in sessionSubmissions" :key="sub.id">
                <td><strong>{{ sub.order_number }}</strong></td>
                <td class="col-num">{{ sub.items.length }}</td>
                <td>{{ sub.expected_delivery }}</td>
                <td>{{ sub.warehouse }}</td>
                <td class="col-num">{{ formatCurrencyWithDecimals(sub.total_value, currentCurrency, 2) }}</td>
              </tr>
            </tbody>
          </table>
        </div>
      </div>

    </div>
  </div>
</template>

<script>
import { ref, computed, watch, onMounted } from 'vue'
import { api } from '../api'
import { useFilters } from '../composables/useFilters'
import { useI18n } from '../composables/useI18n'
import { formatCurrencyWithDecimals } from '../utils/currency'

const TREND_MULT = { increasing: 1.3, stable: 1.0, decreasing: 0.7 }
const LEAD_TIMES = { Sensors: 7, 'Circuit Boards': 14, Controllers: 21, Actuators: 10 }

export default {
  name: 'Restocking',
  setup() {
    const { t, currentCurrency, translateProductName } = useI18n()
    const { selectedLocation, selectedCategory, getCurrentFilters } = useFilters()

    const loading = ref(true)
    const error = ref(null)
    const budget = ref(25000)

    const allForecasts = ref([])
    const inventoryItems = ref([])
    const orphanSkuCount = ref(0)

    const submitting = ref(false)
    const submitSuccess = ref(false)
    const submitError = ref(null)
    const lastOrderNumber = ref('')
    const sessionSubmissions = ref([])

    // --- Data loading ---
    const loadData = async () => {
      loading.value = true
      error.value = null
      try {
        const filters = getCurrentFilters()
        const [forecastsData, inventoryData] = await Promise.all([
          api.getDemandForecasts(),
          api.getInventory({ warehouse: filters.warehouse, category: filters.category })
        ])
        allForecasts.value = forecastsData
        inventoryItems.value = inventoryData
      } catch (err) {
        error.value = t('common.error') + ': ' + err.message
      } finally {
        loading.value = false
      }
    }

    watch([selectedLocation, selectedCategory], () => {
      loadData()
    })

    onMounted(loadData)

    // --- Join + score ---
    const scoredItems = computed(() => {
      const invMap = {}
      for (const inv of inventoryItems.value) {
        invMap[inv.sku] = inv
      }

      let dropped = 0
      const joined = []
      for (const fc of allForecasts.value) {
        const inv = invMap[fc.item_sku]
        if (!inv) { dropped++; continue }
        const gap = fc.forecasted_demand - fc.current_demand
        if (gap <= 0) continue
        const mult = TREND_MULT[fc.trend] ?? 1.0
        const leadTime = LEAD_TIMES[inv.category] ?? 14
        const score = (gap * mult) / inv.unit_cost
        joined.push({
          sku: inv.sku,
          name: inv.name,
          category: inv.category,
          warehouse: inv.warehouse,
          current_demand: fc.current_demand,
          forecasted_demand: fc.forecasted_demand,
          trend: fc.trend,
          unit_cost: inv.unit_cost,
          gap,
          score,
          leadTime
        })
      }
      orphanSkuCount.value = dropped

      joined.sort((a, b) => b.score - a.score)

      // Greedy cart build
      const cartSet = new Set()
      let running = 0
      for (const item of joined) {
        const sub = item.gap * item.unit_cost
        if (running + sub <= budget.value) {
          cartSet.add(item.sku + ':full')
          running += sub
        } else {
          const capQty = Math.floor((budget.value - running) / item.unit_cost)
          if (capQty > 0) {
            cartSet.add(item.sku + ':cap:' + capQty)
            running += capQty * item.unit_cost
          }
          break
        }
      }

      // Build final display list with included/capped flags
      const result = []
      const cartMap = {}
      for (const key of cartSet) {
        const [sku, type, capQty] = key.split(':')
        cartMap[sku] = { type, capQty: capQty ? parseInt(capQty) : null }
      }

      for (const item of joined) {
        const entry = cartMap[item.sku]
        if (entry) {
          const qty = entry.type === 'cap' ? entry.capQty : item.gap
          result.push({
            ...item,
            qty,
            subtotal: qty * item.unit_cost,
            included: true,
            capped: entry.type === 'cap'
          })
        } else {
          result.push({
            ...item,
            qty: item.gap,
            subtotal: item.gap * item.unit_cost,
            included: false,
            capped: false
          })
        }
      }

      return result
    })

    // --- Cart (included rows only) ---
    const cart = computed(() => scoredItems.value.filter(r => r.included))

    const cartTotal = computed(() => cart.value.reduce((sum, r) => sum + r.subtotal, 0))

    const maxLeadTime = computed(() => {
      if (cart.value.length === 0) return 0
      return Math.max(...cart.value.map(r => r.leadTime))
    })

    const expectedDelivery = computed(() => {
      if (cart.value.length === 0) return ''
      const d = new Date()
      d.setDate(d.getDate() + maxLeadTime.value)
      return d.toISOString().slice(0, 10)
    })

    const consolidatedWarehouse = computed(() => {
      const warehouses = [...new Set(cart.value.map(r => r.warehouse))]
      if (warehouses.length === 0) return ''
      if (warehouses.length === 1) return warehouses[0]
      return 'Mixed'
    })

    // --- Submit ---
    const placeOrder = async () => {
      if (cart.value.length === 0 || submitting.value) return
      submitting.value = true
      submitSuccess.value = false
      submitError.value = null
      try {
        const payload = {
          items: cart.value.map(c => ({
            sku: c.sku,
            name: c.name,
            quantity: c.qty,
            unit_price: c.unit_cost
          })),
          warehouse: consolidatedWarehouse.value,
          expected_delivery: expectedDelivery.value
        }
        const order = await api.submitRestockOrder(payload)
        sessionSubmissions.value.push(order)
        lastOrderNumber.value = order.order_number
        submitSuccess.value = true
      } catch (err) {
        submitError.value = err.message || 'Failed to submit order'
      } finally {
        submitting.value = false
      }
    }

    return {
      t,
      currentCurrency,
      translateProductName,
      formatCurrencyWithDecimals,
      loading,
      error,
      budget,
      orphanSkuCount,
      scoredItems,
      cart,
      cartTotal,
      maxLeadTime,
      expectedDelivery,
      consolidatedWarehouse,
      submitting,
      submitSuccess,
      submitError,
      lastOrderNumber,
      sessionSubmissions,
      placeOrder
    }
  }
}
</script>

<style scoped>
/* Budget card */
.budget-card {
  margin-bottom: 1.25rem;
  display: flex;
  flex-direction: column;
  gap: 0.75rem;
}

.budget-label {
  font-size: 0.875rem;
  font-weight: 600;
  color: #64748b;
  text-transform: uppercase;
  letter-spacing: 0.05em;
}

.budget-controls {
  display: flex;
  align-items: center;
  gap: 1rem;
}

.budget-slider {
  flex: 1;
  accent-color: #64748b;
  height: 6px;
  cursor: pointer;
}

.budget-input {
  width: 120px;
  padding: 0.375rem 0.625rem;
  border: 1px solid #e2e8f0;
  border-radius: 6px;
  font-size: 0.875rem;
  color: #0f172a;
  outline: none;
}

.budget-input:focus {
  border-color: #3b82f6;
  box-shadow: 0 0 0 2px rgba(59, 130, 246, 0.15);
}

.budget-display {
  font-size: 1.5rem;
  font-weight: 700;
  color: #0f172a;
}

.budget-hint {
  font-size: 0.8125rem;
  color: #94a3b8;
}

/* Orphan chip */
.orphan-chip {
  display: inline-block;
  margin-bottom: 1rem;
  padding: 0.375rem 0.875rem;
  background: #fef3c7;
  border: 1px solid #fbbf24;
  border-radius: 9999px;
  font-size: 0.8125rem;
  color: #92400e;
  font-weight: 500;
}

/* Table */
.restock-table {
  width: 100%;
  border-collapse: collapse;
}

.col-num {
  text-align: right;
}

.row-excluded td {
  color: #94a3b8;
}

.row-excluded strong {
  color: #cbd5e1;
}

/* Trend badges */
.trend-badge {
  display: inline-block;
  padding: 0.2rem 0.625rem;
  border-radius: 9999px;
  font-size: 0.75rem;
  font-weight: 600;
  white-space: nowrap;
}

.trend-badge.increasing {
  background: #d1fae5;
  color: #065f46;
}

.trend-badge.stable {
  background: #e2e8f0;
  color: #475569;
}

.trend-badge.decreasing {
  background: #fee2e2;
  color: #991b1b;
}

/* Capped badge */
.badge-capped {
  display: inline-block;
  margin-left: 0.5rem;
  padding: 0.125rem 0.5rem;
  background: #fef9c3;
  border: 1px solid #fbbf24;
  border-radius: 4px;
  font-size: 0.7rem;
  font-weight: 600;
  color: #78350f;
  vertical-align: middle;
}

/* CTA bar */
.cta-bar {
  display: flex;
  justify-content: space-between;
  align-items: flex-end;
  gap: 1.5rem;
  padding: 1.25rem 0 0;
  border-top: 1px solid #e2e8f0;
  margin-top: 1rem;
  flex-wrap: wrap;
}

.cta-summary {
  display: flex;
  flex-direction: column;
  gap: 0.5rem;
}

.summary-line {
  display: flex;
  align-items: center;
  gap: 0.5rem;
  font-size: 0.875rem;
}

.summary-label {
  color: #64748b;
  min-width: 110px;
}

.summary-value {
  font-weight: 600;
  color: #0f172a;
}

.summary-value.over-budget {
  color: #ef4444;
}

.summary-of {
  color: #94a3b8;
  font-size: 0.8125rem;
}

.summary-sub {
  color: #94a3b8;
  font-size: 0.8125rem;
}

.cta-actions {
  display: flex;
  flex-direction: column;
  align-items: flex-end;
  gap: 0.5rem;
}

.btn-primary {
  padding: 0.625rem 1.5rem;
  background: #3b82f6;
  color: white;
  border: none;
  border-radius: 8px;
  font-size: 0.9375rem;
  font-weight: 600;
  cursor: pointer;
  transition: background 0.15s;
  white-space: nowrap;
}

.btn-primary:hover:not(:disabled) {
  background: #2563eb;
}

.btn-primary:disabled {
  opacity: 0.5;
  cursor: not-allowed;
}

.submit-success {
  font-size: 0.875rem;
  color: #059669;
  font-weight: 500;
}

.submit-error {
  font-size: 0.875rem;
  color: #ef4444;
  font-weight: 500;
}

/* Empty state */
.empty-cell {
  text-align: center;
  color: #94a3b8;
  padding: 2rem 0;
  font-size: 0.9375rem;
}

/* Submissions table */
.submissions-table {
  width: 100%;
  border-collapse: collapse;
}
</style>
