<script setup lang="ts">
import { ref, computed, watch, onUnmounted } from 'vue'

const N = ref(100) // 頂点数 100〜40000
const MAX_HISTORY = 5000 // プロットに保持する最大点数

// 頂点0 = 頂点1（常に黒、更新しない）
const colors = ref<number[]>([])
const h = ref(5) // 1〜N（つまみは奇数、テキストで任意の整数）

// 1〜N の最大奇数（つまみの max）
const maxOdd = computed(() => {
  const n = N.value
  return n >= 1 ? (n % 2 === 1 ? n : n - 1) : 1
})

// つまみ用：現在の h を奇数に丸めた値（表示・操作用）
const hSliderValue = computed({
  get: () => {
    const odd = h.value % 2 === 1 ? h.value : h.value - 1
    return Math.max(1, Math.min(maxOdd.value, odd))
  },
  set: (v: number) => {
    h.value = v
  },
})

function onHInput(e: Event) {
  const v = parseInt((e.target as HTMLInputElement).value, 10)
  if (!isNaN(v)) h.value = Math.max(1, Math.min(N.value, v))
}
const initialBlackRatio = ref(0.5)
const mode = ref<'sync' | 'async'>('sync')
const running = ref(false)
const fps = ref(8) // アニメーションのFPS（1〜30）
const roundCount = ref(0) // 経過ラウンド数
const history = ref<{ round: number; ratio: number }[]>([]) // ラウンドごとの黒割合（プロット用）
let timerId: ReturnType<typeof setInterval> | null = null

// 全体の黒頂点数を initialBlackRatio に合わせる（頂点1は常に黒のため含める）
function initState() {
  const n = N.value
  const totalBlack = Math.max(1, Math.min(n, Math.round(initialBlackRatio.value * n)))
  const blackAmongRest = totalBlack - 1 // 頂点2〜nのうち黒にする数（頂点1が1つ黒）
  const indices = Array.from({ length: n - 1 }, (_, i) => i + 1)
  for (let i = 0; i < n - 1; i++) {
    const j = i + Math.floor(Math.random() * (n - 1 - i))
    ;[indices[i], indices[j]] = [indices[j], indices[i]]
  }
  const blackSet = new Set(indices.slice(0, blackAmongRest))
  const arr = Array.from({ length: n }, (_, i) => (i === 0 ? 1 : (blackSet.has(i) ? 1 : 0)))
  colors.value = arr
}

// 初期黒割合のつまみに合わせて、全体の黒割合が一致するよう頂点2〜nを再割り当て
function applyInitialBlackRatio() {
  const n = colors.value.length
  if (n === 0) return
  const totalBlack = Math.max(1, Math.min(n, Math.round(initialBlackRatio.value * n)))
  const blackAmongRest = totalBlack - 1
  const indices = Array.from({ length: n - 1 }, (_, i) => i + 1)
  for (let i = 0; i < n - 1; i++) {
    const j = i + Math.floor(Math.random() * (n - 1 - i))
    ;[indices[i], indices[j]] = [indices[j], indices[i]]
  }
  const blackSet = new Set(indices.slice(0, blackAmongRest))
  colors.value = colors.value.map((_, i) => (i === 0 ? 1 : (blackSet.has(i) ? 1 : 0)))
}

watch(initialBlackRatio, () => {
  applyInitialBlackRatio()
})

watch(N, () => {
  initState()
  roundCount.value = 0
  const c = colors.value
  if (c.length) history.value = [{ round: 0, ratio: (c.filter((x) => x === 1).length / c.length) * 100 }]
  h.value = Math.max(1, Math.min(N.value, h.value))
})

watch(fps, () => {
  if (running.value) restartTimer()
})

function getMinorityColor(indices: number[]): number {
  const sampled = indices.map((i) => colors.value[i])
  const black = sampled.filter((c) => c === 1).length
  const white = sampled.length - black
  if (black === 0) return 0
  if (white === 0) return 1
  if (black === white) return Math.random() < 0.5 ? 1 : 0 // tie はランダムに break
  return black < white ? 1 : 0
}

// h個の頂点を重複を許して一様ランダムにサンプリング（with replacement）
function sampleWithReplacement(n: number, k: number): number[] {
  return Array.from({ length: k }, () => Math.floor(Math.random() * n))
}

function step() {
  const n = N.value
  const next = [...colors.value]
  next[0] = 1 // 頂点1は常に黒（更新しない）
  if (mode.value === 'sync') {
    for (let v = 1; v < n; v++) {
      const indices = sampleWithReplacement(n, h.value)
      next[v] = getMinorityColor(indices)
    }
  } else {
    const v = 1 + Math.floor(Math.random() * (n - 1)) // 頂点2〜nのみ
    const indices = sampleWithReplacement(n, h.value)
    next[v] = getMinorityColor(indices)
  }
  colors.value = next
  roundCount.value++
  const ratio = (next.filter((c) => c === 1).length / n) * 100
  history.value.push({ round: roundCount.value, ratio })
  if (history.value.length > MAX_HISTORY) history.value.shift()
  // 全頂点が黒になったら停止
  if (next.every((c) => c === 1)) {
    stop()
  }
}

function start() {
  if (timerId) return
  running.value = true
  const intervalMs = Math.round(1000 / fps.value)
  timerId = setInterval(step, intervalMs)
}

function restartTimer() {
  if (!timerId) return
  clearInterval(timerId)
  const intervalMs = Math.round(1000 / fps.value)
  timerId = setInterval(step, intervalMs)
}

function stop() {
  if (timerId) {
    clearInterval(timerId)
    timerId = null
  }
  running.value = false
}

function reset() {
  stop()
  roundCount.value = 0
  initState()
  const n = colors.value.length
  const black = colors.value.filter((c) => c === 1).length
  history.value = [{ round: 0, ratio: n ? (black / n) * 100 : 0 }]
}

// プロット用パス（時刻=ラウンド vs 黒頂点割合%）
const plotWidth = 1000
const plotHeight = 280
const plotMargin = { left: 44, top: 20, right: 20, bottom: 36 }

const plotPath = computed(() => {
  const pts = history.value
  if (pts.length === 0) return ''
  const maxR = Math.max(1, ...pts.map((p) => p.round))
  const x = (r: number) => plotMargin.left + (r / maxR) * (plotWidth - plotMargin.left - plotMargin.right)
  const y = (ratio: number) =>
    plotMargin.top + (1 - ratio / 100) * (plotHeight - plotMargin.top - plotMargin.bottom)
  return pts.map((p) => `${x(p.round)},${y(p.ratio)}`).join(' ')
})

const plotViewBox = `${0} ${0} ${plotWidth} ${plotHeight}`

// 黒頂点の割合（%）
const blackRatio = computed(() => {
  const c = colors.value
  if (!c.length) return '0'
  const black = c.filter((x) => x === 1).length
  return ((black / c.length) * 100).toFixed(1)
})

initState()
// 初期状態をプロットに1点追加
const c0 = colors.value
if (c0.length) {
  history.value = [{ round: 0, ratio: (c0.filter((x) => x === 1).length / c0.length) * 100 }]
}
onUnmounted(() => stop())
</script>

<template>
  <div class="minority-graph">
    <div class="controls-row">
      <div class="mode-buttons">
        <button
          class="btn"
          :class="{ active: mode === 'sync' }"
          @click="mode = 'sync'"
        >
          sync
        </button>
        <button
          class="btn"
          :class="{ active: mode === 'async' }"
          @click="mode = 'async'"
        >
          async
        </button>
      </div>
      <div class="sliders">
        <label class="slider-label">
          <span>頂点数</span>
          <input
            v-model.number="N"
            type="range"
            min="100"
            max="40000"
            step="500"
            class="slider slider-n"
          />
          <span class="value">{{ N }}</span>
        </label>
        <label class="slider-label">
          <span>h（1〜nの奇数）</span>
          <input
            v-model.number="hSliderValue"
            type="range"
            min="1"
            :max="maxOdd"
            step="2"
            class="slider"
          />
          <span class="value">{{ h }}</span>
          <input
            type="number"
            :value="h"
            :min="1"
            :max="N"
            class="h-input"
            @input="onHInput"
          />
        </label>
        <label class="slider-label">
          <span>初期黒割合</span>
          <input
            v-model.number="initialBlackRatio"
            type="range"
            min="0"
            max="1"
            step="0.01"
            class="slider"
          />
          <span class="value">{{ (initialBlackRatio * 100).toFixed(0) }}%</span>
        </label>
      </div>
      <div class="action-buttons">
        <button class="btn action" :disabled="running" @click="start">
          開始
        </button>
        <button class="btn action" @click="reset">リセット</button>
        <button class="btn action" :disabled="!running" @click="stop">
          停止
        </button>
        <label class="slider-label fps-slider">
          <span>FPS</span>
          <input
            v-model.number="fps"
            type="range"
            min="1"
            max="30"
            step="1"
            class="slider slider-fps"
          />
          <span class="value">{{ fps }}</span>
        </label>
      </div>
    </div>
    <div class="plot-container">
      <svg
        :viewBox="plotViewBox"
        class="plot-svg"
        preserveAspectRatio="xMidYMid meet"
      >
        <!-- 枠・軸 -->
        <rect
          :x="plotMargin.left"
          :y="plotMargin.top"
          :width="plotWidth - plotMargin.left - plotMargin.right"
          :height="plotHeight - plotMargin.top - plotMargin.bottom"
          fill="none"
          stroke="#ccc"
          stroke-width="1"
        />
        <text :x="plotMargin.left - 8" :y="plotMargin.top + 4" class="axis-label">100%</text>
        <text :x="plotMargin.left - 8" :y="plotHeight - plotMargin.bottom + 4" class="axis-label">0%</text>
        <text :x="plotMargin.left" :y="plotHeight - 4" class="axis-label">0</text>
        <text :x="plotWidth - plotMargin.right - 56" :y="plotHeight - 4" class="axis-label">ラウンド →</text>
        <!-- プロット線 -->
        <polyline
          v-if="history.length > 0"
          :points="plotPath"
          fill="none"
          stroke="#1967d2"
          stroke-width="1.5"
          class="plot-line"
        />
      </svg>
      <div class="plot-legend">
        ラウンド {{ roundCount }}　黒 {{ blackRatio }}%
      </div>
    </div>
  </div>
</template>

<style scoped>
.minority-graph {
  width: 100%;
  display: flex;
  flex-direction: column;
  align-items: stretch;
  gap: 12px;
  padding: 8px;
  box-sizing: border-box;
}
.controls-row {
  display: flex;
  flex-wrap: wrap;
  align-items: center;
  justify-content: center;
  gap: 16px;
}
.mode-buttons {
  display: flex;
  gap: 8px;
}
.btn {
  padding: 6px 14px;
  border: 1px solid #333;
  border-radius: 6px;
  background: #f5f5f5;
  cursor: pointer;
  font-size: 14px;
}
.btn:hover:not(:disabled) {
  background: #eee;
}
.btn.active {
  background: #333;
  color: #fff;
}
.btn:disabled {
  opacity: 0.5;
  cursor: not-allowed;
}
.sliders {
  display: flex;
  gap: 20px;
  align-items: center;
}
.slider-label {
  display: flex;
  align-items: center;
  gap: 8px;
  font-size: 13px;
}
.slider-label span:first-child {
  min-width: 70px;
}
.slider {
  width: 80px;
}
.value {
  min-width: 36px;
  font-variant-numeric: tabular-nums;
}
.h-input {
  width: 56px;
  padding: 4px 6px;
  font-size: 13px;
  border: 1px solid #999;
  border-radius: 4px;
  box-sizing: border-box;
}
.action-buttons {
  display: flex;
  gap: 8px;
  align-items: center;
  flex-wrap: wrap;
}
.fps-slider {
  margin-left: 8px;
}
.slider-fps {
  width: 60px;
}
.btn.action {
  background: #fff;
}
.plot-container {
  width: 100%;
  border: 1px solid #ddd;
  border-radius: 8px;
  background: #fafafa;
  padding: 8px;
  box-sizing: border-box;
}
.plot-svg {
  width: 100%;
  height: auto;
  display: block;
}
.axis-label {
  font-size: 11px;
  fill: #666;
  font-family: inherit;
}
.plot-legend {
  margin-top: 6px;
  font-size: 14px;
  color: #333;
  text-align: right;
}
.slider-n {
  width: 120px;
}
</style>
