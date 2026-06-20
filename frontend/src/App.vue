<template>
  <div class="min-h-screen bg-slate-900 text-slate-200">
    <div class="sr-only" aria-live="polite" aria-atomic="true">{{ statusAnnouncement }}</div>
    <div class="sr-only" aria-live="assertive" aria-atomic="true">{{ resultAnnouncement }}</div>

    <header class="border-b border-slate-700 px-6 py-4" role="banner">
      <h1 class="text-2xl font-bold text-cyan-400">蒙特卡洛模拟与统计假设检验平台</h1>
      <p class="text-sm text-slate-500 mt-1">随机采样模拟 · 6种MC场景 · 假设检验 · 置信区间可视化</p>
    </header>

    <main id="main-content" class="flex flex-col lg:flex-row gap-4 p-4" role="main">
      <aside class="lg:w-1/4 space-y-4" aria-label="控制面板">
        <nav id="scenario-nav" class="bg-slate-800 rounded-lg p-4 border border-slate-700" aria-label="模拟场景选择">
          <h2 class="text-sm font-bold text-slate-400 mb-3">模拟场景</h2>
          <div class="space-y-1" role="listbox" aria-label="选择模拟场景">
            <div
              v-for="(s, idx) in SCENARIOS"
              :key="s.id"
              role="option"
              tabindex="0"
              :data-scenario-id="s.id"
              :aria-posinset="idx + 1"
              :aria-setsize="SCENARIOS.length"
              :aria-selected="store.currentScenario.id === s.id"
              :aria-label="`${s.name}，${s.description}`"
              @click="selectScenario(s)"
              @keydown.enter.prevent="selectScenario(s)"
              @keydown.space.prevent="selectScenario(s)"
              @keydown.arrow-down.prevent="focusScenarioByOffset($event, s.id, 1)"
              @keydown.arrow-up.prevent="focusScenarioByOffset($event, s.id, -1)"
              :class="[
                'cursor-pointer p-2 rounded border text-sm transition-all',
                store.currentScenario.id === s.id
                  ? 'border-cyan-500 bg-cyan-900/30 text-cyan-400'
                  : 'border-slate-700 text-slate-300 hover:border-slate-500'
              ]"
            >
              <div class="font-bold">{{ s.name }}</div>
              <div class="text-xs text-slate-500 mt-0.5">{{ s.description }}</div>
            </div>
          </div>
        </nav>

        <section class="bg-slate-800 rounded-lg p-4 border border-slate-700" aria-labelledby="params-heading">
          <h2 id="params-heading" class="text-sm font-bold text-slate-400 mb-3">参数控制</h2>
          <div class="mb-3">
            <label for="iterations-range" class="text-xs text-slate-500 block mb-1">
              迭代次数: <span class="text-cyan-400 font-bold">{{ store.iterations }}</span>
            </label>
            <span id="iterations-help" class="sr-only">范围从100到5000，步长100</span>
            <input
              id="iterations-range"
              type="range"
              min="100"
              max="5000"
              step="100"
              v-model.number="store.iterations"
              aria-describedby="iterations-help"
              aria-valuemin="100"
              aria-valuemax="5000"
              :aria-valuenow="store.iterations"
              class="w-full accent-cyan-500"
            />
            <div class="flex justify-between text-xs text-slate-600 mt-1" aria-hidden="true">
              <span>100</span>
              <span>5000</span>
            </div>
          </div>
          <button
            @click="handleRunSimulation"
            :disabled="store.isRunning"
            :aria-busy="store.isRunning"
            class="w-full py-2 bg-cyan-600 hover:bg-cyan-500 disabled:opacity-50 disabled:cursor-not-allowed rounded text-sm font-bold"
          >
            <span class="sr-only">{{ store.isRunning ? '模拟进行中，请稍候' : '执行蒙特卡洛模拟' }}</span>
            <span aria-hidden="true">{{ store.isRunning ? '运行中...' : '▶ 开始模拟' }}</span>
          </button>
        </section>

        <section
          v-if="store.result"
          class="bg-slate-800 rounded-lg p-4 border border-slate-700 text-sm"
          aria-labelledby="result-heading"
        >
          <h2 id="result-heading" class="text-sm font-bold text-slate-400 mb-3">模拟结果</h2>
          <dl class="space-y-2">
            <div class="flex justify-between">
              <dt class="text-slate-500">估算值</dt>
              <dd class="text-cyan-400 font-bold font-mono">{{ store.result.estimate.toFixed(6) }}</dd>
            </div>
            <div v-if="store.result.trueValue !== undefined" class="flex justify-between">
              <dt class="text-slate-500">真实值</dt>
              <dd class="text-green-400 font-mono">{{ store.result.trueValue.toFixed(6) }}</dd>
            </div>
            <div v-if="store.result.error !== undefined" class="flex justify-between">
              <dt class="text-slate-500">误差</dt>
              <dd class="text-orange-400 font-mono">{{ store.result.error.toFixed(6) }}</dd>
            </div>
            <div class="flex justify-between">
              <dt class="text-slate-500">样本数</dt>
              <dd class="text-slate-300">{{ store.result.iterations }}</dd>
            </div>
          </dl>
        </section>
      </aside>

      <div id="charts-section" class="lg:w-3/4 space-y-4">
        <section
          class="bg-slate-800 rounded-lg p-4 border border-slate-700"
          aria-labelledby="convergence-heading"
        >
          <h2 id="convergence-heading" class="text-sm font-bold text-slate-400 mb-3">收敛过程</h2>
          <div
            ref="convergenceRef"
            role="img"
            :aria-label="convergenceChartAlt"
            class="w-full rounded"
            style="height:240px;background:#0f172a;"
          ></div>
          <p v-if="store.result" class="text-xs text-slate-500 mt-2 sr-only">
            收敛图显示模拟值随迭代次数变化的趋势。当前估算值为 {{ store.result.estimate.toFixed(6) }}，
            共执行 {{ store.result.iterations }} 次迭代。
          </p>
        </section>

        <section
          class="bg-slate-800 rounded-lg p-4 border border-slate-700"
          aria-labelledby="histogram-heading"
        >
          <h2 id="histogram-heading" class="text-sm font-bold text-slate-400 mb-3">样本分布直方图</h2>
          <div
            ref="histogramRef"
            role="img"
            :aria-label="histogramChartAlt"
            class="w-full rounded"
            style="height:220px;background:#0f172a;"
          ></div>
          <p v-if="store.result" class="text-xs text-slate-500 mt-2 sr-only">
            样本分布直方图展示了 {{ store.result.samples.length }} 个采样值的频率分布。
          </p>
        </section>

        <section
          class="bg-slate-800 rounded-lg p-4 border border-slate-700"
          aria-labelledby="test-heading"
        >
          <h2 id="test-heading" class="text-sm font-bold text-slate-400 mb-3">假设检验 (独立样本 T 检验)</h2>
          <div class="grid grid-cols-1 md:grid-cols-2 gap-4 mb-3">
            <div>
              <label for="group1-input" class="text-xs text-slate-500 block mb-1">
                样本组A (逗号分隔)
              </label>
              <span id="group1-help" class="sr-only">请输入用逗号分隔的数值，例如 5.1,4.8,5.3</span>
              <textarea
                id="group1-input"
                v-model="group1Input"
                rows="2"
                aria-describedby="group1-help"
                class="w-full mt-1 bg-slate-900 border border-slate-600 rounded px-2 py-1 text-xs font-mono focus:outline-none focus:border-cyan-500 resize-none"
              ></textarea>
            </div>
            <div>
              <label for="group2-input" class="text-xs text-slate-500 block mb-1">
                样本组B (逗号分隔)
              </label>
              <span id="group2-help" class="sr-only">请输入用逗号分隔的数值，例如 4.6,4.2,4.9</span>
              <textarea
                id="group2-input"
                v-model="group2Input"
                rows="2"
                aria-describedby="group2-help"
                class="w-full mt-1 bg-slate-900 border border-slate-600 rounded px-2 py-1 text-xs font-mono focus:outline-none focus:border-cyan-500 resize-none"
              ></textarea>
            </div>
          </div>
          <button
            @click="runTest"
            class="px-4 py-1.5 bg-purple-600 hover:bg-purple-500 rounded text-sm"
            aria-label="执行独立样本T检验，比较样本组A和样本组B的均值差异"
          >
            执行T检验
          </button>

          <div
            v-if="store.testResult"
            class="mt-3 grid grid-cols-2 md:grid-cols-4 gap-3 text-sm"
            role="region"
            aria-label="T检验结果"
          >
            <div class="bg-slate-900 rounded p-2 text-center">
              <div class="text-xs text-slate-500 mb-1">统计量 t</div>
              <div class="text-cyan-400 font-bold font-mono" aria-label="t统计量值">{{ store.testResult.statistic }}</div>
            </div>
            <div class="bg-slate-900 rounded p-2 text-center">
              <div class="text-xs text-slate-500 mb-1">p 值</div>
              <div
                class="font-bold font-mono"
                :class="store.testResult.significant ? 'text-red-400' : 'text-green-400'"
                :aria-label="`p值为${store.testResult.pValue}，${store.testResult.significant ? '具有统计显著性' : '不具有统计显著性'}`"
              >
                {{ store.testResult.pValue }}
              </div>
            </div>
            <div class="bg-slate-900 rounded p-2 text-center">
              <div class="text-xs text-slate-500 mb-1">自由度 df</div>
              <div class="text-slate-300 font-mono" aria-label="自由度">{{ store.testResult.df }}</div>
            </div>
            <div class="bg-slate-900 rounded p-2 text-center">
              <div class="text-xs text-slate-500 mb-1">显著性</div>
              <div
                class="text-xs font-bold"
                :class="store.testResult.significant ? 'text-red-400' : 'text-green-400'"
                aria-live="polite"
              >
                <span class="sr-only">{{ store.testResult.significant ? '结果显著' : '结果不显著' }}</span>
                <span aria-hidden="true">{{ store.testResult.significant ? '显著(p<0.05)' : '不显著' }}</span>
              </div>
            </div>
          </div>
        </section>
      </div>
    </main>
  </div>
</template>

<script setup lang="ts">
import { ref, watch, onMounted, computed, nextTick } from 'vue'
import * as echarts from 'echarts'
import { useMCStore, SCENARIOS } from './store/mc'

const store = useMCStore()
const convergenceRef = ref<HTMLDivElement | null>(null)
const histogramRef = ref<HTMLDivElement | null>(null)
const group1Input = ref('5.1,4.8,5.3,4.9,5.2,5.0,4.7,5.1,5.4,4.8')
const group2Input = ref('4.6,4.2,4.9,4.3,4.5,4.7,4.4,4.8,4.1,4.6')
const statusAnnouncement = ref('')
const resultAnnouncement = ref('')
let convChart: echarts.ECharts | null = null
let histChart: echarts.ECharts | null = null

function selectScenario(s: typeof SCENARIOS[0]) {
  store.setScenario(s)
  statusAnnouncement.value = `已选择场景：${s.name}，${s.description}`
  handleRunSimulation()
}

function focusScenarioByOffset(_event: KeyboardEvent, currentId: string, offset: number) {
  const listbox = document.querySelector('[role="listbox"]')
  if (!listbox) return
  const options = listbox.querySelectorAll<HTMLElement>('[role="option"]')
  const currentIdx = SCENARIOS.findIndex(s => s.id === currentId)
  if (currentIdx === -1) return
  const nextIdx = (currentIdx + offset + SCENARIOS.length) % SCENARIOS.length
  nextTick(() => {
    options[nextIdx]?.focus()
  })
}

function handleRunSimulation() {
  statusAnnouncement.value = `正在执行${store.currentScenario.name}模拟，迭代次数${store.iterations}次`
  store.runSimulation()
}

const convergenceChartAlt = computed(() => {
  if (!store.result) {
    return '收敛过程图表，等待模拟运行后显示数据'
  }
  const r = store.result
  let alt = `收敛过程图，展示${r.iterations}次迭代的收敛趋势。当前估算值为${r.estimate.toFixed(6)}`
  if (r.trueValue !== undefined) {
    alt += `，真实值为${r.trueValue.toFixed(6)}`
  }
  if (r.error !== undefined) {
    alt += `，误差为${r.error.toFixed(6)}`
  }
  return alt
})

const histogramChartAlt = computed(() => {
  if (!store.result) {
    return '样本分布直方图，等待模拟运行后显示数据'
  }
  const samples = store.result.samples
  const mn = Math.min(...samples).toFixed(4)
  const mx = Math.max(...samples).toFixed(4)
  const avg = (samples.reduce((a, b) => a + b, 0) / samples.length).toFixed(4)
  return `样本分布直方图，共${samples.length}个样本。最小值${mn}，最大值${mx}，平均值${avg}`
})

function initCharts() {
  if (convergenceRef.value) convChart = echarts.init(convergenceRef.value, 'dark')
  if (histogramRef.value) histChart = echarts.init(histogramRef.value, 'dark')
}

function updateCharts() {
  if (convChart && store.convergenceData.length > 0) {
    convChart.setOption({
      backgroundColor: '#0f172a',
      grid: { top: 20, bottom: 35, left: 65, right: 20 },
      xAxis: { type: 'value', axisLabel: { color: '#94a3b8', fontSize: 10 } },
      yAxis: { type: 'value', axisLabel: { color: '#94a3b8', fontSize: 10 } },
      series: [{ type: 'line', data: store.convergenceData, smooth: true, lineStyle: { color: '#06b6d4', width: 2 }, areaStyle: { color: 'rgba(6,182,212,0.1)' }, symbol: 'none' }],
      tooltip: { trigger: 'axis', backgroundColor: '#1e293b', borderColor: '#475569' }
    })
  }
  if (histChart && store.histogramData.xAxis.length > 0) {
    histChart.setOption({
      backgroundColor: '#0f172a',
      grid: { top: 15, bottom: 40, left: 55, right: 15 },
      xAxis: { type: 'category', data: store.histogramData.xAxis, axisLabel: { color: '#94a3b8', fontSize: 9, rotate: 30 } },
      yAxis: { type: 'value', axisLabel: { color: '#94a3b8', fontSize: 10 } },
      series: [{ type: 'bar', data: store.histogramData.data, itemStyle: { color: '#8b5cf6' } }],
      tooltip: { trigger: 'axis', backgroundColor: '#1e293b', borderColor: '#475569' }
    })
  }
}

function runTest() {
  const g1 = group1Input.value.split(',').map(s => parseFloat(s.trim())).filter(n => !isNaN(n))
  const g2 = group2Input.value.split(',').map(s => parseFloat(s.trim())).filter(n => !isNaN(n))
  if (g1.length < 2 || g2.length < 2) {
    statusAnnouncement.value = '请输入至少两个有效数值，每组样本需要两个以上数据点'
    return
  }
  statusAnnouncement.value = `正在执行T检验，样本组A ${g1.length}个数据，样本组B ${g2.length}个数据`
  store.runTest(g1, g2)
}

watch(() => store.result, (newResult) => {
  if (newResult) {
    let msg = `模拟完成：${newResult.scenario}，迭代${newResult.iterations}次，估算值${newResult.estimate.toFixed(6)}`
    if (newResult.trueValue !== undefined) {
      msg += `，真实值${newResult.trueValue.toFixed(6)}`
    }
    if (newResult.error !== undefined) {
      msg += `，误差${newResult.error.toFixed(6)}`
    }
    resultAnnouncement.value = msg
  }
}, { deep: true })

watch(() => store.testResult, (tr) => {
  if (tr) {
    resultAnnouncement.value = `T检验完成：t=${tr.statistic}，p=${tr.pValue}，自由度${tr.df}，${tr.significant ? '结果显著p小于0.05' : '结果不显著'}`
  }
}, { deep: true })

onMounted(() => { initCharts(); handleRunSimulation() })
</script>
