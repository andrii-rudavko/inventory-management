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
        <div class="budget-header">
          <label class="budget-label">{{ t('restocking.budgetLabel') }}</label>
          <span class="budget-value">{{ currencySymbol }}{{ budget.toLocaleString() }}</span>
        </div>
        <input
          type="range"
          class="budget-slider"
          min="0"
          :max="maxBudget"
          step="250"
          v-model.number="budget"
        />
        <div class="budget-range-labels">
          <span>{{ currencySymbol }}0</span>
          <span>{{ currencySymbol }}{{ maxBudget.toLocaleString() }}</span>
        </div>
      </div>

      <div class="card">
        <div class="card-header">
          <h3 class="card-title">{{ t('restocking.recommendations') }}</h3>
        </div>
        <p class="recommendations-subtitle">{{ t('restocking.recommendationsSubtitle') }}</p>

        <div v-if="recommendations.length === 0" class="empty-state">
          {{ t('restocking.noRecommendations') }}
        </div>
        <div v-else class="table-container">
          <table>
            <thead>
              <tr>
                <th></th>
                <th>{{ t('restocking.table.item') }}</th>
                <th>{{ t('restocking.table.trend') }}</th>
                <th>{{ t('restocking.table.currentDemand') }}</th>
                <th>{{ t('restocking.table.forecastedDemand') }}</th>
                <th>{{ t('restocking.table.restockQty') }}</th>
                <th>{{ t('restocking.table.unitCost') }}</th>
                <th>{{ t('restocking.table.leadTime') }}</th>
                <th>{{ t('restocking.table.estimatedCost') }}</th>
              </tr>
            </thead>
            <tbody>
              <tr
                v-for="rec in recommendations"
                :key="rec.item_sku"
                :class="{ selected: isSelected(rec.item_sku) }"
              >
                <td>
                  <input
                    type="checkbox"
                    :checked="isSelected(rec.item_sku)"
                    :disabled="!isSelected(rec.item_sku) && !fitsInRemainingBudget(rec)"
                    @change="toggleSelection(rec)"
                  />
                </td>
                <td>{{ rec.item_name }}</td>
                <td><span :class="['badge', rec.trend]">{{ rec.trend }}</span></td>
                <td>{{ rec.current_demand }}</td>
                <td>{{ rec.forecasted_demand }}</td>
                <td>{{ rec.restock_quantity }}</td>
                <td>{{ currencySymbol }}{{ rec.unit_cost.toFixed(2) }}</td>
                <td>{{ t('orders.submittedOrders.leadTimeDays', { count: rec.lead_time_days }) }}</td>
                <td>{{ currencySymbol }}{{ rec.estimated_cost.toLocaleString() }}</td>
              </tr>
            </tbody>
          </table>
        </div>

        <div class="order-summary">
          <div class="summary-stats">
            <span>{{ t('restocking.selectedCount', { count: selectedItems.length }) }}</span>
            <span><strong>{{ t('restocking.totalCost') }}:</strong> {{ currencySymbol }}{{ totalSelectedCost.toLocaleString() }}</span>
            <span><strong>{{ t('restocking.remainingBudget') }}:</strong> {{ currencySymbol }}{{ remainingBudget.toLocaleString() }}</span>
          </div>
          <button
            class="place-order-btn"
            :disabled="selectedItems.length === 0 || submitting || remainingBudget < 0"
            @click="placeOrder"
          >
            {{ submitting ? t('restocking.placingOrder') : t('restocking.placeOrder') }}
          </button>
        </div>

        <div v-if="submitError" class="error submit-error">{{ submitError }}</div>
        <div v-if="successMessage" class="success-message">{{ successMessage }}</div>
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
    const { t, currentCurrency } = useI18n()

    const currencySymbol = computed(() => {
      return currentCurrency.value === 'JPY' ? '¥' : '$'
    })

    const loading = ref(true)
    const error = ref(null)
    const recommendations = ref([])
    const selectedSkus = ref(new Set())
    const budget = ref(10000)
    const maxBudget = ref(20000)
    const submitting = ref(false)
    const successMessage = ref(null)
    const submitError = ref(null)

    const loadRecommendations = async () => {
      try {
        loading.value = true
        error.value = null
        recommendations.value = await api.getRestockingRecommendations()
      } catch (err) {
        error.value = 'Failed to load restocking recommendations: ' + err.message
      } finally {
        loading.value = false
      }
    }

    const selectedItems = computed(() =>
      recommendations.value.filter(rec => selectedSkus.value.has(rec.item_sku))
    )

    const totalSelectedCost = computed(() =>
      selectedItems.value.reduce((sum, rec) => sum + rec.estimated_cost, 0)
    )

    const remainingBudget = computed(() => budget.value - totalSelectedCost.value)

    const isSelected = (sku) => selectedSkus.value.has(sku)

    const fitsInRemainingBudget = (rec) => rec.estimated_cost <= remainingBudget.value

    const toggleSelection = (rec) => {
      const next = new Set(selectedSkus.value)
      if (next.has(rec.item_sku)) {
        next.delete(rec.item_sku)
      } else {
        next.add(rec.item_sku)
      }
      selectedSkus.value = next
    }

    // Auto re-select highest priority items that still fit whenever the budget changes
    const autoSelectByBudget = () => {
      const next = new Set()
      let remaining = budget.value
      for (const rec of recommendations.value) {
        if (rec.estimated_cost <= remaining) {
          next.add(rec.item_sku)
          remaining -= rec.estimated_cost
        }
      }
      selectedSkus.value = next
    }

    const placeOrder = async () => {
      if (selectedItems.value.length === 0) return
      submitting.value = true
      successMessage.value = null
      submitError.value = null
      try {
        await api.submitRestockingOrder({
          budget: budget.value,
          items: selectedItems.value.map(rec => ({
            item_sku: rec.item_sku,
            item_name: rec.item_name,
            quantity: rec.restock_quantity,
            unit_cost: rec.unit_cost,
            lead_time_days: rec.lead_time_days
          }))
        })
        successMessage.value = t('restocking.orderSuccess')
        selectedSkus.value = new Set()
      } catch (err) {
        submitError.value = 'Failed to submit restocking order: ' + err.message
      } finally {
        submitting.value = false
      }
    }

    watch(budget, autoSelectByBudget)

    onMounted(async () => {
      await loadRecommendations()
      autoSelectByBudget()
    })

    return {
      t,
      loading,
      error,
      recommendations,
      budget,
      maxBudget,
      selectedItems,
      totalSelectedCost,
      remainingBudget,
      isSelected,
      fitsInRemainingBudget,
      toggleSelection,
      placeOrder,
      submitting,
      successMessage,
      submitError,
      currencySymbol
    }
  }
}
</script>

<style scoped>
.budget-card {
  padding: 1.5rem 1.25rem;
}

.budget-header {
  display: flex;
  justify-content: space-between;
  align-items: baseline;
  margin-bottom: 0.75rem;
}

.budget-label {
  font-weight: 600;
  color: #334155;
}

.budget-value {
  font-size: 1.5rem;
  font-weight: 700;
  color: #0f172a;
}

.budget-slider {
  width: 100%;
  accent-color: #2563eb;
}

.budget-range-labels {
  display: flex;
  justify-content: space-between;
  color: #94a3b8;
  font-size: 0.75rem;
  margin-top: 0.25rem;
}

.recommendations-subtitle {
  color: #64748b;
  font-size: 0.875rem;
  margin-bottom: 1rem;
}

.empty-state {
  padding: 2rem;
  text-align: center;
  color: #64748b;
}

tr.selected {
  background: #eff6ff;
}

.badge.increasing {
  background: #d1fae5;
  color: #065f46;
}

.order-summary {
  display: flex;
  justify-content: space-between;
  align-items: center;
  margin-top: 1.25rem;
  padding-top: 1rem;
  border-top: 1px solid #e2e8f0;
  flex-wrap: wrap;
  gap: 1rem;
}

.summary-stats {
  display: flex;
  gap: 1.5rem;
  color: #334155;
  font-size: 0.938rem;
  flex-wrap: wrap;
}

.place-order-btn {
  background: #2563eb;
  color: white;
  border: none;
  padding: 0.625rem 1.5rem;
  border-radius: 8px;
  font-weight: 600;
  font-size: 0.938rem;
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

.submit-error {
  margin-top: 1rem;
}

.success-message {
  margin-top: 1rem;
  padding: 0.75rem 1rem;
  background: #d1fae5;
  color: #065f46;
  border-radius: 8px;
  font-size: 0.875rem;
}
</style>
