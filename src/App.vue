<script setup>
import { computed, onBeforeUnmount, ref } from 'vue'

import { tasks } from './data/tasks'

const levelImages = {
  3: new URL('./assets/level3.png', import.meta.url).href,
  4: new URL('./assets/level4.png', import.meta.url).href,
  5: new URL('./assets/level5.png', import.meta.url).href,
}

const typeImages = {
  drawing: new URL('./assets/drawing.png', import.meta.url).href,
  performance: new URL('./assets/performance.png', import.meta.url).href,
  talk: new URL('./assets/talk.png', import.meta.url).href,
}

const levelCards = [
  { value: 3, title: 'Level 3', subtitle: 'Einfach', image: levelImages[3] },
  { value: 4, title: 'Level 4', subtitle: 'Mittel', image: levelImages[4] },
  { value: 5, title: 'Level 5', subtitle: 'Schwer', image: levelImages[5] },
]

const typeCards = [
  { value: 'drawing', title: 'Zeichnen', detail: 'Papier & Stift', image: typeImages.drawing },
  { value: 'performance', title: 'Pantomime', detail: 'Nur Gesten', image: typeImages.performance },
  { value: 'talk', title: 'Umschreiben', detail: 'Ohne verbotene Woerter', image: typeImages.talk },
]

const selectedLevel = ref(null)
const selectedType = ref(null)
const currentTask = ref(null)
const usedTaskIds = ref([])
const publicNoticeOpen = ref(false)
const noTasksMessage = ref('')
const timerSeconds = ref(0)
const currentTaskHidden = ref(false)
const phase = ref('choose-level') // choose-level | choose-type | public-notice | show-task | hide-task | ask-result | reveal | exhausted

let timerHandle = null

const remainingForSelection = computed(() => {
  if (!selectedLevel.value || !selectedType.value) return null
  return tasks.filter(
    (task) =>
      task.level === selectedLevel.value &&
      task.type === selectedType.value &&
      !usedTaskIds.value.includes(task.id),
  ).length
})

const currentLevelLabel = computed(
  () => levelCards.find((l) => l.value === selectedLevel.value)?.subtitle ?? 'Waehle Level',
)
const currentTypeLabel = computed(
  () => typeCards.find((t) => t.value === selectedType.value)?.title ?? 'Waehle Typ',
)

function typeLabel(type) {
  return typeCards.find((t) => t.value === type)?.title ?? type
}

function selectLevel(level) {
  selectedLevel.value = level
  selectedType.value = null
  currentTask.value = null
  publicNoticeOpen.value = false
  noTasksMessage.value = ''
  currentTaskHidden.value = false
  resetTimer()
  phase.value = 'choose-type'
}

function selectType(type) {
  selectedType.value = type
  drawTask()
}

function drawTask() {
  resetTimer()
  currentTaskHidden.value = false

  if (!selectedLevel.value || !selectedType.value) return

  const available = tasks.filter(
    (task) =>
      task.level === selectedLevel.value &&
      task.type === selectedType.value &&
      !usedTaskIds.value.includes(task.id),
  )

  if (!available.length) {
    currentTask.value = null
    publicNoticeOpen.value = false
    noTasksMessage.value = 'Keine Aufgaben mehr verfuegbar. Starte eine neue Runde oder setze den Pool zurueck.'
    phase.value = 'exhausted'
    return
  }

  noTasksMessage.value = ''
  const picked = available[Math.floor(Math.random() * available.length)]
  usedTaskIds.value = [...usedTaskIds.value, picked.id]
  currentTask.value = picked

  const isPublicRound = rollPublicRound()
  publicNoticeOpen.value = isPublicRound

  if (isPublicRound) {
    phase.value = 'public-notice'
  } else {
    startShowPhase()
  }
}

function startShowPhase() {
  currentTaskHidden.value = false
  phase.value = 'show-task'
  startCountdown(10, () => {
    playRing()
    currentTaskHidden.value = true
    startPlayPhase()
  })
}

function startPlayPhase() {
  phase.value = 'hide-task'
  startCountdown(60, () => {
    playRing()
    phase.value = 'ask-result'
  })
}

function acknowledgePublic() {
  publicNoticeOpen.value = false
  startShowPhase()
}

function confirmGuess() {
  phase.value = 'reveal'
  currentTaskHidden.value = false
  resetTimer()
}

function skipPlayWait() {
  resetTimer()
  playRing()
  phase.value = 'ask-result'
}

function startCountdown(seconds, onDone) {
  resetTimer()
  timerSeconds.value = seconds
  timerHandle = window.setInterval(() => {
    if (timerSeconds.value <= 1) {
      timerSeconds.value = 0
      resetTimer(true)
      onDone?.()
    } else {
      timerSeconds.value -= 1
    }
  }, 1000)
}

function rollPublicRound() {
  return Math.floor(Math.random() * 6) === 0
}

function playRing() {
  const audio = new Audio(`${import.meta.env.BASE_URL}ring.mp3`)
  audio.play().catch(() => {})
}

function resetTimer(keepValue = false) {
  if (timerHandle) {
    clearInterval(timerHandle)
    timerHandle = null
  }
  if (!keepValue) {
    timerSeconds.value = 0
  }
}

function formatTime() {
  const minutes = String(Math.floor(timerSeconds.value / 60)).padStart(2, '0')
  const seconds = String(timerSeconds.value % 60).padStart(2, '0')
  return `${minutes}:${seconds}`
}

function nextRound() {
  resetTimer()
  selectedLevel.value = null
  selectedType.value = null
  currentTask.value = null
  publicNoticeOpen.value = false
  noTasksMessage.value = ''
  currentTaskHidden.value = false
  phase.value = 'choose-level'
}

function resetPool() {
  usedTaskIds.value = []
  nextRound()
}

onBeforeUnmount(() => resetTimer())
</script>

<template>
  <v-app>
    <v-main class="app-surface">
      <v-container class="py-8">
        <v-row class="mb-6" align="center">
          <v-col cols="12" md="7">
            <h2 class="text-primary mb-2">XMAS-tivity</h2>
            <div class="text-h4 font-weight-bold mb-2">Runde vorbereiten</div>
            <p class="text-body-1 text-medium-emphasis mb-4">
              Waehle zuerst den Schwierigkeitsgrad (Level 3-5) und dann den Aufgabentyp. Danach
              zieht die App einen zufaelligen Task, der in dieser Session noch nicht benutzt wurde.
            </p>
            <div class="d-flex flex-wrap" style="gap: 10px;">
              <v-chip color="primary" variant="elevated" prepend-icon="mdi-flag-checkered">
                {{ currentLevelLabel }}
              </v-chip>
              <v-chip color="secondary" variant="outlined" prepend-icon="mdi-shape-outline">
                {{ currentTypeLabel }}
              </v-chip>
              <v-chip v-if="remainingForSelection !== null" color="success" variant="tonal" prepend-icon="mdi-database-outline">
                {{ remainingForSelection }} verfuegbar
              </v-chip>
            </div>
          </v-col>
          <v-col cols="12" md="5">
            <v-card elevation="2" class="pa-4 status-card">
              <div class="text-subtitle-1 font-weight-medium mb-2">Rundenstatus</div>
              <div class="d-flex flex-wrap" style="gap: 8px;">
                <v-chip size="small" color="primary" variant="tonal">
                  Level: {{ selectedLevel ?? '---' }}
                </v-chip>
                <v-chip size="small" color="secondary" variant="tonal">
                  Typ: {{ selectedType ?? '---' }}
                </v-chip>
                <v-chip size="small" color="info" variant="tonal">
                  Aufgaben genutzt: {{ usedTaskIds.length }}
                </v-chip>
                <v-chip size="small" color="success" variant="tonal">
                  Timer: {{ formatTime() }}
                </v-chip>
              </div>
              <div class="d-flex flex-wrap mt-3" style="gap: 8px;">
                <v-btn size="small" color="primary" variant="tonal" prepend-icon="mdi-refresh" @click="nextRound">
                  Neue Auswahl
                </v-btn>
                <v-btn size="small" color="error" variant="text" prepend-icon="mdi-restart-alert" @click="resetPool">
                  Pool zuruecksetzen
                </v-btn>
              </div>
            </v-card>
          </v-col>
        </v-row>

        <v-row v-if="phase === 'choose-level'" class="mt-2" dense>
          <v-col cols="12" md="4" v-for="level in levelCards" :key="level.value">
            <v-card
              class="level-card choice-card"
              elevation="6"
              rounded="lg"
              @click="selectLevel(level.value)"
            >
              <v-img :src="level.image" cover class="level-img">
                <template #placeholder>
                  <v-row class="fill-height ma-0" align="center" justify="center">
                    <v-progress-circular indeterminate color="primary" />
                  </v-row>
                </template>
                <div class="level-overlay">
                  <div class="text-h6 font-weight-bold text-white">{{ level.title }}</div>
                  <div class="text-body-2 text-white text-medium-emphasis">{{ level.subtitle }}</div>
                </div>
              </v-img>
            </v-card>
          </v-col>
        </v-row>

        <v-row v-else-if="phase === 'choose-type'" class="mt-2" dense>
          <v-col cols="12" md="4" v-for="type in typeCards" :key="type.value">
            <v-card
              class="type-card choice-card"
              elevation="6"
              rounded="lg"
              @click="selectType(type.value)"
            >
              <v-img :src="type.image" cover class="type-img">
                <template #placeholder>
                  <v-row class="fill-height ma-0" align="center" justify="center">
                    <v-progress-circular indeterminate color="secondary" />
                  </v-row>
                </template>
                <div class="type-overlay">
                  <div class="text-overline text-white text-medium-emphasis">Aufgabentyp</div>
                  <div class="text-h6 font-weight-bold text-white">{{ type.title }}</div>
                  <div class="text-body-2 text-white text-medium-emphasis">{{ type.detail }}</div>
                </div>
              </v-img>
            </v-card>
          </v-col>
        </v-row>

        <v-row v-else-if="phase === 'exhausted'" class="mt-4">
          <v-col cols="12" md="10" lg="8" class="mx-auto">
            <v-alert type="warning" border="start" prominent>
              {{ noTasksMessage }}
            </v-alert>
            <div class="d-flex flex-wrap mt-3" style="gap: 10px;">
              <v-btn color="primary" prepend-icon="mdi-restart" @click="resetPool">Pool leeren und neu starten</v-btn>
              <v-btn variant="text" prepend-icon="mdi-arrow-left" @click="nextRound">Andere Auswahl treffen</v-btn>
            </div>
          </v-col>
        </v-row>

        <v-row v-else-if="phase === 'public-notice'" class="mt-4">
          <v-col cols="12" md="10" lg="8" class="mx-auto">
            <v-card class="pa-8 public-card" elevation="8">
              <div class="text-overline text-warning mb-2">Hinweis</div>
              <div class="text-h4 font-weight-bold mb-3">Öffentliche Runde</div>
              <p class="text-body-1 text-medium-emphasis mb-6">
                Diese Runde wurde zufaellig als öffentlich ausgewaehlt. Bitte
                bestaetige, bevor der Task angezeigt wird.
              </p>
              <div class="d-flex flex-wrap" style="gap: 10px;">
                <v-btn color="primary" size="large" prepend-icon="mdi-check-decagram" @click="acknowledgePublic">
                  Bestaetigen und anzeigen
                </v-btn>
                <v-btn color="secondary" size="large" variant="tonal" prepend-icon="mdi-dice-multiple" @click="drawTask">
                  Andere Aufgabe ziehen
                </v-btn>
              </div>
            </v-card>
          </v-col>
        </v-row>

        <v-row v-else-if="phase === 'show-task' || phase === 'hide-task' || phase === 'reveal'" class="mt-4">
          <v-col cols="12" md="10" lg="8" class="mx-auto">
            <v-card class="pa-8 task-card" elevation="6">
              <div class="d-flex flex-wrap justify-space-between align-center mb-4" style="gap: 16px;">
                <div>
                  <div class="text-overline text-primary">Aufgabe</div>
                 <div class="text-body-2 text-medium-emphasis">
                    Level {{ currentTask?.level }} · {{ typeLabel(currentTask?.type) }}
                  </div>
                </div>
                <div class="timer-chip">
                  <v-icon color="primary" class="mr-2">mdi-timer-outline</v-icon>
                  <span class="text-h5 font-weight-bold">{{ formatTime() }}</span>
                </div>
              </div>
              <v-divider class="my-4" />

              <div v-if="phase === 'show-task'">
                <div class="text-h3 font-weight-bold mb-4">{{ currentTask?.title }}</div>
                <div class="text-body-1 text-medium-emphasis">
                  Anzeige endet in {{ formatTime() }}. Danach wird der Begriff ausgeblendet.
                </div>
              </div>

              <div v-else-if="phase === 'hide-task'" class="text-center">
                <div class="text-overline text-secondary mb-2">Begriff ausgeblendet</div>
                <div class="text-h2 font-weight-bold mb-2">{{ formatTime() }}</div>
                <div class="text-body-1 text-medium-emphasis">
                  60 Sekunden bis zur Ratungsphase. Das Wort bleibt versteckt.
                </div>
                <div class="d-flex justify-center mt-4">
                  <v-btn color="primary" size="large" prepend-icon="mdi-fast-forward" @click="skipPlayWait">
                    Wartezeit ueberspringen
                  </v-btn>
                </div>
              </div>

              <div v-else-if="phase === 'reveal'">
                <div class="text-overline text-success mb-2">Aufgedeckt</div>
                <div class="text-h3 font-weight-bold mb-4">{{ currentTask?.title }}</div>
                <div class="text-body-2 text-medium-emphasis">
                  Starte eine neue Runde oder ziehe eine neue Aufgabe mit gleichem Level/Typ.
                </div>
              </div>

              <v-divider class="my-4" />
              <div class="d-flex flex-wrap" style="gap: 12px;">
                <v-btn color="primary" size="large" prepend-icon="mdi-refresh" @click="nextRound">
                  Naechste Runde
                </v-btn>
                <v-btn color="secondary" size="large" variant="tonal" prepend-icon="mdi-dice-multiple" @click="drawTask">
                  Neue Aufgabe (gleiches Level/Typ)
                </v-btn>
              </div>
            </v-card>
          </v-col>
        </v-row>

        <v-row v-else-if="phase === 'ask-result'" class="mt-4">
          <v-col cols="12" md="10" lg="8" class="mx-auto">
            <v-card class="pa-8 public-card" elevation="6">
              <div class="text-overline text-primary mb-2">Zeit abgelaufen</div>
              <div class="text-h4 font-weight-bold mb-3">Wurde der Begriff erraten?</div>
              <p class="text-body-1 text-medium-emphasis mb-6">
                Unabhaengig von Ja oder Nein: Du kannst dir den Begriff jetzt wieder anzeigen lassen
                und eine neue Runde starten.
              </p>
              <v-btn color="primary" size="large" prepend-icon="mdi-eye-check" @click="confirmGuess">
                Begriff anzeigen
              </v-btn>
            </v-card>
          </v-col>
        </v-row>
      </v-container>
    </v-main>
  </v-app>
</template>

<style scoped>
.app-surface {
  min-height: 100vh;
  background-color: rgba(255, 255, 255, 0.82);
  backdrop-filter: blur(2px);
}

.choice-card {
  cursor: pointer;
  transition: transform 180ms ease, box-shadow 180ms ease;
}

.choice-card:hover {
  transform: translateY(-4px);
  box-shadow: 0 12px 30px rgba(37, 99, 235, 0.18);
}

.level-card {
  overflow: hidden;
}

.level-img {
  border-radius: 12px;
  height: 320px;
  max-height: 320px;
}

@media (orientation: landscape) {
  .level-img {
    height: 60vh;
    max-height: 60vh;
  }
}

.type-card {
  overflow: hidden;
}

.type-img {
  border-radius: 12px;
  height: 320px;
  max-height: 320px;
}

@media (orientation: landscape) {
  .type-img {
    height: 60vh;
    max-height: 60vh;
  }
}

.type-overlay {
  position: absolute;
  bottom: 0;
  left: 0;
  right: 0;
  padding: 14px 16px;
  background: linear-gradient(180deg, rgba(0, 0, 0, 0.05) 0%, rgba(0, 0, 0, 0.7) 100%);
}

.level-overlay {
  position: absolute;
  bottom: 0;
  left: 0;
  right: 0;
  padding: 14px 16px;
  background: linear-gradient(180deg, rgba(0, 0, 0, 0.05) 0%, rgba(0, 0, 0, 0.65) 100%);
}

.status-card {
  backdrop-filter: blur(4px);
  background-color: rgba(255, 255, 255, 0.82);
}

.public-card {
  border: 2px dashed rgba(252, 211, 77, 0.8);
  background-color: #fff8e1;
}

.task-card {
  background-color: rgba(255, 255, 255, 0.92);
  border: 1px solid rgba(37, 99, 235, 0.08);
}

.timer-chip {
  display: inline-flex;
  align-items: center;
  padding: 10px 14px;
  border-radius: 999px;
  background-color: #e0e7ff;
  border: 1px solid #c7d2fe;
}
</style>
