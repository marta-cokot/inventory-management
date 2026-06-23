<template>
  <Teleport to="body">
    <Transition name="modal">
      <div v-if="isOpen && backlogItem" class="modal-overlay" @click="close">
        <div class="modal-container" @click.stop>
          <div class="modal-header">
            <h3 class="modal-title">
              {{ mode === 'create' ? 'Create Purchase Order' : 'Purchase Order Details' }}
            </h3>
            <button class="close-button" @click="close">
              <svg width="20" height="20" viewBox="0 0 20 20" fill="none">
                <path d="M15 5L5 15M5 5L15 15" stroke="currentColor" stroke-width="2" stroke-linecap="round"/>
              </svg>
            </button>
          </div>

          <div class="modal-body">
            <!-- Item context header -->
            <div class="item-context">
              <div class="item-context-label">Backlog Item</div>
              <div class="item-context-name">{{ translateProductName(backlogItem.item_name) }}</div>
              <div class="item-context-sku">SKU: {{ backlogItem.item_sku }}</div>
            </div>

            <!-- Create mode: form -->
            <form v-if="mode === 'create'" class="po-form" @submit.prevent="handleSubmit">
              <div class="form-group">
                <label class="form-label" for="supplier-name">Supplier Name <span class="required">*</span></label>
                <input
                  id="supplier-name"
                  v-model="form.supplier_name"
                  type="text"
                  class="form-input"
                  placeholder="Enter supplier name"
                  required
                />
              </div>

              <div class="form-row">
                <div class="form-group">
                  <label class="form-label" for="quantity">Quantity <span class="required">*</span></label>
                  <input
                    id="quantity"
                    v-model.number="form.quantity"
                    type="number"
                    class="form-input"
                    min="1"
                    required
                  />
                </div>

                <div class="form-group">
                  <label class="form-label" for="unit-cost">Unit Cost <span class="required">*</span></label>
                  <input
                    id="unit-cost"
                    v-model.number="form.unit_cost"
                    type="number"
                    class="form-input"
                    min="0"
                    step="0.01"
                    placeholder="0.00"
                    required
                  />
                </div>
              </div>

              <div class="form-group">
                <label class="form-label" for="expected-delivery">Expected Delivery Date <span class="required">*</span></label>
                <input
                  id="expected-delivery"
                  v-model="form.expected_delivery_date"
                  type="date"
                  class="form-input"
                  required
                />
              </div>

              <div class="form-group">
                <label class="form-label" for="notes">Notes</label>
                <textarea
                  id="notes"
                  v-model="form.notes"
                  class="form-textarea"
                  rows="3"
                  placeholder="Optional notes..."
                ></textarea>
              </div>

              <div v-if="formError" class="form-error">{{ formError }}</div>
            </form>

            <!-- View mode: loading / not found / data -->
            <div v-else>
              <div v-if="viewLoading" class="state-loading">Loading purchase order...</div>
              <div v-else-if="viewNotFound" class="state-empty">No purchase order found for this item.</div>
              <div v-else-if="viewError" class="state-error">{{ viewError }}</div>
              <div v-else-if="purchaseOrder" class="po-details">
                <div class="info-grid">
                  <div class="info-item">
                    <div class="info-label">PO ID</div>
                    <div class="info-value mono">{{ purchaseOrder.id }}</div>
                  </div>

                  <div class="info-item">
                    <div class="info-label">Status</div>
                    <div class="info-value">
                      <span class="status-badge" :class="purchaseOrder.status">{{ purchaseOrder.status }}</span>
                    </div>
                  </div>

                  <div class="info-item">
                    <div class="info-label">Supplier</div>
                    <div class="info-value">{{ purchaseOrder.supplier_name }}</div>
                  </div>

                  <div class="info-item">
                    <div class="info-label">Quantity</div>
                    <div class="info-value">{{ purchaseOrder.quantity }} units</div>
                  </div>

                  <div class="info-item">
                    <div class="info-label">Unit Cost</div>
                    <div class="info-value">{{ formatCost(purchaseOrder.unit_cost) }}</div>
                  </div>

                  <div class="info-item">
                    <div class="info-label">Total Cost</div>
                    <div class="info-value total-cost">{{ formatCost(purchaseOrder.unit_cost * purchaseOrder.quantity) }}</div>
                  </div>

                  <div class="info-item">
                    <div class="info-label">Expected Delivery</div>
                    <div class="info-value">{{ formatDate(purchaseOrder.expected_delivery_date) }}</div>
                  </div>

                  <div class="info-item">
                    <div class="info-label">Created</div>
                    <div class="info-value">{{ formatDate(purchaseOrder.created_date) }}</div>
                  </div>
                </div>

                <div v-if="purchaseOrder.notes" class="notes-section">
                  <div class="info-label">Notes</div>
                  <div class="notes-content">{{ purchaseOrder.notes }}</div>
                </div>
              </div>
            </div>
          </div>

          <div class="modal-footer">
            <button class="btn-secondary" @click="close" :disabled="formLoading">Cancel</button>
            <button
              v-if="mode === 'create'"
              class="btn-primary"
              :disabled="formLoading"
              @click="handleSubmit"
            >
              <span v-if="formLoading">Creating...</span>
              <span v-else>Create Purchase Order</span>
            </button>
          </div>
        </div>
      </div>
    </Transition>
  </Teleport>
</template>

<script>
import { ref, watch } from 'vue'
import { useI18n } from '../composables/useI18n'
import { api } from '../api'

export default {
  name: 'PurchaseOrderModal',

  props: {
    isOpen: {
      type: Boolean,
      default: false
    },
    backlogItem: {
      type: Object,
      default: null
    },
    mode: {
      type: String,
      default: 'create'
    }
  },

  emits: ['close', 'po-created'],

  setup(props, { emit }) {
    const { translateProductName } = useI18n()

    // Create mode state
    const form = ref({
      supplier_name: '',
      quantity: 0,
      unit_cost: '',
      expected_delivery_date: '',
      notes: ''
    })
    const formLoading = ref(false)
    const formError = ref(null)

    // View mode state
    const purchaseOrder = ref(null)
    const viewLoading = ref(false)
    const viewError = ref(null)
    const viewNotFound = ref(false)

    const resetForm = () => {
      form.value = {
        supplier_name: '',
        quantity: props.backlogItem ? props.backlogItem.quantity_needed : 0,
        unit_cost: '',
        expected_delivery_date: '',
        notes: ''
      }
      formError.value = null
    }

    const loadPurchaseOrder = async () => {
      if (!props.backlogItem) return
      viewLoading.value = true
      viewError.value = null
      viewNotFound.value = false
      purchaseOrder.value = null
      try {
        purchaseOrder.value = await api.getPurchaseOrderByBacklogItem(props.backlogItem.id)
      } catch (err) {
        if (err.response && err.response.status === 404) {
          viewNotFound.value = true
        } else {
          viewError.value = 'Failed to load purchase order.'
        }
      } finally {
        viewLoading.value = false
      }
    }

    // When modal opens, initialize state based on mode
    watch(
      () => props.isOpen,
      (opened) => {
        if (opened) {
          if (props.mode === 'create') {
            resetForm()
          } else {
            loadPurchaseOrder()
          }
        }
      }
    )

    const close = () => {
      emit('close')
    }

    const handleSubmit = async () => {
      if (formLoading.value) return
      formError.value = null
      formLoading.value = true
      try {
        const payload = {
          backlog_item_id: props.backlogItem.id,
          supplier_name: form.value.supplier_name,
          quantity: form.value.quantity,
          unit_cost: form.value.unit_cost,
          expected_delivery_date: form.value.expected_delivery_date,
          notes: form.value.notes || undefined
        }
        const result = await api.createPurchaseOrder(payload)
        emit('po-created', result)
        emit('close')
      } catch (err) {
        formError.value =
          err.response?.data?.detail || 'Failed to create purchase order. Please try again.'
      } finally {
        formLoading.value = false
      }
    }

    const formatDate = (dateString) => {
      if (!dateString) return 'N/A'
      const date = new Date(dateString)
      if (isNaN(date.getTime())) return dateString
      return date.toLocaleDateString('en-US', {
        year: 'numeric',
        month: 'long',
        day: 'numeric'
      })
    }

    const formatCost = (value) => {
      if (value == null) return 'N/A'
      return new Intl.NumberFormat('en-US', { style: 'currency', currency: 'USD' }).format(value)
    }

    return {
      form,
      formLoading,
      formError,
      purchaseOrder,
      viewLoading,
      viewError,
      viewNotFound,
      translateProductName,
      close,
      handleSubmit,
      formatDate,
      formatCost
    }
  }
}
</script>

<style scoped>
.modal-overlay {
  position: fixed;
  top: 0;
  left: 0;
  right: 0;
  bottom: 0;
  background: rgba(0, 0, 0, 0.5);
  display: flex;
  align-items: center;
  justify-content: center;
  z-index: 2000;
  padding: 1rem;
}

.modal-container {
  background: white;
  border-radius: 12px;
  box-shadow: 0 20px 50px rgba(0, 0, 0, 0.15);
  max-width: 600px;
  width: 100%;
  max-height: 90vh;
  overflow: hidden;
  display: flex;
  flex-direction: column;
}

.modal-header {
  display: flex;
  align-items: center;
  justify-content: space-between;
  padding: 1.5rem;
  border-bottom: 1px solid #e2e8f0;
}

.modal-title {
  font-size: 1.25rem;
  font-weight: 700;
  color: #0f172a;
  letter-spacing: -0.025em;
}

.close-button {
  background: none;
  border: none;
  color: #64748b;
  cursor: pointer;
  padding: 0.5rem;
  display: flex;
  align-items: center;
  justify-content: center;
  border-radius: 6px;
  transition: all 0.15s ease;
}

.close-button:hover {
  background: #f1f5f9;
  color: #0f172a;
}

.modal-body {
  flex: 1;
  overflow-y: auto;
  padding: 2rem;
}

.modal-footer {
  padding: 1.5rem;
  border-top: 1px solid #e2e8f0;
  display: flex;
  justify-content: flex-end;
  gap: 0.75rem;
}

/* Item context */
.item-context {
  background: #f8fafc;
  border: 1px solid #e2e8f0;
  border-radius: 8px;
  padding: 1rem 1.25rem;
  margin-bottom: 1.75rem;
}

.item-context-label {
  font-size: 0.75rem;
  font-weight: 600;
  text-transform: uppercase;
  letter-spacing: 0.05em;
  color: #64748b;
  margin-bottom: 0.25rem;
}

.item-context-name {
  font-size: 1rem;
  font-weight: 600;
  color: #0f172a;
  margin-bottom: 0.25rem;
}

.item-context-sku {
  font-size: 0.813rem;
  color: #64748b;
  font-family: 'Monaco', 'Courier New', monospace;
}

/* Form */
.po-form {
  display: flex;
  flex-direction: column;
  gap: 1.25rem;
}

.form-row {
  display: grid;
  grid-template-columns: 1fr 1fr;
  gap: 1rem;
}

.form-group {
  display: flex;
  flex-direction: column;
  gap: 0.5rem;
}

.form-label {
  font-size: 0.813rem;
  font-weight: 600;
  color: #0f172a;
  letter-spacing: 0.01em;
}

.required {
  color: #ef4444;
}

.form-input,
.form-textarea {
  padding: 0.625rem 0.875rem;
  border: 1px solid #e2e8f0;
  border-radius: 8px;
  font-size: 0.938rem;
  color: #0f172a;
  background: white;
  font-family: inherit;
  transition: border-color 0.15s ease, box-shadow 0.15s ease;
  width: 100%;
  box-sizing: border-box;
}

.form-input:focus,
.form-textarea:focus {
  outline: none;
  border-color: #2563eb;
  box-shadow: 0 0 0 3px rgba(37, 99, 235, 0.1);
}

.form-textarea {
  resize: vertical;
  min-height: 80px;
}

.form-error {
  padding: 0.75rem 1rem;
  background: #fef2f2;
  border: 1px solid #fecaca;
  border-radius: 8px;
  font-size: 0.875rem;
  color: #dc2626;
}

/* View mode states */
.state-loading,
.state-empty,
.state-error {
  text-align: center;
  padding: 2rem 1rem;
  font-size: 0.938rem;
}

.state-loading {
  color: #64748b;
}

.state-empty {
  color: #64748b;
}

.state-error {
  color: #dc2626;
}

/* PO detail view */
.info-grid {
  display: grid;
  grid-template-columns: repeat(auto-fit, minmax(200px, 1fr));
  gap: 1.5rem;
  margin-bottom: 1.5rem;
}

.info-item {
  display: flex;
  flex-direction: column;
  gap: 0.375rem;
}

.info-label {
  font-size: 0.75rem;
  font-weight: 600;
  text-transform: uppercase;
  letter-spacing: 0.05em;
  color: #64748b;
}

.info-value {
  font-size: 0.938rem;
  color: #0f172a;
  font-weight: 500;
}

.info-value.mono {
  font-family: 'Monaco', 'Courier New', monospace;
  color: #2563eb;
}

.info-value.total-cost {
  font-weight: 700;
  font-size: 1.063rem;
}

/* Status badge */
.status-badge {
  display: inline-block;
  padding: 0.25rem 0.625rem;
  border-radius: 6px;
  font-size: 0.813rem;
  font-weight: 600;
  text-transform: capitalize;
}

.status-badge.pending {
  background: #fef9c3;
  color: #92400e;
}

.status-badge.ordered {
  background: #dbeafe;
  color: #1e40af;
}

.status-badge.delivered {
  background: #dcfce7;
  color: #166534;
}

.status-badge.cancelled {
  background: #fee2e2;
  color: #991b1b;
}

/* Notes */
.notes-section {
  border-top: 1px solid #e2e8f0;
  padding-top: 1.25rem;
}

.notes-content {
  margin-top: 0.5rem;
  font-size: 0.938rem;
  color: #334155;
  line-height: 1.6;
  white-space: pre-wrap;
}

/* Buttons */
.btn-secondary {
  padding: 0.625rem 1.25rem;
  background: #f1f5f9;
  border: 1px solid #e2e8f0;
  border-radius: 8px;
  font-weight: 500;
  font-size: 0.875rem;
  color: #334155;
  cursor: pointer;
  transition: all 0.15s ease;
  font-family: inherit;
}

.btn-secondary:hover:not(:disabled) {
  background: #e2e8f0;
  border-color: #cbd5e1;
}

.btn-secondary:disabled {
  opacity: 0.6;
  cursor: not-allowed;
}

.btn-primary {
  padding: 0.625rem 1.25rem;
  background: #2563eb;
  border: 1px solid #2563eb;
  border-radius: 8px;
  font-weight: 600;
  font-size: 0.875rem;
  color: white;
  cursor: pointer;
  transition: all 0.15s ease;
  font-family: inherit;
}

.btn-primary:hover:not(:disabled) {
  background: #1d4ed8;
  border-color: #1d4ed8;
}

.btn-primary:disabled {
  opacity: 0.6;
  cursor: not-allowed;
}

/* Modal transition animations */
.modal-enter-active,
.modal-leave-active {
  transition: opacity 0.2s ease;
}

.modal-enter-from,
.modal-leave-to {
  opacity: 0;
}

.modal-enter-active .modal-container,
.modal-leave-active .modal-container {
  transition: transform 0.2s ease;
}

.modal-enter-from .modal-container,
.modal-leave-to .modal-container {
  transform: scale(0.95);
}
</style>
