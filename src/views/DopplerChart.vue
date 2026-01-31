<template>
  <div class="chart-container">
    <div class="header">
      <div class="title-group">
        <h2>卫星多普勒分析系统</h2>
        <span class="badge" v-if="loading">LOADING...</span>
        <span class="badge error" v-if="errorMsg">{{ errorMsg }}</span>
      </div>
      
      <div class="search-bar">
        <div class="input-group">
          <label>Start:</label>
          <input type="datetime-local" v-model="searchStart" step="1">
        </div>
        <div class="input-group">
          <label>End:</label>
          <input type="datetime-local" v-model="searchEnd" step="1">
        </div>
        
        <button @click="handleManualSearch" :disabled="loading" class="btn-search">
          查询范围
        </button>
        
        <div class="quick-actions">
          <button @click="setRange(3)">近3天</button>
          <button @click="setRange(7)">近7天</button>
          <button @click="setRange(15)">近15天</button>
          <button @click="setRange(30)">近30天</button>
          <button @click="setRange(60)">近60天</button>
          <button @click="setRange(90)">近90天</button>
        </div>
      </div>
    </div>

    <div class="meta-info">
      <span>当前视窗点数: <strong>{{ currentPoints }}</strong></span>
      <span>后端压缩阈值: <strong>2000</strong></span>
      <span class="hint">(LTTB 动态降采样已激活)</span>
    </div>

    <div ref="chartRef" class="chart-box"></div>
  </div>
</template>

<script setup>
import { ref, onMounted, shallowRef } from 'vue';
import * as echarts from 'echarts';
import axios from 'axios';

// --- 1. 状态定义 ---
const chartRef = ref(null);
const chartInstance = shallowRef(null); // 使用 shallowRef 优化 ECharts 性能
const loading = ref(false);
const errorMsg = ref('');
const currentPoints = ref(0);

// 时间输入框绑定
const searchStart = ref('');
const searchEnd = ref('');

// 全局时间范围锁 (修复缩放无法还原的关键)
const globalStartTs = ref(null);
const globalEndTs = ref(null);

const API_URL = '/api/spectrum/GetSpectrumData';

// --- 2. 辅助函数 ---
// 格式化 Date 对象为 input type="datetime-local" 需要的格式 (yyyy-MM-ddTHH:mm:ss)
const formatDateForInput = (date) => {
  const pad = (n) => (n < 10 ? '0' + n : n);
  return (
    date.getFullYear() +
    '-' +
    pad(date.getMonth() + 1) +
    '-' +
    pad(date.getDate()) +
    'T' +
    pad(date.getHours()) +
    ':' +
    pad(date.getMinutes()) +
    ':' +
    pad(date.getSeconds())
  );
};

// --- 3. 初始化图表 ---
const initChart = () => {
  if (!chartRef.value) return;
  
  chartInstance.value = echarts.init(chartRef.value, 'dark');

  const option = {
    backgroundColor: '#0b1221',
    animation: false, // 大数据量下建议关闭动画以提升性能
    tooltip: {
      trigger: 'axis',
      axisPointer: { type: 'cross' },
      formatter: (params) => {
        if (!params[0]) return '';
        const date = new Date(params[0].value[0]);
        const val = params[0].value[1];
        return `${date.toLocaleString()}<br/>Doppler: <b>${val} Hz</b>`;
      }
    },
    grid: { top: 40, bottom: 40, left: 60, right: 20 },
    xAxis: {
      type: 'time',
      boundaryGap: false,
      // 注意：这里不设 min/max，由 handleManualSearch 动态设置
      axisLine: { lineStyle: { color: '#4a657a' } },
      splitLine: { show: false }
    },
    yAxis: {
      type: 'value',
      name: 'Shift (Hz)',
      min: -6000,
      max: 6000,
      splitLine: { lineStyle: { color: '#1f2d40', type: 'dashed' } }
    },
    // 缩放组件配置
    dataZoom: [
      {
        type: 'slider',
        show: true,
        bottom: 5,
        height: 20,
        borderColor: '#4a657a',
        fillerColor: 'rgba(0, 242, 255, 0.2)'
      },
      {
        type: 'inside' // 支持鼠标滚轮缩放
      }
    ],
    series: [
      {
        name: 'Doppler',
        type: 'line',
        // 关键性能配置
        showSymbol: false, 
        smooth: true, // LTTB后点少，开启平滑视觉更好
        data: [], 
        itemStyle: { color: '#00f2ff' },
        lineStyle: { width: 1 },
        areaStyle: {
          color: new echarts.graphic.LinearGradient(0, 0, 0, 1, [
            { offset: 0, color: 'rgba(0, 242, 255, 0.3)' },
            { offset: 1, color: 'rgba(0, 242, 255, 0.0)' }
          ])
        }
      }
    ]
  };

  chartInstance.value.setOption(option);

  // 监听缩放事件 (防抖处理)
  let zoomTimer = null;
  chartInstance.value.on('dataZoom', () => {
    if (zoomTimer) clearTimeout(zoomTimer);
    zoomTimer = setTimeout(handleChartZoom, 500); // 停止操作 500ms 后才请求
  });

  // 默认加载近3天
  setRange(3);
};

// --- 4. 业务逻辑：手动查询 (宏观) ---
const handleManualSearch = async () => {
  if (!searchStart.value || !searchEnd.value) return;
  
  const startTs = new Date(searchStart.value).getTime();
  const endTs = new Date(searchEnd.value).getTime();

  if (isNaN(startTs) || isNaN(endTs)) {
    errorMsg.value = "时间格式无效";
    return;
  }

  // 【核心修复】：锁定全局坐标系范围
  globalStartTs.value = startTs;
  globalEndTs.value = endTs;

  // 在加载数据前，强制设置 X 轴的 min 和 max
  // 并重置 dataZoom 到 0-100% (看全貌)
  chartInstance.value.setOption({
    xAxis: {
      min: startTs,
      max: endTs
    },
    dataZoom: [
      { start: 0, end: 100 },
      { start: 0, end: 100 }
    ]
  });

  // 加载该范围的“概览”数据
  await loadData(startTs, endTs);
};

// --- 5. 业务逻辑：图表缩放 (微观) ---
const handleChartZoom = () => {
  const option = chartInstance.value.getOption();
  
  // 获取当前视窗对应的时间戳范围
  // 注意：在 type: 'time' 轴中，startValue/endValue 直接就是时间戳
  const startTs = option.dataZoom[0].startValue;
  const endTs = option.dataZoom[0].endValue;

  // 如果当前视窗无效，或视窗几乎等于全局范围（没必要重复加载），则跳过
  // 这里做一个简单的判断：如果 start/end 存在且有效
  if (startTs != null && endTs != null) {
    loadData(startTs, endTs);
  }
};

// --- 6. 核心数据请求 ---
const loadData = async (startTs, endTs) => {
  try {
    loading.value = true;
    errorMsg.value = '';

    // 向后端请求数据
    // 逻辑：无论请求范围是 30天 还是 1小时，threshold 始终限制点数为 2000
    // 这保证了前端渲染永远流畅
    const res = await axios.get(API_URL, {
      params: {
        startTime: Math.floor(startTs),
        endTime: Math.floor(endTs),
        threshold: 2000 
      }
    });

    // 更新状态
    currentPoints.value = res.data.count;

    // 【重要】：只更新 series.data，绝对不要碰 xAxis 的 min/max
    // ECharts 会自动将这批新数据放置在已锁定的 X 轴上的正确位置
    chartInstance.value.setOption({
      series: [{
        data: res.data.data
      }]
    });

  } catch (err) {
    console.error(err);
    errorMsg.value = '数据加载失败: ' + (err.message || '未知错误');
  } finally {
    loading.value = false;
  }
};

// --- 7. 快捷设置 ---
const setRange = (days) => {
  const end = new Date();
  const start = new Date();
  start.setDate(end.getDate() - days);

  // 更新输入框
  searchStart.value = formatDateForInput(start);
  searchEnd.value = formatDateForInput(end);

  // 触发查询
  handleManualSearch();
};

// --- 生命周期 ---
onMounted(() => {
  initChart();
  window.addEventListener('resize', () => chartInstance.value?.resize());
});
</script>

<style scoped>
/* 布局样式 */
.chart-container {
  height: 100vh;
  display: flex;
  flex-direction: column;
  background: #080f1a;
  color: #fff;
  padding: 20px;
  box-sizing: border-box;
  font-family: 'Segoe UI', sans-serif;
}

.header {
  border-bottom: 1px solid #1f2d40;
  padding-bottom: 15px;
  margin-bottom: 10px;
}

.title-group {
  display: flex;
  align-items: center;
  margin-bottom: 15px;
}

h2 { margin: 0; font-size: 1.4rem; letter-spacing: 1px; color: #e0e6ed; }

/* 搜索栏样式 */
.search-bar {
  display: flex;
  flex-wrap: wrap;
  gap: 15px;
  align-items: center;
  background: #0f1826;
  padding: 10px;
  border-radius: 4px;
}

.input-group {
  display: flex;
  align-items: center;
  gap: 8px;
  font-size: 13px;
  color: #889bb3;
}

input[type="datetime-local"] {
  background: #162233;
  border: 1px solid #4a657a;
  color: #00f2ff;
  padding: 4px 8px;
  border-radius: 3px;
  outline: none;
  font-family: monospace;
}

/* 按钮样式 */
button {
  background: transparent;
  border: 1px solid #4a657a;
  color: #c0c6d1;
  padding: 5px 12px;
  border-radius: 3px;
  cursor: pointer;
  font-size: 13px;
  transition: all 0.2s;
}

button:hover { border-color: #00f2ff; color: #00f2ff; }
button:disabled { opacity: 0.5; cursor: not-allowed; }

.btn-search {
  background: rgba(0, 242, 255, 0.15);
  border-color: #00f2ff;
  color: #00f2ff;
  font-weight: 600;
  padding: 5px 20px;
}

.quick-actions { display: flex; gap: 8px; margin-left: auto; }

/* 信息栏与图表 */
.meta-info {
  text-align: right;
  font-size: 12px;
  color: #58697a;
  margin-bottom: 5px;
}
.meta-info strong { color: #00f2ff; margin: 0 5px; }
.hint { font-style: italic; margin-left: 10px; }

.chart-box {
  flex: 1;
  width: 100%;
  border: 1px solid #1f2d40;
  background: #0b1221;
}

/* Badge 样式 */
.badge {
  background: #00f2ff;
  color: #000;
  font-size: 10px;
  font-weight: bold;
  padding: 2px 6px;
  border-radius: 2px;
  margin-left: 15px;
}
.badge.error { background: #ff4d4f; color: white; }
</style>