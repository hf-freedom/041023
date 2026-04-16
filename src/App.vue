<script setup lang="ts">
import { ref, computed, onMounted, watch } from 'vue'
import type { ExerciseRecord, RecordType, ExerciseOption } from './types'
import { PRESET_EXERCISES } from './types'

const STORAGE_KEY = 'exercise-records'

const records = ref<ExerciseRecord[]>([])
const selectedExercise = ref<string>('')
const customExerciseName = ref<string>('')
const recordType = ref<RecordType>('duration')
const duration = ref<number>(0)
const count = ref<number>(0)
const customUnit = ref<string>('')

// 编辑模式
const editingRecordId = ref<string | null>(null)

// 筛选和视图状态
const filterExerciseType = ref<string>('')
const currentView = ref<'today' | 'history' | 'calendar' | 'stats'>('today')
const selectedDate = ref<string>('')
const currentCalendarMonth = ref<Date>(new Date())

// 历史记录筛选
const historyFilterMonth = ref<string>(() => {
  const now = new Date()
  return `${now.getFullYear()}-${String(now.getMonth() + 1).padStart(2, '0')}`
})

const exerciseOptions = computed(() => {
  return PRESET_EXERCISES.map(e => e.name)
})

const selectedExerciseInfo = computed<ExerciseOption | null>(() => {
  return PRESET_EXERCISES.find(e => e.name === selectedExercise.value) || null
})

const currentUnit = computed(() => {
  if (selectedExerciseInfo.value) {
    return selectedExerciseInfo.value.unit
  }
  if (customUnit.value) {
    return customUnit.value
  }
  return recordType.value === 'duration' ? '分钟' : '次'
})

const isCustomExercise = computed(() => {
  return selectedExercise.value === '__custom__'
})

const exerciseName = computed(() => {
  if (isCustomExercise.value) {
    return customExerciseName.value
  }
  return selectedExercise.value
})

const canSubmit = computed(() => {
  if (!exerciseName.value.trim()) return false
  if (recordType.value === 'duration') {
    return duration.value > 0
  }
  return count.value > 0
})

// 今日记录
const todayRecords = computed(() => {
  const today = new Date().toISOString().split('T')[0]
  return records.value.filter(r => r.date === today)
})

// 按运动类型筛选的今日记录
const filteredTodayRecords = computed(() => {
  if (!filterExerciseType.value) return todayRecords.value
  return todayRecords.value.filter(r => r.name === filterExerciseType.value)
})

// 今日汇总数据
const todaySummary = computed(() => {
  const today = new Date().toISOString().split('T')[0]
  const todayRecs = records.value.filter(r => r.date === today)
  
  let totalDuration = 0
  todayRecs.forEach(r => {
    if (r.type === 'duration' && r.duration) {
      totalDuration += r.duration
    }
  })
  
  const uniqueTypes = new Set(todayRecs.map(r => r.name))
  
  return {
    totalDuration,
    totalCount: todayRecs.length,
    uniqueTypesCount: uniqueTypes.size
  }
})

// 今日所有运动类型（用于筛选）
const todayExerciseTypes = computed(() => {
  const types = new Set<string>()
  todayRecords.value.forEach(r => types.add(r.name))
  return Array.from(types)
})

// 历史记录（按日期分组）
const groupedHistoryRecords = computed(() => {
  const groups: Record<string, ExerciseRecord[]> = {}
  
  let filteredRecords = records.value
  
  // 按月筛选
  if (historyFilterMonth.value) {
    filteredRecords = filteredRecords.filter(r => r.date.startsWith(historyFilterMonth.value))
  }
  
  filteredRecords.forEach(record => {
    if (!groups[record.date]) {
      groups[record.date] = []
    }
    groups[record.date].push(record)
  })
  
  // 按日期降序排序
  return Object.keys(groups)
    .sort((a, b) => b.localeCompare(a))
    .map(date => ({
      date,
      records: groups[date].sort((a, b) => b.createdAt - a.createdAt)
    }))
})

// 日历数据
const calendarDays = computed(() => {
  const year = currentCalendarMonth.value.getFullYear()
  const month = currentCalendarMonth.value.getMonth()
  
  const firstDay = new Date(year, month, 1)
  const lastDay = new Date(year, month + 1, 0)
  const daysInMonth = lastDay.getDate()
  const startDayOfWeek = firstDay.getDay()
  
  const days: Array<{ date: string; day: number; hasRecord: boolean; records: ExerciseRecord[] }> = []
  
  // 填充月初空白
  for (let i = 0; i < startDayOfWeek; i++) {
    days.push({ date: '', day: 0, hasRecord: false, records: [] })
  }
  
  // 填充日期
  for (let day = 1; day <= daysInMonth; day++) {
    const dateStr = `${year}-${String(month + 1).padStart(2, '0')}-${String(day).padStart(2, '0')}`
    const dayRecords = records.value.filter(r => r.date === dateStr)
    days.push({
      date: dateStr,
      day,
      hasRecord: dayRecords.length > 0,
      records: dayRecords
    })
  }
  
  return days
})

const calendarMonthLabel = computed(() => {
  const year = currentCalendarMonth.value.getFullYear()
  const month = currentCalendarMonth.value.getMonth() + 1
  return `${year}年${month}月`
})

// 周统计
const weekStats = computed(() => {
  const now = new Date()
  const dayOfWeek = now.getDay()
  const diff = now.getDate() - dayOfWeek + (dayOfWeek === 0 ? -6 : 1)
  const monday = new Date(now.setDate(diff))
  monday.setHours(0, 0, 0, 0)
  
  const sunday = new Date(monday)
  sunday.setDate(monday.getDate() + 6)
  sunday.setHours(23, 59, 59, 999)
  
  const weekRecords = records.value.filter(r => {
    const recordDate = new Date(r.date)
    return recordDate >= monday && recordDate <= sunday
  })
  
  let totalDuration = 0
  weekRecords.forEach(r => {
    if (r.type === 'duration' && r.duration) {
      totalDuration += r.duration
    }
  })
  
  return {
    totalCount: weekRecords.length,
    totalDuration
  }
})

// 月统计
const monthStats = computed(() => {
  const now = new Date()
  const year = now.getFullYear()
  const month = now.getMonth() + 1
  const monthPrefix = `${year}-${String(month).padStart(2, '0')}`
  
  const monthRecords = records.value.filter(r => r.date.startsWith(monthPrefix))
  
  let totalDuration = 0
  monthRecords.forEach(r => {
    if (r.type === 'duration' && r.duration) {
      totalDuration += r.duration
    }
  })
  
  return {
    totalCount: monthRecords.length,
    totalDuration
  }
})

// 选中日期的记录
const selectedDateRecords = computed(() => {
  if (!selectedDate.value) return []
  return records.value
    .filter(r => r.date === selectedDate.value)
    .sort((a, b) => b.createdAt - a.createdAt)
})

function onExerciseChange() {
  if (selectedExerciseInfo.value) {
    recordType.value = selectedExerciseInfo.value.type
  }
}

function generateId(): string {
  return Date.now().toString(36) + Math.random().toString(36).substr(2)
}

function formatDate(dateStr: string): string {
  const date = new Date(dateStr)
  const year = date.getFullYear()
  const month = String(date.getMonth() + 1).padStart(2, '0')
  const day = String(date.getDate()).padStart(2, '0')
  const hours = String(date.getHours()).padStart(2, '0')
  const minutes = String(date.getMinutes()).padStart(2, '0')
  return `${year}-${month}-${day} ${hours}:${minutes}`
}

function formatDateShort(dateStr: string): string {
  const date = new Date(dateStr)
  const month = String(date.getMonth() + 1).padStart(2, '0')
  const day = String(date.getDate()).padStart(2, '0')
  return `${month}-${day}`
}

function formatWeekDay(dateStr: string): string {
  const date = new Date(dateStr)
  const days = ['周日', '周一', '周二', '周三', '周四', '周五', '周六']
  return days[date.getDay()]
}

function submitRecord() {
  if (!canSubmit.value) return

  if (editingRecordId.value) {
    // 更新记录
    const index = records.value.findIndex(r => r.id === editingRecordId.value)
    if (index !== -1) {
      records.value[index] = {
        ...records.value[index],
        name: exerciseName.value.trim(),
        type: recordType.value,
        duration: recordType.value === 'duration' ? duration.value : undefined,
        count: recordType.value === 'count' ? count.value : undefined,
        unit: currentUnit.value
      }
    }
    editingRecordId.value = null
  } else {
    // 新建记录
    const record: ExerciseRecord = {
      id: generateId(),
      name: exerciseName.value.trim(),
      type: recordType.value,
      date: new Date().toISOString().split('T')[0],
      createdAt: Date.now()
    }

    if (recordType.value === 'duration') {
      record.duration = duration.value
      record.unit = currentUnit.value
    } else {
      record.count = count.value
      record.unit = currentUnit.value
    }

    records.value.unshift(record)
  }
  
  resetForm()
  saveToStorage()
}

function resetForm() {
  selectedExercise.value = ''
  customExerciseName.value = ''
  duration.value = 0
  count.value = 0
  customUnit.value = ''
  recordType.value = 'duration'
  editingRecordId.value = null
}

function editRecord(record: ExerciseRecord) {
  editingRecordId.value = record.id
  
  // 检查是否是预设运动
  const preset = PRESET_EXERCISES.find(e => e.name === record.name)
  if (preset) {
    selectedExercise.value = record.name
  } else {
    selectedExercise.value = '__custom__'
    customExerciseName.value = record.name
    customUnit.value = record.unit || ''
  }
  
  recordType.value = record.type
  
  if (record.type === 'duration') {
    duration.value = record.duration || 0
  } else {
    count.value = record.count || 0
  }
}

function cancelEdit() {
  resetForm()
}

function deleteRecord(id: string) {
  if (confirm('确定要删除这条记录吗？')) {
    records.value = records.value.filter(r => r.id !== id)
    saveToStorage()
  }
}

function getRecordDetail(record: ExerciseRecord): string {
  if (record.type === 'duration') {
    return `时长: ${record.duration} ${record.unit}`
  }
  return `数量: ${record.count} ${record.unit}`
}

function saveToStorage() {
  localStorage.setItem(STORAGE_KEY, JSON.stringify(records.value))
}

function loadFromStorage() {
  const data = localStorage.getItem(STORAGE_KEY)
  if (data) {
    try {
      records.value = JSON.parse(data)
    } catch {
      records.value = []
    }
  }
}

function prevMonth() {
  currentCalendarMonth.value = new Date(currentCalendarMonth.value.getFullYear(), currentCalendarMonth.value.getMonth() - 1, 1)
}

function nextMonth() {
  currentCalendarMonth.value = new Date(currentCalendarMonth.value.getFullYear(), currentCalendarMonth.value.getMonth() + 1, 1)
}

function selectDate(dateStr: string) {
  if (!dateStr) return
  selectedDate.value = dateStr
}

function closeDateDetail() {
  selectedDate.value = ''
}

onMounted(() => {
  loadFromStorage()
  
  // 初始化历史记录筛选月份为当前月
  const now = new Date()
  historyFilterMonth.value = `${now.getFullYear()}-${String(now.getMonth() + 1).padStart(2, '0')}`
})
</script>

<template>
  <h1>每日运动打卡</h1>
  
  <!-- 导航标签 -->
  <div class="nav-tabs">
    <button 
      class="nav-tab" 
      :class="{ active: currentView === 'today' }"
      @click="currentView = 'today'"
    >
      今日打卡
    </button>
    <button 
      class="nav-tab" 
      :class="{ active: currentView === 'history' }"
      @click="currentView = 'history'"
    >
      历史记录
    </button>
    <button 
      class="nav-tab" 
      :class="{ active: currentView === 'calendar' }"
      @click="currentView = 'calendar'"
    >
      打卡日历
    </button>
    <button 
      class="nav-tab" 
      :class="{ active: currentView === 'stats' }"
      @click="currentView = 'stats'"
    >
      统计数据
    </button>
  </div>

  <!-- 今日打卡视图 -->
  <template v-if="currentView === 'today'">
    <div class="container">
      <h2>{{ editingRecordId ? '编辑运动记录' : '添加运动记录' }}</h2>
      
      <div class="form-group">
        <label>选择运动项目</label>
        <select v-model="selectedExercise" @change="onExerciseChange">
          <option value="">请选择...</option>
          <option v-for="name in exerciseOptions" :key="name" :value="name">
            {{ name }}
          </option>
          <option value="__custom__">自定义运动...</option>
        </select>
        
        <div v-if="isCustomExercise" class="custom-input">
          <input 
            v-model="customExerciseName" 
            type="text" 
            placeholder="请输入运动名称"
          />
        </div>
      </div>

      <div class="form-group">
        <label>记录类型</label>
        <div class="radio-group">
          <label 
            class="radio-label" 
            :class="{ active: recordType === 'duration' }"
          >
            <input 
              type="radio" 
              v-model="recordType" 
              value="duration"
            />
            记录时长
          </label>
          <label 
            class="radio-label" 
            :class="{ active: recordType === 'count' }"
          >
            <input 
              type="radio" 
              v-model="recordType" 
              value="count"
            />
            记录次数/组数
          </label>
        </div>
      </div>

      <div class="form-group">
        <label v-if="recordType === 'duration'">运动时长</label>
        <label v-else>运动数量</label>
        
        <div class="input-row">
          <input 
            v-if="recordType === 'duration'"
            v-model.number="duration" 
            type="number" 
            min="1"
            placeholder="请输入时长"
          />
          <input 
            v-else
            v-model.number="count" 
            type="number" 
            min="1"
            placeholder="请输入数量"
          />
          <span>{{ currentUnit }}</span>
        </div>
        
        <div v-if="isCustomExercise" class="custom-input">
          <input 
            v-model="customUnit" 
            type="text" 
            placeholder="自定义单位（可选）"
          />
        </div>
      </div>

      <div class="button-group">
        <button 
          class="btn btn-primary" 
          :disabled="!canSubmit"
          @click="submitRecord"
        >
          {{ editingRecordId ? '保存修改' : '打卡记录' }}
        </button>
        <button 
          v-if="editingRecordId"
          class="btn btn-secondary"
          @click="cancelEdit"
        >
          取消
        </button>
      </div>
    </div>

    <!-- 今日汇总 -->
    <div v-if="todaySummary.totalCount > 0" class="container summary-container">
      <h2>今日汇总</h2>
      <div class="summary-grid">
        <div class="summary-item">
          <div class="summary-value">{{ todaySummary.totalDuration }}</div>
          <div class="summary-label">运动总时长(分钟)</div>
        </div>
        <div class="summary-item">
          <div class="summary-value">{{ todaySummary.totalCount }}</div>
          <div class="summary-label">总打卡次数</div>
        </div>
        <div class="summary-item">
          <div class="summary-value">{{ todaySummary.uniqueTypesCount }}</div>
          <div class="summary-label">运动类型数</div>
        </div>
      </div>
    </div>

    <div class="container">
      <div class="section-header">
        <h2>今日记录</h2>
        <select v-if="todayExerciseTypes.length > 0" v-model="filterExerciseType" class="filter-select">
          <option value="">全部类型</option>
          <option v-for="type in todayExerciseTypes" :key="type" :value="type">
            {{ type }}
          </option>
        </select>
      </div>
      
      <div v-if="filteredTodayRecords.length === 0" class="empty-state">
        {{ filterExerciseType ? '该类型暂无记录' : '暂无运动记录，开始打卡吧！' }}
      </div>
      
      <div v-else>
        <div 
          v-for="record in filteredTodayRecords" 
          :key="record.id" 
          class="record-item"
        >
          <div class="record-header">
            <span class="record-title">{{ record.name }}</span>
            <div class="record-actions">
              <span class="record-date">{{ formatDate(record.date) }}</span>
              <button 
                class="edit-btn" 
                @click="editRecord(record)"
              >
                编辑
              </button>
              <button 
                class="delete-btn" 
                @click="deleteRecord(record.id)"
              >
                删除
              </button>
            </div>
          </div>
          <div class="record-detail">
            {{ getRecordDetail(record) }}
          </div>
        </div>
      </div>
    </div>
  </template>

  <!-- 历史记录视图 -->
  <template v-if="currentView === 'history'">
    <div class="container">
      <div class="section-header">
        <h2>历史记录</h2>
        <input 
          type="month" 
          v-model="historyFilterMonth"
          class="month-input"
        />
      </div>
      
      <div v-if="groupedHistoryRecords.length === 0" class="empty-state">
        该月份暂无运动记录
      </div>
      
      <div v-else class="history-list">
        <div 
          v-for="group in groupedHistoryRecords" 
          :key="group.date" 
          class="history-group"
        >
          <div class="history-date-header">
            <span class="history-date">{{ group.date }}</span>
            <span class="history-weekday">{{ formatWeekDay(group.date) }}</span>
            <span class="history-count">{{ group.records.length }} 条记录</span>
          </div>
          <div 
            v-for="record in group.records" 
            :key="record.id" 
            class="record-item history-item"
          >
            <div class="record-header">
              <span class="record-title">{{ record.name }}</span>
              <div class="record-actions">
                <button 
                  class="edit-btn" 
                  @click="editRecord(record)"
                >
                  编辑
                </button>
                <button 
                  class="delete-btn" 
                  @click="deleteRecord(record.id)"
                >
                  删除
                </button>
              </div>
            </div>
            <div class="record-detail">
              {{ getRecordDetail(record) }}
            </div>
          </div>
        </div>
      </div>
    </div>
  </template>

  <!-- 打卡日历视图 -->
  <template v-if="currentView === 'calendar'">
    <div class="container">
      <h2>打卡日历</h2>
      
      <div class="calendar-header">
        <button class="calendar-nav" @click="prevMonth">&lt;</button>
        <span class="calendar-month">{{ calendarMonthLabel }}</span>
        <button class="calendar-nav" @click="nextMonth">&gt;</button>
      </div>
      
      <div class="calendar-grid">
        <div class="calendar-weekday">日</div>
        <div class="calendar-weekday">一</div>
        <div class="calendar-weekday">二</div>
        <div class="calendar-weekday">三</div>
        <div class="calendar-weekday">四</div>
        <div class="calendar-weekday">五</div>
        <div class="calendar-weekday">六</div>
        
        <div 
          v-for="(day, index) in calendarDays" 
          :key="index"
          class="calendar-day"
          :class="{ 
            'has-record': day.hasRecord,
            'empty': day.day === 0,
            'selected': day.date === selectedDate
          }"
          @click="day.date && selectDate(day.date)"
        >
          <template v-if="day.day > 0">
            <span class="day-number">{{ day.day }}</span>
            <span v-if="day.hasRecord" class="record-dot"></span>
          </template>
        </div>
      </div>
      
      <div class="calendar-legend">
        <span class="legend-item">
          <span class="legend-dot"></span>
          有打卡
        </span>
      </div>
    </div>

    <!-- 日期详情列表（页面内展示） -->
    <div v-if="selectedDate" class="container date-detail-container">
      <div class="date-detail-header">
        <h3>{{ selectedDate }} {{ formatWeekDay(selectedDate) }} 运动记录</h3>
        <button class="close-detail-btn" @click="closeDateDetail">&times;</button>
      </div>
      <div v-if="selectedDateRecords.length === 0" class="empty-state">
        该日期暂无运动记录
      </div>
      <div v-else>
        <div 
          v-for="record in selectedDateRecords" 
          :key="record.id" 
          class="record-item"
        >
          <div class="record-header">
            <span class="record-title">{{ record.name }}</span>
            <div class="record-actions">
              <button 
                class="edit-btn" 
                @click="editRecord(record); closeDateDetail(); currentView = 'today'"
              >
                编辑
              </button>
              <button 
                class="delete-btn" 
                @click="deleteRecord(record.id)"
              >
                删除
              </button>
            </div>
          </div>
          <div class="record-detail">
            {{ getRecordDetail(record) }}
          </div>
        </div>
      </div>
    </div>
  </template>

  <!-- 统计数据视图 -->
  <template v-if="currentView === 'stats'">
    <div class="container">
      <h2>本周统计</h2>
      <div class="stats-grid">
        <div class="stat-card">
          <div class="stat-value">{{ weekStats.totalCount }}</div>
          <div class="stat-label">运动总次数</div>
        </div>
        <div class="stat-card">
          <div class="stat-value">{{ weekStats.totalDuration }}</div>
          <div class="stat-label">累计时长(分钟)</div>
        </div>
      </div>
    </div>

    <div class="container">
      <h2>本月统计</h2>
      <div class="stats-grid">
        <div class="stat-card">
          <div class="stat-value">{{ monthStats.totalCount }}</div>
          <div class="stat-label">运动总次数</div>
        </div>
        <div class="stat-card">
          <div class="stat-value">{{ monthStats.totalDuration }}</div>
          <div class="stat-label">累计时长(分钟)</div>
        </div>
      </div>
    </div>
  </template>
</template>
