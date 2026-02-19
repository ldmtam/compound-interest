<template>
  <div class="app">
    <!-- Header -->
    <div class="header">
      <div class="header-icon">📈</div>
      <div>
        <h1>So sánh lãi kép theo kỳ đầu tư</h1>
        <p>Cùng số tiền — khác kỳ hạn — chênh lệch bao nhiêu?</p>
      </div>
    </div>

    <div class="main">
      <!-- Sidebar -->
      <div class="sidebar">
        <p class="compare-note">
          Nhập <strong>tổng số tiền đầu tư mỗi năm</strong>. Tool sẽ chia đều
          theo từng kỳ và so sánh kết quả giữa
          <strong>tuần · tháng · quý · năm</strong>.
        </p>

        <SliderInput
          label="Số tiền khởi điểm ban đầu"
          v-model="principal"
          :min="0"
          :max="1_000_000_000"
          :step="10_000_000"
          unit="VND"
          :format="fmtNumber"
        />
        <SliderInput
          label="Tổng đóng góp mỗi năm"
          v-model="yearlyAmount"
          :min="0"
          :max="500_000_000"
          :step="1_000_000"
          unit="VND"
          :format="fmtNumber"
        />
        <SliderInput
          label="Thời gian đầu tư"
          v-model="time"
          :min="1"
          :max="50"
          :step="1"
          unit="năm"
          :format="(v) => String(v)"
        />
        <SliderInput
          label="Lãi suất kỳ vọng (%/năm)"
          v-model="rate"
          :min="1"
          :max="100"
          :step="0.5"
          unit="%"
          :format="(v) => String(v)"
        />

        <!-- Breakdown -->
        <div class="breakdown-box">
          <div class="breakdown-title">📌 Phân bổ đóng góp</div>
          <div
            v-for="f in FREQS"
            :key="f.key"
            class="breakdown-row"
          >
            <span class="bd-icon-label">{{ f.icon }} {{ f.shortLabel }}</span>
            <span
              class="bd-val"
              :style="{ color: f.color }"
            >
              {{ fmtNumber(Math.round(yearlyAmount / f.periodsPerYear)) }} VND
            </span>
          </div>
        </div>

        <button
          class="calc-btn"
          @click="runCompare"
        >
          ⚖️ So sánh ngay
        </button>
      </div>

      <!-- Content -->
      <div class="content">
        <template v-if="results">
          <!-- Winner banner -->
          <div
            class="winner-banner"
            :style="{
              borderColor: results.winner.color,
              background: results.winner.color + '12',
            }"
          >
            <span class="winner-icon">🏆</span>
            <div>
              <div class="winner-title">Kỳ đầu tư hiệu quả nhất</div>
              <div
                class="winner-name"
                :style="{ color: results.winner.color }"
              >
                {{ results.winner.icon }} {{ results.winner.label }}
              </div>
            </div>
            <div class="winner-gain">
              +{{ fmtNumber(Math.round(results.gain)) }} VND
              <span class="winner-vs">so với kỳ kém nhất</span>
            </div>
          </div>

          <!-- Compare cards -->
          <div class="compare-cards">
            <div
              v-for="item in results.items"
              :key="item.key"
              :class="['cmp-card', { best: item.isBest, worst: item.isWorst }]"
              :style="
                item.isBest
                  ? { borderColor: item.color, background: item.color + '08' }
                  : {}
              "
            >
              <div
                class="cmp-badge"
                :style="{ color: item.color }"
                v-if="item.isBest"
              >
                👑 Tốt nhất
              </div>
              <div
                class="cmp-badge worst-badge"
                v-else-if="item.isWorst"
              >
                📉 Kém nhất
              </div>
              <div
                class="cmp-rank"
                v-else
              >
                {{ item.rank }}
              </div>

              <div class="cmp-label">{{ item.icon }} {{ item.label }}</div>
              <div class="cmp-contribution">
                {{ fmtNumber(Math.round(item.perPeriod)) }} VND / kỳ
              </div>

              <div class="cmp-divider"></div>

              <div class="cmp-row">
                <span>Tổng đầu tư</span>
                <span class="cmp-val neutral">{{
                  fmtNumber(Math.round(item.totalInvested))
                }}</span>
              </div>
              <div class="cmp-row">
                <span>Lợi nhuận</span>
                <span class="cmp-val green">{{
                  fmtNumber(Math.round(item.profit))
                }}</span>
              </div>
              <div class="cmp-row total-row">
                <span>Kết quả cuối</span>
                <span
                  class="cmp-val"
                  :style="{ color: item.color }"
                  >{{ fmtNumber(Math.round(item.finalBalance)) }}</span
                >
              </div>

              <div
                v-if="!item.isBest"
                class="cmp-diff"
              >
                −
                {{
                  fmtNumber(
                    Math.round(
                      results.items[0].finalBalance - item.finalBalance,
                    ),
                  )
                }}
                VND so với tốt nhất
              </div>
            </div>
          </div>

          <!-- Chart -->
          <div class="chart-area">
            <div class="chart-header">
              <p class="chart-unit">
                Đơn vị: triệu đồng — Kết quả tích lũy theo thời gian
              </p>
              <div class="chart-hints">
                <span>🖱️ Cuộn để zoom</span>
                <span>🖐️ Kéo để di chuyển</span>
                <button
                  class="reset-zoom-btn"
                  @click="resetZoom"
                >
                  ↺ Reset zoom
                </button>
              </div>
            </div>
            <canvas ref="chartCanvas"></canvas>
          </div>

          <!-- Insight -->
          <div class="insight-box">
            <div class="insight-title">💡 Phân tích</div>
            <p class="insight-text">{{ results.insight }}</p>
          </div>

          <div class="disclaimer">
            <span>⚠️</span>
            <span
              >Kết quả chỉ mang tính minh họa. Việc đầu tư luôn có các yếu tố
              rủi ro nên nhà đầu tư cần nghiên cứu kỹ trước khi quyết
              định.</span
            >
          </div>
        </template>

        <template v-else>
          <div class="empty-state">
            <div class="empty-icon">⚖️</div>
            <p>
              Nhập thông tin và nhấn <strong>So sánh ngay</strong> để xem kết
              quả
            </p>
          </div>
        </template>
      </div>
    </div>
  </div>
</template>

<script setup>
import { ref, nextTick } from 'vue';
import SliderInput from './SliderInput.vue';

// ─── Config ──────────────────────────────────────────────────────────────────
const FREQS = [
  {
    key: 'weekly',
    icon: '📅',
    label: 'Theo tuần',
    shortLabel: 'Tuần',
    periodsPerYear: 52,
    color: '#16a34a',
  },
  {
    key: 'monthly',
    icon: '🗓️',
    label: 'Theo tháng',
    shortLabel: 'Tháng',
    periodsPerYear: 12,
    color: '#2563eb',
  },
  {
    key: 'quarterly',
    icon: '📆',
    label: 'Theo quý',
    shortLabel: 'Quý',
    periodsPerYear: 4,
    color: '#ea580c',
  },
  {
    key: 'yearly',
    icon: '📅',
    label: 'Theo năm',
    shortLabel: 'Năm',
    periodsPerYear: 1,
    color: '#7c3aed',
  },
];

const RANK_LABELS = ['', '🥇', '🥈', '🥉', '4️⃣'];

// ─── State ───────────────────────────────────────────────────────────────────
const principal = ref(100_000_000);
const yearlyAmount = ref(24_000_000);
const rate = ref(10);
const time = ref(10);
const results = ref(null);
const chartCanvas = ref(null);
let chartInstance = null;

// ─── Helpers ─────────────────────────────────────────────────────────────────
const fmtNumber = (n) => new Intl.NumberFormat('vi-VN').format(Math.round(n));
const fmtShort = (n) => {
  if (n >= 1_000_000_000) return (n / 1_000_000_000).toFixed(2) + ' tỷ';
  if (n >= 1_000_000) return (n / 1_000_000).toFixed(1) + ' tr';
  return fmtNumber(n);
};

// ─── Compound calc ───────────────────────────────────────────────────────────
function finalBalance(p, perPeriod, periodsPerYear, years, annualRate) {
  const r = annualRate / 100 / periodsPerYear;
  let bal = p;
  for (let i = 0; i < periodsPerYear * years; i++)
    bal = bal * (1 + r) + perPeriod;
  return bal;
}

function buildSeries(p, perPeriod, periodsPerYear, years, annualRate) {
  const r = annualRate / 100 / periodsPerYear;
  const n = periodsPerYear * years;
  const sampleEvery = Math.max(1, Math.round(periodsPerYear / 4));
  const series = [];
  let bal = p;
  for (let i = 0; i <= n; i++) {
    if (i > 0) bal = bal * (1 + r) + perPeriod;
    if (i % sampleEvery === 0) series.push(+(bal / 1_000_000).toFixed(2));
  }
  return series;
}

// ─── Compare ─────────────────────────────────────────────────────────────────
function runCompare() {
  const p = principal.value;
  const yly = yearlyAmount.value;
  const yrs = time.value;
  const r = rate.value;

  const items = FREQS.map((f) => {
    const perPeriod = yly / f.periodsPerYear;
    const bal = finalBalance(p, perPeriod, f.periodsPerYear, yrs, r);
    const totalInvested = p + perPeriod * f.periodsPerYear * yrs;
    return {
      ...f,
      perPeriod,
      finalBalance: bal,
      totalInvested,
      profit: bal - totalInvested,
      isBest: false,
      isWorst: false,
      rank: '',
    };
  }).sort((a, b) => b.finalBalance - a.finalBalance);

  items.forEach((item, i) => {
    item.rank = RANK_LABELS[i + 1] || `${i + 1}`;
    item.isBest = i === 0;
    item.isWorst = i === items.length - 1;
  });

  const gain = items[0].finalBalance - items[items.length - 1].finalBalance;

  // Chart labels (quarterly axis)
  const totalSamples = yrs * 4 + 1;
  const labels = Array.from({ length: totalSamples }, (_, i) => {
    const yr = Math.floor(i / 4);
    const q = i % 4;
    return q === 0 ? `${2026 + yr}` : `Q${q + 1}/${2026 + yr}`;
  });

  const seriesMap = {};
  for (const f of FREQS) {
    const raw = buildSeries(
      p,
      yly / f.periodsPerYear,
      f.periodsPerYear,
      yrs,
      r,
    );
    seriesMap[f.key] = Array.from(
      { length: totalSamples },
      (_, i) => raw[i] ?? null,
    );
  }

  // Insight
  const weeklyFinal = items.find((i) => i.key === 'weekly').finalBalance;
  const monthlyFinal = items.find((i) => i.key === 'monthly').finalBalance;
  const diff = weeklyFinal - monthlyFinal;
  const pct = ((diff / monthlyFinal) * 100).toFixed(2);
  const insight = `Đầu tư theo tuần vượt trội hơn theo tháng khoảng ${fmtNumber(Math.round(diff))} VND (${pct}%) sau ${yrs} năm. Nguyên nhân: vốn được đưa vào thị trường sớm hơn và tái đầu tư qua nhiều chu kỳ hơn trong năm, giúp lãi kép tích lũy nhanh hơn. Khoảng cách này càng lớn khi lãi suất và thời gian tăng.`;

  results.value = {
    items,
    winner: { ...items[0] },
    gain,
    insight,
    labels,
    seriesMap,
  };
  nextTick(() => renderChart(labels, seriesMap, items));
}

// ─── Chart ───────────────────────────────────────────────────────────────────
function renderChart(labels, seriesMap, items) {
  if (!chartCanvas.value) return;
  chartInstance?.destroy();
  const Chart = window.Chart;
  if (!Chart) return;

  const datasets = FREQS.map((f) => {
    const item = items.find((i) => i.key === f.key);
    return {
      label: `${f.icon} ${f.label}`,
      data: seriesMap[f.key],
      borderColor: f.color,
      borderWidth: item?.isBest ? 3 : 1.8,
      backgroundColor: 'transparent',
      fill: false,
      tension: 0.4,
      pointRadius: 0,
      pointHoverRadius: 5,
      borderDash: item?.isWorst ? [5, 4] : [],
    };
  });

  chartInstance = new Chart(chartCanvas.value, {
    type: 'line',
    data: { labels, datasets },
    options: {
      responsive: true,
      maintainAspectRatio: true,
      interaction: { mode: 'index', intersect: false },
      plugins: {
        legend: {
          labels: {
            color: '#475569',
            font: { family: "'Be Vietnam Pro', sans-serif", size: 12 },
            usePointStyle: true,
          },
        },
        tooltip: {
          backgroundColor: '#ffffff',
          borderColor: '#e2e8f0',
          borderWidth: 1,
          titleColor: '#64748b',
          bodyColor: '#1e293b',
          titleFont: { family: "'Be Vietnam Pro', sans-serif" },
          bodyFont: { family: "'Be Vietnam Pro', sans-serif" },
          callbacks: {
            label: (v) =>
              ` ${v.dataset.label}: ${fmtShort(v.parsed.y * 1_000_000)} VND`,
          },
        },
        zoom: {
          pan: { enabled: true, mode: 'x' },
          zoom: {
            wheel: { enabled: true },
            pinch: { enabled: true },
            mode: 'x',
          },
        },
      },
      scales: {
        x: {
          ticks: {
            color: '#94a3b8',
            font: { family: "'Be Vietnam Pro', sans-serif", size: 11 },
            maxTicksLimit: 10,
          },
          grid: { color: '#f1f5f9' },
        },
        y: {
          ticks: {
            color: '#94a3b8',
            font: { family: "'Be Vietnam Pro', sans-serif", size: 11 },
          },
          grid: { color: '#f1f5f9' },
        },
      },
    },
  });
}

function resetZoom() {
  chartInstance?.resetZoom();
}
</script>

<style scoped>
@import url('https://fonts.googleapis.com/css2?family=Be+Vietnam+Pro:wght@300;400;500;600;700&display=swap');

.app {
  font-family: 'Be Vietnam Pro', sans-serif;
  background: #ffffff;
  border: 1px solid #e2e8f0;
  border-radius: 16px;
  width: 100%;
  max-width: 1200px;
  overflow: hidden;
  box-shadow: 0 8px 40px rgba(15, 23, 42, 0.1);
}

/* ── Header ── */
.header {
  background: linear-gradient(135deg, #eff6ff 0%, #f8fafc 100%);
  border-bottom: 1px solid #e2e8f0;
  padding: 22px 32px;
  display: flex;
  align-items: center;
  gap: 14px;
}
.header-icon {
  width: 42px;
  height: 42px;
  background: linear-gradient(135deg, #3b82f6, #06b6d4);
  border-radius: 11px;
  display: flex;
  align-items: center;
  justify-content: center;
  font-size: 20px;
  box-shadow: 0 4px 12px rgba(59, 130, 246, 0.3);
}
.header h1 {
  font-size: 19px;
  font-weight: 700;
  color: #0f172a;
  letter-spacing: -0.3px;
  margin: 0;
}
.header p {
  font-size: 13px;
  color: #64748b;
  margin-top: 2px;
}

/* ── Layout ── */
.main {
  display: grid;
  grid-template-columns: 320px 1fr;
  min-height: 640px;
}

/* ── Sidebar ── */
.sidebar {
  background: #f8fafc;
  border-right: 1px solid #e2e8f0;
  padding: 26px 22px;
  display: flex;
  flex-direction: column;
}

.compare-note {
  font-size: 12px;
  color: #64748b;
  line-height: 1.65;
  background: #eff6ff;
  border-radius: 8px;
  padding: 12px;
  margin-bottom: 20px;
  border: 1px solid #bfdbfe;
}
.compare-note strong {
  color: #1d4ed8;
}

.breakdown-box {
  background: #ffffff;
  border: 1.5px solid #e2e8f0;
  border-radius: 10px;
  padding: 14px;
  margin-bottom: 18px;
}
.breakdown-title {
  font-size: 11px;
  font-weight: 700;
  color: #94a3b8;
  text-transform: uppercase;
  letter-spacing: 0.6px;
  margin-bottom: 10px;
}
.breakdown-row {
  display: flex;
  justify-content: space-between;
  align-items: center;
  padding: 5px 0;
  font-size: 13px;
  color: #475569;
}
.bd-icon-label {
  display: flex;
  align-items: center;
  gap: 6px;
}
.bd-val {
  font-weight: 700;
}

.calc-btn {
  margin-top: auto;
  width: 100%;
  padding: 13px;
  background: linear-gradient(135deg, #3b82f6, #2563eb);
  border: none;
  border-radius: 10px;
  color: white;
  font-family: 'Be Vietnam Pro', sans-serif;
  font-size: 14px;
  font-weight: 700;
  cursor: pointer;
  transition: all 0.2s;
  box-shadow: 0 4px 14px rgba(59, 130, 246, 0.35);
  letter-spacing: 0.3px;
}
.calc-btn:hover {
  transform: translateY(-1px);
  box-shadow: 0 6px 20px rgba(59, 130, 246, 0.45);
}
.calc-btn:active {
  transform: translateY(0);
}

/* ── Content ── */
.content {
  padding: 28px 32px;
  display: flex;
  flex-direction: column;
  gap: 20px;
  overflow-y: auto;
  background: #ffffff;
}

/* ── Winner banner ── */
.winner-banner {
  border: 2px solid;
  border-radius: 12px;
  padding: 16px 20px;
  display: flex;
  align-items: center;
  gap: 16px;
}
.winner-icon {
  font-size: 30px;
  flex-shrink: 0;
}
.winner-title {
  font-size: 11px;
  color: #64748b;
  font-weight: 600;
  text-transform: uppercase;
  letter-spacing: 0.5px;
}
.winner-name {
  font-size: 18px;
  font-weight: 700;
  margin-top: 3px;
}
.winner-gain {
  margin-left: auto;
  text-align: right;
  font-size: 16px;
  font-weight: 700;
  color: #16a34a;
  white-space: nowrap;
}
.winner-vs {
  display: block;
  font-size: 11px;
  color: #64748b;
  font-weight: 400;
  margin-top: 2px;
}

/* ── Compare cards ── */
.compare-cards {
  display: grid;
  grid-template-columns: repeat(4, 1fr);
  gap: 12px;
}
.cmp-card {
  background: #f8fafc;
  border: 1.5px solid #e2e8f0;
  border-radius: 12px;
  padding: 16px;
  transition:
    border-color 0.2s,
    background 0.2s;
}
.cmp-card.worst {
  opacity: 0.65;
}

.cmp-badge {
  font-size: 10px;
  font-weight: 700;
  text-transform: uppercase;
  letter-spacing: 0.5px;
  margin-bottom: 8px;
}
.worst-badge {
  color: #94a3b8;
}
.cmp-rank {
  font-size: 18px;
  margin-bottom: 6px;
}

.cmp-label {
  font-size: 14px;
  font-weight: 700;
  color: #0f172a;
  margin-bottom: 3px;
}
.cmp-contribution {
  font-size: 11px;
  color: #64748b;
  margin-bottom: 10px;
}
.cmp-divider {
  height: 1px;
  background: #e2e8f0;
  margin-bottom: 10px;
}

.cmp-row {
  display: flex;
  justify-content: space-between;
  font-size: 11px;
  color: #64748b;
  margin-bottom: 5px;
}
.cmp-val {
  font-weight: 700;
  font-size: 12px;
}
.cmp-val.neutral {
  color: #334155;
}
.cmp-val.green {
  color: #16a34a;
}

.total-row {
  margin-top: 8px;
  padding-top: 8px;
  border-top: 1px solid #e2e8f0;
}
.total-row span:first-child {
  color: #334155;
  font-weight: 600;
}
.total-row .cmp-val {
  font-size: 13px;
}

.cmp-diff {
  margin-top: 10px;
  font-size: 10px;
  color: #94a3b8;
  border-top: 1px dashed #e2e8f0;
  padding-top: 8px;
  line-height: 1.5;
}

/* ── Chart ── */
.chart-area {
  flex: 1;
  min-height: 0;
}
.chart-header {
  display: flex;
  align-items: center;
  justify-content: space-between;
  margin-bottom: 10px;
  gap: 12px;
  flex-wrap: wrap;
}
.chart-unit {
  font-size: 12px;
  color: #94a3b8;
}
.chart-hints {
  display: flex;
  align-items: center;
  gap: 10px;
  flex-wrap: wrap;
}
.chart-hints span {
  font-size: 11px;
  color: #94a3b8;
}
.reset-zoom-btn {
  padding: 4px 10px;
  background: #f1f5f9;
  border: 1.5px solid #e2e8f0;
  border-radius: 6px;
  font-family: 'Be Vietnam Pro', sans-serif;
  font-size: 11px;
  font-weight: 600;
  color: #475569;
  cursor: pointer;
  transition: all 0.15s;
}
.reset-zoom-btn:hover {
  background: #e2e8f0;
  color: #0f172a;
}

/* ── Insight ── */
.insight-box {
  background: #eff6ff;
  border: 1px solid #bfdbfe;
  border-radius: 10px;
  padding: 14px 16px;
}
.insight-title {
  font-size: 12px;
  font-weight: 700;
  color: #2563eb;
  margin-bottom: 6px;
}
.insight-text {
  font-size: 13px;
  color: #1e40af;
  line-height: 1.65;
}

/* ── Disclaimer ── */
.disclaimer {
  background: #fffbeb;
  border: 1px solid #fde68a;
  border-radius: 8px;
  padding: 12px 16px;
  font-size: 12px;
  color: #92400e;
  display: flex;
  gap: 8px;
  align-items: flex-start;
}

/* ── Empty ── */
.empty-state {
  display: flex;
  flex-direction: column;
  align-items: center;
  justify-content: center;
  height: 100%;
  color: #94a3b8;
  gap: 14px;
}
.empty-icon {
  font-size: 52px;
  opacity: 0.4;
}
.empty-state p {
  font-size: 14px;
}
.empty-state strong {
  color: #3b82f6;
}

@media (max-width: 1000px) {
  .main {
    grid-template-columns: 1fr;
  }
  .sidebar {
    border-right: none;
    border-bottom: 1px solid #e2e8f0;
  }
  .compare-cards {
    grid-template-columns: repeat(2, 1fr);
  }
}
@media (max-width: 600px) {
  .compare-cards {
    grid-template-columns: 1fr;
  }
}
</style>
