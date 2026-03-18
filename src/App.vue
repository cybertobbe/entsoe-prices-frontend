<template>
  <div class="app">
    <header>
      <h1>⚡ Elpriser</h1>
      <p class="subtitle">{{ formatDate(today) }} • Växelkurs: {{ exchangeRate }} SEK/EUR</p>
    </header>

    <div v-if="loading" class="loading">Laddar priser...</div>

    <div v-else-if="summary" class="content">

      <!-- Moms toggle -->
      <div class="vat-toggle">
        <button :class="{ active: !includeVat }" @click="includeVat = false">Exkl moms</button>
        <button :class="{ active: includeVat }" @click="includeVat = true">Inkl moms (25%)</button>
      </div>

      <!-- Aktuellt pris -->
      <div class="current-price-section">
        <div class="current-price-card">
          <span class="label">Just nu ({{ currentHour }}:00)</span>
          <span class="value" :class="getCurrentPriceClass()">{{ formatPrice(displayCurrentPrice) }}</span>
          <span class="unit">SEK/kWh {{ includeVat ? 'inkl moms' : 'exkl moms' }}</span>
          <div class="comparison" v-if="displayCurrentPrice && displayAvgPrice">
            <span v-if="displayCurrentPrice < displayAvgPrice * 0.9" class="under">
              🟢 {{ formatPercent(displayCurrentPrice, displayAvgPrice) }} under snittet
            </span>
            <span v-else-if="displayCurrentPrice > displayAvgPrice * 1.1" class="over">
              🔴 {{ formatPercent(displayCurrentPrice, displayAvgPrice) }} över snittet
            </span>
            <span v-else class="normal">
              🟡 Nära snittet
            </span>
          </div>
          <div class="forecast-comparison" v-if="displayCurrentPrice && displayForecastPrice">
            <span v-if="displayCurrentPrice < displayForecastPrice * 0.95" class="under">
              📉 {{ formatDiff(displayForecastPrice - displayCurrentPrice) }} lägre än igår
            </span>
            <span v-else-if="displayCurrentPrice > displayForecastPrice * 1.05" class="over">
              📈 {{ formatDiff(displayCurrentPrice - displayForecastPrice) }} högre än igår
            </span>
            <span v-else class="normal">
              📊 Samma som igår
            </span>
          </div>
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
          Imorgon {{ !hasTomorrowPrices ? '(publ kl 13.15)' : '' }}
        </button>
      </div>

      <p class="selected-date">{{ getSelectedDateLabel() }}</p>

      <div class="summary-cards">
        <div class="card avg">
          <span class="label">Snitt</span>
          <span class="value">{{ formatPrice(displaySummaryAvg) }}</span>
          <span class="unit">SEK/kWh</span>
          <span class="compare" v-if="getPreviousSummary()">
            {{ getCompareText(displaySummaryAvg, getDisplayPreviousAvg()) }}
          </span>
        </div>
        <div class="card low">
          <span class="label">Lägsta</span>
          <span class="value">{{ formatPrice(displaySummaryMin) }}</span>
          <span class="unit">SEK/kWh</span>
        </div>
        <div class="card high">
          <span class="label">Högsta</span>
          <span class="value">{{ formatPrice(displaySummaryMax) }}</span>
          <span class="unit">SEK/kWh</span>
        </div>
      </div>

      <div class="hours-section">
        <div class="hours-card cheap">
          <h3>🟢 Billigaste timmarna</h3>
          <ul>
            <li v-for="hour in currentSummary?.cheapestHours" :key="hour.timestamp">
              <span class="time">{{ formatTime(hour.timestamp) }}</span>
              <span class="price">{{ formatPrice(getDisplayPrice(hour)) }} SEK/kWh</span>
            </li>
          </ul>
        </div>
        <div class="hours-card expensive">
          <h3>🔴 Dyraste timmarna</h3>
          <ul>
            <li v-for="hour in currentSummary?.expensiveHours" :key="hour.timestamp">
              <span class="time">{{ formatTime(hour.timestamp) }}</span>
              <span class="price">{{ formatPrice(getDisplayPrice(hour)) }} SEK/kWh</span>
            </li>
          </ul>
        </div>
      </div>

      <div class="chart-section">
        <h3>Pris per timme (00-23)</h3>
        <div class="chart">
          <div
              v-for="(price, index) in currentHourlyPrices"
              :key="index"
              class="bar"
              :style="{ height: getBarHeight(getDisplayHourlyPrice(price)) + '%' }"
              :class="getBarClass(getDisplayHourlyPrice(price), price.hour)"
              :title="price.hour + ':00 - ' + formatPrice(getDisplayHourlyPrice(price)) + ' SEK/kWh'"
          >
            <span class="bar-label">{{ price.hour }}</span>
          </div>
        </div>
      </div>

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
            <span class="comp-cell">{{ formatPrice(getDisplayAvg(dayBeforeYesterdaySummary)) }}</span>
            <span class="comp-cell low">{{ formatPrice(getDisplayMin(dayBeforeYesterdaySummary)) }}</span>
            <span class="comp-cell high">{{ formatPrice(getDisplayMax(dayBeforeYesterdaySummary)) }}</span>
          </div>
          <div class="comp-row" v-if="yesterdaySummary">
            <span class="comp-cell">Igår</span>
            <span class="comp-cell">{{ formatPrice(getDisplayAvg(yesterdaySummary)) }}</span>
            <span class="comp-cell low">{{ formatPrice(getDisplayMin(yesterdaySummary)) }}</span>
            <span class="comp-cell high">{{ formatPrice(getDisplayMax(yesterdaySummary)) }}</span>
          </div>
          <div class="comp-row current" v-if="summary">
            <span class="comp-cell">Idag</span>
            <span class="comp-cell">{{ formatPrice(getDisplayAvg(summary)) }}</span>
            <span class="comp-cell low">{{ formatPrice(getDisplayMin(summary)) }}</span>
            <span class="comp-cell high">{{ formatPrice(getDisplayMax(summary)) }}</span>
          </div>
          <div class="comp-row" v-if="tomorrowSummary">
            <span class="comp-cell">Imorgon</span>
            <span class="comp-cell">{{ formatPrice(getDisplayAvg(tomorrowSummary)) }}</span>
            <span class="comp-cell low">{{ formatPrice(getDisplayMin(tomorrowSummary)) }}</span>
            <span class="comp-cell high">{{ formatPrice(getDisplayMax(tomorrowSummary)) }}</span>
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
const includeVat = ref(false)
const today = new Date()

const VAT_MULTIPLIER = 1.25

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

const currentHour = computed(() => {
  return new Date().getHours().toString().padStart(2, '0')
})

const currentPrice = computed(() => {
  const now = new Date()
  const currentH = now.getHours()

  // Hitta priset för aktuell timme
  const price = prices.value.find(p => {
    const priceHour = parseInt(p.timestamp.substring(11, 13))
    return priceHour === currentH
  })
  return price || null
})

const displayCurrentPrice = computed(() => {
  if (!currentPrice.value) return null
  return includeVat.value ? currentPrice.value.priceSekKwhInclVat : currentPrice.value.priceSekKwh
})

const avgPrice = computed(() => summary.value?.averageSekKwh || 0)

const displayAvgPrice = computed(() => {
  return includeVat.value ? avgPrice.value * VAT_MULTIPLIER : avgPrice.value
})

const forecastPrice = computed(() => {
  if (!yesterdayPrices.value?.length) return null
  const currentH = new Date().getHours()
  const price = yesterdayPrices.value.find(p => {
    const priceHour = parseInt(p.timestamp.substring(11, 13))
    return priceHour === currentH
  })
  return price || null
})

const displayForecastPrice = computed(() => {
  if (!forecastPrice.value) return null
  return includeVat.value ? forecastPrice.value.priceSekKwhInclVat : forecastPrice.value.priceSekKwh
})

const displaySummaryAvg = computed(() => {
  if (!currentSummary.value) return null
  return includeVat.value ? currentSummary.value.averageSekKwhInclVat : currentSummary.value.averageSekKwh
})

const displaySummaryMin = computed(() => {
  if (!currentSummary.value) return null
  const min = currentSummary.value.minSekKwh
  return includeVat.value ? min * VAT_MULTIPLIER : min
})

const displaySummaryMax = computed(() => {
  if (!currentSummary.value) return null
  const max = currentSummary.value.maxSekKwh
  return includeVat.value ? max * VAT_MULTIPLIER : max
})

const currentHourlyPrices = computed(() => {
  const priceList = currentPrices.value
  const hourlyMap = new Map()

  // Gruppera per timme
  priceList.forEach(p => {
    const hour = parseInt(p.timestamp.substring(11, 13))
    if (!hourlyMap.has(hour)) {
      hourlyMap.set(hour, [])
    }
    hourlyMap.get(hour).push(p)
  })

  // Beräkna medelvärde per timme
  const hourly = []
  for (let h = 0; h < 24; h++) {
    const hourPrices = hourlyMap.get(h)
    if (hourPrices && hourPrices.length > 0) {
      const avgSek = hourPrices.reduce((sum, p) => sum + p.priceSekKwh, 0) / hourPrices.length
      const avgSekVat = hourPrices.reduce((sum, p) => sum + p.priceSekKwhInclVat, 0) / hourPrices.length
      hourly.push({
        hour: h.toString().padStart(2, '0'),
        priceSekKwh: avgSek,
        priceSekKwhInclVat: avgSekVat
      })
    }
  }
  return hourly
})

const maxPrice = computed(() => {
  const priceValues = currentHourlyPrices.value.map(p => getDisplayHourlyPrice(p))
  return Math.max(...priceValues, 0)
})

function getDisplayPrice(priceObj) {
  return includeVat.value ? priceObj.priceSekKwhInclVat : priceObj.priceSekKwh
}

function getDisplayHourlyPrice(priceObj) {
  return includeVat.value ? priceObj.priceSekKwhInclVat : priceObj.priceSekKwh
}

function getDisplayAvg(summaryObj) {
  if (!summaryObj) return null
  return includeVat.value ? summaryObj.averageSekKwhInclVat : summaryObj.averageSekKwh
}

function getDisplayMin(summaryObj) {
  if (!summaryObj) return null
  return includeVat.value ? summaryObj.minSekKwh * VAT_MULTIPLIER : summaryObj.minSekKwh
}

function getDisplayMax(summaryObj) {
  if (!summaryObj) return null
  return includeVat.value ? summaryObj.maxSekKwh * VAT_MULTIPLIER : summaryObj.maxSekKwh
}

function getDisplayPreviousAvg() {
  const prev = getPreviousSummary()
  return prev ? getDisplayAvg(prev) : null
}

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
  if (!displayCurrentPrice.value || !displayAvgPrice.value) return ''
  if (displayCurrentPrice.value < displayAvgPrice.value * 0.9) return 'low'
  if (displayCurrentPrice.value > displayAvgPrice.value * 1.1) return 'high'
  return 'normal'
}

function getBarHeight(price) {
  if (maxPrice.value === 0) return 0
  return (price / maxPrice.value) * 100
}

function getBarClass(price, hour) {
  const currentH = new Date().getHours().toString().padStart(2, '0')
  const isCurrentHour = hour === currentH && selectedDay.value === 'today'

  let cls = ''
  const avg = displaySummaryAvg.value || displayAvgPrice.value
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

function formatPercent(current, avg) {
  const diff = Math.abs(((current - avg) / avg) * 100)
  return `${diff.toFixed(0)}%`
}

function formatDiff(diff) {
  return `${(diff * 100).toFixed(1)} öre`
}

function formatPrice(price) {
  return price?.toFixed(2) || '—'
}

function formatDate(date) {
  return date.toLocaleDateString('sv-SE', { weekday: 'long', year: 'numeric', month: 'long', day: 'numeric' })
}

function formatTime(timestamp) {
  return timestamp.substring(11, 16)
}

function getDateOffset(offset) {
  const d = new Date()
  d.setDate(d.getDate() + offset)
  return d.toISOString().split('T')[0]
}

const safeJson = async (res) => {
  if (!res.ok) return null
  const text = await res.text()
  if (!text) return null
  try {
    return JSON.parse(text)
  } catch {
    return null
  }
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

    const responses = await Promise.all(requests)

    summary.value = await safeJson(responses[0])
    prices.value = await safeJson(responses[1]) || []
    const rateData = await safeJson(responses[2])
    exchangeRate.value = rateData?.rate?.toFixed(2)

    const yesterdaySum = await safeJson(responses[3])
    if (yesterdaySum?.date) yesterdaySummary.value = yesterdaySum

    const yesterdayPrc = await safeJson(responses[4])
    if (yesterdayPrc?.length > 0) yesterdayPrices.value = yesterdayPrc

    const dayBeforeSum = await safeJson(responses[5])
    if (dayBeforeSum?.date) dayBeforeYesterdaySummary.value = dayBeforeSum

    const dayBeforePrc = await safeJson(responses[6])
    if (dayBeforePrc?.length > 0) dayBeforeYesterdayPrices.value = dayBeforePrc

    const tomorrowSum = await safeJson(responses[7])
    if (tomorrowSum?.date) tomorrowSummary.value = tomorrowSum

    const tomorrowPrc = await safeJson(responses[8])
    if (tomorrowPrc?.length > 0) tomorrowPrices.value = tomorrowPrc

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
  margin-bottom: 1.5rem;
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

.vat-toggle {
  display: flex;
  justify-content: center;
  gap: 0.5rem;
  margin-bottom: 1.5rem;
}

.vat-toggle button {
  padding: 0.5rem 1rem;
  border: 1px solid rgba(255,255,255,0.2);
  background: rgba(255,255,255,0.05);
  color: #888;
  border-radius: 20px;
  cursor: pointer;
  transition: all 0.2s;
  font-size: 0.85rem;
}

.vat-toggle button:hover {
  background: rgba(255,255,255,0.1);
}

.vat-toggle button.active {
  background: rgba(255,215,0,0.2);
  border-color: #ffd700;
  color: #ffd700;
}

.current-price-section {
  margin-bottom: 1.5rem;
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

.current-price-card .comparison,
.current-price-card .forecast-comparison {
  display: block;
  margin-top: 0.75rem;
  font-size: 0.9rem;
}

.current-price-card .under { color: #4ade80; }
.current-price-card .over { color: #f87171; }
.current-price-card .normal { color: #ffd700; }

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
  gap: 3px;
  padding-bottom: 25px;
}

.bar {
  flex: 1;
  background: #ffd700;
  border-radius: 3px 3px 0 0;
  position: relative;
  min-height: 4px;
  transition: all 0.2s;
  cursor: pointer;
}

.bar:hover {
  opacity: 0.8;
  transform: scaleY(1.02);
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
  bottom: -22px;
  left: 50%;
  transform: translateX(-50%);
  font-size: 0.6rem;
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

  .bar-label {
    font-size: 0.5rem;
  }
}
</style>