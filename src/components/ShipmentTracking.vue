<template>
  <v-container class="fill-height">
    <v-row justify="center" align="center">
      <v-col cols="12" sm="10" md="8" lg="6">
        <!-- Logo -->
        <div class="text-center mb-6">
          <img 
            src="/logo.svg" 
            alt="Sporttrader Logo" 
            class="logo"
          />
        </div>

        <v-card elevation="8" class="mx-auto">
          <v-card-title class="text-h4 text-center py-6 bg-primary">
            <v-icon size="large" class="mr-2">mdi-package-variant</v-icon>
            Track Your Shipment
          </v-card-title>

          <v-card-text class="pa-6">
            <v-form ref="formRef" v-model="valid" @submit.prevent="trackShipment">
              <v-text-field
                v-model="orderId"
                label="Order ID"
                placeholder="Enter your order ID"
                :rules="[rules.required]"
                variant="outlined"
                prepend-inner-icon="mdi-identifier"
                class="mb-4"
                :disabled="loading"
              />

              <v-text-field
                v-model="zipcode"
                label="Zipcode"
                placeholder="Enter your zipcode"
                :rules="[rules.required]"
                variant="outlined"
                prepend-inner-icon="mdi-map-marker"
                class="mb-4"
                :disabled="loading"
              />

              <v-btn
                type="submit"
                color="primary"
                size="large"
                block
                :loading="loading"
                :disabled="!valid"
              >
                <v-icon left class="mr-2">mdi-magnify</v-icon>
                Track Shipment
              </v-btn>
            </v-form>

            <!-- Error Alert -->
            <v-alert
              v-if="error"
              type="error"
              variant="tonal"
              class="mt-6"
              closable
              @click:close="error = null"
            >
              <v-alert-title>Error</v-alert-title>
              {{ error }}
            </v-alert>

            <!-- Shipments Results -->
            <div v-if="shipments.length > 0" class="mt-6">
              <v-divider class="mb-4" />
              
              <h3 class="text-h6 mb-4">
                <v-icon class="mr-2">mdi-truck-delivery</v-icon>
                Shipment Details
              </h3>

              <v-list lines="two">
                <v-list-item
                  v-for="(shipment, index) in shipments"
                  :key="index"
                  class="mb-3 border rounded"
                >
                  <template v-slot:prepend>
                    <v-avatar color="primary">
                      <v-icon>mdi-package</v-icon>
                    </v-avatar>
                  </template>

                  <v-list-item-title class="font-weight-bold mb-2">
                    Shipment {{ index + 1 }}
                  </v-list-item-title>

                  <v-list-item-subtitle>
                    <div class="mb-2">
                      <strong>Barcode:</strong> 
                      <v-chip size="small" class="ml-2" color="success">
                        {{ shipment.barcode }}
                      </v-chip>
                    </div>
                  </v-list-item-subtitle>

                  <template v-slot:append>
                    <v-btn
                      :href="shipment.track_trace_url"
                      target="_blank"
                      color="primary"
                      variant="elevated"
                      size="small"
                    >
                      <v-icon left size="small" class="mr-1">mdi-open-in-new</v-icon>
                      Track & Trace
                    </v-btn>
                  </template>
                </v-list-item>
              </v-list>
            </div>

            <!-- No Results Message -->
            <v-alert
              v-if="searched && shipments.length === 0 && !error"
              type="info"
              variant="tonal"
              class="mt-6"
            >
              No shipments found for this order.
            </v-alert>
          </v-card-text>
        </v-card>
      </v-col>
    </v-row>
  </v-container>
</template>

<script setup>
import { ref, onMounted } from 'vue'

const API_BASE_URL = 'https://api.sporttrader.nl'

// Form state
const formRef = ref(null)
const valid = ref(false)
const orderId = ref('')
const zipcode = ref('')
const loading = ref(false)
const error = ref(null)
const shipments = ref([])
const searched = ref(false)

// On mount, check if there's an order ID in the URL path
onMounted(() => {
  const path = window.location.pathname
  // Extract order ID from path (e.g., /pl7737042305-B -> pl7737042305-B)
  if (path && path !== '/') {
    const orderIdFromUrl = path.substring(1).trim() // Remove leading slash
    if (orderIdFromUrl) {
      orderId.value = orderIdFromUrl
    }
  }
})

// Validation rules
const rules = {
  required: (value) => !!value || 'This field is required',
}

// Track shipment function
const trackShipment = async () => {
  if (!valid.value) return

  loading.value = true
  error.value = null
  shipments.value = []
  searched.value = false

  try {
    const params = new URLSearchParams({
      zipcode: zipcode.value.trim()
    })

    const response = await fetch(
      `${API_BASE_URL}/v1/public/shipments/${encodeURIComponent(orderId.value.trim())}?${params}`,
      {
        method: 'GET',
        headers: {
          'Accept': 'application/json',
        },
      }
    )

    const data = await response.json()

    if (!response.ok) {
      // Handle error response
      if (response.status === 404) {
        error.value = 'Order not found. Please check your Order ID and Zipcode.'
      } else if (response.status === 422) {
        error.value = data.detail?.[0]?.msg || 'Invalid input. Please check your Order ID and Zipcode.'
      } else if (response.status === 429) {
        error.value = 'Rate limit exceeded. Please try again later.'
      } else {
        error.value = data.detail || 'An error occurred while fetching shipment details.'
      }
    } else {
      // Success
      shipments.value = data.shipments || []
      searched.value = true
    }
  } catch (err) {
    error.value = `Network error: ${err.message}. Please check your connection and try again.`
  } finally {
    loading.value = false
  }
}
</script>

<style scoped>
.border {
  border: 1px solid rgba(0, 0, 0, 0.12);
}

.logo {
  max-width: 200px;
  height: auto;
  transition: transform 0.3s ease;
}

.logo:hover {
  transform: scale(1.05);
}
</style>

