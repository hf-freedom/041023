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

function submitRecord() {
  if (!canSubmit.value) return

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

onMounted(() => {
  loadFromStorage()
})
</script>

<template>
  <h1>每日运动打卡</h1>
  
  <div class="container">
    <h2>添加运动记录</h2>
    
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

    <button 
      class="btn btn-primary" 
      :disabled="!canSubmit"
      @click="submitRecord"
    >
      打卡记录
    </button>
  </div>

  <div class="container">
    <h2>运动记录</h2>
    
    <div v-if="records.length === 0" class="empty-state">
      暂无运动记录，开始打卡吧！
    </div>
    
    <div v-else>
      <div 
        v-for="record in records" 
        :key="record.id" 
        class="record-item"
      >
        <div class="record-header">
          <span class="record-title">{{ record.name }}</span>
          <div>
            <span class="record-date">{{ formatDate(record.date) }}</span>
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
