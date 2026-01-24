<template>
  <div class="app-wrapper dark">
    <el-container class="main-layout">

      <el-header class="sys-header">
        <div class="brand">
          <el-icon class="logo-icon">
            <Dish />
          </el-icon> SAT-COMMAND 任务调度中心
        </div>
        <div class="sys-info">
          <el-tag type="success" effect="dark" round>矩阵状态: 正常</el-tag>
          <el-tag :type="receivers.filter(r => r.status === 'idle').length > 0 ? 'info' : 'warning'" effect="dark" round>
            接收机可用: {{receivers.filter(r => r.status === 'idle').length}}/{{ receivers.length }}
          </el-tag>
          <span class="time-text">UTC: {{ utcTime }}</span>
        </div>
      </el-header>

      <el-container class="content-body">

        <el-aside width="240px" class="left-panel">
          <div class="panel-title">天线阵列 ({{ antennas.length }})</div>
          <div class="antenna-list">
            <el-card v-for="ant in antennas" :key="ant.id" class="sci-fi-card antenna-card"
              :class="{ 'active-border': ant.status !== 'idle' }" shadow="hover">
              <div class="card-header-flex">
                <span class="ant-name">{{ ant.name }}</span>
                <el-tag size="small" :type="getBandColor(ant.band)">{{ ant.band }}-Band</el-tag>
              </div>
              <div class="ant-status" :class="getStatusColorClass(ant.status)">
                <div class="status-dot"></div>
                <span>{{ ant.statusText }}</span>
              </div>
              <div class="ant-params" v-if="ant.az">
                AZ: {{ ant.az }}° <span v-if="ant.el">EL: {{ ant.el }}°</span>
              </div>
            </el-card>
          </div>
        </el-aside>

        <el-main class="center-panel">
          <div class="topology-wrapper">
            <div class="panel-title-overlay">信号路由拓扑 (MATRIX ROUTING)</div>

            <svg class="connections-layer">
              <template v-for="(link, index) in activeLinks" :key="index">
                <line :x1="50" :y1="getAntennaY(link.antId)" :x2="400" :y2="200" class="link-line" />
                <line :x1="480" :y1="200" :x2="800" :y2="getReceiverY(link.rcvId)" class="link-line" />
              </template>
            </svg>

            <div class="topology-nodes">
              <div class="matrix-box">
                <div class="matrix-label">RF SWITCH<br>4 x 10</div>
                <div class="matrix-leds">
                  <span v-for="n in 4" :key="n" class="led blink"></span>
                </div>
              </div>

              <div class="receiver-pool">
                <div class="pool-label">RECEIVER POOL (10)</div>
                <div class="rcv-grid">
                  <div v-for="rcv in receivers" :key="rcv.id" class="rcv-item"
                    :class="{ 'is-busy': rcv.status === 'busy', 'is-selected': selectedRcvId === rcv.id }"
                    @click="selectReceiver(rcv.id)">
                    <span class="rcv-id">{{ rcv.name }}</span>
                    <span class="rcv-led" :class="rcv.status"></span>
                  </div>
                </div>
              </div>
            </div>
          </div>

          <div class="analysis-wrapper">
            <div class="panel-header-sm">
              <span>实时信号分析 - {{ selectedReceiver?.name || '未选择' }}</span>
              <el-tag size="small" type="warning" v-if="selectedReceiver?.target">TARGET: {{ selectedReceiver.target
                }}</el-tag>
            </div>
            <div class="spectrum-chart">
              <div class="wave-bg"></div>
              <div class="wave-line" v-if="selectedReceiver?.status === 'busy'"></div>
              <div class="empty-state" v-else>无信号输入</div>

              <transition name="el-zoom-in-center">
                <div v-if="hasAlert" class="alert-overlay">
                  <el-alert title="检测到特征信号: BPSK_Mod" type="error" description="匹配度 98% - 建议立即查证" show-icon
                    :closable="false" effect="dark" />
                </div>
              </transition>
            </div>
          </div>
        </el-main>

        <el-aside width="300px" class="right-panel">
          <div class="panel-header-flex">
            <span>任务调度队列</span>
            <el-button type="primary" size="small" icon="Plus" @click="openNewTaskDialog">新建任务</el-button>
          </div>

          <el-scrollbar>
            <div class="task-list">
              <div class="list-label">正在运行 ({{ runningTasks.length }})</div>
              <transition-group name="list">
                <el-card v-for="task in runningTasks" :key="task.id" class="sci-fi-card task-card"
                  :class="{ 'alert-mode': task.isAlert }" body-style="padding: 12px;">
                  <div class="task-row">
                    <span class="task-id">{{ task.id }}</span>
                    <span class="task-timer">{{ task.time }}</span>
                  </div>
                  <div class="task-title">{{ task.name }}</div>
                  <div class="task-resource">资源: {{ task.resourceName }}</div>

                  <div v-if="task.isAlert" class="alert-text">
                    <el-icon>
                      <Warning />
                    </el-icon> 触发特征告警
                  </div>

                  <el-progress :percentage="task.progress" :status="task.isAlert ? 'exception' : ''" :stroke-width="4"
                    :show-text="false" class="mt-2" />
                </el-card>
              </transition-group>

              <div class="list-label mt-4">等待队列</div>
              <el-card class="sci-fi-card task-card pending-card" shadow="never">
                <div class="task-row">
                  <span class="task-id">TASK-1026</span>
                  <el-tag size="small" type="info">待定</el-tag>
                </div>
                <div class="task-title">全频段扫描 (Ka)</div>
                <div class="task-resource">等待 ANT-04 维护结束</div>
              </el-card>
            </div>
          </el-scrollbar>
        </el-aside>
      </el-container>
    </el-container>

    <el-dialog v-model="dialogVisible" title="新建侦测调度任务" width="500px" class="custom-dialog"
      :close-on-click-modal="false">
      <el-form ref="taskFormRef" :model="newTaskForm" :rules="rules" label-width="100px" status-icon>

        <el-form-item label="任务名称" prop="name">
          <el-input v-model="newTaskForm.name" placeholder="例如: SAT-X 重点监视" />
        </el-form-item>

        <el-form-item label="目标卫星" prop="satellite">
          <el-select v-model="newTaskForm.satellite" placeholder="请选择目标卫星" style="width: 100%">
            <el-option label="SAT-ASIA-3 (通信)" value="SAT-ASIA-3" />
            <el-option label="SAT-GPS-05 (导航)" value="SAT-GPS-05" />
            <el-option label="SAT-UNKNOWN-X (不明)" value="SAT-UNKNOWN-X" />
          </el-select>
        </el-form-item>

        <el-form-item label="使用天线" prop="antennaId">
          <el-select v-model="newTaskForm.antennaId" placeholder="自动匹配 / 手动选择" style="width: 100%">
            <el-option v-for="ant in antennas" :key="ant.id"
              :label="`${ant.name} (${ant.band}-Band) - ${ant.status === 'idle' ? '空闲' : '忙碌'}`" :value="ant.id"
              :disabled="ant.status === 'maint'" />
          </el-select>
        </el-form-item>

        <el-row :gutter="20">
          <el-col :span="12">
            <el-form-item label="调度模式" prop="mode">
              <el-select v-model="newTaskForm.mode">
                <el-option label="单次执行" value="single" />
                <el-option label="周期轮询" value="loop" />
                <el-option label="持续监视" value="persist" />
              </el-select>
            </el-form-item>
          </el-col>
          <el-col :span="12">
            <el-form-item label="监视时长" prop="duration">
              <el-input-number v-model="newTaskForm.duration" :min="1" :max="120" controls-position="right"
                style="width: 100%">
                <template #suffix>分钟</template>
              </el-input-number>
            </el-form-item>
          </el-col>
        </el-row>

        <el-form-item label="优先级">
          <el-radio-group v-model="newTaskForm.priority">
            <el-radio label="normal">普通</el-radio>
            <el-radio label="high"><span style="color:var(--neon-yellow)">高优先</span></el-radio>
            <el-radio label="critical"><span style="color:var(--neon-red)">紧急</span></el-radio>
          </el-radio-group>
        </el-form-item>

      </el-form>

      <template #footer>
        <span class="dialog-footer">
          <el-button @click="dialogVisible = false">取消</el-button>
          <el-button type="primary" @click="submitTask(taskFormRef)" :loading="submitting">
            立即调度
          </el-button>
        </span>
      </template>
    </el-dialog>

  </div>
</template>

<script setup>
import { ref, reactive, computed, onMounted } from 'vue'
import { Dish, Plus, Warning, VideoPlay } from '@element-plus/icons-vue'
import { ElMessage } from 'element-plus'

// --- 基础数据 ---
const utcTime = ref('2023-10-27 14:30:55')

// 天线数据
const antennas = ref([
  { id: 1, name: 'ANT-01', band: 'C', status: 'busy', statusText: '跟踪中: SAT-ASIA-3', az: 120.5, el: 45.2 },
  { id: 2, name: 'ANT-02', band: 'X', status: 'idle', statusText: '待机 (收藏位)', az: 0, el: 0 },
  { id: 3, name: 'ANT-03', band: 'Ku', status: 'moving', statusText: '对星中...', az: 90.0, el: 30 },
  { id: 4, name: 'ANT-04', band: 'Ka', status: 'maint', statusText: '维护模式', az: null },
])

// 接收机数据
const receivers = ref(Array.from({ length: 10 }, (_, i) => ({
  id: i + 1,
  name: `RCV-${String(i + 1).padStart(2, '0')}`,
  status: 'idle',
  target: null
})))

// 初始化模拟占用
receivers.value[0].status = 'busy'; receivers.value[0].target = 'SAT-ASIA-3';
receivers.value[2].status = 'busy'; receivers.value[2].target = 'SAT-ASIA-3';

// 任务列表
const tasks = ref([
  { id: 'TASK-1024', name: 'SAT-ASIA-3 频谱监视', time: '00:15:30', resourceName: 'ANT-01(C) -> RCV-01', antId: 1, rcvId: 1, progress: 60, isAlert: false, status: 'running' },
  { id: 'TASK-1025', name: '高优先级干扰查证', time: '00:02:10', resourceName: 'ANT-03(Ku) -> RCV-03', antId: 3, rcvId: 3, progress: 25, isAlert: true, status: 'running' }
])

const runningTasks = computed(() => tasks.value.filter(t => t.status === 'running'))

// --- 动态拓扑连线逻辑 ---
// 核心：连线不再写死，而是根据 tasks 数组中的 antId 和 rcvId 自动生成
const activeLinks = computed(() => {
  return tasks.value
    .filter(t => t.status === 'running')
    .map(t => ({ antId: t.antId, rcvId: t.rcvId }))
})

// 坐标映射
const getAntennaY = (id) => 60 + (id - 1) * 110
const getReceiverY = (id) => 100 + Math.floor((id - 1) / 2) * 40

// 选中的接收机逻辑
const selectedRcvId = ref(3)
const selectedReceiver = computed(() => receivers.value.find(r => r.id === selectedRcvId.value))
const hasAlert = computed(() => selectedReceiver.value?.id === 3 && tasks.value.find(t => t.rcvId === 3)?.isAlert)

const selectReceiver = (id) => {
  selectedRcvId.value = id
}

// --- 新建任务逻辑 ---
const dialogVisible = ref(false)
const submitting = ref(false)
const taskFormRef = ref(null)

const newTaskForm = reactive({
  name: '',
  satellite: '',
  antennaId: null,
  mode: 'persist',
  duration: 30,
  priority: 'normal'
})

const rules = {
  name: [{ required: true, message: '请输入任务名称', trigger: 'blur' }],
  satellite: [{ required: true, message: '请选择目标卫星', trigger: 'change' }],
  antennaId: [{ required: true, message: '请分配天线资源', trigger: 'change' }],
}

const openNewTaskDialog = () => {
  // 重置表单
  newTaskForm.name = ''
  newTaskForm.satellite = ''
  newTaskForm.antennaId = null
  newTaskForm.mode = 'persist'
  newTaskForm.duration = 30
  newTaskForm.priority = 'normal'

  dialogVisible.value = true
}

const submitTask = async (formEl) => {
  if (!formEl) return
  await formEl.validate((valid) => {
    if (valid) {
      submitting.value = true

      // 1. 寻找空闲接收机
      const freeReceiver = receivers.value.find(r => r.status === 'idle')
      if (!freeReceiver) {
        ElMessage.error('系统资源不足：无可用接收机！')
        submitting.value = false
        return
      }

      // 模拟网络请求延迟
      setTimeout(() => {
        // 2. 更新资源状态
        // 锁定接收机
        freeReceiver.status = 'busy'
        freeReceiver.target = newTaskForm.satellite

        // 更新天线状态
        const targetAnt = antennas.value.find(a => a.id === newTaskForm.antennaId)
        if (targetAnt) {
          targetAnt.status = 'busy'
          targetAnt.statusText = `跟踪中: ${newTaskForm.satellite}`
        }

        // 3. 创建新任务
        const taskId = `TASK-${Math.floor(Math.random() * 8999) + 1000}`
        const newTask = {
          id: taskId,
          name: newTaskForm.name,
          time: '00:00:00', // 初始时间
          resourceName: `${targetAnt.name} -> ${freeReceiver.name}`,
          antId: newTaskForm.antennaId,
          rcvId: freeReceiver.id,
          progress: 0,
          isAlert: false,
          status: 'running'
        }

        // 推入队列头部
        tasks.value.unshift(newTask)

        // 自动选中新任务对应的接收机，方便查看
        selectReceiver(freeReceiver.id)

        ElMessage.success(`任务 ${taskId} 下发成功，资源已就绪`)
        dialogVisible.value = false
        submitting.value = false
      }, 800)
    }
  })
}

// 辅助样式
const getBandColor = (band) => ({ 'C': 'warning', 'X': 'danger', 'Ku': 'primary', 'Ka': 'success' }[band] || 'info')
const getStatusColorClass = (status) => ({
  'busy': 'text-warning', 'idle': 'text-success', 'moving': 'text-primary', 'maint': 'text-info'
}[status])

// 时间模拟
onMounted(() => setInterval(() => { /* Update Time Logic */ }, 1000))
</script>

<style scoped>
/* 此处保留上一个版本的所有样式，并添加以下 Dialog 相关样式 */

/* ... (粘贴上一个版本的 :root 和其他样式) ... */
:root {
  --bg-dark: #0f1115;
  --panel-bg: #181b21;
  --header-bg: #121418;
  --border-col: #2d333b;
  --accent: #2f81f7;
  --text-main: #e6edf3;
  --neon-red: #f85149;
  --neon-yellow: #d29922;
}

.app-wrapper {
  background-color: #0f1115;
  height: 100vh;
  color: #e6edf3;
  font-family: 'Segoe UI', monospace;
  overflow: hidden;
}

/* 深度覆盖 Element Plus */
:deep(.el-card) {
  background-color: #21262d;
  border: 1px solid #30363d;
  color: #e6edf3;
}

:deep(.el-header) {
  padding: 0 20px;
}

:deep(.el-dialog) {
  background-color: #161b22;
  border: 1px solid #30363d;
}

:deep(.el-dialog__title) {
  color: #e6edf3;
}

:deep(.el-dialog__body) {
  color: #e6edf3;
  padding-top: 10px;
}

:deep(.el-form-item__label) {
  color: #8b949e;
}

:deep(.el-input__wrapper),
:deep(.el-select__wrapper) {
  background-color: #0d1117;
  box-shadow: 0 0 0 1px #30363d inset;
}

/* 布局样式复用 */
.sys-header {
  background: #121418;
  border-bottom: 1px solid #30363d;
  display: flex;
  justify-content: space-between;
  align-items: center;
  height: 60px;
}

.brand {
  font-size: 20px;
  font-weight: bold;
  color: #2f81f7;
  display: flex;
  align-items: center;
  gap: 10px;
}

.sys-info {
  display: flex;
  gap: 15px;
  align-items: center;
}

.time-text {
  font-size: 14px;
  color: #8b949e;
  font-family: monospace;
}

.content-body {
  height: calc(100vh - 60px);
}

.left-panel,
.right-panel {
  background: #0d1117;
  border-right: 1px solid #30363d;
  padding: 10px;
  display: flex;
  flex-direction: column;
}

.right-panel {
  border-left: 1px solid #30363d;
  border-right: none;
}

.panel-title {
  font-size: 12px;
  color: #8b949e;
  margin-bottom: 10px;
  text-transform: uppercase;
}

/* 天线列表 */
.antenna-list {
  display: flex;
  flex-direction: column;
  gap: 10px;
}

.sci-fi-card {
  border-radius: 4px;
  transition: all 0.3s;
}

.active-border {
  border-color: #d29922 !important;
  box-shadow: 0 0 8px rgba(210, 153, 34, 0.1);
}

.card-header-flex {
  display: flex;
  justify-content: space-between;
  align-items: center;
  margin-bottom: 8px;
}

.ant-name {
  font-weight: bold;
  font-size: 14px;
}

.ant-status {
  font-size: 12px;
  display: flex;
  align-items: center;
  gap: 6px;
}

.status-dot {
  width: 6px;
  height: 6px;
  border-radius: 50%;
  background: currentColor;
}

.text-warning {
  color: #d29922;
}

.text-success {
  color: #3fb950;
}

.text-primary {
  color: #2f81f7;
}

.text-info {
  color: #8b949e;
}

.ant-params {
  font-size: 11px;
  color: #8b949e;
  margin-top: 5px;
  font-family: monospace;
}

/* 中间面板 */
.center-panel {
  background: #0f1115;
  padding: 10px;
  display: flex;
  flex-direction: column;
  gap: 10px;
  position: relative;
}

.topology-wrapper {
  flex: 3;
  background: #080a0e;
  border: 1px solid #30363d;
  border-radius: 6px;
  position: relative;
  overflow: hidden;
  display: flex;
  align-items: center;
  justify-content: center;
}

.panel-title-overlay {
  position: absolute;
  top: 10px;
  left: 15px;
  font-size: 12px;
  color: #8b949e;
}

.connections-layer {
  position: absolute;
  width: 100%;
  height: 100%;
  z-index: 1;
  pointer-events: none;
}

.link-line {
  stroke: #d29922;
  stroke-width: 2;
  stroke-dasharray: 5;
  animation: dash 1s linear infinite;
}

@keyframes dash {
  to {
    stroke-dashoffset: -10;
  }
}

.topology-nodes {
  z-index: 2;
  display: flex;
  width: 100%;
  justify-content: space-evenly;
  align-items: center;
  padding-left: 200px;
}

.matrix-box {
  width: 100px;
  height: 180px;
  border: 2px solid #555;
  background: #161b22;
  display: flex;
  flex-direction: column;
  align-items: center;
  justify-content: center;
  color: #888;
  font-size: 12px;
  text-align: center;
}

.matrix-leds {
  margin-top: 10px;
  display: flex;
  gap: 4px;
}

.led {
  width: 4px;
  height: 4px;
  background: #2f81f7;
  border-radius: 50%;
}

.receiver-pool {
  background: #161b22;
  border: 1px solid #30363d;
  padding: 15px;
  border-radius: 4px;
}

.pool-label {
  font-size: 10px;
  color: #888;
  margin-bottom: 10px;
}

.rcv-grid {
  display: grid;
  grid-template-columns: repeat(2, 1fr);
  gap: 10px;
}

.rcv-item {
  width: 80px;
  height: 30px;
  background: #0d1117;
  border: 1px solid #30363d;
  display: flex;
  align-items: center;
  justify-content: space-between;
  padding: 0 8px;
  font-size: 11px;
  cursor: pointer;
  transition: all 0.2s;
}

.rcv-item:hover {
  border-color: #2f81f7;
}

.rcv-item.is-busy {
  border-left: 3px solid #d29922;
}

.rcv-item.is-selected {
  background: rgba(47, 129, 247, 0.15);
  border-color: #2f81f7;
}

.rcv-led {
  width: 6px;
  height: 6px;
  border-radius: 50%;
  background: #3fb950;
}

.rcv-led.busy {
  background: #d29922;
}

/* 信号分析 */
.analysis-wrapper {
  flex: 2;
  background: #0d1117;
  border: 1px solid #30363d;
  border-radius: 6px;
  display: flex;
  flex-direction: column;
}

.panel-header-sm {
  padding: 8px 15px;
  background: #161b22;
  border-bottom: 1px solid #30363d;
  font-size: 12px;
  display: flex;
  justify-content: space-between;
  align-items: center;
}

.spectrum-chart {
  flex: 1;
  position: relative;
  background: linear-gradient(180deg, rgba(0, 0, 0, 0.5) 0%, rgba(0, 30, 60, 0.3) 100%);
  overflow: hidden;
  display: flex;
  align-items: center;
  justify-content: center;
}

.empty-state {
  color: #555;
  font-size: 12px;
}

.wave-line {
  position: absolute;
  bottom: 0;
  left: 0;
  width: 100%;
  height: 100px;
  background: linear-gradient(to top, rgba(47, 129, 247, 0.3), transparent);
  clip-path: polygon(0% 100%, 10% 90%, 20% 100%, 30% 95%, 40% 100%, 50% 40%, 60% 100%, 100% 100%);
}

.alert-overlay {
  position: absolute;
  top: 20px;
  right: 20px;
  z-index: 10;
}

/* 任务列表 */
.panel-header-flex {
  display: flex;
  justify-content: space-between;
  align-items: center;
  margin-bottom: 15px;
  font-size: 13px;
  font-weight: bold;
}

.task-list {
  padding-right: 5px;
}

.list-label {
  font-size: 11px;
  color: #888;
  margin-bottom: 8px;
}

.task-card {
  margin-bottom: 10px;
  border-left: 3px solid #30363d;
}

.task-card.alert-mode {
  border-left-color: #f85149 !important;
  background: rgba(248, 81, 73, 0.05) !important;
}

.task-row {
  display: flex;
  justify-content: space-between;
  font-size: 11px;
  color: #8b949e;
  margin-bottom: 4px;
}

.task-title {
  font-weight: bold;
  font-size: 13px;
  margin-bottom: 4px;
}

.task-resource {
  font-size: 11px;
  color: #8b949e;
}

.alert-text {
  color: #f85149;
  font-size: 11px;
  margin-top: 5px;
  display: flex;
  align-items: center;
  gap: 4px;
  font-weight: bold;
}

.mt-2 {
  margin-top: 8px;
}

.mt-4 {
  margin-top: 16px;
}

.pending-card {
  opacity: 0.7;
}

/* 列表动画 */
.list-enter-active,
.list-leave-active {
  transition: all 0.5s ease;
}

.list-enter-from,
.list-leave-to {
  opacity: 0;
  transform: translateX(30px);
}
</style>