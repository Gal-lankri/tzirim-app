<template>
  <div class="container">
    <!-- Header -->
    <div class="header">
      <h1>צירים</h1>
      <p>אפליקציה לזיהוי צירי לידה</p>
      <div class="time-display">
        <div class="clock-icon"></div>
        {{ currentTime }}
      </div>
    </div>

    <!-- Instructions -->
    <div class="instructions">
      <h3>מתי להגיע לחדר לידה?</h3>
      <ul>
        <li>צירים סדירים כל 5 דקות או פחות</li>
        <li>כל ציר נמשך יותר מ-45 שניות</li>
        <li>צירים כל 4 דקות במשך שעה</li>
        <li>ירידת מים - יש להגיע לבדיקה בכל מקרה</li>
      </ul>
    </div>

    <!-- Timer Card -->
    <div class="card">
      <h2 style="text-align: center; margin-bottom: 20px; color: #667eea;">טיימר צירים</h2>
      
      <!-- Timer Display -->
      <div class="timer">
        <div v-if="!isTracking">00:00</div>
        <div v-else>{{ contractionTimer }}</div>
      </div>
      
      <!-- Status Message -->
      <div v-if="statusMessage" :class="['status', statusClass]">
        {{ statusMessage }}
      </div>
      
      <!-- Control Buttons -->
      <div v-if="!isTracking">
        <button @click="startContraction" class="btn">
          התחלת ציר
        </button>
        <button @click="recordWaterBreaking" class="btn secondary">
          ירידת מים
        </button>
      </div>
      
      <div v-else>
        <button @click="endContraction" class="btn success">
          סיום ציר
        </button>
        <button @click="cancelContraction" class="btn danger">
          ביטול
        </button>
      </div>
    </div>

    <!-- Recent Contractions -->
    <div class="card">
      <h3 style="margin-bottom: 16px; color: #667eea;">צירים אחרונים</h3>
      <div v-if="contractions.length === 0" style="text-align: center; color: #666; padding: 20px;">
        עדיין לא נרשמו צירים
      </div>
      <div v-else class="contraction-list">
        <div v-for="(contraction, index) in recentContractions" :key="index" class="contraction-item">
          <div>
            <div class="contraction-time">{{ formatTime(contraction.startTime) }}</div>
            <div class="contraction-duration">{{ contraction.duration }} שניות</div>
          </div>
          <div style="color: #999; font-size: 12px;">
            {{ formatTimeAgo(contraction.startTime) }}
          </div>
        </div>
      </div>
      
      <button v-if="contractions.length > 0" @click="clearHistory" class="btn secondary" style="margin-top: 16px;">
        ניקוי היסטוריה
      </button>
    </div>

    <!-- Statistics -->
    <div v-if="contractions.length > 0" class="card">
      <h3 style="margin-bottom: 16px; color: #667eea;">סטטיסטיקות</h3>
      <div style="display: grid; grid-template-columns: 1fr 1fr; gap: 16px;">
        <div style="text-align: center;">
          <div style="font-size: 24px; font-weight: 700; color: #667eea;">{{ averageFrequency }}</div>
          <div style="font-size: 14px; color: #666;">דקות בין צירים</div>
        </div>
        <div style="text-align: center;">
          <div style="font-size: 24px; font-weight: 700; color: #667eea;">{{ averageDuration }}</div>
          <div style="font-size: 14px; color: #666;">שניות ציר ממוצע</div>
        </div>
      </div>
    </div>
  </div>
</template>

<script>
import { ref, computed, onMounted, onUnmounted } from 'vue'

export default {
  name: 'App',
  setup() {
    const contractions = ref([])
    const isTracking = ref(false)
    const startTime = ref(null)
    const currentTime = ref('00:00')
    const contractionTimer = ref('00:00')
    const statusMessage = ref('')
    const statusClass = ref('')
    const timerInterval = ref(null)
    const contractionInterval = ref(null)

    // Load data from localStorage
    onMounted(() => {
      const saved = localStorage.getItem('tzirim-contractions')
      if (saved) {
        const parsed = JSON.parse(saved)
        // Convert date strings back to Date objects
        contractions.value = parsed.map(contraction => ({
          ...contraction,
          startTime: new Date(contraction.startTime),
          endTime: new Date(contraction.endTime)
        }))
      }
      
      // Start clock
      updateClock()
      timerInterval.value = setInterval(updateClock, 1000)
    })

    onUnmounted(() => {
      if (timerInterval.value) {
        clearInterval(timerInterval.value)
      }
      if (contractionInterval.value) {
        clearInterval(contractionInterval.value)
      }
    })

    const updateClock = () => {
      const now = new Date()
      const hours = String(now.getHours()).padStart(2, '0')
      const minutes = String(now.getMinutes()).padStart(2, '0')
      currentTime.value = `${hours}:${minutes}`
    }

    const updateContractionTimer = () => {
      if (!startTime.value) return
      
      const now = new Date()
      const elapsed = Math.floor((now - startTime.value) / 1000)
      const minutes = Math.floor(elapsed / 60)
      const seconds = elapsed % 60
      
      contractionTimer.value = `${String(minutes).padStart(2, '0')}:${String(seconds).padStart(2, '0')}`
    }

    const startContraction = () => {
      isTracking.value = true
      startTime.value = new Date()
      contractionTimer.value = '00:00'
      statusMessage.value = 'התחלת ציר - לחצי על "סיום ציר" כשהציר נגמר'
      statusClass.value = 'warning'
      
      // Start contraction timer
      updateContractionTimer()
      contractionInterval.value = setInterval(updateContractionTimer, 1000)
    }

    const endContraction = () => {
      if (!startTime.value) return
      
      const endTime = new Date()
      const duration = Math.round((endTime - startTime.value) / 1000)
      
      const contraction = {
        startTime: startTime.value,
        endTime: endTime,
        duration: duration
      }
      
      contractions.value.unshift(contraction)
      
      // Save to localStorage
      localStorage.setItem('tzirim-contractions', JSON.stringify(contractions.value))
      
      // Check if should go to hospital
      checkHospitalRecommendation()
      
      // Reset
      isTracking.value = false
      startTime.value = null
      statusMessage.value = ''
      statusClass.value = ''
      
      // Clear contraction timer
      if (contractionInterval.value) {
        clearInterval(contractionInterval.value)
        contractionInterval.value = null
      }
    }

    const cancelContraction = () => {
      isTracking.value = false
      startTime.value = null
      statusMessage.value = ''
      statusClass.value = ''
      
      // Clear contraction timer
      if (contractionInterval.value) {
        clearInterval(contractionInterval.value)
        contractionInterval.value = null
      }
    }

    const recordWaterBreaking = () => {
      statusMessage.value = 'ירידת מים! יש להגיע לבדיקת רופא מיד!'
      statusClass.value = 'danger'
      
      // Clear message after 10 seconds
      setTimeout(() => {
        statusMessage.value = ''
        statusClass.value = ''
      }, 10000)
    }

    const checkHospitalRecommendation = () => {
      if (contractions.value.length < 3) return
      
      const recent = contractions.value.slice(0, 3)
      const now = new Date()
      
      // Check if contractions are every 4 minutes or less
      let shouldGo = true
      for (let i = 1; i < recent.length; i++) {
        const timeDiff = (recent[i-1].startTime - recent[i].startTime) / 1000 / 60 // minutes
        if (timeDiff > 4) {
          shouldGo = false
          break
        }
      }
      
      // Check if contractions last more than 45 seconds
      const longContractions = recent.filter(c => c.duration > 45)
      
      if (shouldGo && longContractions.length >= 2) {
        statusMessage.value = 'צירים כל 4 דקות! יש להגיע לחדר לידה!'
        statusClass.value = 'danger'
      } else if (recent[0].duration > 45) {
        statusMessage.value = 'ציר ארוך! יש לעקוב אחר התדירות'
        statusClass.value = 'warning'
      }
    }

    const clearHistory = () => {
      if (confirm('האם למחוק את כל ההיסטוריה?')) {
        contractions.value = []
        localStorage.removeItem('tzirim-contractions')
        statusMessage.value = ''
        statusClass.value = ''
      }
    }

    const recentContractions = computed(() => {
      return contractions.value.slice(0, 10)
    })

    const averageFrequency = computed(() => {
      if (contractions.value.length < 2) return '-'
      
      let totalMinutes = 0
      let count = 0
      
      for (let i = 1; i < contractions.value.length; i++) {
        const timeDiff = (contractions.value[i-1].startTime - contractions.value[i].startTime) / 1000 / 60
        totalMinutes += timeDiff
        count++
      }
      
      return Math.round(totalMinutes / count)
    })

    const averageDuration = computed(() => {
      if (contractions.value.length === 0) return '-'
      
      const total = contractions.value.reduce((sum, c) => sum + c.duration, 0)
      return Math.round(total / contractions.value.length)
    })

    const formatTime = (date) => {
      return date.toLocaleTimeString('he-IL', { 
        hour: '2-digit', 
        minute: '2-digit',
        hour12: false 
      })
    }

    const formatTimeAgo = (date) => {
      const now = new Date()
      const diff = Math.round((now - date) / 1000 / 60) // minutes
      
      if (diff < 1) return 'עכשיו'
      if (diff < 60) return `לפני ${diff} דקות`
      if (diff < 1440) return `לפני ${Math.round(diff / 60)} שעות`
      return `לפני ${Math.round(diff / 1440)} ימים`
    }

    return {
      contractions,
      isTracking,
      currentTime,
      contractionTimer,
      statusMessage,
      statusClass,
      recentContractions,
      averageFrequency,
      averageDuration,
      startContraction,
      endContraction,
      cancelContraction,
      recordWaterBreaking,
      clearHistory,
      formatTime,
      formatTimeAgo
    }
  }
}
</script> 