<template>
  <div class="restocking">
    <div class="page-header">
      <h2>Restocking Planner</h2>
      <p>Allocate your budget to restock high-demand items</p>
    </div>

    <!-- Budget Slider Card -->
    <div class="card">
      <div class="card-header">
        <h3 class="card-title">Available Budget</h3>
      </div>
      <div class="budget-display">{{ formatBudget(budget) }}</div>
      <input
        type="range"
        min="10000"
        max="500000"
        step="10000"
        v-model.number="budget"
        class="budget-slider"
      />
      <div class="util-bar-wrap">
        <div
          class="util-bar"
          :style="{ width: utilizationPct + '%', backgroundColor: utilizationColor }"
        ></div>
      </div>
      <div class="util-label">
        {{ formatBudget(totalCost) }} used of {{ formatBudget(budget) }} ({{ utilizationPct }}%)
      </div>
    </div>

    <!-- Recommendations Table Card -->
    <div class="card">
      <div class="card-header">
        <h3 class="card-title">
          Recommended Restocking
          <span class="count-badge">{{ recommendations.length }}</span>
        </h3>
      </div>

      <div v-if="loading" class="loading">Loading recommendations...</div>

      <div v-else-if="recommendations.length === 0" class="empty-state">
        Increase your budget to unlock restocking recommendations.
      </div>

      <div v-else>
        <div class="table-container">
          <table>
            <thead>
              <tr>
                <th>SKU</th>
                <th>Item Name</th>
                <th>Trend</th>
                <th>Restock Qty</th>
                <th>Unit Cost</th>
                <th>Total Cost</th>
              </tr>
            </thead>
            <tbody>
              <tr v-for="item in recommendations" :key="item.item_sku">
                <td>{{ item.item_sku }}</td>
                <td>{{ item.item_name }}</td>
                <td>
                  <span :class="['badge', item.trend]">{{ item.trend }}</span>
                </td>
                <td>{{ item.restock_quantity.toLocaleString() }}</td>
                <td>{{ formatCurrency(item.unit_cost) }}</td>
                <td><strong>{{ formatCurrency(item.total_cost) }}</strong></td>
              </tr>
            </tbody>
            <tfoot>
              <tr>
                <td colspan="5" class="tfoot-total">Total Cost</td>
                <td><strong>{{ formatCurrency(totalCost) }}</strong></td>
              </tr>
            </tfoot>
          </table>
        </div>

        <div v-if="!orderPlaced" class="action-bar">
          <button
            class="btn-primary"
            :disabled="recommendations.length === 0 || loading || orderPlaced || submitting"
            @click="placeOrder"
          >
            {{ submitting ? 'Placing Order...' : 'Place Order' }}
          </button>
        </div>
      </div>
    </div>

    <!-- Success State -->
    <div v-if="orderPlaced && placedOrder" class="success-card">
      <div class="success-title">Order Placed Successfully</div>
      <div class="order-number">{{ placedOrder.order_number }}</div>
      <div class="success-meta">
        Expected delivery: {{ formatDelivery(placedOrder.expected_delivery) }}
      </div>
      <div style="margin-bottom: 1rem;">
        <router-link to="/orders" class="view-link">View in Orders tab</router-link>
      </div>
      <button class="btn-secondary" @click="resetOrder">Place Another Order</button>
    </div>
  </div>
</template>

<script>
import { ref, watch, computed, onMounted } from 'vue'
import { api } from '../api'

export default {
  name: 'Restocking',
  setup() {
    const budget = ref(100000)

    const recommendations = ref([])
    const totalCost = ref(0)
    const budgetRemaining = ref(0)

    const loading = ref(false)
    const submitting = ref(false)
    const orderPlaced = ref(false)
    const placedOrder = ref(null)

    let debounceTimer = null
    function debounce(fn, delay) {
      return (...args) => {
        clearTimeout(debounceTimer)
        debounceTimer = setTimeout(() => fn(...args), delay)
      }
    }

    const loadRecommendations = async (val) => {
      loading.value = true
      try {
        const res = await api.getRestockingRecommendations(val)
        recommendations.value = res.recommendations
        totalCost.value = res.total_cost
        budgetRemaining.value = res.budget_remaining
      } catch (e) {
        recommendations.value = []
      } finally {
        loading.value = false
      }
    }

    const debouncedLoad = debounce(loadRecommendations, 300)

    watch(budget, (val) => debouncedLoad(val))

    onMounted(() => loadRecommendations(budget.value))

    const placeOrder = async () => {
      submitting.value = true
      try {
        const res = await api.createRestockingOrder(budget.value)
        placedOrder.value = res
        orderPlaced.value = true
      } finally {
        submitting.value = false
      }
    }

    const resetOrder = () => {
      orderPlaced.value = false
      placedOrder.value = null
      loadRecommendations(budget.value)
    }

    const formatCurrency = (val) =>
      '$' + val.toLocaleString('en-US', { minimumFractionDigits: 2, maximumFractionDigits: 2 })

    const formatBudget = (val) => '$' + val.toLocaleString('en-US')

    const utilizationPct = computed(() =>
      budget.value > 0 ? Math.min(100, Math.round((totalCost.value / budget.value) * 100)) : 0
    )

    const utilizationColor = computed(() =>
      utilizationPct.value >= 100
        ? '#dc2626'
        : utilizationPct.value >= 80
        ? '#ea580c'
        : '#059669'
    )

    const formatDelivery = (isoStr) => {
      if (!isoStr) return ''
      return new Date(isoStr).toLocaleDateString('en-US', {
        year: 'numeric',
        month: 'long',
        day: 'numeric'
      })
    }

    return {
      budget,
      recommendations,
      totalCost,
      budgetRemaining,
      loading,
      submitting,
      orderPlaced,
      placedOrder,
      placeOrder,
      resetOrder,
      formatCurrency,
      formatBudget,
      utilizationPct,
      utilizationColor,
      formatDelivery
    }
  }
}
</script>

<style scoped>
.budget-display {
  font-size: 2.5rem;
  font-weight: 700;
  color: #0f172a;
  margin-bottom: 0.75rem;
  letter-spacing: -0.03em;
}

.budget-slider {
  width: 100%;
  accent-color: #2563eb;
  margin-bottom: 1rem;
}

.util-bar-wrap {
  background: #f1f5f9;
  border-radius: 6px;
  height: 8px;
  margin-bottom: 0.5rem;
  overflow: hidden;
}

.util-bar {
  height: 100%;
  border-radius: 6px;
  transition: width 0.3s ease, background-color 0.3s ease;
}

.util-label {
  font-size: 0.875rem;
  color: #64748b;
}

.btn-primary {
  background: #2563eb;
  color: white;
  border: none;
  padding: 0.625rem 1.5rem;
  border-radius: 8px;
  font-size: 0.938rem;
  font-weight: 600;
  cursor: pointer;
  transition: background 0.2s;
}

.btn-primary:hover:not(:disabled) {
  background: #1d4ed8;
}

.btn-primary:disabled {
  opacity: 0.5;
  cursor: not-allowed;
}

.btn-secondary {
  background: #f1f5f9;
  color: #0f172a;
  border: 1px solid #e2e8f0;
  padding: 0.625rem 1.5rem;
  border-radius: 8px;
  font-size: 0.938rem;
  font-weight: 600;
  cursor: pointer;
}

.empty-state {
  text-align: center;
  padding: 3rem;
  color: #64748b;
}

.success-card {
  background: #f0fdf4;
  border: 1px solid #bbf7d0;
  border-radius: 10px;
  padding: 1.5rem;
  margin-top: 1rem;
}

.success-title {
  font-size: 1.25rem;
  font-weight: 700;
  color: #065f46;
  margin-bottom: 0.5rem;
}

.success-meta {
  color: #047857;
  font-size: 0.938rem;
  margin-bottom: 1rem;
}

.order-number {
  font-size: 1.5rem;
  font-weight: 700;
  color: #0f172a;
  margin: 0.5rem 0;
}

.view-link {
  color: #2563eb;
  text-decoration: none;
  font-weight: 500;
}

.view-link:hover {
  text-decoration: underline;
}

.action-bar {
  display: flex;
  justify-content: flex-end;
  margin-top: 1rem;
}

.tfoot-total {
  text-align: right;
  font-weight: 700;
  color: #0f172a;
  font-size: 1rem;
}

.count-badge {
  background: #f1f5f9;
  color: #64748b;
  border-radius: 12px;
  padding: 0.125rem 0.625rem;
  font-size: 0.813rem;
  font-weight: 600;
  margin-left: 0.5rem;
}
</style>
