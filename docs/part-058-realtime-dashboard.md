# Part 58: Real-time Dashboard ด้วย Vue.js

## Dashboard Architecture

Real-time Dashboard ต้องการ architecture ที่รองรับการ update ข้อมูลต่อเนื่อง และแสดงผลให้ผู้ใช้เห็นทันที

```
┌─────────────────────────────────────────────┐
│              Dashboard Client                │
│  ┌─────────┐  ┌─────────┐  ┌─────────────┐ │
│  │ Charts  │  │ Metrics │  │  Data Table │ │
│  └────┬────┘  └────┬────┘  └──────┬──────┘ │
│       └─────────────┴──────────────┘        │
│              Pinia Store                     │
│       ┌──────────────────────────┐          │
│       │    WebSocket Manager     │          │
│       └──────────────────────────┘          │
└─────────────────────────────────────────────┘
                    │
              WebSocket
                    │
┌─────────────────────────────────────────────┐
│              Backend Server                  │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐  │
│  │  Nitro   │  │  Redis   │  │Database  │  │
│  │  Events  │  │  Pub/Sub │  │          │  │
│  └──────────┘  └──────────┘  └──────────┘  │
└─────────────────────────────────────────────┘
```

## WebSocket Data Streaming

```typescript
// composables/useWebSocket.ts
import { ref, onUnmounted, watch } from 'vue'

interface WebSocketOptions {
  url: string
  reconnectInterval?: number
  maxReconnectAttempts?: number
  onMessage?: (data: any) => void
  onConnect?: () => void
  onDisconnect?: () => void
  onError?: (error: Event) => void
}

export type ConnectionStatus = 'connecting' | 'connected' | 'disconnected' | 'error'

export function useWebSocket(options: WebSocketOptions) {
  const {
    url,
    reconnectInterval = 3000,
    maxReconnectAttempts = 5,
    onMessage,
    onConnect,
    onDisconnect,
    onError
  } = options

  const ws = ref<WebSocket | null>(null)
  const status = ref<ConnectionStatus>('disconnected')
  const reconnectAttempts = ref(0)
  const lastMessage = ref<any>(null)
  let reconnectTimer: ReturnType<typeof setTimeout> | null = null

  const connect = () => {
    if (ws.value?.readyState === WebSocket.OPEN) return

    status.value = 'connecting'
    ws.value = new WebSocket(url)

    ws.value.onopen = () => {
      status.value = 'connected'
      reconnectAttempts.value = 0
      onConnect?.()
    }

    ws.value.onmessage = (event) => {
      try {
        const data = JSON.parse(event.data)
        lastMessage.value = data
        onMessage?.(data)
      } catch (e) {
        console.warn('Failed to parse WebSocket message:', event.data)
      }
    }

    ws.value.onclose = () => {
      status.value = 'disconnected'
      onDisconnect?.()
      
      // Auto-reconnect
      if (reconnectAttempts.value < maxReconnectAttempts) {
        reconnectAttempts.value++
        reconnectTimer = setTimeout(connect, reconnectInterval * reconnectAttempts.value)
      }
    }

    ws.value.onerror = (error) => {
      status.value = 'error'
      onError?.(error)
    }
  }

  const disconnect = () => {
    if (reconnectTimer) {
      clearTimeout(reconnectTimer)
      reconnectTimer = null
    }
    reconnectAttempts.value = maxReconnectAttempts // Prevent auto-reconnect
    ws.value?.close()
    ws.value = null
    status.value = 'disconnected'
  }

  const send = (data: any) => {
    if (ws.value?.readyState === WebSocket.OPEN) {
      ws.value.send(JSON.stringify(data))
      return true
    }
    return false
  }

  onUnmounted(() => {
    disconnect()
  })

  return {
    status,
    lastMessage,
    reconnectAttempts,
    connect,
    disconnect,
    send
  }
}
```

## Pinia Store สำหรับ Dashboard

```typescript
// stores/dashboard.ts
import { defineStore } from 'pinia'
import { ref, computed } from 'vue'

interface MetricPoint {
  timestamp: number
  value: number
}

interface DashboardMetric {
  id: string
  name: string
  value: number
  previousValue: number
  unit: string
  history: MetricPoint[]
  trend: 'up' | 'down' | 'stable'
}

interface AlertItem {
  id: string
  level: 'info' | 'warning' | 'error'
  message: string
  timestamp: number
  acknowledged: boolean
}

export const useDashboardStore = defineStore('dashboard', () => {
  const metrics = ref<Map<string, DashboardMetric>>(new Map())
  const alerts = ref<AlertItem[]>([])
  const isConnected = ref(false)
  const lastUpdated = ref<Date | null>(null)
  const maxHistoryPoints = 60 // Keep 60 data points

  const activeAlerts = computed(() =>
    alerts.value.filter(a => !a.acknowledged).sort((a, b) => b.timestamp - a.timestamp)
  )

  const metricsList = computed(() => Array.from(metrics.value.values()))

  const updateMetric = (id: string, value: number) => {
    const existing = metrics.value.get(id)
    if (!existing) return

    const previousValue = existing.value
    const history = [
      ...existing.history.slice(-(maxHistoryPoints - 1)),
      { timestamp: Date.now(), value }
    ]

    const trend = value > previousValue ? 'up' : value < previousValue ? 'down' : 'stable'

    metrics.value.set(id, {
      ...existing,
      previousValue,
      value,
      history,
      trend
    })

    lastUpdated.value = new Date()
  }

  const addAlert = (alert: Omit<AlertItem, 'id' | 'acknowledged'>) => {
    alerts.value.unshift({
      ...alert,
      id: crypto.randomUUID(),
      acknowledged: false
    })

    // Keep only last 100 alerts
    if (alerts.value.length > 100) {
      alerts.value.splice(100)
    }
  }

  const acknowledgeAlert = (id: string) => {
    const alert = alerts.value.find(a => a.id === id)
    if (alert) {
      alert.acknowledged = true
    }
  }

  const initMetric = (metric: Omit<DashboardMetric, 'history' | 'trend'>) => {
    metrics.value.set(metric.id, {
      ...metric,
      history: [{ timestamp: Date.now(), value: metric.value }],
      trend: 'stable'
    })
  }

  const processWebSocketMessage = (message: any) => {
    switch (message.type) {
      case 'metric_update':
        updateMetric(message.metricId, message.value)
        break
      case 'alert':
        addAlert({
          level: message.level,
          message: message.message,
          timestamp: message.timestamp
        })
        break
      case 'bulk_update':
        message.metrics.forEach((m: { id: string; value: number }) => {
          updateMetric(m.id, m.value)
        })
        break
    }
  }

  return {
    metrics,
    alerts,
    isConnected,
    lastUpdated,
    activeAlerts,
    metricsList,
    updateMetric,
    addAlert,
    acknowledgeAlert,
    initMetric,
    processWebSocketMessage
  }
})
```

## Chart.js กับ Vue

```vue
<!-- components/dashboard/RealtimeLineChart.vue -->
<template>
  <div class="chart-container">
    <div class="chart-header">
      <h3>{{ title }}</h3>
      <div class="chart-controls">
        <button
          v-for="range in timeRanges"
          :key="range.value"
          :class="['range-btn', { active: selectedRange === range.value }]"
          @click="selectedRange = range.value"
        >
          {{ range.label }}
        </button>
      </div>
    </div>
    <canvas ref="chartRef" :height="height"></canvas>
  </div>
</template>

<script setup lang="ts">
import { ref, onMounted, onUnmounted, watch, computed } from 'vue'
import {
  Chart,
  LineController,
  LineElement,
  PointElement,
  LinearScale,
  TimeScale,
  Tooltip,
  Legend,
  Filler
} from 'chart.js'
import 'chartjs-adapter-date-fns'

Chart.register(
  LineController,
  LineElement,
  PointElement,
  LinearScale,
  TimeScale,
  Tooltip,
  Legend,
  Filler
)

interface DataPoint {
  timestamp: number
  value: number
}

const props = defineProps<{
  title: string
  data: DataPoint[]
  color?: string
  height?: number
  unit?: string
}>()

const chartRef = ref<HTMLCanvasElement | null>(null)
let chart: Chart | null = null
const selectedRange = ref('1h')

const timeRanges = [
  { label: '1H', value: '1h' },
  { label: '6H', value: '6h' },
  { label: '24H', value: '24h' },
  { label: '7D', value: '7d' }
]

const filteredData = computed(() => {
  const now = Date.now()
  const ranges: Record<string, number> = {
    '1h': 3600000,
    '6h': 21600000,
    '24h': 86400000,
    '7d': 604800000
  }
  const cutoff = now - ranges[selectedRange.value]
  return props.data.filter(d => d.timestamp >= cutoff)
})

const chartColor = computed(() => props.color || '#4CAF50')

const createChart = () => {
  if (!chartRef.value) return

  const ctx = chartRef.value.getContext('2d')
  if (!ctx) return

  chart = new Chart(ctx, {
    type: 'line',
    data: {
      datasets: [{
        label: props.title,
        data: filteredData.value.map(d => ({ x: d.timestamp, y: d.value })),
        borderColor: chartColor.value,
        backgroundColor: `${chartColor.value}20`,
        fill: true,
        tension: 0.4,
        pointRadius: 2,
        pointHoverRadius: 5
      }]
    },
    options: {
      responsive: true,
      maintainAspectRatio: false,
      animation: {
        duration: 200
      },
      scales: {
        x: {
          type: 'time',
          time: {
            displayFormats: {
              minute: 'HH:mm',
              hour: 'HH:mm',
              day: 'MM/dd'
            }
          },
          grid: {
            color: 'rgba(255,255,255,0.1)'
          }
        },
        y: {
          beginAtZero: false,
          grid: {
            color: 'rgba(255,255,255,0.1)'
          },
          ticks: {
            callback: (value) => `${value}${props.unit || ''}`
          }
        }
      },
      plugins: {
        legend: { display: false },
        tooltip: {
          callbacks: {
            label: (context) => `${context.parsed.y}${props.unit || ''}`
          }
        }
      }
    }
  })
}

const updateChart = () => {
  if (!chart) return
  
  chart.data.datasets[0].data = filteredData.value.map(d => ({
    x: d.timestamp,
    y: d.value
  }))
  
  chart.update('none') // No animation for real-time updates
}

onMounted(() => {
  createChart()
})

onUnmounted(() => {
  chart?.destroy()
})

watch(filteredData, () => {
  updateChart()
}, { deep: true })

watch(selectedRange, () => {
  updateChart()
})
</script>
```

## ApexCharts กับ Vue

```vue
<!-- components/dashboard/ApexGaugeChart.vue -->
<template>
  <div class="gauge-chart">
    <VueApexCharts
      type="radialBar"
      :height="200"
      :options="chartOptions"
      :series="[percentage]"
    />
    <div class="gauge-info">
      <span class="gauge-value">{{ value }}</span>
      <span class="gauge-unit">{{ unit }}</span>
    </div>
  </div>
</template>

<script setup lang="ts">
import { computed } from 'vue'
import VueApexCharts from 'vue3-apexcharts'

const props = defineProps<{
  value: number
  maxValue: number
  unit: string
  label: string
  thresholds?: { warning: number; critical: number }
}>()

const percentage = computed(() => Math.min(100, (props.value / props.maxValue) * 100))

const color = computed(() => {
  if (!props.thresholds) return '#4CAF50'
  const pct = percentage.value
  if (pct >= props.thresholds.critical) return '#f44336'
  if (pct >= props.thresholds.warning) return '#FF9800'
  return '#4CAF50'
})

const chartOptions = computed(() => ({
  chart: {
    type: 'radialBar',
    background: 'transparent'
  },
  plotOptions: {
    radialBar: {
      startAngle: -135,
      endAngle: 135,
      hollow: {
        size: '70%'
      },
      track: {
        background: '#2a2a2a'
      },
      dataLabels: {
        name: {
          show: true,
          offsetY: 20,
          color: '#888',
          fontSize: '12px'
        },
        value: {
          show: false
        }
      }
    }
  },
  fill: {
    colors: [color.value]
  },
  labels: [props.label],
  theme: {
    mode: 'dark'
  }
}))
</script>
```

## Real-time Updates

```typescript
// composables/useDashboardConnection.ts
import { onMounted, onUnmounted } from 'vue'
import { useWebSocket } from './useWebSocket'
import { useDashboardStore } from '@/stores/dashboard'

export function useDashboardConnection() {
  const store = useDashboardStore()
  const wsUrl = `${import.meta.env.VITE_WS_URL}/dashboard`

  const { status, connect, disconnect, send } = useWebSocket({
    url: wsUrl,
    reconnectInterval: 5000,
    maxReconnectAttempts: 10,
    onConnect: () => {
      store.isConnected = true
      // Subscribe to metrics
      send({
        type: 'subscribe',
        channels: ['metrics', 'alerts']
      })
    },
    onDisconnect: () => {
      store.isConnected = false
    },
    onMessage: (data) => {
      store.processWebSocketMessage(data)
    }
  })

  onMounted(() => {
    // Initialize metrics
    store.initMetric({ id: 'cpu', name: 'CPU Usage', value: 0, previousValue: 0, unit: '%' })
    store.initMetric({ id: 'memory', name: 'Memory', value: 0, previousValue: 0, unit: 'GB' })
    store.initMetric({ id: 'requests', name: 'Requests/s', value: 0, previousValue: 0, unit: 'req/s' })
    store.initMetric({ id: 'errors', name: 'Error Rate', value: 0, previousValue: 0, unit: '%' })
    
    connect()
  })

  onUnmounted(() => {
    disconnect()
  })

  return { status, send }
}
```

## Data Aggregation

```typescript
// utils/dataAggregation.ts
interface DataPoint {
  timestamp: number
  value: number
}

export type AggregationType = 'avg' | 'sum' | 'min' | 'max' | 'count'

export function aggregateData(
  data: DataPoint[],
  windowMs: number,
  type: AggregationType = 'avg'
): DataPoint[] {
  if (data.length === 0) return []

  const buckets = new Map<number, number[]>()

  data.forEach(point => {
    const bucket = Math.floor(point.timestamp / windowMs) * windowMs
    if (!buckets.has(bucket)) {
      buckets.set(bucket, [])
    }
    buckets.get(bucket)!.push(point.value)
  })

  return Array.from(buckets.entries())
    .sort(([a], [b]) => a - b)
    .map(([timestamp, values]) => ({
      timestamp,
      value: aggregate(values, type)
    }))
}

function aggregate(values: number[], type: AggregationType): number {
  switch (type) {
    case 'avg': return values.reduce((a, b) => a + b, 0) / values.length
    case 'sum': return values.reduce((a, b) => a + b, 0)
    case 'min': return Math.min(...values)
    case 'max': return Math.max(...values)
    case 'count': return values.length
    default: return 0
  }
}

export function calculateMovingAverage(data: DataPoint[], window: number): DataPoint[] {
  return data.map((point, index) => {
    const start = Math.max(0, index - window + 1)
    const windowData = data.slice(start, index + 1)
    const avg = windowData.reduce((sum, p) => sum + p.value, 0) / windowData.length
    return { timestamp: point.timestamp, value: Math.round(avg * 100) / 100 }
  })
}
```

## Export Feature

```typescript
// composables/useExport.ts
import { ref } from 'vue'

type ExportFormat = 'csv' | 'json' | 'xlsx'

interface ExportOptions {
  filename?: string
  format?: ExportFormat
}

export function useExport() {
  const isExporting = ref(false)

  const exportToCSV = (data: Record<string, any>[], filename = 'export') => {
    const headers = Object.keys(data[0])
    const rows = data.map(row => 
      headers.map(h => {
        const val = row[h]
        return typeof val === 'string' && val.includes(',') ? `"${val}"` : val
      }).join(',')
    )
    
    const csv = [headers.join(','), ...rows].join('\n')
    downloadFile(csv, `${filename}.csv`, 'text/csv')
  }

  const exportToJSON = (data: any, filename = 'export') => {
    const json = JSON.stringify(data, null, 2)
    downloadFile(json, `${filename}.json`, 'application/json')
  }

  const downloadFile = (content: string, filename: string, mimeType: string) => {
    const blob = new Blob([content], { type: mimeType })
    const url = URL.createObjectURL(blob)
    const link = document.createElement('a')
    link.href = url
    link.download = filename
    document.body.appendChild(link)
    link.click()
    document.body.removeChild(link)
    URL.revokeObjectURL(url)
  }

  const exportDashboard = async (
    metricsData: any[],
    options: ExportOptions = {}
  ) => {
    isExporting.value = true
    try {
      const filename = options.filename || `dashboard-${new Date().toISOString().split('T')[0]}`
      const format = options.format || 'csv'
      
      if (format === 'csv') {
        exportToCSV(metricsData, filename)
      } else if (format === 'json') {
        exportToJSON(metricsData, filename)
      }
    } finally {
      isExporting.value = false
    }
  }

  return { isExporting, exportToCSV, exportToJSON, exportDashboard }
}
```

## ตัวอย่าง: Live Analytics Dashboard

```vue
<!-- pages/dashboard.vue -->
<template>
  <div class="dashboard">
    <!-- Header -->
    <header class="dashboard-header">
      <h1>Live Analytics Dashboard</h1>
      <div class="header-actions">
        <span :class="['connection-badge', connectionStatus]">
          {{ connectionLabels[connectionStatus] }}
        </span>
        <span class="last-updated" v-if="store.lastUpdated">
          อัพเดตล่าสุด: {{ formatTime(store.lastUpdated) }}
        </span>
        <button @click="exportData" :disabled="isExporting">
          {{ isExporting ? 'กำลัง Export...' : 'Export Data' }}
        </button>
      </div>
    </header>

    <!-- Alert Banner -->
    <div v-if="store.activeAlerts.length" class="alert-banner">
      <div 
        v-for="alert in store.activeAlerts.slice(0, 3)" 
        :key="alert.id"
        :class="['alert-item', `alert--${alert.level}`]"
      >
        <span>{{ alert.message }}</span>
        <button @click="store.acknowledgeAlert(alert.id)">✕</button>
      </div>
    </div>

    <!-- Metric Cards -->
    <div class="metric-grid">
      <div 
        v-for="metric in store.metricsList" 
        :key="metric.id"
        class="metric-card"
      >
        <div class="metric-header">
          <span class="metric-name">{{ metric.name }}</span>
          <span :class="['trend-badge', `trend--${metric.trend}`]">
            {{ metric.trend === 'up' ? '↑' : metric.trend === 'down' ? '↓' : '→' }}
          </span>
        </div>
        <div class="metric-value">
          <span class="value">{{ metric.value.toFixed(1) }}</span>
          <span class="unit">{{ metric.unit }}</span>
        </div>
        <div class="metric-change" :class="{ positive: metric.value > metric.previousValue }">
          {{ ((metric.value - metric.previousValue) / metric.previousValue * 100).toFixed(1) }}%
        </div>
      </div>
    </div>

    <!-- Charts -->
    <div class="charts-grid">
      <!-- CPU Chart -->
      <div class="chart-card">
        <RealtimeLineChart
          title="CPU Usage"
          :data="cpuHistory"
          color="#4CAF50"
          unit="%"
          :height="200"
        />
      </div>

      <!-- Memory Chart -->
      <div class="chart-card">
        <RealtimeLineChart
          title="Memory Usage"
          :data="memoryHistory"
          color="#2196F3"
          unit=" GB"
          :height="200"
        />
      </div>

      <!-- Request Rate -->
      <div class="chart-card">
        <RealtimeLineChart
          title="Request Rate"
          :data="requestHistory"
          color="#9C27B0"
          unit=" req/s"
          :height="200"
        />
      </div>
    </div>

    <!-- Gauges -->
    <div class="gauge-grid">
      <ApexGaugeChart
        v-if="cpuMetric"
        :value="cpuMetric.value"
        :max-value="100"
        unit="%"
        label="CPU"
        :thresholds="{ warning: 70, critical: 90 }"
      />
      <ApexGaugeChart
        v-if="memoryMetric"
        :value="memoryMetric.value"
        :max-value="32"
        unit="GB"
        label="Memory"
        :thresholds="{ warning: 24, critical: 29 }"
      />
    </div>

    <!-- Data Table -->
    <div class="table-section">
      <h2>Recent Events</h2>
      <table class="data-table">
        <thead>
          <tr>
            <th>เวลา</th>
            <th>Metric</th>
            <th>ค่า</th>
            <th>Status</th>
          </tr>
        </thead>
        <tbody>
          <tr v-for="event in recentEvents" :key="event.id">
            <td>{{ formatTime(new Date(event.timestamp)) }}</td>
            <td>{{ event.metric }}</td>
            <td>{{ event.value }}</td>
            <td :class="`status--${event.status}`">{{ event.status }}</td>
          </tr>
        </tbody>
      </table>
    </div>
  </div>
</template>

<script setup lang="ts">
import { computed, ref } from 'vue'
import { useDashboardStore } from '@/stores/dashboard'
import { useDashboardConnection } from '@/composables/useDashboardConnection'
import { useExport } from '@/composables/useExport'
import RealtimeLineChart from '@/components/dashboard/RealtimeLineChart.vue'
import ApexGaugeChart from '@/components/dashboard/ApexGaugeChart.vue'

const store = useDashboardStore()
const { status: connectionStatus } = useDashboardConnection()
const { isExporting, exportToCSV } = useExport()

const connectionLabels: Record<string, string> = {
  connecting: 'กำลังเชื่อมต่อ...',
  connected: 'เชื่อมต่อแล้ว',
  disconnected: 'ขาดการเชื่อมต่อ',
  error: 'เกิดข้อผิดพลาด'
}

const cpuMetric = computed(() => store.metrics.get('cpu'))
const memoryMetric = computed(() => store.metrics.get('memory'))

const cpuHistory = computed(() => cpuMetric.value?.history || [])
const memoryHistory = computed(() => memoryMetric.value?.history || [])
const requestHistory = computed(() => store.metrics.get('requests')?.history || [])

const recentEvents = ref<any[]>([])

const formatTime = (date: Date) => date.toLocaleTimeString('th-TH')

const exportData = () => {
  const data = store.metricsList.map(m => ({
    name: m.name,
    value: m.value,
    unit: m.unit,
    trend: m.trend,
    timestamp: new Date().toISOString()
  }))
  exportToCSV(data, `dashboard-${Date.now()}`)
}
</script>

<style scoped>
.dashboard {
  padding: 20px;
  background: #1a1a2e;
  min-height: 100vh;
  color: #fff;
}

.metric-grid {
  display: grid;
  grid-template-columns: repeat(auto-fit, minmax(200px, 1fr));
  gap: 16px;
  margin: 20px 0;
}

.metric-card {
  background: #16213e;
  border-radius: 12px;
  padding: 20px;
  border: 1px solid #0f3460;
}

.charts-grid {
  display: grid;
  grid-template-columns: repeat(auto-fit, minmax(350px, 1fr));
  gap: 16px;
}

.connection-badge.connected { color: #4CAF50; }
.connection-badge.disconnected { color: #f44336; }
.connection-badge.connecting { color: #FF9800; }
</style>
```

## Nuxt Server-Sent Events (SSE)

```typescript
// server/routes/events/stream.ts
export default defineEventHandler(async (event) => {
  const eventStream = createEventStream(event)

  // Simulate real-time data
  const interval = setInterval(async () => {
    await eventStream.push({
      data: JSON.stringify({
        type: 'metric_update',
        metricId: 'cpu',
        value: Math.random() * 100
      })
    })
  }, 1000)

  eventStream.onClosed(async () => {
    clearInterval(interval)
    await eventStream.close()
  })

  return eventStream.send()
})
```

## สรุป

Real-time Dashboard ต้องการ:
1. WebSocket/SSE สำหรับ data streaming
2. Efficient state management ด้วย Pinia
3. Chart libraries (Chart.js หรือ ApexCharts)
4. Data aggregation สำหรับ performance
5. Export functionality
6. Connection status management
