<template>
    <div class="chart-container">
        <div class="header">
            <h2>卫星多普勒频移监控 (Satellite Doppler Monitor)</h2>
            <div class="status-bar">
                <span class="status-item">Status: <span class="live">● LIVE</span></span>
                <span class="status-item">Data Points: {{ totalPoints.toLocaleString() }}</span>
                <span class="status-item">Time Range: 3 Days</span>
            </div>
        </div>

        <div ref="chartRef" class="chart-box"></div>

        <div class="controls">
            <button @click="toggleSimulation">{{ isRunning ? '暂停接收' : '继续接收' }}</button>
            <button @click="resetView">重置视图 (查看3天全貌)</button>
        </div>
    </div>
</template>

<script setup>
import { ref, onMounted, onUnmounted, shallowRef } from 'vue';
import * as echarts from 'echarts';

// --- 1. 状态定义 ---
const chartRef = ref(null);
const chartInstance = shallowRef(null);
const isRunning = ref(true);
const totalPoints = ref(0);
let timer = null;

// 配置常量
const TOTAL_DAYS = 3;
const SAMPLE_RATE = 1; // 1秒1个点
const TOTAL_SECONDS = TOTAL_DAYS * 24 * 3600;

// --- 2. 核心：模拟卫星多普勒数据 (S-Curve) ---
// 真实的卫星多普勒是一个S形曲线：接近时频率变高(正偏移)，远离时频率变低(负偏移)
function generateDopplerData(startTime, count) {
    const data = [];
    let currentTime = startTime;

    // 模拟参数：卫星轨道周期约90分钟 (5400秒)
    const orbitPeriod = 5400;

    for (let i = 0; i < count; i++) {
        // 算出当前处于轨道的哪个位置
        const t = (i % orbitPeriod) / orbitPeriod;

        // 制造只有在"过顶"时才有信号的效果 (假设只有20%的时间能收到信号)
        // 这里的 Math.sin 模拟多普勒的S曲线变化
        let value = 0;

        // 模拟多普勒S曲线：使用tanh函数或者sin函数的一部分来模拟频移
        // 当 t 在 0.4 到 0.6 之间时（卫星过顶），产生剧烈的频移
        if (t > 0.4 && t < 0.6) {
            // 归一化时间 -1 到 1
            const normalizedT = (t - 0.5) * 10;
            // 核心多普勒公式模拟：-1 * frequency * tanh(t) + 噪声
            const baseSignal = -5000 * Math.tanh(normalizedT);
            const noise = (Math.random() - 0.5) * 200; // 加入随机噪声
            value = baseSignal + noise;
        } else {
            // 没有信号时，只有底噪
            value = (Math.random() - 0.5) * 50;
        }

        data.push({
            name: new Date(currentTime).toString(),
            value: [
                currentTime, // X轴：时间戳
                Math.floor(value) // Y轴：频移 Hz
            ]
        });

        currentTime += 1000; // 加1秒
    }
    return data;
}

// --- 3. 初始化图表 ---
const initChart = () => {
    if (!chartRef.value) return;

    chartInstance.value = echarts.init(chartRef.value, 'dark');

    // 生成初始的3天历史数据 (模拟历史回放)
    const now = new Date().getTime();
    const startTime = now - (TOTAL_SECONDS * 1000);
    const initialData = generateDopplerData(startTime, TOTAL_SECONDS);
    totalPoints.value = initialData.length;

    const option = {
        backgroundColor: '#0b1221', // 配合你图片的深色背景
        title: {
            show: false
        },
        tooltip: {
            trigger: 'axis',
            axisPointer: { type: 'cross' },
            formatter: (params) => {
                const date = new Date(params[0].value[0]);
                const freq = params[0].value[1];
                return `时间: ${date.toLocaleString()}<br/>频移: <span style="color:#00f2ff;font-weight:bold">${freq} Hz</span>`;
            }
        },
        grid: {
            top: '15%',
            left: '3%',
            right: '4%',
            bottom: '15%', // 留给滑动条的空间
            containLabel: true
        },
        // X轴配置
        xAxis: {
            type: 'time',
            boundaryGap: false,
            axisLine: { lineStyle: { color: '#4a657a' } },
            axisLabel: { color: '#aab2bd' },
            splitLine: { show: true, lineStyle: { color: '#1f2d40' } }
        },
        // Y轴配置
        yAxis: {
            type: 'value',
            name: 'Doppler Shift (Hz)',
            min: -6000,
            max: 6000,
            axisLine: { show: true, lineStyle: { color: '#4a657a' } },
            splitLine: { show: true, lineStyle: { color: '#1f2d40', type: 'dashed' } },
            axisLabel: { formatter: '{value} Hz' }
        },
        // --- 核心优化配置 ---
        dataZoom: [
            {
                type: 'slider', // 底部滑动条
                show: true,
                start: 95, // 默认显示最后5%的数据
                end: 100,
                handleSize: '100%',
                backgroundColor: '#162233',
                fillerColor: 'rgba(0, 242, 255, 0.2)',
                borderColor: '#4a657a'
            },
            {
                type: 'inside', // 支持鼠标滚轮缩放
                start: 95,
                end: 100
            }
        ],
        series: [
            {
                name: 'Doppler Shift',
                type: 'line',
                showSymbol: false, // 几十万个点时，必须关掉数据点的小圆圈，只显示线

                // --- 性能优化关键参数 ---
                sampling: 'lttb', // 开启 LTTB 降采样算法 (视觉无损压缩)
                large: true,      // 开启海量数据模式
                largeThreshold: 5000, // 超过5000个点就自动启用优化

                itemStyle: {
                    color: '#00f2ff' // 你的图片里的青色
                },
                lineStyle: {
                    width: 1,
                    color: new echarts.graphic.LinearGradient(0, 0, 1, 0, [
                        { offset: 0, color: '#0066ff' },
                        { offset: 1, color: '#00f2ff' }
                    ]),
                    shadowBlur: 5,
                    shadowColor: '#00f2ff'
                },
                areaStyle: {
                    color: new echarts.graphic.LinearGradient(0, 0, 0, 1, [
                        { offset: 0, color: 'rgba(0, 242, 255, 0.3)' },
                        { offset: 1, color: 'rgba(0, 242, 255, 0.01)' }
                    ])
                },
                data: initialData
            }
        ]
    };

    chartInstance.value.setOption(option);

    // 开启实时模拟
    startSimulation();
};

// --- 4. 实时数据模拟 ---
const startSimulation = () => {
    if (timer) clearInterval(timer);

    timer = setInterval(() => {
        if (!isRunning.value) return;

        // 生成当前这一秒的新数据
        const now = new Date().getTime();
        const newDataPoints = generateDopplerData(now, 1); // 生成1个点

        // 获取当前的数据
        // 注意：ECharts 5+ 推荐使用 appendData 但为了演示简单，这里演示全量更新的逻辑
        // 在生产环境处理超大数据流时，通常只保留最近 N 天，移除头部数据

        // 这里我们简单做一个模拟：只向 series[0] 追加数据
        // 实际项目中，应该维护一个 data 数组，shift() 旧数据，push() 新数据

        const option = chartInstance.value.getOption();
        const currentData = option.series[0].data;

        // 保持总长度不超过3天 (滑动窗口)
        if (currentData.length >= TOTAL_SECONDS) {
            currentData.shift();
        }
        currentData.push(newDataPoints[0]);
        totalPoints.value = currentData.length;

        chartInstance.value.setOption({
            series: [{
                data: currentData
            }]
        });
    }, 1000); // 1秒刷新一次
};

const toggleSimulation = () => {
    isRunning.value = !isRunning.value;
};

const resetView = () => {
    chartInstance.value.dispatchAction({
        type: 'dataZoom',
        start: 0,
        end: 100
    });
};

// 响应式布局
const handleResize = () => {
    chartInstance.value?.resize();
};

onMounted(() => {
    initChart();
    window.addEventListener('resize', handleResize);
});

onUnmounted(() => {
    clearInterval(timer);
    window.removeEventListener('resize', handleResize);
    chartInstance.value?.dispose();
});
</script>

<style scoped>
.chart-container {
    width: 100%;
    height: 100vh;
    background-color: #080f1a;
    color: #fff;
    display: flex;
    flex-direction: column;
    padding: 20px;
    box-sizing: border-box;
    font-family: 'Segoe UI', Tahoma, Geneva, Verdana, sans-serif;
}

.header {
    display: flex;
    justify-content: space-between;
    align-items: center;
    margin-bottom: 20px;
    border-bottom: 1px solid #1f2d40;
    padding-bottom: 10px;
}

h2 {
    margin: 0;
    color: #e0e6ed;
    font-size: 1.5rem;
    letter-spacing: 1px;
}

.status-bar {
    display: flex;
    gap: 20px;
    font-size: 0.9rem;
    color: #889bb3;
}

.live {
    color: #00ff6a;
    animation: blink 1s infinite;
}

.chart-box {
    flex: 1;
    /* 占满剩余空间 */
    width: 100%;
    border: 1px solid #1f2d40;
    border-radius: 4px;
    background-color: #0b1221;
}

.controls {
    margin-top: 15px;
    display: flex;
    gap: 10px;
}

button {
    background: transparent;
    border: 1px solid #00f2ff;
    color: #00f2ff;
    padding: 8px 16px;
    cursor: pointer;
    transition: all 0.3s;
    font-family: monospace;
}

button:hover {
    background: rgba(0, 242, 255, 0.1);
    box-shadow: 0 0 10px rgba(0, 242, 255, 0.5);
}

@keyframes blink {
    0% {
        opacity: 1;
    }

    50% {
        opacity: 0.5;
    }

    100% {
        opacity: 1;
    }
}
</style>