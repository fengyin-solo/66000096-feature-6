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

    <!-- 组合筛选：分组 + 状态 + 电量风险 -->
    <div style="margin-bottom:12px;padding:10px;background:#fafafa;border:1px solid #eee;border-radius:8px">
      <div style="margin-bottom:8px">
        <div style="font-size:11px;color:#888;margin-bottom:4px">分组</div>
        <div style="display:flex;flex-wrap:wrap;gap:4px">
          <button v-for="opt in groupOptions" :key="opt.value" @click="store.setDeviceGroupFilter(opt.value)"
            :style="filterPillStyle(store.deviceFilters.groupId === opt.value, opt.color)">
            {{ opt.label }}
          </button>
        </div>
      </div>
      <div style="margin-bottom:8px">
        <div style="font-size:11px;color:#888;margin-bottom:4px">状态</div>
        <div style="display:flex;flex-wrap:wrap;gap:4px">
          <button v-for="opt in statusOptions" :key="opt.value" @click="store.setDeviceStatusFilter(opt.value)"
            :style="filterPillStyle(store.deviceFilters.status === opt.value, opt.color)">
            {{ opt.label }}
          </button>
        </div>
      </div>
      <div>
        <div style="font-size:11px;color:#888;margin-bottom:4px">电量风险</div>
        <div style="display:flex;flex-wrap:wrap;gap:4px">
          <button v-for="opt in batteryOptions" :key="opt.value" @click="store.setDeviceBatteryRiskFilter(opt.value)"
            :style="filterPillStyle(store.deviceFilters.batteryRisk === opt.value, opt.color)">
            {{ opt.label }}
          </button>
        </div>
      </div>
    </div>

    <!-- 列表计数（与筛选结果保持一致） -->
    <div style="display:flex;align-items:center;justify-content:space-between;margin-bottom:8px;font-size:11px;color:#888">
      <span>匹配 <b style="color:#333">{{ store.filteredDeviceCount }}</b> / {{ store.deviceCount }} 台设备</span>
      <button v-if="store.hasActiveFilters" @click="store.clearDeviceFilters()"
        style="padding:2px 8px;border:1px solid #e0e0e0;background:#fff;border-radius:10px;cursor:pointer;font-size:11px;color:#1976d2">
        ✕ 清除条件
      </button>
    </div>

    <!-- 空态：无匹配结果 -->
    <div v-if="store.filteredDevices.length === 0"
      style="text-align:center;padding:32px 16px;border:1px dashed #ddd;border-radius:8px;background:#fafafa">
      <div style="font-size:32px;margin-bottom:8px">🔍</div>
      <div style="font-size:13px;color:#666;margin-bottom:4px">没有符合条件的设备</div>
      <div style="font-size:11px;color:#999;margin-bottom:12px">试试调整或清除筛选条件</div>
      <button @click="store.clearDeviceFilters()"
        style="padding:6px 14px;background:#1976d2;color:#fff;border:none;border-radius:6px;cursor:pointer;font-size:12px">
        清除筛选条件
      </button>
    </div>

    <div v-for="d in store.filteredDevices" :key="d.id"
      :ref="(el) => setRowEl(d.id, el)"
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
      <div style="flex:1">
        <div style="font-weight: store.highlightedDeviceId === d.id ? 700 : 500;font-size:13px;color:#333;display:flex;align-items:center;gap:6px">
          {{ d.name }}
          <span v-if="d.groupId && getGroup(d.groupId)"
            :style="{ fontSize:'10px', padding:'1px 6px', borderRadius:'8px', background: getGroup(d.groupId)!.color + '20', color: getGroup(d.groupId)!.color }">
            {{ getGroup(d.groupId)!.name }}
          </span>
        </div>
        <div style="font-size:11px;color:#888">🔋 {{ d.battery }}% · 🌡 {{ d.temperature }}°C</div>
      </div>
      <span v-if="d.status === 'alert'" style="font-size:10px;color:#ff9800">⚠️</span>
    </div>
  </div>
</template>

<script setup lang="ts">
import { computed, nextTick, onMounted } from 'vue';
import { useIotStore } from '../stores/iot';
import type { DeviceStatusFilter, BatteryRiskFilter, DeviceGroupFilter } from '../types';
const store = useIotStore();

const emit = defineEmits<{
  (e: 'add-device'): void;
}>();

const groupOptions = computed(() => [
  { value: 'all' as DeviceGroupFilter, label: '全部', color: '#1976d2' },
  { value: 'none' as DeviceGroupFilter, label: '未分组', color: '#757575' },
  ...store.groups.map(g => ({ value: g.id as DeviceGroupFilter, label: g.name, color: g.color })),
]);

const statusOptions: Array<{ value: DeviceStatusFilter; label: string; color: string }> = [
  { value: 'all', label: '全部', color: '#1976d2' },
  { value: 'online', label: '在线', color: '#4caf50' },
  { value: 'alert', label: '告警', color: '#ff9800' },
  { value: 'offline', label: '离线', color: '#9e9e9e' },
];

const batteryOptions: Array<{ value: BatteryRiskFilter; label: string; color: string }> = [
  { value: 'all', label: '全部', color: '#1976d2' },
  { value: 'critical', label: '风险 <20%', color: '#e53935' },
  { value: 'low', label: '偏低 20-49%', color: '#f57c00' },
  { value: 'normal', label: '正常 ≥50%', color: '#4caf50' },
];

function filterPillStyle(active: boolean, color?: string) {
  return {
    padding: '3px 10px',
    borderRadius: '12px',
    fontSize: '11px',
    cursor: 'pointer',
    border: '1px solid ' + (active ? (color || '#1976d2') : '#e0e0e0'),
    background: active ? (color || '#1976d2') : '#fff',
    color: active ? '#fff' : '#666',
    fontWeight: active ? 600 : 400,
  };
}

// 设备行 DOM 引用，用于定位（滚动）到设备行
const rowEls = new Map<string, HTMLElement>();
function setRowEl(id: string, el: Element | { $el?: Element } | null) {
  if (el instanceof HTMLElement) {
    rowEls.set(id, el);
  } else {
    rowEls.delete(id);
  }
}

function scrollToDeviceRow(id: string) {
  nextTick(() => {
    rowEls.get(id)?.scrollIntoView({ behavior: 'smooth', block: 'nearest' });
  });
}

function getGroup(groupId: string) {
  return store.getGroupById(groupId);
}

function handleDeviceClick(id: string) {
  store.setHighlightedDevice(id);
  scrollToDeviceRow(id);
}

function handleHover(id: string | null) {
  if (!store.highlightedDeviceId) {
    store.setHighlightedDevice(id);
  }
}

// 返回再进入时：保留上次筛选，并定位到上次高亮的设备行（详情由地图弹窗呈现）
onMounted(() => {
  const id = store.highlightedDeviceId;
  if (id && store.filteredDevices.some(d => d.id === id)) {
    scrollToDeviceRow(id);
  }
});
</script>
