<template>
  <div class="chart-container">
    <div class="header">
      <div class="title-group">
        <h2>🛰️ 卫星多普勒实时监控中心</h2>
        <span class="badge live" v-if="realtimeStatus === 'connected'">● LIVE SIGNAL</span>
        <span class="badge error" v-else-if="realtimeStatus === 'error'">● ERROR</span>
        <span class="badge" v-else>● CONNECTING...</span>
      </div>

      <div class="control-bar">
        <div class="control-item">
          <label>目标卫星:</label>
          <select v-model="selectedSat" @change="handleSatChange" :disabled="loading">
            <option v-for="sat in satelliteList" :key="sat" :value="sat">{{ sat }}</option>
          </select>
        </div>

        <div class="control-item freq-group">
          <label>中心频率 (多选):</label>
          <div class="checkbox-wrapper">
            <label v-for="freq in availableFreqs" :key="freq" class="checkbox-label">
              <input type="checkbox" :value="freq" v-model="selectedFreqs" @change="handleFreqChange"
                :disabled="loading">
              <span class="freq-text">{{ freq }}</span>
            </label>
          </div>
        </div>

        <div class="control-item">
          <label>监控周期:</label>
          <div class="btn-group">
            <button v-for="days in [1, 3, 7, 15, 30]" :key="days" :class="{ active: currentRangeDays === days }"
              @click="setRange(days)">
              近{{ days }}天
            </button>
          </div>
        </div>
      </div>
    </div>

    <div class="meta-info">
      <span>卫星: <strong>{{ selectedSat }}</strong></span>
      <span v-for="freq in selectedFreqs" :key="freq" class="freq-tag">
        {{ freq }}
      </span>
      <span v-if="isZooming" class="warn-text">⚠️ 缩放中 (暂停滚屏)</span>
    </div>

    <div ref="chartRef" class="chart-box"></div>
  </div>
</template>

<script setup>
import { ref, onMounted, onUnmounted, shallowRef } from 'vue';
import * as echarts from 'echarts';
import axios from 'axios';
import { HubConnectionBuilder } from '@microsoft/signalr';

const API_BASE = '/api/Spectrum';
const HUB_URL = '/api/spectrumHub';

// --- 状态 ---
const chartRef = ref(null);
const chartInstance = shallowRef(null);
const loading = ref(false);

const satelliteList = ref([]);
const selectedSat = ref('');

// 频率相关状态
const availableFreqs = ref([]); // 当前卫星支持的所有频率
const selectedFreqs = ref([]);  // 用户选中的频率数组

const currentRangeDays = ref(3);
const realtimeStatus = ref('disconnected');
const isZooming = ref(false);
let connection = null;

const globalStartTs = ref(null);
const globalEndTs = ref(null);

// 定义一组霓虹配色，用于区分不同曲线
const lineColors = ['#00f2ff', '#ff00ee', '#ffee00', '#00ff6a', '#ff9900'];

// --- 1. SignalR ---
const initSignalR = async () => {
  connection = new HubConnectionBuilder()
    .withUrl(HUB_URL)
    .withAutomaticReconnect()
    .build();

  // 监听新签名: (satId, freq, timestamp, value)
  connection.on("ReceiveNewData", (satId, freq, timestamp, value) => {
    // 只有当: 
    // 1. 卫星匹配
    // 2. 该频率被用户选中
    // 才更新图表
    if (satId === selectedSat.value && selectedFreqs.value.includes(freq)) {
      handleRealtimeData(freq, timestamp, value);
    }
  });

  try {
    await connection.start();
    realtimeStatus.value = 'connected';
    console.log("SignalR Connected");
  } catch (err) {
    console.error("SignalR Connection Error", err);
    realtimeStatus.value = 'error';
  }
};

const handleRealtimeData = (freq, timestamp, value) => {
  if (!chartInstance.value) return;

  const option = chartInstance.value.getOption();

  // 安全检查
  if (!option.series) return;

  // 找到对应频率的 Series 索引
  const seriesIndex = option.series.findIndex(s => s.name === freq);

  if (seriesIndex !== -1) {
    const currentData = option.series[seriesIndex].data;
    currentData.push([timestamp, value]);

    // 局部更新 series
    chartInstance.value.setOption({
      series: [{
        name: freq,
        data: currentData
      }]
    });
  }

  // 滚屏逻辑
  if (!isZooming.value) {
    globalEndTs.value = timestamp;
    chartInstance.value.setOption({
      xAxis: { min: globalStartTs.value, max: timestamp }
    });
  }
};

// --- 2. 业务逻辑 ---

// 初始化卫星列表
const fetchSatellites = async () => {
  try {
    const res = await axios.get(`${API_BASE}/satellites`);
    satelliteList.value = res.data;
    if (satelliteList.value.length > 0) {
      selectedSat.value = satelliteList.value[0];
      await handleSatChange(); // 加载第一个卫星的配置
    }
  } catch (e) {
    console.error(e);
  }
}

// 获取当前卫星支持的频率
const fetchFrequencies = async () => {
  try {
    const res = await axios.get(`${API_BASE}/frequencies`, {
      params: { satelliteId: selectedSat.value }
    });
    availableFreqs.value = res.data;
    // 默认全选
    selectedFreqs.value = [...res.data];
  } catch (e) {
    console.error("获取频率失败", e);
  }
};

const handleSatChange = async () => {
  // 切换卫星 -> 重新获取频率 -> 默认全选 -> 重新加载数据
  await fetchFrequencies();
  await setRange(currentRangeDays.value);
};

const handleFreqChange = () => {
  // 勾选变化 -> 重新加载数据 
  setRange(currentRangeDays.value);
};

// 初始化图表
const initChart = () => {
  debugger
  chartInstance.value = echarts.init(chartRef.value, 'dark');
  const option = {
    backgroundColor: '#0b1221',
    animation: false,
    tooltip: {
      trigger: 'axis',
      axisPointer: { type: 'cross' },
      // 复杂的 tooltip，显示所有系列的值
      formatter: (params) => {
        if (!params.length) return '';
        const date = new Date(params[0].value[0]).toLocaleTimeString();
        let html = `<div>${date}</div>`;
        params.forEach(p => {
          html += `<div style="color:${p.color}">● ${p.seriesName}: <b>${p.value[1]} Hz</b></div>`;
        });
        return html;
      }
    },
    legend: { show: true, top: 5, textStyle: { color: '#fff' } }, // 显示图例
    grid: { top: 40, bottom: 40, left: 60, right: 20 },
    xAxis: {
      type: 'time',
      boundaryGap: false,
      axisLine: { lineStyle: { color: '#4a657a' } },
      splitLine: { show: false }
    },
    yAxis: {
      type: 'value',
      name: 'Hz',
      min: -6000, max: 6000,
      splitLine: { lineStyle: { color: '#1f2d40', type: 'dashed' } }
    },
    dataZoom: [
      { type: 'slider', show: true, bottom: 5, height: 20 },
      { type: 'inside' }
    ],
    series: [] // 动态生成
  };

  // 【DEBUG】: 打印看看 axis 是否存在，以及 data 是否为空
  console.log('Debug ECharts Option:', JSON.stringify(option));
  if (option.series && option.series.length > 0) {
    chartInstance.value.setOption(option);

  }


  chartInstance.value.on('dataZoom', (evt) => {
    const opt = chartInstance.value.getOption();
    const end = opt.dataZoom[0].end;
    isZooming.value = end < 99.5;
  });
};

const setRange = async (days) => {
  if (loading.value) return;
  currentRangeDays.value = days;
  loading.value = true;
  isZooming.value = false;

  const end = new Date().getTime();
  const start = end - (days * 24 * 3600 * 1000);
  globalStartTs.value = start;
  globalEndTs.value = end;

  // 重置坐标轴，清空 Series
  chartInstance.value.setOption({
    // 必须补全所有基础配置
    grid: { top: 40, bottom: 40, left: 60, right: 20 }, // 之前的样式
    tooltip: { trigger: 'axis' },
    xAxis: {
      type: 'time', // ✅ 必须加回来
      boundaryGap: false,
      min: start,
      max: end
    },
    yAxis: { // ✅ 必须加回来
      type: 'value',
      min: -6000,
      max: 6000
    },
    dataZoom: [{ start: 0, end: 100 }, { type: 'inside' }],
    series: []
  }, { notMerge: true }); // 这样才不会报错

  // 如果没有选频率，直接返回
  if (selectedFreqs.value.length === 0) {
    loading.value = false;
    return;
  }

  try {
    // 构造参数: 
    // 使用 URLSearchParams 手动序列化数组，兼容 ASP.NET Core 的 Model Binding
    const res = await axios.get(`${API_BASE}/data`, {
      params: {
        satelliteId: selectedSat.value,
        frequencies: selectedFreqs.value,
        startTime: start,
        endTime: end,
        threshold: 2000
      },
      paramsSerializer: params => {
        const p = new URLSearchParams();
        p.append("satelliteId", params.satelliteId);
        p.append("startTime", params.startTime);
        p.append("endTime", params.endTime);
        p.append("threshold", params.threshold);
        // 手动处理数组
        if (Array.isArray(params.frequencies)) {
          params.frequencies.forEach(f => p.append("frequencies", f));
        }
        return p.toString();
      }
    });

    const dataDict = res.data.data || {}; // { "2200MHz": [...], "8450MHz": [...] }

    // 构造多条 Series
    const newSeries = Object.keys(dataDict).map((freq, index) => ({
      name: freq,
      type: 'line',
      showSymbol: false,
      smooth: true,
      data: dataDict[freq],
      itemStyle: { color: lineColors[index % lineColors.length] }, // 轮询颜色
      lineStyle: { width: 1 },
      areaStyle: {
        opacity: 0.1,
        color: new echarts.graphic.LinearGradient(0, 0, 0, 1, [
          { offset: 0, color: lineColors[index % lineColors.length] },
          { offset: 1, color: 'transparent' }
        ])
      }
    }));

    chartInstance.value.setOption({
      series: newSeries
    });

  } catch (err) {
    console.error(err);
  } finally {
    loading.value = false;
  }
};

onMounted(async () => {
  initChart();
  initSignalR();
  await fetchSatellites(); // 获取卫星列表并触发加载
  window.addEventListener('resize', () => chartInstance.value?.resize());
});

onUnmounted(() => {
  if (connection) connection.stop();
  chartInstance.value?.dispose();
});
</script>

<style scoped>
.chart-container {
  height: 100vh;
  background: #080f1a;
  color: white;
  padding: 20px;
  display: flex;
  flex-direction: column;
}

.header {
  display: flex;
  justify-content: space-between;
  align-items: flex-start;
  border-bottom: 1px solid #1f2d40;
  padding-bottom: 15px;
  margin-bottom: 10px;
}

.title-group h2 {
  margin: 0;
  font-size: 1.5rem;
  color: #e0e6ed;
}

.control-bar {
  display: flex;
  gap: 20px;
  align-items: flex-end;
  flex-wrap: wrap;
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
  padding: 5px;
  border-radius: 4px;
  outline: none;
}

/* 复选框样式 */
.checkbox-wrapper {
  display: flex;
  gap: 10px;
  align-items: center;
  background: #162233;
  padding: 5px 10px;
  border-radius: 4px;
  border: 1px solid #4a657a;
  height: 26px;
}

.checkbox-label {
  display: flex;
  align-items: center;
  gap: 5px;
  cursor: pointer;
  user-select: none;
}

.freq-text {
  font-size: 13px;
  color: #c0c6d1;
}

input[type="checkbox"] {
  accent-color: #00f2ff;
}

.btn-group {
  display: flex;
  border: 1px solid #4a657a;
  border-radius: 4px;
}

.btn-group button {
  background: #162233;
  border: none;
  color: #c0c6d1;
  padding: 6px 12px;
  cursor: pointer;
  border-right: 1px solid #4a657a;
  transition: 0.2s;
}

.btn-group button:last-child {
  border-right: none;
}

.btn-group button.active {
  background: #00f2ff;
  color: #000;
  font-weight: bold;
}

.meta-info {
  display: flex;
  gap: 10px;
  justify-content: flex-end;
  font-size: 12px;
  margin-bottom: 5px;
  align-items: center;
}

.freq-tag {
  background: rgba(0, 242, 255, 0.1);
  color: #00f2ff;
  padding: 2px 6px;
  border-radius: 3px;
  border: 1px solid rgba(0, 242, 255, 0.3);
}

.warn-text {
  color: #ff9900;
  animation: blink 2s infinite;
}

.chart-box {
  flex: 1;
  border: 1px solid #1f2d40;
  background: #0b1221;
}

.badge {
  font-size: 10px;
  padding: 2px 6px;
  border-radius: 4px;
  background: #555;
}

.badge.live {
  background: #00ff6a;
  color: #000;
}

.badge.error {
  background: #ff4d4f;
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