<template>
  <div class="input-group">
    <label>{{ label }}</label>
    <div
      class="input-row"
      :class="{ focused: isFocused }"
    >
      <input
        ref="inputEl"
        type="text"
        class="text-input"
        :value="displayValue"
        @focus="onFocus"
        @blur="onBlur"
        @input="onInput"
        @keydown.enter="inputEl.blur()"
        inputmode="numeric"
      />
      <span class="unit">{{ unit }}</span>
    </div>
    <div class="slider-row">
      <button
        class="adj-btn"
        @click="decrement"
      >
        −
      </button>
      <input
        type="range"
        class="slider"
        :min="min"
        :max="max"
        :step="step"
        :value="modelValue"
        @input="onSliderInput"
      />
      <button
        class="adj-btn"
        @click="increment"
      >
        +
      </button>
    </div>
    <div class="slider-labels">
      <span>{{ format(min) }}</span>
      <span>{{ format(max) }}</span>
    </div>
  </div>
</template>

<script setup>
import { ref, computed, watch } from 'vue';

const props = defineProps({
  label: String,
  modelValue: Number,
  min: { type: Number, default: 0 },
  max: { type: Number, default: 100 },
  step: { type: Number, default: 1 },
  unit: String,
  format: {
    type: Function,
    default: (v) => new Intl.NumberFormat('vi-VN').format(Math.round(v)),
  },
});

const emit = defineEmits(['update:modelValue']);
const inputEl = ref(null);
const isFocused = ref(false);
const rawText = ref(''); // giá trị người dùng đang gõ

// Giá trị hiển thị: khi focus → raw text; khi blur → formatted
const displayValue = computed(() =>
  isFocused.value ? rawText.value : props.format(props.modelValue),
);

const clamp = (v) => Math.min(props.max, Math.max(props.min, v));

function onFocus() {
  isFocused.value = true;
  // Hiển thị số nguyên không dấu phẩy để dễ sửa
  rawText.value = String(Math.round(props.modelValue));
  // Chọn hết text để tiện gõ đè
  setTimeout(() => inputEl.value?.select(), 0);
}

function onInput(e) {
  // Cho phép gõ tự do, chỉ lọc ký tự không phải số
  rawText.value = e.target.value.replace(/[^0-9]/g, '');
  const n = parseInt(rawText.value, 10);
  if (!isNaN(n)) emit('update:modelValue', clamp(n));
}

function onBlur() {
  isFocused.value = false;
  const n = parseInt(rawText.value, 10);
  if (!isNaN(n)) emit('update:modelValue', clamp(n));
  rawText.value = '';
}

function onSliderInput(e) {
  emit('update:modelValue', Number(e.target.value));
}

function decrement() {
  emit('update:modelValue', clamp(props.modelValue - props.step));
}

function increment() {
  emit('update:modelValue', clamp(props.modelValue + props.step));
}
</script>

<style scoped>
.input-group {
  margin-bottom: 20px;
}

label {
  display: block;
  font-size: 11px;
  font-weight: 700;
  color: #64748b;
  text-transform: uppercase;
  letter-spacing: 0.6px;
  margin-bottom: 7px;
}

.input-row {
  display: flex;
  align-items: center;
  background: #ffffff;
  border: 1.5px solid #e2e8f0;
  border-radius: 9px;
  overflow: hidden;
  transition:
    border-color 0.18s,
    box-shadow 0.18s;
}

.input-row.focused {
  border-color: #3b82f6;
  box-shadow: 0 0 0 3px rgba(59, 130, 246, 0.12);
}

.text-input {
  flex: 1;
  background: transparent;
  border: none;
  outline: none;
  padding: 10px 12px;
  font-family: 'Be Vietnam Pro', sans-serif;
  font-size: 14px;
  font-weight: 700;
  color: #1e293b;
}

.unit {
  padding: 10px 12px;
  font-size: 12px;
  font-weight: 600;
  color: #94a3b8;
  background: #f8fafc;
  border-left: 1.5px solid #e2e8f0;
  white-space: nowrap;
}

.slider-row {
  display: flex;
  align-items: center;
  gap: 8px;
  margin-top: 8px;
}

.adj-btn {
  width: 24px;
  height: 24px;
  background: #f1f5f9;
  border: 1.5px solid #e2e8f0;
  border-radius: 6px;
  color: #64748b;
  font-size: 15px;
  line-height: 1;
  cursor: pointer;
  display: flex;
  align-items: center;
  justify-content: center;
  transition: all 0.15s;
  flex-shrink: 0;
  font-family: monospace;
}
.adj-btn:hover {
  background: #e2e8f0;
  color: #1e293b;
}

.slider {
  flex: 1;
  height: 4px;
  background: #e2e8f0;
  border-radius: 2px;
  outline: none;
  cursor: pointer;
}
.slider::-webkit-slider-thumb {
  -webkit-appearance: none;
  width: 16px;
  height: 16px;
  background: #3b82f6;
  border-radius: 50%;
  cursor: pointer;
  box-shadow: 0 0 0 3px rgba(59, 130, 246, 0.18);
  transition: box-shadow 0.15s;
}
.slider::-webkit-slider-thumb:hover {
  box-shadow: 0 0 0 5px rgba(59, 130, 246, 0.25);
}

.slider-labels {
  display: flex;
  justify-content: space-between;
  font-size: 11px;
  color: #94a3b8;
  margin-top: 4px;
}
</style>
