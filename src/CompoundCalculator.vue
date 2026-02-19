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
          <!-- Compare cards -->
          <div class="compare-cards">
            <div
              v-for="item in results.items"
              :key="item.key"
              class="cmp-card"
            >
              <div
                class="cmp-freq-dot"
                :style="{ background: item.color }"
              ></div>
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
            </div>
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
import { ref } from 'vue';
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

// ─── State ───────────────────────────────────────────────────────────────────
const principal = ref(100_000_000);
const yearlyAmount = ref(24_000_000);
const rate = ref(10);
const time = ref(10);
const results = ref(null);

// ─── Helpers ─────────────────────────────────────────────────────────────────
const fmtNumber = (n) => new Intl.NumberFormat('vi-VN').format(Math.round(n));

// ─── Compound calc ───────────────────────────────────────────────────────────
function finalBalance(p, perPeriod, periodsPerYear, years, annualRate) {
  const r = annualRate / 100 / periodsPerYear;
  let bal = p;
  for (let i = 0; i < periodsPerYear * years; i++)
    bal = bal * (1 + r) + perPeriod;
  return bal;
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
    };
  });

  // Insight (giữ nguyên logic so sánh nhưng ko hiện trên UI card)
  const weeklyFinal = items.find((i) => i.key === 'weekly').finalBalance;
  const monthlyFinal = items.find((i) => i.key === 'monthly').finalBalance;
  const diff = weeklyFinal - monthlyFinal;
  const pct = ((diff / monthlyFinal) * 100).toFixed(2);
  const insight = `Đầu tư theo tuần vượt trội hơn theo tháng khoảng ${fmtNumber(Math.round(diff))} VND (${pct}%) sau ${yrs} năm. Nguyên nhân: vốn được đưa vào thị trường sớm hơn và tái đầu tư qua nhiều chu kỳ hơn trong năm, giúp lãi kép tích lũy nhanh hơn. Khoảng cách này càng lớn khi lãi suất và thời gian tăng.`;

  results.value = { items, insight };
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
  max-width: 1400px;
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
  min-height: 560px;
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
  background: #ffffff;
}

/* ── Compare cards ── */
.compare-cards {
  display: grid;
  grid-template-columns: repeat(4, minmax(180px, 1fr));
  gap: 14px;
}

.cmp-card {
  background: #f8fafc;
  border: 1.5px solid #e2e8f0;
  border-radius: 12px;
  padding: 18px 20px;
  position: relative;
  min-width: 0;
}

.cmp-freq-dot {
  width: 8px;
  height: 8px;
  border-radius: 50%;
  margin-bottom: 10px;
}

.cmp-label {
  font-size: 14px;
  font-weight: 700;
  color: #0f172a;
  margin-bottom: 3px;
}
.cmp-contribution {
  font-size: 11px;
  color: #94a3b8;
  margin-bottom: 12px;
}
.cmp-divider {
  height: 1px;
  background: #e2e8f0;
  margin-bottom: 12px;
}

.cmp-row {
  display: flex;
  justify-content: space-between;
  align-items: baseline;
  font-size: 12px;
  color: #64748b;
  margin-bottom: 6px;
  gap: 8px;
}
.cmp-val {
  font-weight: 700;
  font-size: 13px;
  white-space: nowrap;
}
.cmp-val.neutral {
  color: #334155;
}
.cmp-val.green {
  color: #16a34a;
}

.total-row {
  margin-top: 10px;
  padding-top: 10px;
  border-top: 1px solid #e2e8f0;
}
.total-row span:first-child {
  font-size: 12px;
  font-weight: 600;
  color: #334155;
}
.total-row .cmp-val {
  font-size: 15px;
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
