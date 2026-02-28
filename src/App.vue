<template>
  <div class="app">
    <header>
      <h1>⚡ Elpriser</h1>
      <p class="subtitle">{{ formatDate(today) }} • Växelkurs: {{ exchangeRate }} SEK/EUR</p>
    </header>

    <div v-if="loading" class="loading">Laddar priser...</div>

    <div v-else-if="summary" class="content">

      <!-- Aktuellt pris -->
      <div class="current-price-section">
        <div class="current-price-card">
          <span class="label">Just nu ({{ formatTime(currentHour) }})</span>
          <span class="value" :class="getCurrentPriceClass()">{{ formatPrice(currentPrice) }}</span>
          <span class="unit">SEK/kWh inkl moms</span>
          <span class="comparison" v-if="avgPrice">
            <span v-if="currentPrice < avgPrice * 0.9">🟢 Under snitt</span>
            <span v-else-if="currentPrice > avgPrice * 1.1">🔴 Över snitt</span>
            <span v-else>🟡 Nära snitt</span>
          </span>
        </div>
      </div>

      <!-- Dagsknappar -->
      <div class="day-tabs">
        <button
            :class="{ active: selectedDay === 'dayBeforeYesterday' }"
            @click="selectedDay = 'dayBeforeYesterday'"
            :disabled="!hasDayBeforeYesterdayPrices"
        >
          I förrgår
        </button>
        <button
            :class="{ active: selectedDay === 'yesterday' }"
            @click="selectedDay = 'yesterday'"
            :disabled="!hasYesterdayPrices"
        >
          Igår
        </button>
        <button
            :class="{ active: selectedDay === 'today' }"
            @click="selectedDay = 'today'"
        >
          Idag
        </button>
        <button
            :class="{ active: selectedDay === 'tomorrow' }"
            @click="selectedDay = 'tomorrow'"
            :disabled="!hasTomorrowPrices"
        >
          Imorgon {{ !hasTomorrowPrices ? '(ej publ.)' : '' }}
        </button>
      </div>

      <!-- Valt datum -->
      <p class="selected-date">{{ getSelectedDateLabel() }}</p>

      <div class="summary-cards">
        <div class="card avg">
          <span class="label">Snitt</span>
          <span class="value">{{ formatPrice(currentSummary?.averageSekKwhInclVat) }}</span>
          <span class="unit">SEK/kWh</span>
          <span class="compare" v-if="getPreviousSummary()">
            {{ getCompareText(currentSummary?.averageSekKwhInclVat, getPreviousSummary()?.averageSekKwhInclVat) }}
          </span>
        </div>
        <div class="card low">
          <span class="label">Lägsta</span>
          <span class="value">{{ formatPrice(currentSummary?.minSekKwh * 1.25) }}</span>
          <span class="unit">SEK/kWh</span>
        </div>
        <div class="card high">
          <span class="label">Högsta</span>
          <span class="value">{{ formatPrice(currentSummary?.maxSekKwh * 1.25) }}</span>
          <span class="unit">SEK/kWh</span>
        </div>
      </div>

      <div class="hours-section">
        <div class="hours-card cheap">
          <h3>🟢 Billigaste timmarna</h3>
          <ul>
            <li v-for="hour in currentSummary?.cheapestHours" :key="hour.timestamp">
              <span class="time">{{ formatTime(hour.timestamp) }}</span>
              <span class="price">{{ formatPrice(hour.priceSekKwhInclVat) }} SEK/kWh</span>
            </li>
          </ul>
        </div>
        <div class="hours-card expensive">
          <h3>🔴 Dyraste timmarna</h3>
          <ul>
            <li v-for="hour in currentSummary?.expensiveHours" :key="hour.timestamp">
              <span class="time">{{ formatTime(hour.timestamp) }}</span>
              <span class="price">{{ formatPrice(hour.priceSekKwhInclVat) }} SEK/kWh</span>
            </li>
          </ul>
        </div>
      </div>

      <div class="chart-section">
        <h3>Pris per timme</h3>
        <div class="chart">
          <div
              v-for="(price, index) in currentHourlyPrices"
              :key="index"
              class="bar"
              :style="{ height: getBarHeight(price.priceSekKwhInclVat) + '%' }"
              :class="getBarClass(price.priceSekKwhInclVat, price.timestamp)"
              :title="formatTime(price.timestamp) + ': ' + formatPrice(price.priceSekKwhInclVat) + ' SEK/kWh'"
          >
            <span class="bar-label" v-if="index % 4 === 0">{{ formatHour(price.timestamp) }}</span>
          </div>
        </div>
      </div>

      <!-- Jämförelse över dagar -->
      <div class="comparison-section" v-if="hasMultipleDays">
        <h3>📊 Jämförelse</h3>
        <div class="comparison-table">
          <div class="comp-row header">
            <span class="comp-cell">Dag</span>
            <span class="comp-cell">Snitt</span>
            <span class="comp-cell">Min</span>
            <span class="comp-cell">Max</span>
          </div>
          <div class="comp-row" v-if="dayBeforeYesterdaySummary">
            <span class="comp-cell">I förrgår</span>
            <span class="comp-cell">{{ formatPrice(dayBeforeYesterdaySummary.averageSekKwhInclVat) }}</span>
            <span class="comp-cell low">{{ formatPrice(dayBeforeYesterdaySummary.minSekKwh * 1.25) }}</span>
            <span class="comp-cell high">{{ formatPrice(dayBeforeYesterdaySummary.maxSekKwh * 1.25) }}</span>
          </div>
          <div class="comp-row" v-if="yesterdaySummary">
            <span class="comp-cell">Igår</span>
            <span class="comp-cell">{{ formatPrice(yesterdaySummary.averageSekKwhInclVat) }}</span>
            <span class="comp-cell low">{{ formatPrice(yesterdaySummary.minSekKwh * 1.25) }}</span>
            <span class="comp-cell high">{{ formatPrice(yesterdaySummary.maxSekKwh * 1.25) }}</span>
          </div>
          <div class="comp-row current" v-if="summary">
            <span class="comp-cell">Idag</span>
            <span class="comp-cell">{{ formatPrice(summary.averageSekKwhInclVat) }}</span>
            <span class="comp-cell low">{{ formatPrice(summary.minSekKwh * 1.25) }}</span>
            <span class="comp-cell high">{{ formatPrice(summary.maxSekKwh * 1.25) }}</span>
          </div>
          <div class="comp-row" v-if="tomorrowSummary">
            <span class="comp-cell">Imorgon</span>
            <span class="comp-cell">{{ formatPrice(tomorrowSummary.averageSekKwhInclVat) }}</span>
            <span class="comp-cell low">{{ formatPrice(tomorrowSummary.minSekKwh * 1.25) }}</span>
            <span class="comp-cell high">{{ formatPrice(tomorrowSummary.maxSekKwh * 1.25) }}</span>
          </div>
        </div>
      </div>
    </div>

    <div v-else class="error">Kunde inte ladda priser</div>
  </div>
</template>

<script setup>
import { ref, onMounted, computed } from 'vue'

const API_URL = import.meta.env.VITE_API_URL || ''

const loading = ref(true)
const summary = ref(null)
const tomorrowSummary = ref(null)
const yesterdaySummary = ref(null)
const dayBeforeYesterdaySummary = ref(null)
const prices = ref([])
const tomorrowPrices = ref([])
const yesterdayPrices = ref([])
const dayBeforeYesterdayPrices = ref([])
const exchangeRate = ref(null)
const selectedDay = ref('today')
const today = new Date()

const currentSummary = computed(() => {
  switch (selectedDay.value) {
    case 'tomorrow': return tomorrowSummary.value
    case 'yesterday': return yesterdaySummary.value
    case 'dayBeforeYesterday': return dayBeforeYesterdaySummary.value
    default: return summary.value
  }
})

const currentPrices = computed(() => {
  switch (selectedDay.value) {
    case 'tomorrow': return tomorrowPrices.value
    case 'yesterday': return yesterdayPrices.value
    case 'dayBeforeYesterday': return dayBeforeYesterdayPrices.value
    default: return prices.value
  }
})

const hasTomorrowPrices = computed(() => tomorrowPrices.value?.length > 0)
const hasYesterdayPrices = computed(() => yesterdayPrices.value?.length > 0)
const hasDayBeforeYesterdayPrices = computed(() => dayBeforeYesterdayPrices.value?.length > 0)
const hasMultipleDays = computed(() => yesterdaySummary.value || tomorrowSummary.value)

const currentHour = computed(() => new Date().toISOString())

const currentPrice = computed(() => {
  const now = new Date()
  const price = prices.value.find(p => {
    const priceTime = new Date(p.timestamp)
    return priceTime.getHours() === now.getHours() && priceTime.getDate() === now.getDate()
  })
  return price?.priceSekKwhInclVat || null
})

const avgPrice = computed(() => summary.value?.averageSekKwhInclVat || 0)

const currentHourlyPrices = computed(() => {
  const priceList = currentPrices.value
  const hourly = []
  for (let i = 0; i < priceList.length; i += 4) {
    const slice = priceList.slice(i, i + 4)
    if (slice.length > 0) {
      const avg = slice.reduce((sum, p) => sum + p.priceSekKwhInclVat, 0) / slice.length
      hourly.push({ timestamp: slice[0].timestamp, priceSekKwhInclVat: avg })
    }
  }
  return hourly
})

const maxPrice = computed(() => Math.max(...currentHourlyPrices.value.map(p => p.priceSekKwhInclVat), 0))

function getPreviousSummary() {
  switch (selectedDay.value) {
    case 'tomorrow': return summary.value
    case 'today': return yesterdaySummary.value
    case 'yesterday': return dayBeforeYesterdaySummary.value
    default: return null
  }
}

function getSelectedDateLabel() {
  const d = new Date()
  switch (selectedDay.value) {
    case 'tomorrow':
      d.setDate(d.getDate() + 1)
      break
    case 'yesterday':
      d.setDate(d.getDate() - 1)
      break
    case 'dayBeforeYesterday':
      d.setDate(d.getDate() - 2)
      break
  }
  return d.toLocaleDateString('sv-SE', { weekday: 'long', day: 'numeric', month: 'long' })
}

function getCurrentPriceClass() {
  if (!currentPrice.value || !avgPrice.value) return ''
  if (currentPrice.value < avgPrice.value * 0.9) return 'low'
  if (currentPrice.value > avgPrice.value * 1.1) return 'high'
  return 'normal'
}

function getBarHeight(price) {
  if (maxPrice.value === 0) return 0
  return (price / maxPrice.value) * 100
}

function getBarClass(price, timestamp) {
  const now = new Date()
  const priceTime = new Date(timestamp)
  const isCurrentHour = priceTime.getHours() === now.getHours() &&
      priceTime.getDate() === now.getDate() &&
      selectedDay.value === 'today'

  let cls = ''
  const avg = currentSummary.value?.averageSekKwhInclVat || avgPrice.value
  if (price < avg * 0.8) cls = 'low'
  else if (price > avg * 1.2) cls = 'high'
  else cls = 'normal'

  if (isCurrentHour) cls += ' current'
  return cls
}

function getCompareText(current, previous) {
  if (!current || !previous) return ''
  const diff = ((current - previous) / previous) * 100
  if (diff > 0) return `↑ ${diff.toFixed(1)}%`
  if (diff < 0) return `↓ ${Math.abs(diff).toFixed(1)}%`
  return '→ 0%'
}

function formatPrice(price) {
  return price?.toFixed(2) || '—'
}

function formatDate(date) {
  return date.toLocaleDateString('sv-SE', { weekday: 'long', year: 'numeric', month: 'long', day: 'numeric' })
}

function formatTime(timestamp) {
  return new Date(timestamp).toLocaleTimeString('sv-SE', { hour: '2-digit', minute: '2-digit' })
}

function formatHour(timestamp) {
  return new Date(timestamp).toLocaleTimeString('sv-SE', { hour: '2-digit' })
}

function getDateOffset(offset) {
  const d = new Date()
  d.setDate(d.getDate() + offset)
  return d.toISOString().split('T')[0]
}

onMounted(async () => {
  try {
    const requests = [
      fetch(`${API_URL}/api/analysis/summary/today`),
      fetch(`${API_URL}/api/analysis/today`),
      fetch(`${API_URL}/api/analysis/exchange-rate`),
      fetch(`${API_URL}/api/analysis/summary/${getDateOffset(-1)}`),
      fetch(`${API_URL}/api/analysis/date/${getDateOffset(-1)}`),
      fetch(`${API_URL}/api/analysis/summary/${getDateOffset(-2)}`),
      fetch(`${API_URL}/api/analysis/date/${getDateOffset(-2)}`),
      fetch(`${API_URL}/api/analysis/summary/${getDateOffset(1)}`),
      fetch(`${API_URL}/api/analysis/date/${getDateOffset(1)}`)
    ]

    const [
      summaryRes, pricesRes, rateRes,
      yesterdaySummaryRes, yesterdayPricesRes,
      dayBeforeYesterdaySummaryRes, dayBeforeYesterdayPricesRes,
      tomorrowSummaryRes, tomorrowPricesRes
    ] = await Promise.all(requests)

    summary.value = await summaryRes.json()
    prices.value = await pricesRes.json()
    const rateData = await rateRes.json()
    exchangeRate.value = rateData.rate?.toFixed(2)

    if (yesterdaySummaryRes.ok) {
      const data = await yesterdaySummaryRes.json()
      if (data?.date) yesterdaySummary.value = data
    }
    if (yesterdayPricesRes.ok) {
      const data = await yesterdayPricesRes.json()
      if (data?.length > 0) yesterdayPrices.value = data
    }

    if (dayBeforeYesterdaySummaryRes.ok) {
      const data = await dayBeforeYesterdaySummaryRes.json()
      if (data?.date) dayBeforeYesterdaySummary.value = data
    }
    if (dayBeforeYesterdayPricesRes.ok) {
      const data = await dayBeforeYesterdayPricesRes.json()
      if (data?.length > 0) dayBeforeYesterdayPrices.value = data
    }

    if (tomorrowSummaryRes.ok) {
      const data = await tomorrowSummaryRes.json()
      if (data?.date) tomorrowSummary.value = data
    }
    if (tomorrowPricesRes.ok) {
      const data = await tomorrowPricesRes.json()
      if (data?.length > 0) tomorrowPrices.value = data
    }
  } catch (e) {
    console.error('Failed to fetch data:', e)
  } finally {
    loading.value = false
  }
})
</script>

<style>
* {
  box-sizing: border-box;
  margin: 0;
  padding: 0;
}

body {
  font-family: -apple-system, BlinkMacSystemFont, 'Segoe UI', Roboto, sans-serif;
  background: linear-gradient(135deg, #1a1a2e 0%, #16213e 100%);
  min-height: 100vh;
  color: #fff;
}

.app {
  max-width: 900px;
  margin: 0 auto;
  padding: 2rem 1rem;
}

header {
  text-align: center;
  margin-bottom: 2rem;
}

header h1 {
  font-size: 2rem;
  margin-bottom: 0.5rem;
}

.subtitle {
  color: #888;
  font-size: 0.9rem;
}

.loading, .error {
  text-align: center;
  padding: 3rem;
  color: #888;
}

.current-price-section {
  margin-bottom: 2rem;
}

.current-price-card {
  background: linear-gradient(135deg, #2d2d44 0%, #1f1f35 100%);
  border-radius: 16px;
  padding: 2rem;
  text-align: center;
  border: 2px solid rgba(255,215,0,0.3);
}

.current-price-card .label {
  display: block;
  font-size: 1rem;
  color: #888;
  margin-bottom: 0.5rem;
}

.current-price-card .value {
  display: block;
  font-size: 3.5rem;
  font-weight: bold;
  color: #ffd700;
}

.current-price-card .value.low { color: #4ade80; }
.current-price-card .value.high { color: #f87171; }

.current-price-card .unit {
  display: block;
  font-size: 0.9rem;
  color: #666;
  margin-top: 0.25rem;
}

.current-price-card .comparison {
  display: block;
  margin-top: 1rem;
  font-size: 0.9rem;
}

.day-tabs {
  display: flex;
  gap: 0.5rem;
  margin-bottom: 0.5rem;
}

.day-tabs button {
  flex: 1;
  padding: 0.75rem 0.5rem;
  border: 1px solid rgba(255,255,255,0.2);
  background: rgba(255,255,255,0.05);
  color: #888;
  border-radius: 8px;
  cursor: pointer;
  transition: all 0.2s;
  font-size: 0.85rem;
}

.day-tabs button:hover:not(:disabled) {
  background: rgba(255,255,255,0.1);
}

.day-tabs button.active {
  background: rgba(255,215,0,0.2);
  border-color: #ffd700;
  color: #ffd700;
}

.day-tabs button:disabled {
  opacity: 0.5;
  cursor: not-allowed;
}

.selected-date {
  text-align: center;
  color: #888;
  font-size: 0.9rem;
  margin-bottom: 1.5rem;
  text-transform: capitalize;
}

.summary-cards {
  display: grid;
  grid-template-columns: repeat(3, 1fr);
  gap: 1rem;
  margin-bottom: 2rem;
}

.card {
  background: rgba(255,255,255,0.05);
  border-radius: 12px;
  padding: 1.5rem;
  text-align: center;
  border: 1px solid rgba(255,255,255,0.1);
}

.card .label {
  display: block;
  font-size: 0.8rem;
  color: #888;
  text-transform: uppercase;
  letter-spacing: 1px;
}

.card .value {
  display: block;
  font-size: 2rem;
  font-weight: bold;
  margin: 0.5rem 0;
}

.card .unit {
  font-size: 0.8rem;
  color: #666;
}

.card .compare {
  display: block;
  font-size: 0.75rem;
  color: #888;
  margin-top: 0.5rem;
}

.card.avg .value { color: #ffd700; }
.card.low .value { color: #4ade80; }
.card.high .value { color: #f87171; }

.hours-section {
  display: grid;
  grid-template-columns: repeat(2, 1fr);
  gap: 1rem;
  margin-bottom: 2rem;
}

.hours-card {
  background: rgba(255,255,255,0.05);
  border-radius: 12px;
  padding: 1.5rem;
  border: 1px solid rgba(255,255,255,0.1);
}

.hours-card h3 {
  font-size: 1rem;
  margin-bottom: 1rem;
}

.hours-card ul {
  list-style: none;
}

.hours-card li {
  display: flex;
  justify-content: space-between;
  padding: 0.5rem 0;
  border-bottom: 1px solid rgba(255,255,255,0.05);
}

.hours-card li:last-child {
  border-bottom: none;
}

.hours-card .time {
  color: #888;
}

.hours-card.cheap .price { color: #4ade80; }
.hours-card.expensive .price { color: #f87171; }

.chart-section {
  background: rgba(255,255,255,0.05);
  border-radius: 12px;
  padding: 1.5rem;
  border: 1px solid rgba(255,255,255,0.1);
  margin-bottom: 2rem;
}

.chart-section h3 {
  font-size: 1rem;
  margin-bottom: 1rem;
}

.chart {
  display: flex;
  align-items: flex-end;
  height: 200px;
  gap: 2px;
}

.bar {
  flex: 1;
  background: #ffd700;
  border-radius: 2px 2px 0 0;
  position: relative;
  min-height: 4px;
  transition: opacity 0.2s;
}

.bar:hover {
  opacity: 0.8;
}

.bar.low { background: #4ade80; }
.bar.high { background: #f87171; }
.bar.normal { background: #ffd700; }

.bar.current {
  box-shadow: 0 0 10px rgba(255,255,255,0.5);
  border: 2px solid #fff;
}

.bar-label {
  position: absolute;
  bottom: -20px;
  left: 50%;
  transform: translateX(-50%);
  font-size: 0.7rem;
  color: #666;
}

.comparison-section {
  background: rgba(255,255,255,0.05);
  border-radius: 12px;
  padding: 1.5rem;
  border: 1px solid rgba(255,255,255,0.1);
}

.comparison-section h3 {
  font-size: 1rem;
  margin-bottom: 1rem;
}

.comparison-table {
  width: 100%;
}

.comp-row {
  display: grid;
  grid-template-columns: 1.5fr 1fr 1fr 1fr;
  gap: 0.5rem;
  padding: 0.75rem 0;
  border-bottom: 1px solid rgba(255,255,255,0.05);
}

.comp-row.header {
  font-size: 0.8rem;
  color: #888;
  text-transform: uppercase;
  letter-spacing: 1px;
}

.comp-row.current {
  background: rgba(255,215,0,0.1);
  border-radius: 8px;
  margin: 0 -0.5rem;
  padding-left: 0.5rem;
  padding-right: 0.5rem;
}

.comp-cell {
  text-align: center;
}

.comp-cell:first-child {
  text-align: left;
}

.comp-cell.low { color: #4ade80; }
.comp-cell.high { color: #f87171; }

@media (max-width: 600px) {
  .summary-cards {
    grid-template-columns: 1fr;
  }

  .hours-section {
    grid-template-columns: 1fr;
  }

  .card .value {
    font-size: 1.5rem;
  }

  .current-price-card .value {
    font-size: 2.5rem;
  }

  .day-tabs {
    flex-wrap: wrap;
  }

  .day-tabs button {
    flex: 1 1 45%;
  }
}
</style>