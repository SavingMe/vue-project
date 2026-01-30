<template>
    <div class="chart-container">
        <div class="header">
            <h2>卫星多普勒频移监控 (后端 LTTB 加速版)</h2>
            <div class="status-bar">
                <span class="status-item">Points: {{ currentPoints }} / Total: ~2.6M</span>
                <span class="status-item" v-if="loading">加载中...</span>
            </div>
        </div>
        <div ref="chartRef" class="chart-box"></div>
    </div>
</template>

<script setup>
import { ref, onMounted, shallowRef } from 'vue';
import * as echarts from 'echarts';
import axios from 'axios';

const chartRef = ref(null);
const chartInstance = shallowRef(null);
const loading = ref(false);
const currentPoints = ref(0);

// API 地址
const API_URL = '/api/Spectrum/GetSpectrumData';

// 初始化图表
const initChart = async () => {
    chartInstance.value = echarts.init(chartRef.value, 'dark');

    const option = {
        backgroundColor: '#0b1221',
        tooltip: { trigger: 'axis' },
        grid: { top: 50, bottom: 50, left: 50, right: 20 },
        xAxis: {
            type: 'time',
            boundaryGap: false
        },
        yAxis: {
            type: 'value',
            name: 'Hz',
            min: -6000,
            max: 6000
        },
        dataZoom: [
            { type: 'slider', show: true },
            { type: 'inside' }
        ],
        series: [{
            type: 'line',
            showSymbol: false,
            data: [], // 初始为空
            itemStyle: { color: '#00f2ff' },
            areaStyle: { opacity: 0.2 }
        }]
    };

    chartInstance.value.setOption(option);

    // 1. 首次加载：加载最近3天的数据概览 (后端会压缩成2000个点)
    const now = new Date().getTime();
    const threeDaysAgo = now - (3 * 24 * 3600 * 1000);
    await fetchData(threeDaysAgo, now);

    // 2. 监听缩放事件 (实现分级渲染/按需加载)
    // 加上防抖，避免拖动时请求过于频繁
    let timer = null;
    chartInstance.value.on('dataZoom', (params) => {
        if (timer) clearTimeout(timer);
        timer = setTimeout(() => {
            handleZoom();
        }, 500); // 停止缩放 500ms 后才请求数据
    });
};

// 获取当前视口的时间范围，并请求更高精度的数据
const handleZoom = () => {
    const option = chartInstance.value.getOption();
    // 获取当前 x 轴的时间范围
    const start = option.dataZoom[0].startValue;
    const end = option.dataZoom[0].endValue;

    // 如果 start 和 end 存在（有时初始化可能为空），则请求数据
    if (start && end) {
        fetchData(start, end);
    }
};

// 请求后端数据核心方法
const fetchData = async (start, end) => {
    try {
        loading.value = true;

        // threshold: 告诉后端，无论这一段时间里有多少真实数据，我只要 2000 个点用于显示
        // 2000 个点在 1920px 的屏幕上足够画出完美的曲线
        const res = await axios.get(API_URL, {
            params: {
                startTime: Math.floor(start),
                endTime: Math.floor(end),
                threshold: 2000
            }
        });

        const newData = res.data.data;
        currentPoints.value = res.data.count;

        // 更新图表
        chartInstance.value.setOption({
            series: [{
                data: newData
            }]
        });
    } catch (err) {
        console.error("加载数据失败", err);
    } finally {
        loading.value = false;
    }
};

onMounted(() => {
    initChart();
});
</script>

<style scoped>
/* 样式同上，省略 */
.chart-container {
    height: 100vh;
    background: #080f1a;
    color: white;
    padding: 20px;
}

.chart-box {
    height: 90%;
}
</style>