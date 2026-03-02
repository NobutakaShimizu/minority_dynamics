<script setup lang="ts">
import { ref, computed, watch, onUnmounted } from 'vue'

const N = ref(100) // 頂点数 100〜1000
const radius = 180
const centerX = 220
const centerY = 220
const nodeR = 3

// 頂点0 = 頂点1（常に黒、更新しない）
const colors = ref<number[]>([])
const h = ref(5) // 奇数のみ 1,3,...,99
const initialBlackRatio = ref(0.5)
const mode = ref<'sync' | 'async'>('sync')
const running = ref(false)
const fps = ref(8) // アニメーションのFPS（1〜30）
const roundCount = ref(0) // 経過ラウンド数
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
}

const nodePositions = computed(() => {
  const n = N.value
  const positions: { x: number; y: number }[] = []
  // 頂点1（index 0）は中央に配置
  positions[0] = { x: centerX, y: centerY }
  // 頂点2〜n を floor(n/100) 個の同心円に年輪状に配置
  const nCircle = n - 1
  const numRings = Math.max(1, Math.floor(n / 100))
  const base = Math.floor(nCircle / numRings)
  const remainder = nCircle % numRings
  const counts: number[] = []
  for (let c = 0; c < numRings; c++) {
    counts.push(base + (c < remainder ? 1 : 0))
  }
  const cumsum: number[] = []
  let s = 0
  for (let c = 0; c < numRings; c++) {
    s += counts[c]
    cumsum.push(s)
  }
  for (let i = 1; i < n; i++) {
    let c = 0
    while (c < numRings && cumsum[c] < i) c++
    const prev = c === 0 ? 0 : cumsum[c - 1]
    const countThisRing = counts[c]
    const j = i - 1 - prev
    const angle = (2 * Math.PI * j) / countThisRing - Math.PI / 2
    const r = radius * (c + 1) / numRings
    positions[i] = {
      x: centerX + r * Math.cos(angle),
      y: centerY + r * Math.sin(angle),
    }
  }
  return positions
})

// 完全グラフの一部の辺のみ描画（中央-最内円、各円環上の隣接）
const edgePairs = computed(() => {
  const n = N.value
  const pos = nodePositions.value
  const pairs: { x1: number; y1: number; x2: number; y2: number }[] = []
  const numRings = Math.max(1, Math.floor(n / 100))
  const nCircle = n - 1
  const base = Math.floor(nCircle / numRings)
  const remainder = nCircle % numRings
  const counts: number[] = []
  for (let c = 0; c < numRings; c++) counts.push(base + (c < remainder ? 1 : 0))
  const cumsum: number[] = []
  let s = 0
  for (let c = 0; c < numRings; c++) {
    s += counts[c]
    cumsum.push(s)
  }
  // 中央の頂点1から最内円の頂点へ（見やすさのため最内円のみ）
  const firstRingEnd = cumsum[0]
  for (let i = 1; i <= firstRingEnd; i++) {
    pairs.push({ x1: pos[0].x, y1: pos[0].y, x2: pos[i].x, y2: pos[i].y })
  }
  // 各円環内で隣接する頂点同士を結ぶ
  let idx = 1
  for (let c = 0; c < numRings; c++) {
    const countThisRing = counts[c]
    for (let j = 0; j < countThisRing; j++) {
      const i = idx + j
      const nextJ = (j + 1) % countThisRing
      const iNext = idx + nextJ
      pairs.push({ x1: pos[i].x, y1: pos[i].y, x2: pos[iNext].x, y2: pos[iNext].y })
    }
    idx += countThisRing
  }
  return pairs
})

// 黒頂点の割合（%）
const blackRatio = computed(() => {
  const c = colors.value
  if (!c.length) return '0'
  const black = c.filter((x) => x === 1).length
  return ((black / c.length) * 100).toFixed(1)
})

initState()
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
            max="1000"
            step="10"
            class="slider"
          />
          <span class="value">{{ N }}</span>
        </label>
        <label class="slider-label">
          <span>h（奇数）</span>
          <input
            v-model.number="h"
            type="range"
            min="1"
            max="99"
            step="2"
            class="slider"
          />
          <span class="value">{{ h }}</span>
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
    <svg
      :width="440"
      :height="440"
      viewBox="0 0 440 440"
      class="graph-svg"
    >
      <!-- 完全グラフの辺（隣接する頂点同士のみ薄く表示） -->
      <g stroke="#e0e0e0" stroke-width="0.2" opacity="0.35">
        <line
          v-for="(pair, idx) in edgePairs"
          :key="`e-${idx}`"
          :x1="pair.x1"
          :y1="pair.y1"
          :x2="pair.x2"
          :y2="pair.y2"
        />
      </g>
      <!-- 頂点 -->
      <g>
        <circle
          v-for="(pos, i) in nodePositions"
          :key="i"
          :cx="pos.x"
          :cy="pos.y"
          :r="nodeR"
          :fill="colors[i] ? '#111' : '#fff'"
          stroke="#555"
          stroke-width="0.4"
        />
      </g>
      <!-- 経過ラウンド数・黒頂点割合（右下） -->
      <text
        x="430"
        y="428"
        text-anchor="end"
        class="round-count-text"
      >
        ラウンド {{ roundCount }}　黒 {{ blackRatio }}%
      </text>
    </svg>
  </div>
</template>

<style scoped>
.minority-graph {
  display: flex;
  flex-direction: column;
  align-items: center;
  gap: 12px;
  padding: 8px;
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
.graph-svg {
  border: 1px solid #ddd;
  border-radius: 8px;
  background: #fafafa;
}
.round-count-text {
  font-size: 14px;
  fill: #333;
  font-family: inherit;
}
</style>
