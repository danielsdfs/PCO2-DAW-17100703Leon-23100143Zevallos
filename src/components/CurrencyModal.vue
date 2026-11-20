<template>
  <q-dialog
    :model-value="modelValue"
    @update:model-value="$emit('update:modelValue', $event)"
  >
    <q-card style="width: 460px; max-width: 95vw; border-radius: 16px">
      <q-card-section class="row items-center justify-between q-pb-sm">
        <div class="row items-center q-gutter-x-sm">
          <q-avatar color="green-1" text-color="green-9" icon="currency_exchange" size="36px" />
          <div>
            <div class="text-subtitle1 text-weight-bold text-grey-10">
              {{ currency?.name || 'Conversor de moneda' }}
            </div>
            <div class="text-caption text-grey-7">
              Código: {{ currency?.code }} ({{ currency?.symbol }})
            </div>
          </div>
        </div>

        <q-btn icon="close" flat round dense v-close-popup />
      </q-card-section>

      <q-separator />

      <!-- Cargando tasa -->
      <q-card-section v-if="loadingRate" class="q-pa-lg text-center">
        <q-spinner-dots color="green-8" size="36px" />
        <div class="text-caption text-grey-7 q-mt-xs">
          Consultando tasa de cambio...
        </div>
      </q-card-section>

      <!-- Error al obtener tasa -->
      <q-card-section v-else-if="rateError">
        <q-banner rounded class="bg-red-1 text-red-9">
          <template #avatar>
            <q-icon name="warning" color="red-8" />
          </template>
          {{ rateError }}
          <template #action>
            <q-btn flat color="red-9" label="Reintentar" @click="loadRate" />
          </template>
        </q-banner>
      </q-card-section>

      <!-- Tasa y conversor -->
      <q-card-section v-else class="q-gutter-y-md">
        <!-- Tasa referencial -->
        <div class="bg-green-1 q-pa-md rounded-borders">
          <div class="text-caption text-green-9 text-weight-medium">
            Tasa de cambio referencial
          </div>
          <div class="text-h6 text-weight-bold text-green-10">
            1 {{ currency?.code }} = {{ usdRateText }} USD
          </div>
        </div>

        <!-- Formulario de conversión -->
        <div>
          <div class="text-caption text-weight-medium text-grey-8 q-mb-xs">
            Monto en {{ currency?.code }}
          </div>
          <div class="row q-col-gutter-sm">
            <div class="col-8">
              <q-input
                v-model.number="amount"
                type="number"
                min="0"
                outlined
                dense
                bg-color="white"
                placeholder="Ej. 100"
                @keyup.enter="handleConvert"
              >
                <template #prepend>
                  <span class="text-caption text-weight-bold text-grey-7">
                    {{ currency?.symbol }}
                  </span>
                </template>
              </q-input>
            </div>
            <div class="col-4">
              <q-btn
                color="primary"
                label="Convertir"
                class="full-width full-height"
                unelevated
                :loading="converting"
                :disable="!amount || amount <= 0"
                @click="handleConvert"
              />
            </div>
          </div>
        </div>

        <!-- Resultado de conversión -->
        <div
          v-if="convertedUsd !== null"
          class="bg-grey-2 q-pa-md rounded-borders row items-center justify-between"
        >
          <div class="text-body2 text-grey-8">Equivalente en dólares:</div>
          <div class="text-h6 text-weight-bold text-grey-10">
            $ {{ convertedUsd }} USD
          </div>
        </div>
      </q-card-section>
    </q-card>
  </q-dialog>
</template>

<script setup>
import { computed, ref, watch } from 'vue'
import {
  convertCurrencyToUsd,
  fetchCurrencyRate
} from '../services/restCountriesService.js'

const props = defineProps({
  modelValue: {
    type: Boolean,
    default: false
  },
  currency: {
    type: Object,
    default: () => ({ code: 'USD', name: 'Dólar estadounidense', symbol: '$' })
  }
})

defineEmits(['update:modelValue'])

const loadingRate = ref(false)
const rateError = ref(null)
const usdRate = ref(null)

const amount = ref(100)
const converting = ref(false)
const convertedUsd = ref(null)

const usdRateText = computed(() => {
  if (usdRate.value === null || usdRate.value === undefined) return 'N/A'
  return Number(usdRate.value).toFixed(4)
})

async function loadRate() {
  const code = props.currency?.code
  if (!code) return

  if (code.toUpperCase() === 'USD') {
    usdRate.value = 1
    convertedUsd.value = Number(amount.value || 0).toFixed(2)
    return
  }

  loadingRate.value = true
  rateError.value = null
  convertedUsd.value = null

  try {
    const res = await fetchCurrencyRate(code)
    const rate =
      res?.rates?.USD ??
      res?.data?.rates?.USD ??
      res?.rate ??
      res?.usd_rate ??
      null
    usdRate.value = rate
  } catch (err) {
    rateError.value =
      err?.response?.data?.message ||
      `No se pudo obtener la tasa de cambio para ${code}.`
  } finally {
    loadingRate.value = false
  }
}

async function handleConvert() {
  const code = props.currency?.code
  const numericAmount = Number(amount.value)
  if (!code || !numericAmount || numericAmount <= 0) return

  if (code.toUpperCase() === 'USD') {
    convertedUsd.value = numericAmount.toFixed(2)
    return
  }

  converting.value = true
  try {
    const res = await convertCurrencyToUsd(code, numericAmount)
    const result =
      res?.result ??
      res?.convertedAmount ??
      res?.converted_amount ??
      res?.data?.result ??
      (usdRate.value ? numericAmount * Number(usdRate.value) : null)

    convertedUsd.value = result !== null ? Number(result).toFixed(2) : 'N/A'
  } catch (err) {
    if (usdRate.value) {
      convertedUsd.value = (numericAmount * Number(usdRate.value)).toFixed(2)
    } else {
      rateError.value = 'Error al realizar la conversión a USD.'
    }
  } finally {
    converting.value = false
  }
}

watch(
  () => [props.modelValue, props.currency?.code],
  ([isOpen]) => {
    if (isOpen) {
      amount.value = 100
      convertedUsd.value = null
      loadRate()
    }
  },
  { immediate: true }
)
</script>
