<template>
  <div class="restocking">
    <div class="page-header">
      <h2>{{ t('restocking.title') }}</h2>
      <p>{{ t('restocking.description') }}</p>
    </div>

    <div class="budget-control card">
      <div class="budget-label">{{ t('restocking.budget') }}</div>
      <div class="budget-row">
        <input
          type="range"
          min="0"
          max="500000"
          step="1000"
          v-model.number="budget"
          class="budget-slider"
        />
        <span class="budget-display">{{ currencySymbol }}{{ budget.toLocaleString() }}</span>
      </div>
    </div>

    <div v-if="orderPlacedMessage" class="success-message">{{ orderPlacedMessage }}</div>

    <div v-if="loading" class="loading">{{ t('common.loading') }}</div>
    <div v-else-if="error" class="error">{{ error }}</div>
    <div v-else>
      <div v-if="recommendations.length === 0" class="empty-state">
        {{ t('restocking.noRecommendations') }}
      </div>
      <div v-else class="card">
        <div class="card-header">
          <h3 class="card-title">{{ t('restocking.recommendations') }} ({{ recommendations.length }})</h3>
          <button
            class="btn-place-order"
            :disabled="recommendations.length === 0 || submitting"
            @click="placeOrder"
          >
            {{ submitting ? t('restocking.placing') : t('restocking.placeOrder') }}
          </button>
        </div>
        <div class="table-container">
          <table class="restock-table">
            <thead>
              <tr>
                <th>{{ t('restocking.table.sku') }}</th>
                <th>{{ t('restocking.table.itemName') }}</th>
                <th>{{ t('restocking.table.trend') }}</th>
                <th>{{ t('restocking.table.currentDemand') }}</th>
                <th>{{ t('restocking.table.forecastedDemand') }}</th>
                <th>{{ t('restocking.recommendedQty') }}</th>
                <th>{{ t('common.unitCost') || 'Unit Cost' }}</th>
                <th>{{ t('restocking.lineCost') }}</th>
              </tr>
            </thead>
            <tbody>
              <tr v-for="rec in recommendations" :key="rec.sku">
                <td><strong>{{ rec.sku }}</strong></td>
                <td>{{ rec.name }}</td>
                <td>
                  <span :class="['badge', getTrendClass(rec.trend)]">
                    {{ t(`trends.${rec.trend}`) || rec.trend }}
                  </span>
                </td>
                <td>{{ rec.current_demand }}</td>
                <td><strong>{{ rec.forecasted_demand }}</strong></td>
                <td>{{ rec.recommended_qty }}</td>
                <td>{{ currencySymbol }}{{ rec.unit_cost.toLocaleString() }}</td>
                <td><strong>{{ currencySymbol }}{{ rec.line_cost.toLocaleString() }}</strong></td>
              </tr>
            </tbody>
            <tfoot>
              <tr class="summary-row">
                <td colspan="7" class="summary-label">{{ t('restocking.totalCost') }}</td>
                <td class="summary-value"><strong>{{ currencySymbol }}{{ totalCost.toLocaleString() }}</strong></td>
              </tr>
              <tr class="summary-row remaining">
                <td colspan="7" class="summary-label">{{ t('restocking.remainingBudget') }}</td>
                <td class="summary-value"><strong>{{ currencySymbol }}{{ remainingBudget.toLocaleString() }}</strong></td>
              </tr>
            </tfoot>
          </table>
        </div>
      </div>
    </div>
  </div>
</template>

<script>
import { ref, computed, watch, onMounted } from 'vue'
import { api } from '../api'
import { useI18n } from '../composables/useI18n'

export default {
  name: 'Restocking',
  setup() {
    const { t, currentCurrency } = useI18n()

    const currencySymbol = computed(() => {
      return currentCurrency.value === 'JPY' ? '¥' : '$'
    })

    const budget = ref(50000)
    const recommendations = ref([])
    const totalCost = ref(0)
    const remainingBudget = ref(0)
    const loading = ref(false)
    const error = ref(null)
    const submitting = ref(false)
    const orderPlacedMessage = ref('')

    let debounceTimer = null

    const loadRecommendations = async (budgetValue) => {
      loading.value = true
      error.value = null
      try {
        const data = await api.getRestockRecommendations(budgetValue)
        recommendations.value = data.recommendations || []
        totalCost.value = data.total_cost || 0
        remainingBudget.value = data.remaining_budget || 0
      } catch (err) {
        error.value = 'Failed to load recommendations: ' + err.message
      } finally {
        loading.value = false
      }
    }

    watch(budget, (newVal) => {
      if (debounceTimer) clearTimeout(debounceTimer)
      debounceTimer = setTimeout(() => {
        loadRecommendations(newVal)
      }, 300)
    })

    const getTrendClass = (trend) => {
      const map = {
        increasing: 'badge-success',
        stable: 'badge-info',
        decreasing: 'badge-warning'
      }
      return map[trend] || 'badge-info'
    }

    const placeOrder = async () => {
      if (recommendations.value.length === 0 || submitting.value) return
      submitting.value = true
      try {
        const payload = {
          items: recommendations.value.map(r => ({
            sku: r.sku,
            name: r.name,
            quantity: r.recommended_qty,
            unit_price: r.unit_cost
          })),
          total_value: totalCost.value
        }
        await api.submitRestockOrder(payload)
        orderPlacedMessage.value = t('restocking.orderPlaced')
        setTimeout(() => {
          orderPlacedMessage.value = ''
        }, 3000)
        await loadRecommendations(budget.value)
      } catch (err) {
        error.value = 'Failed to place order: ' + err.message
      } finally {
        submitting.value = false
      }
    }

    onMounted(() => loadRecommendations(budget.value))

    return {
      t,
      currencySymbol,
      budget,
      recommendations,
      totalCost,
      remainingBudget,
      loading,
      error,
      submitting,
      orderPlacedMessage,
      getTrendClass,
      placeOrder
    }
  }
}
</script>

<style scoped>
.restocking {
  padding: 0;
}

.budget-control {
  margin-bottom: 1.5rem;
  padding: 1.25rem 1.5rem;
}

.budget-label {
  font-size: 0.875rem;
  font-weight: 600;
  color: #64748b;
  text-transform: uppercase;
  letter-spacing: 0.05em;
  margin-bottom: 0.75rem;
}

.budget-row {
  display: flex;
  align-items: center;
  gap: 1rem;
}

.budget-slider {
  flex: 1;
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
  background: #3b82f6;
  border-radius: 50%;
  cursor: pointer;
  box-shadow: 0 1px 3px rgba(0,0,0,0.2);
}

.budget-slider::-moz-range-thumb {
  width: 18px;
  height: 18px;
  background: #3b82f6;
  border-radius: 50%;
  cursor: pointer;
  border: none;
  box-shadow: 0 1px 3px rgba(0,0,0,0.2);
}

.budget-display {
  font-size: 1.125rem;
  font-weight: 700;
  color: #0f172a;
  min-width: 120px;
  text-align: right;
}

.success-message {
  background: #f0fdf4;
  border: 1px solid #86efac;
  color: #166534;
  padding: 0.75rem 1rem;
  border-radius: 6px;
  margin-bottom: 1rem;
  font-weight: 500;
}

.empty-state {
  background: white;
  border: 1px solid #e2e8f0;
  border-radius: 8px;
  padding: 3rem;
  text-align: center;
  color: #64748b;
  font-size: 0.9rem;
}

.card-header {
  display: flex;
  align-items: center;
  justify-content: space-between;
}

.btn-place-order {
  background: #3b82f6;
  color: white;
  border: none;
  padding: 0.5rem 1.25rem;
  border-radius: 6px;
  font-size: 0.875rem;
  font-weight: 600;
  cursor: pointer;
  transition: background 0.15s;
}

.btn-place-order:hover:not(:disabled) {
  background: #2563eb;
}

.btn-place-order:disabled {
  background: #94a3b8;
  cursor: not-allowed;
}

.restock-table {
  table-layout: auto;
  width: 100%;
}

.summary-row td {
  background: #f8fafc;
  border-top: 2px solid #e2e8f0;
  padding: 0.625rem 1rem;
}

.summary-row.remaining td {
  background: #eff6ff;
  border-top: 1px solid #bfdbfe;
}

.summary-label {
  text-align: right;
  font-size: 0.875rem;
  color: #64748b;
  font-weight: 600;
  text-transform: uppercase;
  letter-spacing: 0.04em;
}

.summary-value {
  text-align: left;
  font-size: 1rem;
  color: #0f172a;
}

.summary-row.remaining .summary-value strong {
  color: #1d4ed8;
}
</style>
