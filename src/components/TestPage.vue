<template>
  <section class="test-page">
    <div class="container">
          <div class="header-row">
            <h2 style="margin:0 auto; padding-bottom: 30px;">🕒 Timed Reading Test</h2>
          </div>

      


      <!-- Show instructions only before the first sample, and not after -->
      <p v-if="!state.started && state.fontIndex === 0 && !state.finished" class="instructions">
        TEST-TEST Read one paragraph at a time. Click <strong>Start Reading</strong> (or press <kbd>Space</kbd>) to begin the timer. When you're done, click <strong>Done Reading</strong> (or press <kbd>Space</kbd>) to record your time and move to the next font.
      </p>

      <div v-if="state.started" class="text-display" v-html="currentText" ></div>


      <div class="status" v-if="statusMessage" v-html="statusMessage"></div>

      <div class="controls">
        <button v-if="!state.finished" :class="['primary', state.running ? 'done' : 'start']" @click="toggleTimer">{{ buttonLabel }}</button>
        <button class="reset-btn" @click="goHome">Reset</button>
      </div>

      <div v-if="state.finished" class="results" v-html="resultsHtml"></div>
    </div>
  </section>
</template>

<script setup>
import { ref, reactive, computed, onMounted, onUnmounted } from 'vue'
const emit = defineEmits(['back-home'])

const fonts = ["OpenDyslexic", "Arial", "Verdana", "Comic Sans MS"]
const sampleText = "Reading is a skill that helps us learn, explore, and connect with the world. For some people, reading can be challenging, especially when letters seem to move or mix together. Finding the right font can make a big difference by improving comfort, focus, and speed. The goal is not to read faster, but to read with less effort and more understanding."

const state = reactive({ fontIndex: 0, running: false, startTime: null, finished: false, started: false })
// keyboard: spacebar toggles start/done
function onKeydown(e) {
  // ignore if typing in an input/textarea/select or contenteditable
  if (state.finished) return
  const target = e.target
  if (e.code !== 'Space' || e.repeat) return
  if (target && target instanceof HTMLElement) {
    const tag = target.tagName
    if (["INPUT", "TEXTAREA", "SELECT"].includes(tag) || target.isContentEditable) return
  }
  e.preventDefault()
  toggleTimer()
}

onMounted(() => window.addEventListener('keydown', onKeydown))
onUnmounted(() => window.removeEventListener('keydown', onKeydown))
const userTimes = reactive({})
const statusMessage = ref('')
const buttonLabel = ref('Start Reading')

const currentText = computed(() => {
  // only show the paragraph once the user has started the reading for the current font
  if (state.finished || !state.started) return ''
  const f = fonts[state.fontIndex]
  return `<p style="font-family:${f}; font-size:22px;">${sampleText}</p>`
})

function toggleTimer() {
  if (!state.running) {
    state.startTime = Date.now()
    state.running = true
    state.started = true
    buttonLabel.value = 'Done Reading'
    statusMessage.value = '⏳ Click "Done Reading" when finished.'
    return
  }

  // stopping
  if (!state.startTime) {
    statusMessage.value = 'You need to start the timer first.'
    return
  }
  const elapsed = Math.round(((Date.now() - state.startTime) / 1000) * 100) / 100
  const fontName = fonts[state.fontIndex]
  userTimes[fontName] = elapsed

  // advance
  state.fontIndex += 1
  state.running = false
  state.startTime = null
  buttonLabel.value = 'Start Reading'

  // hide the paragraph until the user starts the next font
  state.started = false
  if (state.fontIndex < fonts.length) {
    statusMessage.value = `✅ You have read <b>${fontName}</b> (${state.fontIndex} of ${fonts.length}).`
  } else {
    // finished
    state.finished = true
    statusMessage.value = `🏁 All done! Results below.`
  }
}

function resetApp() {
  for (const k of Object.keys(userTimes)) delete userTimes[k]
  state.fontIndex = 0
  state.running = false
  state.startTime = null
  state.finished = false
  state.started = false
  buttonLabel.value = 'Start Reading'
  statusMessage.value = "App reset. Click 'Start Reading' to begin!"
}

function goHome() {
  // Reset state locally and navigate back to home
  resetApp()
  emit('back-home')
}

const fontStyles = {
  'OpenDyslexic': "'OpenDyslexic', Arial, sans-serif",
  'Arial': 'Arial, sans-serif',
  'Verdana': 'Verdana, Geneva, sans-serif',
  'Comic Sans MS': '"Comic Sans MS", "Comic Sans", cursive, sans-serif'
}

const resultsHtml = computed(() => {
  if (!state.finished) return ''
  const entries = Object.entries(userTimes)
  if (entries.length === 0) return '<p>No results recorded.</p>'
  // align numbers in a column, always 2 decimals
  const summary = entries.map(([f, t]) =>
    `<span style="display:inline-block;min-width:140px;font-family:${fontStyles[f]};font-size:18px;">${f}</span>` +
    `<span style="display:inline-block;width:60px;text-align:right;font-variant-numeric:tabular-nums;font-size:18px;">${t.toFixed(2)}s</span>`
  ).join('<br>')
  // find min
  let bestFont = entries[0][0]
  let bestTime = entries[0][1]
  for (let i = 1; i < entries.length; i++) {
    if (entries[i][1] < bestTime) { bestTime = entries[i][1]; bestFont = entries[i][0] }
  }
  return `<div>${summary}<br><br>💡 Your fastest font: <b style="font-family:${fontStyles[bestFont]}; font-size:20px;">${bestFont}</b></div>`
})
</script>

<style scoped>
.container{ max-width:800px; margin:20px auto; padding:10px }
.header-row{ display:flex; align-items:center; gap:12px }
.header-row h2{ margin:0 }
.status{ margin:12px 0; font-weight:500; text-align: center; padding-top: 20px;}
.text-display{ border:1px solid #eee; padding:18px; border-radius:8px; background:#fff }
.controls{ position:fixed; left:0; right:0; bottom:20px; display:flex; justify-content:center; gap:12px; z-index:1000 }
.primary{ background:#4f46e5; color:white; border:none; padding:12px 22px; border-radius:999px; cursor:pointer; box-shadow:0 6px 18px rgba(79,70,229,0.18) }
.primary{ color:white; border:none; padding:12px 22px; border-radius:999px; cursor:pointer; box-shadow:0 6px 18px rgba(0,0,0,0.12) }
.primary.start{ background:#16a34a }
.primary.start:hover{ background:#15803d }
.primary.done{ background:#dc2626 }
.primary.done:hover{ background:#b91c1c }
.reset-btn{ background:#efefef; border:1px solid #ddd; padding:10px 16px; border-radius:999px; cursor:pointer }
.results{ margin-top:18px; padding:12px; background:#f9fafb; border-radius:8px; min-height:60px }
</style>
