<template>
  <div class="breathetrack-main">
    <header class="breathetrack-header">
      <h1>BreatheTrack</h1>
      <nav>
        <button
          :class="{ active: section === 'test' }"
          @click="section = 'test'"
        >
          BOLT Test
        </button>
        <button
          :class="{ active: section === 'history' }"
          @click="section = 'history'"
        >
          Results
        </button>
        <button
          :class="{ active: section === 'education' }"
          @click="section = 'education'"
        >
          Learn
        </button>
      </nav>
    </header>

    <main>
      <section v-if="section === 'test'" class="bolt-section">
        <h2>BOLT Test</h2>
        <ol class="bolt-instructions">
          <li>Sit calmly and breathe normally for several minutes.</li>
          <li>Take a normal (not deep) breath in, and out.</li>
          <li>
            Pinch your nose with your fingers after exhaling and start the timer.
          </li>
          <li>
            Hold your breath until you feel the first definite desire to breathe
            (diaphragm involuntary movement).
          </li>
          <li>Release your nose and stop the timer. Breathe normally.</li>
        </ol>
        <div class="timer-area">
          <button
            class="primary"
            @click="startTimer"
            :disabled="timerActive"
          >
            Start Test
          </button>
          <div class="timer-display" :class="{ active: timerActive }">
            <span>{{ timerDisplay }}</span>
          </div>
          <button
            v-if="timerActive"
            class="accent"
            @click="stopTimer"
          >
            Stop & Record
          </button>
        </div>
        <div v-if="latestScore !== null" class="latest-score">
          <h3>Your Latest BOLT Score: <span>{{ latestScore }}s</span></h3>
          <p>{{ boltScoreFeedback(latestScore) }}</p>
        </div>
        <div v-if="scores.length > 1" class="progress-graph">
          <h4>Progress Over Time</h4>
          <bolt-graph :scores="scores" />
        </div>
      </section>

      <section v-if="section === 'history'" class="history-section">
        <h2>BOLT Test History</h2>
        <ul v-if="scores.length > 0" class="history-list">
          <li v-for="(score, i) in scores.slice().reverse()" :key="i">
            {{ formatHistoryDate(scores.length - 1 - i) }} — <b>{{ score }}</b> seconds
          </li>
        </ul>
        <p v-else>No BOLT test results yet.</p>
        <div v-if="scores.length > 1" class="progress-graph">
          <h4>Progress Chart</h4>
          <bolt-graph :scores="scores" />
        </div>
      </section>

      <section v-if="section === 'education'" class="education-section">
        <h2>Learn About the BOLT Test</h2>
        <div class="edu-content">
          <h3>What is the BOLT Test?</h3>
          <p>
            The Body Oxygen Level Test (BOLT) measures the length of a comfortable breath-hold after exhalation.
          </p>
          <h3>Why is BOLT Important?</h3>
          <ul>
            <li>Assesses your breathing efficiency and potential for functional breathing issues.</li>
            <li>Helps track progress in breathing rehabilitation and athletic training.</li>
            <li>BOLT scores are linked to overall respiratory health and CO<sub>2</sub> tolerance.</li>
          </ul>
          <h3>Tips to Improve Your BOLT Score:</h3>
          <ul>
            <li>Practice nose breathing in daily life and during exercise.</li>
            <li>Focus on calm, light breathing. Avoid over-breathing (hyperventilation).</li>
            <li>Try breathing exercises designed to raise CO<sub>2</sub> tolerance carefully.</li>
          </ul>
          <h3>Resources</h3>
          <ul>
            <li>
              <a href="https://www.oxygenadvantage.com/" rel="noopener" target="_blank">Oxygen Advantage</a>
            </li>
            <li>
              <a href="https://buteykoclinic.com/" rel="noopener" target="_blank">Buteyko Clinic</a>
            </li>
          </ul>
        </div>
      </section>
    </main>
  </div>
</template>

<script setup lang="ts">
// PUBLIC_INTERFACE
/**
 * Main container for BreatheTrack app.
 * Handles test guidance, records, progress, and educational content tabs.
 */
import { ref, computed, onBeforeUnmount } from 'vue'

// Simple chart component for line graph
const BoltGraph = {
  name: 'BoltGraph',
  props: {
    scores: {
      type: Array,
      required: true
    }
  },
  template: `
    <svg :width="svgWidth" :height="svgHeight" class="bolt-graph-svg">
      <polyline
        :points="polylinePoints"
        :stroke="primary"
        stroke-width="3"
        fill="none"
      />
      <circle
        v-for="(y, i) in circleYs"
        :key="i"
        :cx="leftPad + i * stepX"
        :cy="y"
        r="6"
        :fill="accent"
      />
    </svg>
  `,
  computed: {
    svgWidth() { return 320 },
    svgHeight() { return 100 },
    leftPad() { return 32 },
    stepX() { return (this.svgWidth - this.leftPad * 2) / (this.scores.length - 1 || 1) },
    minScore() { return Math.min(...this.scores) },
    maxScore() { return Math.max(...this.scores) },
    circleYs() {
      const min = Math.min(...this.scores)
      const max = Math.max(...this.scores)
      const range = max - min || 1
      return this.scores.map(score =>
        80 - ((score - min) / range) * 60 + 10
      )
    },
    polylinePoints() {
      return this.scores.map((score, i) => {
        const x = this.leftPad + i * this.stepX
        const min = Math.min(...this.scores)
        const max = Math.max(...this.scores)
        const range = max - min || 1
        const y = 80 - ((score - min) / range) * 60 + 10
        return `${x},${y}`
      }).join(' ')
    },
    primary() { return "#1976D2" },
    accent() { return "#43A047" }
  }
}

const section = ref('test')
const timer = ref(0)
const timerActive = ref(false)
const timerId = ref(null)
const scores = ref(loadScores())
const startedAt = ref(0)

/**
 * Gets the latest BOLT score or null
 */
const latestScore = computed(() =>
  scores.value.length > 0 ? scores.value[scores.value.length - 1] : null
)

function timerDisplayValue(val) {
  const sec = Math.floor(val / 10)
  const d10 = val % 10
  return `${sec}.${d10}s`
}

const timerDisplay = computed(() => timerDisplayValue(timer.value))

function startTimer() {
  timer.value = 0
  startedAt.value = Date.now()
  timerActive.value = true
  timerId.value = setInterval(() => {
    timer.value++
  }, 100)
}

function stopTimer() {
  timerActive.value = false
  clearInterval(timerId.value)
  // Calculate score in seconds (1 decimal)
  const heldSec = Math.round((timer.value) / 10)
  if (heldSec > 0) {
    scores.value = [...scores.value, heldSec]
    saveScores(scores.value)
  }
  timer.value = 0
}

onBeforeUnmount(() => {
  if (timerId.value) clearInterval(timerId.value)
})

function saveScores(list) {
  try {
    localStorage.setItem('bolt_scores', JSON.stringify(list))
  } catch { /* non-blocking */}
}

function loadScores() {
  try {
    const arr = JSON.parse(localStorage.getItem('bolt_scores') || '[]')
    if (Array.isArray(arr)) return arr
    return []
  } catch {
    return []
  }
}

function formatHistoryDate(idx) {
  // Use today's date minus (scores.length-1-idx) days.
  const daysAgo = scores.value.length - 1 - idx
  const date = new Date()
  date.setDate(date.getDate() - daysAgo)
  return date.toLocaleDateString()
}

function boltScoreFeedback(score) {
  if (score >= 20) return 'Excellent! Indicative of functional breathing.'
  if (score >= 10) return 'Room for improvement. Practice breathing exercises for better results.'
  return "Low BOLT score, consider consulting educational resources."
}
</script>

<style scoped>
.breathetrack-main {
  background: #fff;
  color: #222;
  border-radius: 14px;
  box-shadow: 0 8px 24px rgba(25, 118, 210, 0.10);
  max-width: 430px;
  margin: 2rem auto;
  padding: 2rem;
  font-family: inherit;
}
.breathetrack-header {
  margin-bottom: 2rem;
  text-align: center;
}
.breathetrack-header h1 {
  color: #1976D2;
  letter-spacing: 0.03em;
  margin-bottom: 0.3em;
}
.breathetrack-header nav button {
  background: none;
  border: none;
  color: #1976D2;
  padding: 0.6em 1.2em;
  font-size: 1rem;
  border-radius: 4px 4px 0 0;
  transition: background .2s;
  cursor: pointer;
  margin: 0 0.1em;
}
.breathetrack-header nav button.active, .breathetrack-header nav button:hover {
  background: #E3F2FD;
  font-weight: 600;
}
.bolt-section, .history-section, .education-section {
  margin-top: 1.5rem;
}
.bolt-instructions {
  color: #333;
  background: #F9F9F9;
  border-radius: 8px;
  padding: 1em 1.5em;
  margin-bottom: 1em;
  font-size: 0.98em;
  list-style-type: decimal;
}
.timer-area {
  display: flex;
  flex-direction: column;
  align-items: center;
  margin: 1.2em 0;
}
.timer-display {
  font-size: 2.2rem;
  font-weight: 700;
  color: #1976D2;
  background: #E3F2FD;
  margin: 1em 0;
  padding: 0.45em 1.7em;
  border-radius: 1em;
  min-width: 120px;
  transition: background 0.2s;
}
.timer-display.active {
  background: #BBDEFB;
}
.primary {
  background: #1976D2;
  color: #fff;
  border: none;
  border-radius: 999px;
  font-size: 1.12rem;
  font-weight: bold;
  padding: 0.7em 1.7em;
  margin-bottom: 0.75em;
  cursor: pointer;
  box-shadow: 0 2px 11px 0 rgba(25,118,210,0.17);
}
.primary:disabled {
  opacity: 0.4;
  cursor: not-allowed;
}
.accent {
  background: #43A047;
  color: #fff;
  border: none;
  border-radius: 999px;
  font-size: 1.07rem;
  padding: 0.6em 1.3em;
  margin-top: 0.8em;
  cursor: pointer;
}
.latest-score {
  margin-top: 2em;
  text-align: center;
}
.latest-score h3 span {
  color: #43A047;
  font-size: 1.42em;
}
.progress-graph {
  margin-top: 1.4em;
  text-align: center;
}
.bolt-graph-svg {
  width: 100%;
  height: 100px;
  margin: 0.8em 0;
}
.history-list {
  list-style: none;
  padding: 0;
  margin-top: 1em;
}
.history-list li {
  margin-bottom: 0.6em;
  font-size: 1.05em;
  color: #1976D2;
}
.edu-content {
  background: #FAFAFA;
  border-radius: 8px;
  padding: 1.5em 1.1em;
  margin-top: 1em;
  color: #252525;
  line-height: 1.7;
}
.edu-content h3 {
  color: #1976D2;
  margin: 0.7em 0 0.3em 0;
  font-size: 1.12em;
}
.edu-content ul {
  margin-bottom: 0.95em;
}
@media (max-width: 700px) {
  .breathetrack-main {
    padding: 1rem 0.2rem 2.3rem 0.2rem;
    margin: 0.8rem;
    min-width: 0;
  }
  .bolt-graph-svg {
    width: 98vw;
    min-width: 0;
  }
}
</style>
