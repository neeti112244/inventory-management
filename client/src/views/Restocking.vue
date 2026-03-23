<template>
  <div class="restocking">
    <div class="page-header">
      <h2>{{ t('restocking.title') }}</h2>
      <p>{{ t('restocking.description') }}</p>
    </div>

    <div v-if="loading" class="loading">{{ t('common.loading') }}</div>
    <div v-else-if="error" class="error">{{ error }}</div>
    <div v-else>
      <div class="card budget-card">
        <div class="card-header">
          <h3 class="card-title">{{ t('restocking.budget') }}</h3>
        </div>
        <div class="budget-body">
          <div class="budget-display">{{ formatCurrency(budget) }}</div>
          <input
            type="range"
            class="budget-slider"
            min="0"
            max="500000"
            step="1000"
            v-model.number="budget"
          />
          <div class="budget-summary">
            {{ recommendedItems.length }} {{ t('restocking.recommendedItems') }} &nbsp;|&nbsp; Total cost: {{ formatCurrency(totalCost) }}
          </div>
        </div>
      </div>

      <div class="card">
        <div class="card-header">
          <h3 class="card-title">{{ t('restocking.recommendedItems') }} ({{ recommendedItems.length }})</h3>
        </div>
        <div class="table-container">
          <table>
            <thead>
              <tr>
                <th>{{ t('restocking.table.sku') }}</th>
                <th>{{ t('restocking.table.itemName') }}</th>
                <th>{{ t('restocking.table.trend') }}</th>
                <th>{{ t('restocking.table.demandGap') }}</th>
                <th>{{ t('restocking.table.unitCost') }}</th>
                <th>{{ t('restocking.table.quantity') }}</th>
                <th>{{ t('restocking.table.totalCost') }}</th>
              </tr>
            </thead>
            <tbody>
              <tr v-if="recommendedItems.length === 0">
                <td colspan="7" class="empty-state">{{ t('restocking.noRecommendations') }}</td>
              </tr>
              <tr v-for="item in recommendedItems" :key="item.sku">
                <td><strong>{{ item.sku }}</strong></td>
                <td>{{ item.name }}</td>
                <td>
                  <span :class="['badge', item.trend]">{{ item.trend }}</span>
                </td>
                <td>{{ item.demand_gap }}</td>
                <td>{{ formatCurrency(item.unit_cost) }}</td>
                <td>{{ item.quantity }}</td>
                <td><strong>{{ formatCurrency(item.total_cost) }}</strong></td>
              </tr>
            </tbody>
          </table>
        </div>

        <div class="place-order-footer">
          <div class="footer-totals">
            <span>{{ recommendedItems.length }} items &mdash; Total: <strong>{{ formatCurrency(totalCost) }}</strong></span>
          </div>
          <div class="footer-actions">
            <div v-if="orderPlaced" class="success-banner">{{ t('restocking.orderPlaced') }}</div>
            <button
              class="btn-primary"
              :disabled="recommendedItems.length === 0 || orderPlaced || placingOrder"
              @click="placeOrder"
            >
              {{ placingOrder ? t('common.loading') : t('restocking.placeOrder') }}
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
    const { t } = useI18n()

    const budget = ref(100000)
    const demandForecasts = ref([])
    const inventoryItems = ref([])
    const loading = ref(true)
    const error = ref(null)
    const orderPlaced = ref(false)
    const placingOrder = ref(false)

    const formatCurrency = (value) =>
      value.toLocaleString('en-US', { style: 'currency', currency: 'USD' })

    // Computed recommendation list based on current budget
    const recommendedItems = computed(() => {
      if (!demandForecasts.value.length || !inventoryItems.value.length) return []

      // Build inventory map by SKU
      const inventoryMap = {}
      inventoryItems.value.forEach(item => { inventoryMap[item.sku] = item })

      // Join forecasts with inventory, filter out zero/negative gaps
      const candidates = []
      demandForecasts.value.forEach(forecast => {
        const inv = inventoryMap[forecast.item_sku]
        if (!inv) return
        const gap = Math.max(0, forecast.forecasted_demand - forecast.current_demand)
        if (gap <= 0) return
        candidates.push({
          sku: forecast.item_sku,
          name: forecast.item_name,
          trend: forecast.trend,
          demand_gap: gap,
          unit_cost: inv.unit_cost,
          item_cost: inv.unit_cost * gap
        })
      })

      // Sort: increasing first, then stable, decreasing last (typically filtered by gap <= 0)
      const trendOrder = { increasing: 0, stable: 1, decreasing: 2 }
      candidates.sort((a, b) => (trendOrder[a.trend] ?? 2) - (trendOrder[b.trend] ?? 2))

      // Greedy budget fill
      let remaining = budget.value
      const result = []
      for (const candidate of candidates) {
        if (remaining <= 0) break
        if (remaining >= candidate.item_cost) {
          result.push({ ...candidate, quantity: candidate.demand_gap, total_cost: candidate.item_cost })
          remaining -= candidate.item_cost
        } else {
          const partialQty = Math.floor(remaining / candidate.unit_cost)
          if (partialQty > 0) {
            result.push({ ...candidate, quantity: partialQty, total_cost: partialQty * candidate.unit_cost })
            remaining -= partialQty * candidate.unit_cost
          }
        }
      }
      return result
    })

    const totalCost = computed(() =>
      recommendedItems.value.reduce((sum, item) => sum + item.total_cost, 0)
    )

    const placeOrder = async () => {
      if (!recommendedItems.value.length) return
      placingOrder.value = true
      try {
        const items = recommendedItems.value.map(item => ({
          sku: item.sku,
          name: item.name,
          quantity: item.quantity,
          unit_price: item.unit_cost
        }))
        await api.submitRestockingOrder(items, totalCost.value)
        orderPlaced.value = true
      } catch (err) {
        error.value = 'Failed to place order: ' + err.message
      } finally {
        placingOrder.value = false
      }
    }

    onMounted(async () => {
      try {
        loading.value = true
        const [forecasts, inventory] = await Promise.all([
          api.getDemandForecasts(),
          api.getInventory({})
        ])
        demandForecasts.value = forecasts
        inventoryItems.value = inventory
      } catch (err) {
        error.value = 'Failed to load data: ' + err.message
      } finally {
        loading.value = false
      }
    })

    return {
      t,
      budget,
      loading,
      error,
      orderPlaced,
      placingOrder,
      recommendedItems,
      totalCost,
      formatCurrency,
      placeOrder
    }
  }
}
</script>

<style scoped>
.restocking {
  padding: 2rem;
}

.budget-card {
  margin-bottom: 1.5rem;
}

.budget-body {
  padding: 1.5rem;
  display: flex;
  flex-direction: column;
  gap: 1rem;
}

.budget-display {
  font-size: 2.25rem;
  font-weight: 700;
  color: #0f172a;
  line-height: 1;
}

.budget-slider {
  width: 100%;
  appearance: none;
  height: 6px;
  border-radius: 3px;
  background: #e2e8f0;
  outline: none;
  cursor: pointer;
}

.budget-slider::-webkit-slider-thumb {
  appearance: none;
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
  box-shadow: 0 1px 3px rgba(0, 0, 0, 0.2);
}

.budget-summary {
  font-size: 0.875rem;
  color: #64748b;
}

.empty-state {
  text-align: center;
  color: #64748b;
  padding: 2rem;
  font-style: italic;
}

.place-order-footer {
  display: flex;
  align-items: center;
  justify-content: space-between;
  padding: 1rem 1.5rem;
  border-top: 1px solid #e2e8f0;
  gap: 1rem;
}

.footer-totals {
  font-size: 0.9rem;
  color: #64748b;
}

.footer-actions {
  display: flex;
  align-items: center;
  gap: 1rem;
}

.btn-primary {
  padding: 0.625rem 1.25rem;
  background: #2563eb;
  color: #fff;
  border: none;
  border-radius: 6px;
  font-size: 0.875rem;
  font-weight: 600;
  cursor: pointer;
  transition: background 0.15s;
}

.btn-primary:hover:not(:disabled) {
  background: #1d4ed8;
}

.btn-primary:disabled {
  background: #94a3b8;
  cursor: not-allowed;
}

.success-banner {
  padding: 0.5rem 1rem;
  background: #dcfce7;
  color: #166534;
  border: 1px solid #bbf7d0;
  border-radius: 6px;
  font-size: 0.875rem;
  font-weight: 500;
}
</style>
