<template>
  <div class="mastermind">
    <h2>Mastermind Game</h2>

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
        :style="{backgroundColor: currentGuess[i] || '#eee'}"
        @click="setColor(i)"
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
            :style="{backgroundColor: color || '#eee'}"
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
      <p v-else>
        You lost! Secret was:
        <span>
          <span
            v-for="(c, i) in secret"
            :key="i"
            :style="{backgroundColor: c, display: 'inline-block', width: '24px', height: '24px', borderRadius: '50%', marginRight: '6px', border: '2px solid #ccc'}"
          ></span>
        </span>
      </p>
      <button @click="resetGame">Play Again</button>
    </div>
  </div>
</template>

<script setup>
import { ref, computed } from 'vue';

const colors = [
  "red", "orange", "yellow", "green", "blue",
  "purple", "pink", "brown", "black", "white"
];
const secretLength = 4;
const maxAttempts = 4;

const secret = ref([]);
const currentGuess = ref(new Array(secretLength).fill(null));
const selectedColor = ref(null);
const attempts = ref([]);
const gameOver = ref(false);
const winner = ref(false);

function generateSecret() {
  // Allow duplicates for a classic mastermind feel
  secret.value = Array.from(
    {length: secretLength},
    () => colors[Math.floor(Math.random() * colors.length)]
  );
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

  // Black peg: right color in right slot
  for (let i = 0; i < secretLength; i++) {
    if (guessCopy[i] === secretCopy[i]) {
      black++;
      secretCopy[i] = guessCopy[i] = null;
    }
  }
  // White peg: right color, wrong slot
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
}

function resetGame() {
  attempts.value = [];
  currentGuess.value = new Array(secretLength).fill(null);
  gameOver.value = false;
  winner.value = false;
  generateSecret();
}

// Helper function for feedback dots in 2x2 grid
function feedbackPegs(black, white) {
  // Black first, then white, then empty until 4 total
  const dots = [];
  for (let i = 0; i < black; i++) dots.push('black');
  for (let i = 0; i < white; i++) dots.push('white');
  while (dots.length < 4) dots.push('empty');
  return dots;
}

generateSecret();
</script>
