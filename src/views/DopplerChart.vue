<template>
  <div class="chart-container">
    <div class="header">
      <div class="title-group">
        <h2>🛰️ 卫星多普勒实时监控中心</h2>
        <span class="badge live" v-if="realtimeStatus === 'connected'">● LIVE SIGNAL</span>
        <span class="badge error" v-else-if="realtimeStatus === 'error'">● CONNECTION ERROR</span>
        <span class="badge" v-else>● CONNECTING...</span>
      </div>

      <div class="control-bar">
        <div class="control-item">
          <label>目标卫星:</label>
          <select v-model="selectedSat" @change="handleSatChange" :disabled="loading">
            <option v-for="sat in satelliteList" :key="sat" :value="sat">{{ sat }}</option>
          </select>
        </div>

        <div class="control-item">
          <label>监控周期:</label>
          <div class="btn-group">
            <button v-for="days in [3, 7, 15, 30, 60, 90]" :key="days" :class="{ active: currentRangeDays === days }"
              @click="setRange(days)">
              近{{ days }}天
            </button>
          </div>
        </div>
      </div>
    </div>

    <div class="meta-info">
      <span>正在接收: <strong>{{ selectedSat }}</strong></span>
      <span>当前缓存点数: <strong>{{ currentPoints }}</strong></span>
      <span v-if="isZooming" class="warn-text">⚠️ 缩放浏览中 (自动滚屏已暂停)</span>
    </div>

    <div ref="chartRef" class="chart-box"></div>
  </div>
</template>

<script setup>
import { ref, onMounted, onUnmounted, shallowRef } from 'vue';
import * as echarts from 'echarts';
import axios from 'axios';
import { HubConnectionBuilder } from '@microsoft/signalr';

// --- 1. 配置常量 ---
// 使用相对路径，触发 vite.config.ts 中的 proxy 规则
// /api/Spectrum -> http://localhost:5000/Spectrum
const API_BASE = '/api/Spectrum';
const HUB_URL = '/api/spectrumHub';

// --- 2. 状态定义 ---
const chartRef = ref(null);
const chartInstance = shallowRef(null);
const loading = ref(false);
const satelliteList = ref(['SAT-A-L-V', 'SAT-A-S-H', 'SAT-B-X-V']);
const selectedSat = ref('SAT-A-L-V');
const currentRangeDays = ref(3);
const currentPoints = ref(0);

// 实时系统状态
const realtimeStatus = ref('disconnected'); // disconnected, connected, error
const isZooming = ref(false); // 是否处于用户交互/缩放状态
let connection = null;

// 全局时间锁 (用于固定 X 轴的总长度)
const globalStartTs = ref(null);
const globalEndTs = ref(null);

// --- 3. SignalR 连接与实时处理 ---
const initSignalR = async () => {
  connection = new HubConnectionBuilder()
    .withUrl(HUB_URL)
    .withAutomaticReconnect() // 自动重连
    .build();

  // 监听后端推送的新数据
  connection.on("ReceiveNewData", (satId, timestamp, value) => {
    // 只有当推送的数据属于当前选中的卫星时，才更新图表
    if (satId === selectedSat.value) {
      handleRealtimeData(timestamp, value);
    }
  });
debugger
  try {
    await connection.start();
    realtimeStatus.value = 'connected';
    console.log("SignalR Connected");
  } catch (err) {
    console.error("SignalR Connection Error", err);
    realtimeStatus.value = 'error';
  }
};

const handleRealtimeData = (timestamp, value) => {
  if (!chartInstance.value) return;

  const option = chartInstance.value.getOption();
  // 获取当前 Series 的数据副本
  const currentData = option.series[0].data;

  // 1. 追加新数据
  currentData.push([timestamp, value]);
  currentPoints.value = currentData.length;

  // 2. 更新图表数据
  chartInstance.value.setOption({
    series: [{ data: currentData }]
  });

  // 3. 处理视图滚动
  // 如果用户不在缩放查看历史，则自动更新 X 轴的最大值，实现“向右滚动”效果
  if (!isZooming.value) {
    globalEndTs.value = timestamp; // 更新全局结束时间
    chartInstance.value.setOption({
      xAxis: {
        min: globalStartTs.value, // 起点固定 (或者也可以随之移动，看需求)
        max: timestamp            // 终点跟随最新时间
      }
    });
  }
};

// --- 4. ECharts 初始化 ---
const initChart = () => {
  chartInstance.value = echarts.init(chartRef.value, 'dark');

  const option = {
    backgroundColor: '#0b1221',
    animation: false, // 实时高频刷新建议关闭动画
    tooltip: {
      trigger: 'axis',
      axisPointer: { type: 'cross' },
      formatter: (params) => {
        if (!params[0]) return '';
        const date = new Date(params[0].value[0]);
        return `${date.toLocaleString()}<br/>Freq: <b>${params[0].value[1].toFixed(2)} Hz</b>`;
      }
    },
    grid: { top: 50, bottom: 40, left: 60, right: 20 },
    // X 轴配置
    xAxis: {
      type: 'time',
      boundaryGap: false,
      axisLine: { lineStyle: { color: '#4a657a' } },
      splitLine: { show: false }
    },
    // Y 轴配置
    yAxis: {
      type: 'value',
      name: 'Doppler Shift (Hz)',
      min: -6000,
      max: 6000,
      splitLine: { lineStyle: { color: '#1f2d40', type: 'dashed' } }
    },
    // 缩放组件
    dataZoom: [
      { type: 'slider', show: true, bottom: 5, height: 20 },
      { type: 'inside' } // 支持鼠标滚轮
    ],
    series: [{
      name: 'Doppler',
      type: 'line',
      showSymbol: false, // 不显示小圆点，提升性能
      smooth: true,      // 开启平滑
      data: [],          // 初始为空
      itemStyle: { color: '#00f2ff' },
      lineStyle: { width: 1 },
      areaStyle: {
        opacity: 0.2,
        color: new echarts.graphic.LinearGradient(0, 0, 0, 1, [
          { offset: 0, color: '#00f2ff' },
          { offset: 1, color: 'transparent' }
        ])
      }
    }]
  };

  chartInstance.value.setOption(option);

  // 监听缩放事件，判断用户是否处于 "Zoom 模式"
  chartInstance.value.on('dataZoom', (evt) => {
    const opt = chartInstance.value.getOption();
    const start = opt.dataZoom[0].start;
    const end = opt.dataZoom[0].end;

    // 如果 end < 99.5，说明用户在查看历史数据，而不是最新的最右侧数据
    // 此时标记 isZooming = true，暂停自动滚屏
    isZooming.value = end < 99.5;
  });
};

// --- 5. 业务逻辑 ---

const handleSatChange = () => {
  // 切换卫星时，重置数据并重新加载
  setRange(currentRangeDays.value);
};

const setRange = async (days) => {
  if (loading.value) return;

  currentRangeDays.value = days;
  loading.value = true;
  isZooming.value = false; // 重置缩放状态，回到 Live 模式

  // 计算时间范围
  const end = new Date().getTime();
  const start = end - (days * 24 * 3600 * 1000);

  // 锁定全局 X 轴范围
  globalStartTs.value = start;
  globalEndTs.value = end;

  // 1. 先设置坐标轴范围，并重置缩放条
  chartInstance.value.setOption({
    xAxis: { min: start, max: end },
    dataZoom: [{ start: 0, end: 100 }, { start: 0, end: 100 }]
  });

  try {
    // 2. 请求历史数据 (LTTB 压缩版)
    // 路径: /api/Spectrum/data
    const res = await axios.get(`${API_BASE}/data`, {
      params: {
        satelliteId: selectedSat.value,
        startTime: start,
        endTime: end,
        threshold: 2000 // 这里的 2000 是告诉后端：无论几天数据，给我压缩成2000个点
      }
    });

    const historicalData = res.data.data;
    currentPoints.value = historicalData.length;

    // 3. 全量更新 Series
    chartInstance.value.setOption({
      series: [{ data: historicalData }]
    });

  } catch (err) {
    console.error("加载历史数据失败", err);
    // 这里可以加一个 UI 提示
  } finally {
    loading.value = false;
  }
};

// --- 6. 生命周期管理 ---
onMounted(() => {
  initChart();     // 1. 初始化空图表
  initSignalR();   // 2. 连接 WebSocket
  setRange(3);     // 3. 加载默认历史数据

  window.addEventListener('resize', () => chartInstance.value?.resize());
});

onUnmounted(() => {
  if (connection) connection.stop();
  chartInstance.value?.dispose();
});
</script>

<style scoped>
/* 保持原有样式，深色科技风 */
.chart-container {
  height: 100vh;
  background: #080f1a;
  color: white;
  padding: 20px;
  display: flex;
  flex-direction: column;
  box-sizing: border-box;
}

.header {
  display: flex;
  justify-content: space-between;
  align-items: flex-start;
  margin-bottom: 15px;
  border-bottom: 1px solid #1f2d40;
  padding-bottom: 15px;
}

.title-group h2 {
  margin: 0;
  font-size: 1.5rem;
  color: #e0e6ed;
  margin-bottom: 5px;
}

.control-bar {
  display: flex;
  gap: 20px;
  align-items: flex-end;
}

.control-item {
  display: flex;
  flex-direction: column;
  gap: 5px;
}

.control-item label {
  font-size: 12px;
  color: #889bb3;
}

select {
  background: #162233;
  color: #00f2ff;
  border: 1px solid #4a657a;
  padding: 5px 10px;
  border-radius: 4px;
  outline: none;
  font-weight: bold;
  cursor: pointer;
}

.btn-group {
  display: flex;
  border: 1px solid #4a657a;
  border-radius: 4px;
  overflow: hidden;
}

.btn-group button {
  background: #162233;
  border: none;
  color: #c0c6d1;
  padding: 6px 12px;
  cursor: pointer;
  border-right: 1px solid #4a657a;
  transition: 0.2s;
  font-size: 13px;
}

.btn-group button:last-child {
  border-right: none;
}

.btn-group button.active {
  background: #00f2ff;
  color: #000;
  font-weight: bold;
}

.btn-group button:hover:not(.active) {
  background: rgba(0, 242, 255, 0.1);
}

.badge {
  font-size: 10px;
  padding: 2px 6px;
  border-radius: 4px;
  font-weight: bold;
  display: inline-block;
  background: #555;
  color: #ccc;
}

.badge.live {
  background: #00ff6a;
  color: #000;
  box-shadow: 0 0 8px rgba(0, 255, 106, 0.4);
}

.badge.error {
  background: #ff4d4f;
  color: white;
}

.meta-info {
  text-align: right;
  font-size: 12px;
  color: #58697a;
  margin-bottom: 5px;
  display: flex;
  gap: 15px;
  justify-content: flex-end;
}

.meta-info strong {
  color: #00f2ff;
}

.warn-text {
  color: #ff9900;
  animation: blink 2s infinite;
}

.chart-box {
  flex: 1;
  border: 1px solid #1f2d40;
  background: #0b1221;
  border-radius: 4px;
}

@keyframes blink {

  0%,
  100% {
    opacity: 1;
  }

  50% {
    opacity: 0.5;
  }
}
</style>