<template>
  <div class="mastermind">
    <h2>Mastermind</h2>

    <div class="secret-code-row">
      <div
        v-for="(peg, i) in secretLength"
        :key="i"
        class="secret-slot"
        :style="gameOver ? { backgroundColor: secret[i] } : { backgroundColor: '#444' }"
      >
        <span v-if="!gameOver" class="secret-mark">?</span>
      </div>
    </div>

    <div class="colors">
      <button
        v-for="(color, index) in colors"
        :key="color"
        :style="{backgroundColor: color}"
        @click="selectedColor = color"
        :class="{selected: selectedColor === color}"
        aria-label="Pick color"
      ></button>
    </div>

    <div class="guess-row">
      <div
        v-for="(slot, i) in secretLength"
        :key="i"
        class="guess-slot"
        :class="{ focused: focusedSlot === i }"
        :style="{backgroundColor: currentGuess[i] || '#444'}"
        @click="focusedSlot = i"
      ></div>
    </div>

    <button @click="submitGuess" :disabled="!isGuessComplete">Check</button>

    <div class="history">
      <div v-for="(attempt, index) in attempts" :key="index" class="attempt">
        <div class="guess">
          <div
            v-for="(color, idx) in attempt.guess"
            :key="idx"
            class="guess-slot"
            :style="{backgroundColor: color || '#444'}"
          ></div>
        </div>
        <div class="feedback-grid">
          <span
            v-for="(peg, idx) in feedbackPegs(attempt.black, attempt.white)"
            :key="idx"
            :class="['feedback-dot', peg]"
          ></span>
        </div>
      </div>
    </div>

    <div v-if="gameOver">
      <p v-if="winner">You won! 🎉</p>
      <p v-else>You lost!</p>
      <button @click="resetGame">Play Again</button>
    </div>
  </div>
</template>

<script setup>
import { ref, computed, onMounted, onUnmounted } from 'vue';

const colors = [
  "red", "orange", "yellow", "green", "blue",
  "purple", "pink", "saddlebrown", "black", "white"
];
const secretLength = 4;
const maxAttempts = 10;

const secret = ref([]);
const currentGuess = ref(new Array(secretLength).fill(null));
const selectedColor = ref(null);
const attempts = ref([]);
const gameOver = ref(false);
const winner = ref(false);
const focusedSlot = ref(null);

function generateSecret() {
  secret.value = Array.from(
    {length: secretLength},
    () => colors[Math.floor(Math.random() * colors.length)]
  );
}

function setSlotColor(index, colorIdx) {
  currentGuess.value[index] = colors[colorIdx];
}

function setColor(index) {
  if (selectedColor.value) {
    currentGuess.value[index] = selectedColor.value;
  }
}

const isGuessComplete = computed(() =>
  currentGuess.value.every(slot => slot !== null)
);

function submitGuess() {
  if (!isGuessComplete.value || gameOver.value) return;
  const guess = currentGuess.value.slice();

  let black = 0;
  let white = 0;
  const secretCopy = secret.value.slice();
  const guessCopy = guess.slice();

  for (let i = 0; i < secretLength; i++) {
    if (guessCopy[i] === secretCopy[i]) {
      black++;
      secretCopy[i] = guessCopy[i] = null;
    }
  }
  for (let i = 0; i < secretLength; i++) {
    if (guessCopy[i] !== null) {
      const idx = secretCopy.indexOf(guessCopy[i]);
      if (idx !== -1) {
        white++;
        secretCopy[idx] = null;
        guessCopy[i] = null;
      }
    }
  }

  attempts.value.push({ guess, black, white });
  if (black === secretLength) {
    gameOver.value = true;
    winner.value = true;
  } else if (attempts.value.length >= maxAttempts) {
    gameOver.value = true;
  }
  currentGuess.value = new Array(secretLength).fill(null);
  focusedSlot.value = null;
}

function resetGame() {
  attempts.value = [];
  currentGuess.value = new Array(secretLength).fill(null);
  gameOver.value = false;
  winner.value = false;
  focusedSlot.value = null;
  generateSecret();
}

function feedbackPegs(black, white) {
  const dots = [];
  for (let i = 0; i < black; i++) dots.push('black');
  for (let i = 0; i < white; i++) dots.push('white');
  while (dots.length < 4) dots.push('empty');
  return dots;
}

function onKeyDown(event) {
  let colorIdx = null;
  if (event.key >= '1' && event.key <= '9') colorIdx = parseInt(event.key, 10) - 1;
  else if (event.key === '0') colorIdx = 9;

  if (colorIdx !== null && colorIdx >= 0 && colorIdx < colors.length && !gameOver.value) {
    if (focusedSlot.value !== null) {
      setSlotColor(focusedSlot.value, colorIdx);
    } else {
      for (let i = 0; i < secretLength; i++) {
        if (!currentGuess.value[i]) {
          setSlotColor(i, colorIdx);
          break;
        }
      }
    }
  } else if (event.key === 'Enter' && isGuessComplete.value && !gameOver.value) {
    submitGuess();
  }
}

onMounted(() => {
  window.addEventListener('keydown', onKeyDown);
});
onUnmounted(() => {
  window.removeEventListener('keydown', onKeyDown);
});

generateSecret();
</script>
