<template>
  <div class="bg-gray-50 text-gray-800 font-sans p-4 min-h-screen flex items-center justify-center">
    <div class="w-full max-w-md bg-white shadow-xl rounded-xl p-6 space-y-6">

      <!-- Step 0: Task -->
      <div v-if="step === 0" class="space-y-4 animate-fade">
        <label for="task" class="font-medium block">What are you working on?</label>
        <input id="task" ref="taskInput" v-model="task" @keyup.enter="nextStep"
               class="w-full p-2 border rounded-md">
        <button @click="nextStep" class="w-full bg-yellow-300 hover:bg-yellow-400 py-2 rounded-md font-medium">Start</button>
      </div>

      <!-- Questions -->
      <div v-else-if="step <= questions.length" class="space-y-4 animate-fade">
        <label :for="'q' + step" class="font-medium block">{{ questions[step - 1] }}</label>
        <input :id="'q' + step" ref="questionInput" v-model="answers[step - 1]" @keyup.enter="nextStep"
               class="w-full p-2 border rounded-md">
        <div class="flex gap-2">
          <button @click="nextStep" class="flex-1 bg-yellow-300 hover:bg-yellow-400 py-2 rounded-md font-medium">Continue</button>
          <button @click="skipStep" class="flex-1 bg-gray-100 hover:bg-gray-200 py-2 rounded-md font-medium">Skip</button>
        </div>
      </div>

      <!-- Summary View -->
      <div v-else-if="mode === 'summary'" class="space-y-4 animate-fade">
        <p class="text-xl font-semibold text-green-700 animate-pop">{{ summaryMessage }}</p>
        <div class="bg-green-50 border border-green-200 rounded-md p-4 space-y-4 shadow-sm">
          <p><span class="italic text-sm text-gray-600">What are you working on?</span><br>
            <strong>{{ task }}</strong></p>
          <div v-for="(question, index) in questions" :key="index">
            <p class="italic text-sm text-gray-600">{{ question }}</p>
            <p class="font-bold">{{ answers[index]?.trim() || '(skipped)' }}</p>
          </div>
        </div>

        <!-- Timer Selection or Active Timer Bar -->
        <div v-if="!timerActive" class="space-y-2">
          <p class="text-sm text-gray-500 mt-4">Pick a timer to get started:</p>
          <div class="flex gap-2">
            <button @click="startTimer(5)" class="flex-1 bg-blue-100 hover:bg-blue-200 py-2 rounded-md">5 min</button>
            <button @click="startTimer(15)" class="flex-1 bg-blue-100 hover:bg-blue-200 py-2 rounded-md">15 min</button>
            <button @click="startTimer(30)" class="flex-1 bg-blue-100 hover:bg-blue-200 py-2 rounded-md">30 min</button>
          </div>
        </div>

        <div v-else class="flex items-center justify-between bg-blue-50 border border-blue-200 px-4 py-3 rounded-md mt-4">
          <div class="text-blue-700 font-semibold text-lg">{{ timeDisplay }}</div>
          <div class="flex gap-2">
            <button @click="toggleTimer" class="text-sm px-3 py-1 bg-yellow-300 hover:bg-yellow-400 rounded-md font-medium">
              {{ running ? 'Pause' : 'Resume' }}
            </button>
            <button @click="confirmStop" class="text-sm px-3 py-1 bg-red-200 hover:bg-red-300 text-red-800 rounded-md font-medium">
              Stop
            </button>
            <button @click="mode = 'timer'" class="text-sm px-3 py-1 bg-green-200 hover:bg-green-300 text-green-800 rounded-md font-medium">
              Focus
            </button>
          </div>
        </div>
      </div>

      <!-- Timer View -->
      <div v-else-if="mode === 'timer'" class="space-y-6 animate-fade text-center">
        <p class="text-xl font-semibold">⏱ Focus Timer</p>
        <div class="text-5xl font-bold text-blue-600 tracking-wider">{{ timeDisplay }}</div>
        <div class="flex gap-2 justify-center">
          <button @click="toggleTimer" class="bg-yellow-300 hover:bg-yellow-400 px-4 py-2 rounded-md font-medium">
            {{ running ? 'Pause' : 'Resume' }}
          </button>
          <button @click="resetTimer" class="bg-gray-200 hover:bg-gray-300 px-4 py-2 rounded-md font-medium">Reset</button>
        </div>
        <button @click="mode = 'summary'" class="mt-4 underline text-blue-500 hover:text-blue-700 text-sm">
          ← Back to Summary
        </button>
      </div>

    </div>
  </div>
</template>

<script setup>
import { ref, computed, onMounted, nextTick } from 'vue'

const step = ref(0)
const task = ref('')
const questions = ref([
  "What’s your energy level today?",
  "How are you feeling in a few words?",
  "What are your top priorities today?",
  "What’s in your way?",
  "What’s one tiny action you could take right now?"
])
const answers = ref(["", "", "", "", ""])
const mode = ref('summary')
const summaryMessage = ref("")
const intervalId = ref(null)
const duration = ref(0)
const timeLeft = ref(0)
const running = ref(false)

const timeDisplay = computed(() => {
  const min = Math.floor(timeLeft.value / 60).toString().padStart(2, '0')
  const sec = (timeLeft.value % 60).toString().padStart(2, '0')
  return `${min}:${sec}`
})

const timerActive = computed(() => duration.value > 0)

function nextStep() {
  if (step.value === 0 && !task.value.trim()) {
    alert("Please enter what you're working on.")
    return
  }
  step.value++
  if (step.value > questions.value.length) setRandomSummaryMessage()
}

function skipStep() {
  answers.value[step.value - 1] = ""
  step.value++
  if (step.value > questions.value.length) setRandomSummaryMessage()
}

function setRandomSummaryMessage() {
  const options = [
    "You’re doing it!",
    "You're starting now.",
    "This is your moment.",
    "Momentum looks good on you.",
    "Little steps, big moves."
  ]
  summaryMessage.value = options[Math.floor(Math.random() * options.length)]
}

function focusInput() {
  nextTick(() => {
    const input = document.querySelector(step.value === 0 ? '#task' : `#q${step.value}`)
    if (input) input.focus()
  })
}

function startTimer(minutes) {
  clearTimer()
  duration.value = minutes * 60
  timeLeft.value = duration.value
  running.value = true
  intervalId.value = setInterval(tick, 1000)
  mode.value = 'timer'
}

function tick() {
  if (!running.value) return
  if (timeLeft.value > 0) {
    timeLeft.value--
  } else {
    clearTimer()
    alert("⏰ Time's up! Great job focusing.")
  }
}

function toggleTimer() {
  running.value = !running.value
}

function resetTimer() {
  clearTimer()
  timeLeft.value = duration.value
  running.value = false
}

function confirmStop() {
  if (confirm("Are you sure you want to stop this timer?")) {
    clearTimer()
    duration.value = 0
    timeLeft.value = 0
    running.value = false
  }
}

function clearTimer() {
  if (intervalId.value) {
    clearInterval(intervalId.value)
    intervalId.value = null
  }
}

onMounted(focusInput)
</script>

<style scoped>
.animate-fade {
  animation: fadeIn 0.4s ease;
}
.animate-pop {
  animation: popIn 0.5s ease;
}
@keyframes fadeIn {
  from { opacity: 0; transform: translateY(10px); }
  to { opacity: 1; transform: translateY(0); }
}
@keyframes popIn {
  0% { opacity: 0; transform: scale(0.9); }
  100% { opacity: 1; transform: scale(1); }
}
</style>
