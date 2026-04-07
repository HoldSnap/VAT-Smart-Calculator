<script setup>
import { computed, ref } from 'vue'

const volumeInput = ref('6100')
const vatRateInput = ref('22')
const pricePerUnitInput = ref('2')

const parsedVolume = computed(() => {
  const normalized = volumeInput.value.trim()

  if (!normalized || !/^\d+$/.test(normalized)) {
    return null
  }

  const volume = Number.parseInt(normalized, 10)

  if (Number.isNaN(volume) || volume < 0) {
    return null
  }

  return volume
})

const parsedVatRate = computed(() => parseDecimal(vatRateInput.value))
const parsedPricePerUnit = computed(() => parseDecimal(pricePerUnitInput.value))

const hasValidInputs = computed(
  () => parsedVolume.value !== null && parsedVatRate.value !== null && parsedPricePerUnit.value !== null,
)

const recommendedStep = computed(() => {
  if (!parsedVatRate.value || !parsedPricePerUnit.value) {
    return null
  }

  const rate = parsedVatRate.value
  const price = parsedPricePerUnit.value

  if (rate.value === 0 || price.value === 0) {
    return 1
  }

  const vatNumerator = price.numerator * rate.numerator
  const vatDenominator = price.denominator * (100 * rate.denominator + rate.numerator)

  return vatDenominator / gcd(vatNumerator, vatDenominator)
})

const rawCalculation = computed(() => {
  if (!hasValidInputs.value) {
    return null
  }

  const volume = parsedVolume.value
  const pricePerUnit = parsedPricePerUnit.value.value
  const vatRate = parsedVatRate.value.value
  const totalPrice = pricePerUnit * volume
  const vat = totalPrice * vatRate / (100 + vatRate)
  const basePrice = totalPrice - vat

  return {
    volume,
    totalPrice,
    vat,
    basePrice,
  }
})

const recommendedCalculation = computed(() => {
  if (!hasValidInputs.value || !recommendedStep.value) {
    return null
  }

  const volume = parsedVolume.value
  const pricePerUnit = parsedPricePerUnit.value.value
  const vatRate = parsedVatRate.value.value
  const step = recommendedStep.value
  const recommendedVolume = Math.ceil(volume / step) * step
  const totalPrice = pricePerUnit * recommendedVolume
  const vat = totalPrice * vatRate / (100 + vatRate)
  const basePrice = totalPrice - vat

  return {
    recommendedVolume,
    totalPrice,
    vat,
    basePrice,
    delta: recommendedVolume - volume,
  }
})

const recommendationText = computed(() => {
  if (parsedVolume.value === null) {
    return 'Введите целый объём в см³, чтобы увидеть расчёт.'
  }

  if (!parsedVatRate.value) {
    return 'Укажите корректную ставку НДС, например 22.'
  }

  if (!parsedPricePerUnit.value) {
    return 'Укажите корректную цену за 1 см³, например 2 или 2.5.'
  }

  if (parsedVatRate.value.value === 0 || parsedPricePerUnit.value.value === 0) {
    return 'При текущих параметрах НДС равен нулю, поэтому любой объём уже подходит.'
  }

  if (!recommendedCalculation.value || !recommendedStep.value) {
    return 'Недостаточно данных для рекомендации.'
  }

  if (recommendedCalculation.value.delta === 0) {
    return `Объём уже оптимален: целый НДС получается при шаге ${formatInteger(recommendedStep.value)} см³.`
  }

  return `Увеличьте объём на ${formatInteger(recommendedCalculation.value.delta)} см³. Для текущих параметров целый НДС получается на шагах по ${formatInteger(recommendedStep.value)} см³.`
})

function parseDecimal(value) {
  const normalized = value.trim().replace(',', '.')

  if (!normalized || !/^\d+(\.\d+)?$/.test(normalized)) {
    return null
  }

  const [integerPart, fractionPart = ''] = normalized.split('.')
  const denominator = 10 ** fractionPart.length
  const numerator = Number.parseInt(`${integerPart}${fractionPart}`, 10)

  if (Number.isNaN(numerator)) {
    return null
  }

  const divisor = gcd(numerator, denominator)

  return {
    value: Number(normalized),
    numerator: numerator / divisor,
    denominator: denominator / divisor,
  }
}

function gcd(a, b) {
  let x = Math.abs(a)
  let y = Math.abs(b)

  while (y !== 0) {
    const remainder = x % y
    x = y
    y = remainder
  }

  return x || 1
}

function formatInteger(value) {
  return new Intl.NumberFormat('ru-RU', {
    maximumFractionDigits: 0,
  }).format(value)
}

function formatNumber(value, minimumFractionDigits = 0, maximumFractionDigits = 2) {
  return new Intl.NumberFormat('ru-RU', {
    minimumFractionDigits,
    maximumFractionDigits,
  }).format(value)
}
</script>

<template>
  <main class="page-shell">
    <section class="hero">
      <div class="hero__copy">
        <p class="eyebrow">VAT Smart Calculator</p>
        <h1>Калькулятор объёма с гибкими настройками НДС и цены</h1>
        <p class="hero__lead">
          Меняйте объём, ставку НДС и цену за единицу объёма. Интерфейс сразу сравнит обычный
          расчёт и оптимальный вариант, при котором НДС получается без копеек.
        </p>

        <div class="hero__badges">
          <span>
            Шаг рекомендации:
            {{ recommendedStep ? `${formatInteger(recommendedStep)} см³` : 'заполните параметры' }}
          </span>
          <span>
            НДС: {{ parsedVatRate ? `${formatNumber(parsedVatRate.value, 0, 2)}%` : 'не задан' }}
          </span>
          <span>
            Цена:
            {{
              parsedPricePerUnit
                ? `${formatNumber(parsedPricePerUnit.value, 0, 4)} руб/см³`
                : 'не задана'
            }}
          </span>
        </div>
      </div>

      <div class="calculator-card">
        <div class="config-grid">
          <div class="config-field config-field--wide">
            <label class="field-label" for="volume">Объём изделия</label>
            <div class="input-wrap">
              <input
                id="volume"
                v-model="volumeInput"
                class="volume-input"
                inputmode="numeric"
                type="text"
                placeholder="Например, 6100"
              />
              <span class="input-unit">см³</span>
            </div>
          </div>

          <div class="config-field">
            <label class="field-label" for="vat-rate">Ставка НДС</label>
            <div class="input-wrap input-wrap--compact">
              <input
                id="vat-rate"
                v-model="vatRateInput"
                class="config-input"
                inputmode="decimal"
                type="text"
                placeholder="22"
              />
              <span class="input-unit">%</span>
            </div>
          </div>

          <div class="config-field">
            <label class="field-label" for="price-per-unit">Цена за 1 см³</label>
            <div class="input-wrap input-wrap--compact">
              <input
                id="price-per-unit"
                v-model="pricePerUnitInput"
                class="config-input"
                inputmode="decimal"
                type="text"
                placeholder="2"
              />
              <span class="input-unit">руб</span>
            </div>
          </div>
        </div>

        <p class="field-hint">
          Поддерживаются целые и десятичные значения для НДС и цены. Можно вводить точку или
          запятую.
        </p>

        <div class="insight" :class="{ 'insight--warning': !hasValidInputs }">
          <p class="insight__label">Рекомендация</p>
          <p class="insight__text">{{ recommendationText }}</p>
        </div>
      </div>
    </section>

    <section class="results-grid">
      <article class="result-card">
        <div class="card-head">
          <p class="card-kicker">Исходный расчёт</p>
          <h2>Как сейчас</h2>
        </div>

        <template v-if="rawCalculation">
          <dl class="metrics">
            <div class="metric">
              <dt>Объём</dt>
              <dd>{{ formatInteger(rawCalculation.volume) }} см³</dd>
            </div>
            <div class="metric">
              <dt>Цена с НДС</dt>
              <dd>{{ formatNumber(rawCalculation.totalPrice) }} руб</dd>
            </div>
            <div class="metric">
              <dt>Цена без НДС</dt>
              <dd>{{ formatNumber(rawCalculation.basePrice) }} руб</dd>
            </div>
            <div class="metric">
              <dt>НДС</dt>
              <dd>{{ formatNumber(rawCalculation.vat) }} руб</dd>
            </div>
          </dl>
        </template>

        <p v-else class="empty-state">
          Нужны корректные значения объёма, ставки НДС и цены за 1 см³.
        </p>
      </article>

      <article class="result-card result-card--featured">
        <div class="card-head">
          <p class="card-kicker">Рекомендуемый расчёт</p>
          <h2>Без копеек в НДС</h2>
        </div>

        <template v-if="recommendedCalculation">
          <dl class="metrics">
            <div class="metric">
              <dt>Рекомендованный объём</dt>
              <dd>{{ formatInteger(recommendedCalculation.recommendedVolume) }} см³</dd>
            </div>
            <div class="metric">
              <dt>Цена с НДС</dt>
              <dd>{{ formatNumber(recommendedCalculation.totalPrice) }} руб</dd>
            </div>
            <div class="metric">
              <dt>Цена без НДС</dt>
              <dd>{{ formatNumber(recommendedCalculation.basePrice) }} руб</dd>
            </div>
            <div class="metric">
              <dt>НДС</dt>
              <dd>{{ formatNumber(recommendedCalculation.vat) }} руб</dd>
            </div>
          </dl>

          <div class="delta-chip" :class="{ 'delta-chip--ok': recommendedCalculation.delta === 0 }">
            <span v-if="recommendedCalculation.delta === 0">Корректировка не нужна</span>
            <span v-else>
              +{{ formatInteger(recommendedCalculation.delta) }} см³ к текущему объёму
            </span>
          </div>
        </template>

        <p v-else class="empty-state">
          После ввода параметров здесь появится объём, при котором НДС выходит без копеек.
        </p>
      </article>
    </section>

    <section class="info-panel">
      <div>
        <p class="card-kicker">Как это работает</p>
        <h2>Формулы остаются прозрачными, а результат видно сразу</h2>
      </div>

      <div class="info-grid">
        <article>
          <h3>1. Общая сумма</h3>
          <p>
            Цена с НДС считается по формуле <code>объём × цена за 1 см³</code>, поэтому можно
            быстро тестировать разные тарифы.
          </p>
        </article>

        <article>
          <h3>2. Выделение НДС</h3>
          <p>
            НДС выделяется из общей суммы через формулу
            <code>сумма × ставка / (100 + ставка)</code>.
          </p>
        </article>

        <article>
          <h3>3. Подбор объёма</h3>
          <p>
            Калькулятор ищет ближайший подходящий объём вверх. Текущий шаг рекомендации:
            <code>{{ recommendedStep ? `${formatInteger(recommendedStep)} см³` : 'не рассчитан' }}</code>.
          </p>
        </article>
      </div>
    </section>
  </main>
</template>

<style scoped>
:global(*) {
  box-sizing: border-box;
}

:global(body) {
  margin: 0;
  min-width: 320px;
  font-family: 'Avenir Next', 'Segoe UI', sans-serif;
  background:
    radial-gradient(circle at top left, rgba(244, 199, 120, 0.3), transparent 30%),
    radial-gradient(circle at 85% 10%, rgba(66, 102, 255, 0.22), transparent 24%),
    linear-gradient(180deg, #fff7ea 0%, #f4f7fb 52%, #e9eef7 100%);
  color: #182033;
}

:global(#app) {
  min-height: 100vh;
}

code {
  padding: 0.15rem 0.45rem;
  border-radius: 999px;
  background: rgba(24, 32, 51, 0.08);
  font-size: 0.92em;
}

.page-shell {
  width: min(1180px, calc(100vw - 32px));
  margin: 0 auto;
  padding: 40px 0 56px;
}

.hero {
  display: grid;
  grid-template-columns: minmax(0, 1.3fr) minmax(320px, 440px);
  gap: 28px;
  align-items: stretch;
}

.hero__copy,
.calculator-card,
.result-card,
.info-panel {
  position: relative;
  overflow: hidden;
  border: 1px solid rgba(255, 255, 255, 0.58);
  border-radius: 28px;
  background: rgba(255, 255, 255, 0.74);
  box-shadow: 0 22px 60px rgba(36, 54, 92, 0.12);
  backdrop-filter: blur(18px);
}

.hero__copy {
  padding: 34px;
}

.hero__copy::after,
.calculator-card::after,
.result-card::after,
.info-panel::after {
  content: '';
  position: absolute;
  inset: auto -10% -45% auto;
  width: 180px;
  height: 180px;
  border-radius: 50%;
  background: radial-gradient(circle, rgba(255, 191, 87, 0.22), transparent 68%);
  pointer-events: none;
}

.eyebrow,
.card-kicker,
.insight__label {
  margin: 0 0 10px;
  font-size: 0.78rem;
  font-weight: 700;
  letter-spacing: 0.14em;
  text-transform: uppercase;
  color: #8c5b17;
}

h1,
h2,
h3 {
  margin: 0;
  font-family: Georgia, 'Times New Roman', serif;
  font-weight: 700;
  letter-spacing: -0.03em;
}

h1 {
  max-width: 11ch;
  font-size: clamp(2.7rem, 5vw, 5rem);
  line-height: 0.98;
}

.hero__lead {
  max-width: 62ch;
  margin: 22px 0 0;
  font-size: 1.08rem;
  line-height: 1.75;
  color: #4a556d;
}

.hero__badges {
  display: flex;
  flex-wrap: wrap;
  gap: 12px;
  margin-top: 28px;
}

.hero__badges span,
.delta-chip {
  display: inline-flex;
  align-items: center;
  justify-content: center;
  min-height: 42px;
  padding: 0 16px;
  border: 1px solid rgba(24, 32, 51, 0.08);
  border-radius: 999px;
  background: rgba(255, 255, 255, 0.78);
  color: #22304f;
  font-size: 0.95rem;
  font-weight: 600;
}

.calculator-card {
  padding: 28px;
}

.config-grid {
  display: grid;
  grid-template-columns: repeat(2, minmax(0, 1fr));
  gap: 16px;
}

.config-field--wide {
  grid-column: 1 / -1;
}

.field-label {
  display: block;
  margin-bottom: 12px;
  font-size: 0.95rem;
  font-weight: 700;
  color: #23314d;
}

.input-wrap {
  display: flex;
  align-items: center;
  gap: 12px;
  padding: 12px 16px;
  border: 1px solid rgba(41, 60, 101, 0.12);
  border-radius: 22px;
  background: rgba(247, 249, 253, 0.9);
}

.input-wrap--compact {
  min-height: 68px;
}

.volume-input,
.config-input {
  width: 100%;
  border: 0;
  outline: none;
  background: transparent;
  color: #182033;
  letter-spacing: -0.04em;
}

.volume-input {
  font-size: 2rem;
  font-weight: 700;
}

.config-input {
  font-size: 1.35rem;
  font-weight: 600;
}

.volume-input::placeholder,
.config-input::placeholder {
  color: #9aa4ba;
}

.input-unit {
  flex-shrink: 0;
  padding: 8px 12px;
  border-radius: 999px;
  background: linear-gradient(135deg, #ffe1a8 0%, #ffd089 100%);
  color: #6c4610;
  font-size: 0.92rem;
  font-weight: 700;
}

.field-hint {
  margin: 12px 0 0;
  color: #64708a;
  line-height: 1.55;
}

.insight {
  margin-top: 24px;
  padding: 18px;
  border-radius: 22px;
  background: linear-gradient(145deg, #1e335e 0%, #3358a3 100%);
  color: #f7f9ff;
}

.insight--warning {
  background: linear-gradient(145deg, #8e5417 0%, #b8741d 100%);
}

.insight__label {
  color: rgba(255, 244, 220, 0.84);
}

.insight__text {
  margin: 0;
  font-size: 1.04rem;
  line-height: 1.6;
}

.results-grid {
  display: grid;
  grid-template-columns: repeat(2, minmax(0, 1fr));
  gap: 24px;
  margin-top: 24px;
}

.result-card {
  padding: 28px;
}

.result-card--featured {
  background:
    linear-gradient(180deg, rgba(255, 255, 255, 0.88), rgba(255, 250, 239, 0.9)),
    rgba(255, 255, 255, 0.8);
}

.card-head {
  display: flex;
  flex-direction: column;
  gap: 6px;
  margin-bottom: 22px;
}

.card-head h2 {
  font-size: clamp(1.8rem, 3vw, 2.5rem);
}

.metrics {
  display: grid;
  gap: 14px;
  margin: 0;
}

.metric {
  display: flex;
  align-items: end;
  justify-content: space-between;
  gap: 16px;
  padding: 16px 0;
  border-bottom: 1px solid rgba(24, 32, 51, 0.08);
}

.metric:last-child {
  border-bottom: 0;
}

.metric dt {
  color: #66738c;
  font-size: 0.97rem;
}

.metric dd {
  margin: 0;
  text-align: right;
  font-size: 1.12rem;
  font-weight: 700;
  color: #1c2740;
}

.delta-chip {
  margin-top: 22px;
  width: fit-content;
  background: linear-gradient(135deg, #ffe8bf 0%, #ffd890 100%);
  color: #744917;
}

.delta-chip--ok {
  background: linear-gradient(135deg, #dff8e6 0%, #bdecc9 100%);
  color: #1f6b37;
}

.empty-state {
  margin: 0;
  color: #6d7890;
  line-height: 1.7;
}

.info-panel {
  margin-top: 24px;
  padding: 30px;
}

.info-panel h2 {
  margin-top: 4px;
  font-size: clamp(1.8rem, 3vw, 2.8rem);
}

.info-grid {
  display: grid;
  grid-template-columns: repeat(3, minmax(0, 1fr));
  gap: 18px;
  margin-top: 24px;
}

.info-grid article {
  padding: 20px;
  border-radius: 24px;
  background: rgba(248, 250, 255, 0.88);
  border: 1px solid rgba(41, 60, 101, 0.08);
}

.info-grid h3 {
  font-size: 1.3rem;
}

.info-grid p {
  margin: 12px 0 0;
  color: #59657c;
  line-height: 1.7;
}

@media (max-width: 980px) {
  .hero,
  .results-grid,
  .info-grid {
    grid-template-columns: 1fr;
  }

  h1 {
    max-width: none;
  }
}

@media (max-width: 640px) {
  .page-shell {
    width: min(100vw - 20px, 100%);
    padding: 20px 0 28px;
  }

  .hero__copy,
  .calculator-card,
  .result-card,
  .info-panel {
    border-radius: 24px;
  }

  .hero__copy,
  .calculator-card,
  .result-card,
  .info-panel,
  .info-grid article {
    padding: 22px;
  }

  .config-grid {
    grid-template-columns: 1fr;
  }

  .config-field--wide {
    grid-column: auto;
  }

  .volume-input {
    font-size: 1.6rem;
  }

  .metric {
    flex-direction: column;
    align-items: start;
  }

  .metric dd {
    text-align: left;
  }
}
</style>
