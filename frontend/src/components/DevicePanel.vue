<template>
  <div style="width:300px;padding:16px;overflow:auto;border-left:1px solid #e0e0e0;height:100vh;box-sizing:border-box">
    <h3 style="margin:0 0 12px;display:flex;align-items:center;justify-content:space-between">
      <span>📡 设备列表</span>
      <button @click="emit('add-device')"
        style="padding:6px 12px;background:#1b5e20;color:#fff;border:none;border-radius:6px;cursor:pointer;font-size:12px;font-weight:500">
        ➕ 注册
      </button>
    </h3>
    <div style="display:flex;gap:8px;margin-bottom:16px">
      <span style="font-size:12px;padding:2px 8px;border-radius:12px;background:#e8f5e9">🟢 {{ store.onlineCount }} 在线</span>
      <span style="font-size:12px;padding:2px 8px;border-radius:12px;background:#ffebee">⚠️ {{ store.alertCount }} 告警</span>
    </div>

    <!-- ============ 设备详情 ============ -->
    <div v-if="detailDevice">
      <div style="display:flex;align-items:center;justify-content:space-between;margin-bottom:12px">
        <button @click="closeDetail"
          style="padding:4px 10px;background:#f5f5f5;color:#666;border:1px solid #e0e0e0;border-radius:6px;cursor:pointer;font-size:12px">
          ← 返回列表
        </button>
        <button @click="locateOnMap"
          style="padding:4px 10px;background:#e3f2fd;color:#1565c0;border:1px solid #90caf9;border-radius:6px;cursor:pointer;font-size:12px">
          📍 地图定位
        </button>
      </div>

      <div :style="{ padding:'14px', borderRadius:'10px', border:'2px solid ' + (store.highlightedDeviceId === detailDevice.id ? '#1976d2' : '#e0e0e0'),
        background: store.highlightedDeviceId === detailDevice.id ? '#f5faff' : '#fff', marginBottom:'12px' }">
        <div style="display:flex;align-items:center;gap:8px;margin-bottom:4px">
          <span :style="{ width:'10px', height:'10px', borderRadius:'50%',
            background: detailDevice.status === 'online' ? '#4caf50' : detailDevice.status === 'alert' ? '#ff9800' : '#9e9e9e' }"></span>
          <span style="font-weight:700;font-size:15px;color:#333">{{ detailDevice.name }}</span>
        </div>
        <div style="display:flex;gap:6px;flex-wrap:wrap;margin-left:18px">
          <span :style="{ fontSize:'10px', padding:'2px 8px', borderRadius:'10px',
            background: statusMeta(detailDevice.status).bg, color: statusMeta(detailDevice.status).color }">
            {{ statusMeta(detailDevice.status).text }}
          </span>
          <span v-if="detailDevice.groupId && getGroup(detailDevice.groupId)"
            :style="{ fontSize:'10px', padding:'2px 8px', borderRadius:'10px',
              background: getGroup(detailDevice.groupId)!.color + '20', color: getGroup(detailDevice.groupId)!.color }">
            {{ getGroup(detailDevice.groupId)!.name }}
          </span>
          <span v-else style="font-size:10px;padding:2px 8px;border-radius:10px;background:#f5f5f5;color:#999">
            未分组
          </span>
        </div>
      </div>

      <div style="display:flex;flex-direction:column;gap:8px;font-size:12px">
        <div style="display:flex;justify-content:space-between;align-items:center;padding:10px 12px;background:#fafafa;border-radius:8px">
          <span style="color:#666">🔋 电量</span>
          <span style="display:flex;align-items:center;gap:6px;font-weight:600;color:#333">
            {{ detailDevice.battery }}%
            <span :style="{ fontSize:'10px', padding:'1px 8px', borderRadius:'10px',
              background: batteryRiskMeta(detailDevice.battery).bg, color: batteryRiskMeta(detailDevice.battery).color }">
              {{ batteryRiskMeta(detailDevice.battery).text }}
            </span>
          </span>
        </div>
        <div style="display:flex;justify-content:space-between;padding:10px 12px;background:#fafafa;border-radius:8px">
          <span style="color:#666">🌡 温度</span>
          <span style="font-weight:600;color:#333">{{ detailDevice.temperature }}°C</span>
        </div>
        <div style="display:flex;justify-content:space-between;padding:10px 12px;background:#fafafa;border-radius:8px">
          <span style="color:#666">📍 位置</span>
          <span style="font-weight:600;color:#333">{{ detailDevice.lat.toFixed(4) }}, {{ detailDevice.lng.toFixed(4) }}</span>
        </div>
        <div style="display:flex;justify-content:space-between;padding:10px 12px;background:#fafafa;border-radius:8px">
          <span style="color:#666">⏱ 最后上报</span>
          <span style="font-weight:500;color:#333">{{ formatLastSeen(detailDevice.lastSeen) }}</span>
        </div>
        <div v-if="detailHealth" style="display:flex;justify-content:space-between;padding:10px 12px;background:#fafafa;border-radius:8px">
          <span style="color:#666">🏥 健康评分</span>
          <span style="font-weight:600;color:#1565c0">{{ detailHealth.healthScore }} 分</span>
        </div>
        <div v-if="detailDevice.thresholds" style="padding:10px 12px;background:#fafafa;border-radius:8px">
          <div style="color:#666;margin-bottom:6px">⚙️ 告警阈值</div>
          <div style="display:flex;justify-content:space-between;color:#888;font-size:11px;margin-bottom:4px">
            <span>低电量阈值</span><span>{{ detailDevice.thresholds.lowBattery }}%</span>
          </div>
          <div style="display:flex;justify-content:space-between;color:#888;font-size:11px;margin-bottom:4px">
            <span>高温阈值</span><span>{{ detailDevice.thresholds.highTemperature }}°C</span>
          </div>
          <div style="display:flex;justify-content:space-between;color:#888;font-size:11px">
            <span>离线超时</span><span>{{ detailDevice.thresholds.offlineTimeout }} 分钟</span>
          </div>
        </div>
      </div>
    </div>

    <!-- ============ 列表 + 筛选 ============ -->
    <template v-else>
      <!-- 组合筛选 -->
      <div style="padding:10px;background:#fafafa;border:1px solid #e0e0e0;border-radius:8px;margin-bottom:12px">
        <div style="font-size:11px;color:#888;margin-bottom:8px;display:flex;justify-content:space-between;align-items:center">
          <span>🔍 组合筛选</span>
          <span :style="{ fontWeight: store.hasActiveFilter ? 600 : 400, color: store.hasActiveFilter ? '#1976d2' : '#888' }">
            匹配 {{ store.filteredDeviceCount }} / {{ store.deviceCount }} 台
          </span>
        </div>
        <div style="display:flex;flex-direction:column;gap:6px">
          <select v-model="filterGroupId"
            style="width:100%;padding:6px 8px;border:1px solid #ddd;border-radius:6px;font-size:12px;background:#fff;box-sizing:border-box;cursor:pointer">
            <option value="">全部分组</option>
            <option v-for="g in store.groups" :key="g.id" :value="g.id">{{ g.name }}</option>
          </select>
          <div style="display:flex;gap:6px">
            <select v-model="filterStatus"
              style="flex:1;padding:6px 8px;border:1px solid #ddd;border-radius:6px;font-size:12px;background:#fff;box-sizing:border-box;cursor:pointer">
              <option value="">全部状态</option>
              <option value="online">🟢 在线</option>
              <option value="alert">⚠️ 告警</option>
              <option value="offline">⚪ 离线</option>
            </select>
            <select v-model="filterBatteryRisk"
              style="flex:1;padding:6px 8px;border:1px solid #ddd;border-radius:6px;font-size:12px;background:#fff;box-sizing:border-box;cursor:pointer">
              <option value="">全部电量</option>
              <option value="high">🔴 高风险 (&lt;20%)</option>
              <option value="medium">🟠 中风险 (20-49%)</option>
              <option value="low">🟢 低风险 (≥50%)</option>
            </select>
          </div>
        </div>
        <button v-if="store.hasActiveFilter" @click="store.resetDeviceFilter()"
          style="margin-top:8px;width:100%;padding:5px;border:1px solid #bbb;background:#fff;color:#666;border-radius:6px;cursor:pointer;font-size:11px">
          ✕ 清除筛选条件
        </button>
      </div>

      <!-- 空态 -->
      <div v-if="store.filteredDevices.length === 0"
        style="text-align:center;padding:40px 16px;background:#fafafa;border:1px dashed #d0d0d0;border-radius:8px">
        <div style="font-size:40px;margin-bottom:10px">🔍</div>
        <div style="font-size:13px;color:#666;margin-bottom:4px">没有符合条件的设备</div>
        <div style="font-size:11px;color:#999;margin-bottom:14px">试试调整或清除筛选条件</div>
        <button @click="store.resetDeviceFilter()"
          style="padding:6px 16px;background:#1976d2;color:#fff;border:none;border-radius:6px;cursor:pointer;font-size:12px;font-weight:500">
          清除筛选条件
        </button>
      </div>

      <!-- 设备卡片（原有卡片与点击/悬停行为保持不变，新增“详情”入口） -->
      <div v-for="d in store.filteredDevices" :key="d.id"
        :ref="(el) => setRowRef(el, d.id)"
        @click="handleDeviceClick(d.id)"
        @mouseenter="handleHover(d.id)"
        @mouseleave="handleHover(null)"
        :style="{ display:'flex', alignItems:'center', gap:'10px', padding:'10px', marginBottom:'8px',
          borderRadius:'8px', border:'2px solid ' + (store.highlightedDeviceId === d.id ? '#1976d2' : (d.status === 'alert' ? '#ffcc80' : '#e0e0e0')),
          background: store.highlightedDeviceId === d.id ? '#e3f2fd' : (d.status === 'alert' ? '#fff3e0' : '#fff'),
          cursor:'pointer', transition:'all 0.2s ease' }">
        <span :style="{ width:'10px', height:'10px', borderRadius:'50%',
          background: d.status === 'online' ? '#4caf50' : d.status === 'alert' ? '#ff9800' : '#9e9e9e',
          boxShadow: store.highlightedDeviceId === d.id ? '0 0 0 3px rgba(25,118,210,0.3)' : 'none' }"></span>
        <div style="flex:1;min-width:0">
          <div style="font-weight: store.highlightedDeviceId === d.id ? 700 : 500;font-size:13px;color:#333;display:flex;align-items:center;gap:6px">
            {{ d.name }}
            <span v-if="d.groupId && getGroup(d.groupId)"
              :style="{ fontSize:'10px', padding:'1px 6px', borderRadius:'8px', background: getGroup(d.groupId)!.color + '20', color: getGroup(d.groupId)!.color }">
              {{ getGroup(d.groupId)!.name }}
            </span>
          </div>
          <div style="font-size:11px;color:#888;display:flex;align-items:center;gap:4px">
            <span>🔋 {{ d.battery }}%</span>
            <span :style="{ fontSize:'9px', padding:'0 5px', borderRadius:'8px',
              background: batteryRiskMeta(d.battery).bg, color: batteryRiskMeta(d.battery).color, flexShrink:0 }">
              {{ batteryRiskMeta(d.battery).shortText }}
            </span>
            <span>· 🌡 {{ d.temperature }}°C</span>
          </div>
        </div>
        <span v-if="d.status === 'alert'" style="font-size:10px;color:#ff9800">⚠️</span>
        <button @click.stop="openDetail(d.id)" :title="'查看设备详情'"
          style="padding:3px 8px;font-size:11px;background:transparent;color:#1976d2;border:1px solid #90caf9;border-radius:5px;cursor:pointer;flexShrink:0">
          详情
        </button>
      </div>
    </template>
  </div>
</template>

<script setup lang="ts">
import { ref, computed, watch, onMounted, nextTick, type ComponentPublicInstance } from 'vue';
import { useIotStore } from '../stores/iot';
import type { BatteryRisk } from '../types';
const store = useIotStore();

const emit = defineEmits<{
  (e: 'add-device'): void;
}>();

const detailDeviceId = ref<string | null>(null);
const detailDevice = computed(() => detailDeviceId.value ? store.getDeviceById(detailDeviceId.value) : undefined);
const detailHealth = computed(() => detailDeviceId.value ? store.getDeviceHealth(detailDeviceId.value) : undefined);

const filterGroupId = computed({
  get: () => store.deviceFilter.groupId ?? '',
  set: (v: string) => store.setDeviceFilter({ groupId: v || null })
});
const filterStatus = computed({
  get: () => store.deviceFilter.status ?? '',
  set: (v: string) => store.setDeviceFilter({ status: (v || null) as DeviceFilterStatus })
});
const filterBatteryRisk = computed({
  get: () => store.deviceFilter.batteryRisk ?? '',
  set: (v: string) => store.setDeviceFilter({ batteryRisk: (v || null) as BatteryRisk | null })
});
type DeviceFilterStatus = 'online' | 'offline' | 'alert';

function getGroup(groupId: string) {
  return store.getGroupById(groupId);
}

function handleDeviceClick(id: string) {
  store.setHighlightedDevice(id);
}

function handleHover(id: string | null) {
  if (!store.highlightedDeviceId) {
    store.setHighlightedDevice(id);
  }
}

function openDetail(id: string) {
  store.setHighlightedDevice(id);
  detailDeviceId.value = id;
}

function closeDetail() {
  const id = detailDeviceId.value;
  detailDeviceId.value = null;
  // 返回列表后定位到对应的设备行
  if (id) scrollToDevice(id);
}

function locateOnMap() {
  if (!detailDeviceId.value) return;
  // 地图仅在高亮设备发生变化时平移，先清除再设置以触发重新定位
  const id = detailDeviceId.value;
  store.setHighlightedDevice(null);
  nextTick(() => store.setHighlightedDevice(id));
}

// 设备行定位
const rowEls = new Map<string, HTMLElement>();
function setRowRef(el: Element | ComponentPublicInstance | null, id: string) {
  if (el instanceof HTMLElement) {
    rowEls.set(id, el);
  } else {
    rowEls.delete(id);
  }
}
function scrollToDevice(id: string) {
  nextTick(() => {
    rowEls.get(id)?.scrollIntoView({ behavior: 'smooth', block: 'center' });
  });
}

// 条件变化导致详情设备被筛掉时，关闭详情以保持列表、高亮与详情一致
watch(() => store.filteredDevices, (list) => {
  if (detailDeviceId.value && !list.some(d => d.id === detailDeviceId.value)) {
    detailDeviceId.value = null;
  }
});

// 进入面板时（如从告警中心跳转过来）定位到当前高亮设备行
onMounted(() => {
  if (store.highlightedDeviceId && store.filteredDevices.some(d => d.id === store.highlightedDeviceId)) {
    scrollToDevice(store.highlightedDeviceId);
  }
});

function batteryRiskMeta(battery: number): { text: string; shortText: string; color: string; bg: string } {
  const risk = store.getBatteryRisk(battery);
  switch (risk) {
    case 'high':
      return { text: '高风险 · 电量 <20%', shortText: '高风险', color: '#c62828', bg: '#ffebee' };
    case 'medium':
      return { text: '中风险 · 电量 20-49%', shortText: '中风险', color: '#ef6c00', bg: '#fff3e0' };
    default:
      return { text: '低风险 · 电量 ≥50%', shortText: '低风险', color: '#2e7d32', bg: '#e8f5e9' };
  }
}

function statusMeta(status: DeviceFilterStatus): { text: string; color: string; bg: string } {
  switch (status) {
    case 'online': return { text: '🟢 在线', color: '#2e7d32', bg: '#e8f5e9' };
    case 'alert': return { text: '⚠️ 告警', color: '#ef6c00', bg: '#fff3e0' };
    default: return { text: '⚪ 离线', color: '#757575', bg: '#f5f5f5' };
  }
}

function formatLastSeen(isoString: string): string {
  const date = new Date(isoString);
  const diff = Date.now() - date.getTime();
  if (diff < 60000) return '刚刚';
  if (diff < 3600000) return Math.floor(diff / 60000) + ' 分钟前';
  if (diff < 86400000) return Math.floor(diff / 3600000) + ' 小时前';
  return date.toLocaleString('zh-CN', { month: '2-digit', day: '2-digit', hour: '2-digit', minute: '2-digit' });
}
</script>
