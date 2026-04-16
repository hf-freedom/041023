<script setup lang="ts">
import { ref, computed, onMounted, watch } from 'vue'
import type { ExerciseRecord, RecordType, ExerciseOption } from './types'
import { PRESET_EXERCISES } from './types'

const STORAGE_KEY = 'exercise-records'
const today = new Date().toISOString().split('T')[0]

const records = ref<ExerciseRecord[]>([])
const selectedExercise = ref<string>('')
const customExerciseName = ref<string>('')
const recordType = ref<RecordType>('duration')
const duration = ref<number>(0)
const count = ref<number>(0)
const customUnit = ref<string>('')
const editingRecord = ref<ExerciseRecord | null>(null)
const exerciseFilter = ref<string>('')
const activeTab = ref<string>('today')
const currentMonth = ref(new Date())
const selectedCalendarDate = ref<string | null>(null)

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

const todayRecords = computed(() => {
  let filtered = records.value.filter(r => r.date === today)
  if (exerciseFilter.value) {
    filtered = filtered.filter(r => r.name === exerciseFilter.value)
  }
  return filtered
})

const todaySummary = computed(() => {
  const todayRecs = records.value.filter(r => r.date === today)
  const totalDuration = todayRecs.reduce((sum, r) => sum + (r.duration || 0), 0)
  const totalCount = todayRecs.length
  const exerciseTypes = new Set(todayRecs.map(r => r.name)).size
  return { totalDuration, totalCount, exerciseTypes }
})

const historyRecords = computed(() => {
  return [...records.value].sort((a, b) => b.createdAt - a.createdAt)
})

const uniqueExerciseNames = computed(() => {
  return [...new Set(records.value.map(r => r.name))]
})

const calendarDays = computed(() => {
  const year = currentMonth.value.getFullYear()
  const month = currentMonth.value.getMonth()
  const firstDay = new Date(year, month, 1)
  const lastDay = new Date(year, month + 1, 0)
  const startDay = firstDay.getDay()
  const days = []
  
  for (let i = 0; i < startDay; i++) {
    days.push({ date: null, hasRecords: false })
  }
  
  for (let i = 1; i <= lastDay.getDate(); i++) {
    const dateStr = `${year}-${String(month + 1).padStart(2, '0')}-${String(i).padStart(2, '0')}`
    const dayRecords = records.value.filter(r => r.date === dateStr)
    days.push({
      date: dateStr,
      hasRecords: dayRecords.length > 0,
      recordCount: dayRecords.length
    })
  }
  
  return days
})

const selectedDateRecords = computed(() => {
  if (!selectedCalendarDate.value) return []
  return records.value.filter(r => r.date === selectedCalendarDate.value)
})

const weekStats = computed(() => {
  const now = new Date()
  const dayOfWeek = now.getDay() || 7
  const startOfWeek = new Date(now)
  startOfWeek.setDate(now.getDate() - dayOfWeek + 1)
  startOfWeek.setHours(0, 0, 0, 0)
  
  const weekRecs = records.value.filter(r => {
    const recordDate = new Date(r.date)
    return recordDate >= startOfWeek
  })
  
  return {
    totalCount: weekRecs.length,
    totalDuration: weekRecs.reduce((sum, r) => sum + (r.duration || 0), 0)
  }
})

const monthStats = computed(() => {
  const year = new Date().getFullYear()
  const month = new Date().getMonth()
  const monthRecs = records.value.filter(r => {
    const d = new Date(r.date)
    return d.getFullYear() === year && d.getMonth() === month
  })
  
  return {
    totalCount: monthRecs.length,
    totalDuration: monthRecs.reduce((sum, r) => sum + (r.duration || 0), 0)
  }
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
  return `${year}-${month}-${day}`
}

function startEdit(record: ExerciseRecord) {
  editingRecord.value = { ...record }
  selectedExercise.value = record.name
  recordType.value = record.type
  duration.value = record.duration || 0
  count.value = record.count || 0
  customUnit.value = record.unit || ''
}

function cancelEdit() {
  editingRecord.value = null
  resetForm()
}

function submitRecord() {
  if (!canSubmit.value) return

  if (editingRecord.value) {
    const index = records.value.findIndex(r => r.id === editingRecord.value!.id)
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
    editingRecord.value = null
  } else {
    const record: ExerciseRecord = {
      id: generateId(),
      name: exerciseName.value.trim(),
      type: recordType.value,
      date: today,
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
}

function deleteRecord(id: string) {
  records.value = records.value.filter(r => r.id !== id)
  saveToStorage()
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
  currentMonth.value = new Date(
    currentMonth.value.getFullYear(),
    currentMonth.value.getMonth() - 1,
    1
  )
}

function nextMonth() {
  currentMonth.value = new Date(
    currentMonth.value.getFullYear(),
    currentMonth.value.getMonth() + 1,
    1
  )
}

function selectCalendarDate(date: string) {
  selectedCalendarDate.value = date
}

function closeCalendarDetail() {
  selectedCalendarDate.value = null
}

onMounted(() => {
  loadFromStorage()
})
</script>

<template>
  <h1>每日运动打卡</h1>

  <div class="stats-container">
    <div class="stat-card">
      <h4>今日统计</h4>
      <p>打卡次数: {{ todaySummary.totalCount }}</p>
      <p>总时长: {{ todaySummary.totalDuration }} 分钟</p>
      <p>运动类型: {{ todaySummary.exerciseTypes }} 种</p>
    </div>
    <div class="stat-card">
      <h4>本周统计</h4>
      <p>打卡次数: {{ weekStats.totalCount }}</p>
      <p>总时长: {{ weekStats.totalDuration }} 分钟</p>
    </div>
    <div class="stat-card">
      <h4>本月统计</h4>
      <p>打卡次数: {{ monthStats.totalCount }}</p>
      <p>总时长: {{ monthStats.totalDuration }} 分钟</p>
    </div>
  </div>
  
  <div class="container">
    <h2>{{ editingRecord ? '编辑记录' : '添加运动记录' }}</h2>
    
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

    <div class="button-row">
      <button 
        class="btn btn-primary" 
        :disabled="!canSubmit"
        @click="submitRecord"
      >
        {{ editingRecord ? '保存修改' : '打卡记录' }}
      </button>
      <button 
        v-if="editingRecord"
        class="btn btn-secondary"
        @click="cancelEdit"
      >
        取消编辑
      </button>
    </div>
  </div>

  <div class="tab-container">
    <button 
      class="tab-btn" 
      :class="{ active: activeTab === 'today' }"
      @click="activeTab = 'today'"
    >
      今日记录
    </button>
    <button 
      class="tab-btn" 
      :class="{ active: activeTab === 'history' }"
      @click="activeTab = 'history'"
    >
      历史记录
    </button>
    <button 
      class="tab-btn" 
      :class="{ active: activeTab === 'calendar' }"
      @click="activeTab = 'calendar'"
    >
      打卡日历
    </button>
  </div>

  <div class="container" v-if="activeTab === 'today'">
    <h2>今日运动记录</h2>
    
    <div class="filter-bar">
      <label>按运动类型筛选:</label>
      <select v-model="exerciseFilter">
        <option value="">全部</option>
        <option v-for="name in uniqueExerciseNames" :key="name" :value="name">
          {{ name }}
        </option>
      </select>
    </div>
    
    <div v-if="todayRecords.length === 0" class="empty-state">
      暂无运动记录，开始打卡吧！
    </div>
    
    <div v-else>
      <div 
        v-for="record in todayRecords" 
        :key="record.id" 
        class="record-item"
      >
        <div class="record-header">
          <span class="record-title">{{ record.name }}</span>
          <div>
            <button 
              class="edit-btn" 
              @click="startEdit(record)"
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

  <div class="container" v-if="activeTab === 'history'">
    <h2>历史记录</h2>
    
    <div v-if="historyRecords.length === 0" class="empty-state">
      暂无历史记录
    </div>
    
    <div v-else>
      <div 
        v-for="record in historyRecords" 
        :key="record.id" 
        class="record-item"
      >
        <div class="record-header">
          <span class="record-title">{{ record.name }}</span>
          <div>
            <span class="record-date">{{ formatDate(record.date) }}</span>
            <button 
              class="edit-btn" 
              @click="startEdit(record)"
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

  <div class="container" v-if="activeTab === 'calendar'">
    <h2>打卡日历</h2>
    
    <div class="calendar-header">
      <button @click="prevMonth" class="btn-nav">&lt;</button>
      <h3>
        {{ currentMonth.getFullYear() }}年{{ currentMonth.getMonth() + 1 }}月
      </h3>
      <button @click="nextMonth" class="btn-nav">&gt;</button>
    </div>
    
    <div class="calendar">
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
          'has-record': day.hasRecords,
          'selected': day.date === selectedCalendarDate
        }"
        @click="day.date && selectCalendarDate(day.date)"
      >
        {{ day.date ? day.date.split('-')[2] : '' }}
        <span v-if="day.hasRecords" class="record-dot"></span>
      </div>
    </div>
    
    <div v-if="selectedCalendarDate" class="calendar-detail">
      <div class="detail-header">
        <h4>{{ selectedCalendarDate }} 的记录</h4>
        <button @click="closeCalendarDetail" class="close-btn">&times;</button>
      </div>
      <div v-if="selectedDateRecords.length === 0" class="empty-state">
        当天没有运动记录
      </div>
      <div v-else>
        <div 
          v-for="record in selectedDateRecords" 
          :key="record.id" 
          class="record-item"
        >
          <div class="record-header">
            <span class="record-title">{{ record.name }}</span>
          </div>
          <div class="record-detail">
            {{ getRecordDetail(record) }}
          </div>
        </div>
      </div>
    </div>
  </div>
</template>