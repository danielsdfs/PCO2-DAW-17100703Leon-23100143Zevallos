
<template>
  <q-page class="bg-grey-2 q-pa-md">
    <div class="container">
      <div class="q-mb-lg">
        <div class="text-h4 text-weight-bold text-primary">
          Country Explorer
        </div>
        <div class="text-grey-7">
          Explora países del mundo, sus capitales y monedas.
        </div>
      </div>

      <!-- Buscador y filtro -->
      <div class="row q-col-gutter-md q-mb-lg">
        <div class="col-12 col-md-8">
          <q-input
            v-model="capital"
            outlined
            rounded
            bg-white
            placeholder="Buscar por capital..."
            debounce="500"
            @update:model-value="buscarPaises"
          >
            <template v-slot:prepend>
              <q-icon name="search" />
            </template>
          </q-input>
        </div>

        <div class="col-12 col-md-4">
          <q-select
            v-model="continente"
            outlined
            rounded
            bg-white
            label="Continente"
            emit-value
            map-options
            :options="continentes"
            @update:model-value="filtrarContinente"
          />
        </div>
      </div>

      <!-- Cargando -->
      <div v-if="cargando" class="row q-col-gutter-md">
        <div
          v-for="n in 8"
          :key="n"
          class="col-12 col-sm-6 col-md-3"
        >
          <q-skeleton height="170px" />
        </div>
      </div>

      <!-- Error -->
      <q-banner
        v-else-if="error"
        class="bg-red-1 text-negative rounded-borders q-mb-md"
      >
        {{ error }}
        <template v-slot:action>
          <q-btn flat label="Reintentar" @click="cargarPaises" />
        </template>
      </q-banner>

      <!-- Listado -->
      <div v-else-if="paises.length" class="row q-col-gutter-md">
        <div
          v-for="pais in paises"
          :key="pais.codes?.alpha_2"
          class="col-12 col-sm-6 col-md-3"
        >
          <q-card class="country-card full-height" flat bordered>
            <q-img
              :src="bandera(pais)"
              height="120px"
              fit="contain"
              class="bg-white"
            />

            <q-card-section>
              <div class="text-h6 text-weight-bold">
                {{ pais.names?.common }}
              </div>

              <div class="text-grey-7 q-mt-sm">
                Capital:
                {{ pais.capitals?.[0]?.name || 'No disponible' }}
              </div>

              <div class="text-grey-7 q-mt-xs">
                Población:
                {{ formatearPoblacion(pais.population) }}
              </div>

              <div class="q-mt-sm">
                <q-badge color="blue-1" text-color="blue-9">
                  {{ pais.subregion || pais.region || 'Sin continente' }}
                </q-badge>
              </div>

              <div class="q-mt-md">
                <q-chip
                  v-for="moneda in pais.currencies || []"
                  :key="moneda.code"
                  color="green-1"
                  text-color="green-9"
                  icon="payments"
                  clickable
                  @click="seleccionarMoneda(moneda)"
                >
                  {{ moneda.code }}
                </q-chip>
              </div>
            </q-card-section>
          </q-card>
        </div>
      </div>

      <!-- Sin resultados -->
      <div v-else class="text-center q-pa-xl">
        <q-icon name="travel_explore" size="60px" color="grey-5" />
        <div class="text-h6 text-grey-7 q-mt-md">
          No se encontraron países
        </div>
        <div class="text-grey-6">
          Intenta con otra capital o continente.
        </div>
      </div>
    </div>
  </q-page>
</template>

<script setup>
import { ref, onMounted } from 'vue'
import { Notify } from 'quasar'

const API_URL = 'https://api.restcountries.com/countries/v5'
const API_KEY = import.meta.env.VITE_RESTCOUNTRIES_API_KEY

const paises = ref([])
const capital = ref('')
const continente = ref(null)
const cargando = ref(false)
const error = ref('')

const continentes = [
  { label: 'Todos los continentes', value: null },
  { label: 'África', value: 'Africa' },
  { label: 'América', value: 'Americas' },
  { label: 'Asia', value: 'Asia' },
  { label: 'Europa', value: 'Europe' },
  { label: 'Oceanía', value: 'Oceania' }
]

async function consultarAPI(endpoint) {
  if (!API_KEY) {
    throw new Error('Configura tu API Key en el archivo .env.local')
  }

  const respuesta = await fetch(`${API_URL}${endpoint}`, {
    headers: {
      Authorization: `Bearer ${API_KEY}`
    }
  })

  if (!respuesta.ok) {
    throw new Error(`Error al consultar la API: ${respuesta.status}`)
  }

  const resultado = await respuesta.json()
  return resultado.data?.objects || []
}

async function cargarPaises() {
  cargando.value = true
  error.value = ''

  try {
    paises.value = await consultarAPI('?limit=100')
  } catch (e) {
    error.value = e.message
  } finally {
    cargando.value = false
  }
}

async function buscarPaises(texto) {
  if (!texto.trim()) {
    await filtrarContinente()
    return
  }

  cargando.value = true
  error.value = ''

  try {
    const consulta = `/capitals?q=${encodeURIComponent(texto)}`
    let resultados = await consultarAPI(consulta)

    if (continente.value) {
      resultados = resultados.filter(
        pais => pais.region === continente.value
      )
    }

    paises.value = resultados
  } catch (e) {
    error.value = e.message
  } finally {
    cargando.value = false
  }
}

async function filtrarContinente() {
  cargando.value = true
  error.value = ''

  try {
    const endpoint = continente.value
      ? `/region/${continente.value}`
      : '?limit=100'

    let resultados = await consultarAPI(endpoint)

    if (capital.value.trim()) {
      resultados = resultados.filter(pais =>
        (pais.capitals || []).some(c =>
          c.name?.toLowerCase().includes(capital.value.toLowerCase())
        )
      )
    }

    paises.value = resultados
  } catch (e) {
    error.value = e.message
  } finally {
    cargando.value = false
  }
}

function bandera(pais) {
  const codigo = pais.codes?.alpha_2?.toLowerCase()
  return codigo ? `https://flagcdn.com/w160/${codigo}.png` : ''
}

function formatearPoblacion(valor) {
  if (valor == null) return 'No disponible'
  return new Intl.NumberFormat('es-ES').format(valor)
}

function seleccionarMoneda(moneda) {
  Notify.create({
    message: `Moneda seleccionada: ${moneda.code}`,
    color: 'positive',
    position: 'top'
  })
  // Tu compañero conectará aquí el modal de conversión.
}

onMounted(cargarPaises)
</script>

<style scoped>
.container {
  max-width: 1400px;
  margin: 0 auto;
}

.country-card {
  border-radius: 14px;
  overflow: hidden;
  transition: transform 0.2s, box-shadow 0.2s;
}

.country-card:hover {
  transform: translateY(-4px);
  box-shadow: 0 5px 18px rgba(0, 0, 0, 0.10);
}
</style>
