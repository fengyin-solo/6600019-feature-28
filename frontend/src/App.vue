<template>
  <div class="flex h-screen">
    <!-- Sidebar -->
    <div class="w-72 bg-gray-900 p-4 flex flex-col gap-3 border-r border-gray-800 overflow-y-auto">
      <h1 class="text-lg font-bold text-cyan-400">地震波形 P/S 波分析</h1>

      <div>
        <label class="block bg-cyan-500 text-black text-center py-2 rounded cursor-pointer hover:bg-cyan-400 text-sm font-medium">
          上传 SAC/miniSEED
          <input type="file" @change="onUpload" class="hidden" />
        </label>
      </div>
      <button @click="store.loadMockData()" class="bg-gray-800 py-2 rounded text-sm hover:bg-gray-700">
        加载模拟数据
      </button>

      <!-- STA/LTA Parameters -->
      <div class="bg-gray-800 rounded-xl p-3 space-y-2">
        <h3 class="text-cyan-300 font-bold text-sm">STA/LTA 参数</h3>
        <div>
          <label class="text-gray-400 text-xs">STA 窗口: {{ store.staWindow.toFixed(1) }}s</label>
          <input type="range" v-model.number="store.staWindow" min="0.5" max="5" step="0.1" class="w-full" />
        </div>
        <div>
          <label class="text-gray-400 text-xs">LTA 窗口: {{ store.ltaWindow.toFixed(1) }}s</label>
          <input type="range" v-model.number="store.ltaWindow" min="5" max="30" step="0.5" class="w-full" />
        </div>
        <div>
          <label class="text-gray-400 text-xs">触发阈值: {{ store.threshold.toFixed(1) }}</label>
          <input type="range" v-model.number="store.threshold" min="1" max="10" step="0.5" class="w-full" />
        </div>
        <button @click="runPick" class="w-full bg-cyan-600 py-2 rounded text-sm hover:bg-cyan-500">
          运行自动拾取
        </button>
      </div>

      <!-- Picks -->
      <div class="bg-gray-800 rounded-xl p-3">
        <h3 class="text-cyan-300 font-bold text-sm mb-2">震相拾取结果</h3>
        <div v-for="p in store.picks" :key="p.id" class="flex justify-between bg-gray-700 rounded p-2 mb-1 text-sm">
          <span :class="p.type === 'P' ? 'text-red-400' : 'text-blue-400'">{{ p.type }} 波</span>
          <span>{{ p.time.toFixed(2) }}s</span>
          <span class="text-gray-400">{{ (p.confidence * 100).toFixed(0) }}%</span>
        </div>
        <div v-if="!store.picks.length" class="text-gray-600 text-xs">
          <span v-if="!store.waveform">请先上传或加载数据</span>
          <span v-else-if="!store.hasRunPicking">点击"运行自动拾取"开始检测</span>
          <span v-else>当前参数未拾取到震相，请调整后重试</span>
        </div>
      </div>

      <!-- Stations -->
      <div class="bg-gray-800 rounded-xl p-3">
        <h3 class="text-cyan-300 font-bold text-sm mb-2">台站分布</h3>
        <div v-for="s in store.stations" :key="s.id"
          @click="store.selectedStation = s"
          class="bg-gray-700 rounded p-2 mb-1 text-sm cursor-pointer hover:bg-gray-600"
          :class="store.selectedStation?.id === s.id ? 'ring-1 ring-cyan-500' : ''">
          {{ s.name }} <span class="text-gray-400 text-xs">({{ s.latitude.toFixed(1) }}, {{ s.longitude.toFixed(1) }})</span>
        </div>
      </div>

      <!-- Events -->
      <div class="bg-gray-800 rounded-xl p-3">
        <h3 class="text-cyan-300 font-bold text-sm mb-2">地震事件目录</h3>
        <div v-for="e in store.events" :key="e.id" class="bg-gray-700 rounded p-2 mb-1 text-xs">
          M{{ e.magnitude }} {{ e.location }}
          <div class="text-gray-500">深度 {{ e.depth }}km | {{ e.originTime.slice(0, 16) }}</div>
        </div>
      </div>
    </div>

    <!-- Main: Waveform Charts -->
    <div class="flex-1 flex flex-col gap-2 p-4 overflow-y-auto">
      <WaveformChart v-if="store.waveform" />
      <div v-else class="flex-1 flex items-center justify-center">
        <div class="text-center max-w-sm">
          <div class="text-5xl mb-4">📊</div>
          <h2 class="text-gray-300 text-lg font-medium mb-2">暂无波形数据</h2>
          <p class="text-gray-500 text-sm mb-4">请先上传 SAC/miniSEED 文件或加载模拟数据以开始分析</p>
          <div class="flex gap-2 justify-center">
            <label class="bg-cyan-500 text-black px-4 py-2 rounded cursor-pointer hover:bg-cyan-400 text-sm font-medium">
              上传文件
              <input type="file" @change="onUpload" class="hidden" />
            </label>
            <button @click="store.loadMockData()" class="bg-gray-800 px-4 py-2 rounded text-sm hover:bg-gray-700">
              加载模拟数据
            </button>
          </div>
        </div>
      </div>
      <div v-if="store.waveform && !store.picks.length && !store.hasRunPicking" class="bg-gray-800/50 rounded-xl p-6 text-center border border-gray-700">
        <div class="text-3xl mb-3">⚙️</div>
        <h3 class="text-gray-300 text-base font-medium mb-2">波形已就绪，尚未运行拾取</h3>
        <p class="text-gray-500 text-sm mb-4">调整左侧 STA/LTA 参数后，点击"运行自动拾取"检测 P/S 波到达时间</p>
        <button @click="runPick" class="bg-cyan-600 px-5 py-2 rounded text-sm hover:bg-cyan-500">
          运行自动拾取
        </button>
      </div>
      <div v-if="store.waveform && !store.picks.length && store.hasRunPicking" class="bg-gray-800/50 rounded-xl p-6 text-center border border-gray-700">
        <div class="text-3xl mb-3">🔍</div>
        <h3 class="text-gray-300 text-base font-medium mb-2">未检测到震相</h3>
        <p class="text-gray-500 text-sm mb-4">当前参数下未拾取到 P/S 波，可尝试降低触发阈值或调整 STA/LTA 窗口后重新运行</p>
        <button @click="runPick" class="bg-cyan-600 px-5 py-2 rounded text-sm hover:bg-cyan-500">
          重新拾取
        </button>
      </div>
    </div>
  </div>
</template>

<script setup lang="ts">
import { useSeismicStore } from './store/seismic'
import WaveformChart from './components/WaveformChart.vue'

const store = useSeismicStore()

function onUpload(e: Event) {
  const file = (e.target as HTMLInputElement).files?.[0]
  if (file) store.uploadAndAnalyze(file)
}

function runPick() {
  store.picks = store.staLtaPicking()
  store.hasRunPicking = true
}
</script>
