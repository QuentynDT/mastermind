<template>
  <div class="mastermind-app">
    <div class="game-container">
      <div class="mastermind-board">
        <h2>Mastermind</h2>
        <div class="settings">
          <label for="num-colors-slider">Colors: {{ numColors }}</label>
          <input
            id="num-colors-slider"
            type="range"
            min="4"
            :max="allColors.length"
            v-model.number="numColors"
            class="slider"
          />
        </div>

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

        <button @click="submitGuess" :disabled="!isGuessComplete || gameOver">Check</button>

        <div class="history player-history">
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

        <div v-if="gameOver && !isKnuthRunning">
          <button @click="resetGame">Play Again</button>
        </div>
      </div>

      <div class="mastermind-board" v-if="gameOver">
          <h2>Knuth's Algorithm</h2>
          
          <div class="secret-code-row">
            <div
              v-for="(peg, i) in secretLength"
              :key="i"
              class="secret-slot"
              :style="{ backgroundColor: secret[i] }"
            ></div>
          </div>
          
          <div class="history knuth-history">
              <div v-if="knuthAttempts.length === 0 && !isKnuthCalculating" class="placeholder-text">
                  Algorithm will begin shortly...
              </div>
              <div v-for="(attempt, index) in knuthAttempts" :key="index" class="attempt">
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
          <p v-if="isKnuthCalculating" class="thinking-text">Algorithm is thinking...</p>
          
          <div v-if="summaryData" class="summary-box">
            <p>{{ summaryData.text1 }}</p>
            <p>{{ summaryData.text2 }}</p>
            <p class="summary-phrase">{{ summaryData.phrase }}</p>
          </div>
      </div>
    </div>
  </div>
</template>

<script setup>
import { ref, computed, onMounted, onUnmounted, watch } from 'vue';

const allColors = Object.freeze([
  "red", "orange", "yellow", "green", "blue",
  "purple", "pink", "saddlebrown", "black", "white"
]);
const secretLength = 4;
const maxAttempts = 6;

const numColors = ref(8);
const colors = computed(() => allColors.slice(0, numColors.value));

const secret = ref([]);
const currentGuess = ref(new Array(secretLength).fill(null));
const selectedColor = ref(null);
const attempts = ref([]);
const gameOver = ref(false);
const winner = ref(false);
const focusedSlot = ref(0);

const knuthAttempts = ref([]);
const isKnuthRunning = ref(false);
const isKnuthCalculating = ref(false);

const summaryData = computed(() => {
    if (gameOver.value && !isKnuthRunning.value && knuthAttempts.value.length > 0) {
        const knuthsAttempts = knuthAttempts.value.length;
        const playerAttempts = winner.value ? attempts.value.length : maxAttempts;

        const text1 = `Knuth's algorithm solved it in ${knuthsAttempts} attempts.`;
        const text2 = (winner.value) ? `You used ${playerAttempts} attempts.` : "You did not finish.";
        const phrase = (!winner.value) ? "Better luck next time...": 
        (playerAttempts - knuthsAttempts <= 0) ? "Outstanding!":
        (playerAttempts - knuthsAttempts <= 2) ? "Not Quite.":
        "Nice Try.";

        return { text1, text2, phrase };
    }
    return null;
});


function generateSecret() {
  secret.value = Array.from(
    {length: secretLength},
    () => colors.value[Math.floor(Math.random() * colors.value.length)]
  );
}

function calculateFeedback(guess, codeToSolve) {
  let black = 0;
  let white = 0;
  const secretCopy = codeToSolve.slice();
  const guessCopy = guess.slice();

  for (let i = 0; i < secretLength; i++) {
    if (guessCopy[i] === secretCopy[i]) {
      black++;
      secretCopy[i] = null;
      guessCopy[i] = null;
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
  return { black, white };
}

function submitGuess() {
  if (!isGuessComplete.value || gameOver.value) return;
  const guess = currentGuess.value.slice();
  
  const { black, white } = calculateFeedback(guess, secret.value);
  attempts.value.push({ guess, black, white });

  if (black === secretLength) {
    gameOver.value = true;
    winner.value = true;
  } else if (attempts.value.length >= maxAttempts) {
    gameOver.value = true;
  }

  currentGuess.value = new Array(secretLength).fill(null);
  focusedSlot.value = 0;
}

const isGuessComplete = computed(() =>
  currentGuess.value.every(slot => slot !== null)
);

function feedbackPegs(black, white) {
  const dots = [];
  for (let i = 0; i < black; i++) dots.push('black');
  for (let i = 0; i < white; i++) dots.push('white');
  while (dots.length < secretLength) dots.push('empty');
  return dots;
}

function setSlotColor(index, color) {
    if (index === null) return;
    currentGuess.value[index] = color;
    let nextEmptySlot = currentGuess.value.indexOf(null);
    focusedSlot.value = nextEmptySlot !== -1 ? nextEmptySlot : null;
}

function resetGame() {
  attempts.value = [];
  currentGuess.value = new Array(secretLength).fill(null);
  gameOver.value = false;
  winner.value = false;
  focusedSlot.value = 0;
  knuthAttempts.value = [];
  isKnuthRunning.value = false;
  isKnuthCalculating.value = false;
  selectedColor.value = colors.value[0];
  generateSecret();
}

function handleKeyDown(event) {
    if (gameOver.value) return;
    let colorIdx = null;
    if (event.key >= '1' && event.key <= '9') {
        colorIdx = parseInt(event.key, 10) - 1;
    } else if (event.key === '0') {
        colorIdx = 9;
    }

    if (colorIdx !== null && colorIdx < colors.value.length) {
        setSlotColor(focusedSlot.value, colors.value[colorIdx]);
    } else if (event.key === 'Enter' && isGuessComplete.value) {
        submitGuess();
    } else if (event.key === 'Backspace') {
        const targetSlot = focusedSlot.value === null ? secretLength - 1 : currentGuess.value[focusedSlot.value] === null ? focusedSlot.value - 1 : focusedSlot.value;
        if(targetSlot >= 0) {
            currentGuess.value[targetSlot] = null;
            focusedSlot.value = targetSlot;
        }
    } else if (event.key === 'ArrowLeft' && focusedSlot.value > 0) {
        focusedSlot.value--;
    } else if (event.key === 'ArrowRight' && focusedSlot.value < secretLength - 1) {
        focusedSlot.value++;
    }
}

watch(gameOver, (isOver) => {
  if (isOver && !isKnuthRunning.value) {
    setTimeout(() => runKnuthAlgorithm(), 500);
  }
});

watch(numColors, () => {
    resetGame();
});

const sleep = ms => new Promise(res => setTimeout(res, ms));

function generateAllPossibleCodes(currentColors) {
    const allCodes = [];
    function generate(prefix) {
        if (prefix.length === secretLength) {
            allCodes.push(prefix);
            return;
        }
        for (const color of currentColors) {
            generate([...prefix, color]);
        }
    }
    generate([]);
    return allCodes;
}

async function runKnuthAlgorithm() {
    isKnuthRunning.value = true;
    let S = generateAllPossibleCodes(colors.value);
    const A = S.slice();
    let guess = [colors.value[0], colors.value[0], colors.value[1], colors.value[1]];

    while(true) {
        if(knuthAttempts.value.length >= maxAttempts) break;

        const { black, white } = calculateFeedback(guess, secret.value);
        knuthAttempts.value.push({ guess, black, white });

        if (black === secretLength) break;

        S = S.filter(code => {
            const feedback = calculateFeedback(guess, code);
            return feedback.black === black && feedback.white === white;
        });

        if (S.length === 0) break;
        
        await sleep(2000);

        isKnuthCalculating.value = true;
        await sleep(10);
        
        let minMax = Infinity;

        const searchSpace = S.length > A.length ? A : S;
        
        const bestGuesses = [];
        for (const possibleGuess of searchSpace) {
            const scoreCounts = {};
            for (const possibleSecret of S) {
                const { black, white } = calculateFeedback(possibleGuess, possibleSecret);
                const scoreKey = `${black},${white}`;
                scoreCounts[scoreKey] = (scoreCounts[scoreKey] || 0) + 1;
            }
            const maxPartitionSize = Math.max(0, ...Object.values(scoreCounts));
            
            if (maxPartitionSize < minMax) {
                minMax = maxPartitionSize;
                bestGuesses.length = 0;
                bestGuesses.push(possibleGuess);
            } else if (maxPartitionSize === minMax) {
                bestGuesses.push(possibleGuess)
            }
        }

        const guessInS = bestGuesses.find(g => S.some(s => s.every((val, i) => val === g[i])));
        guess = guessInS || bestGuesses[0] || S[0];

        isKnuthCalculating.value = false;
        if (!guess) break;
    }
    isKnuthRunning.value = false;
    isKnuthCalculating.value = false;
}

onMounted(() => {
  window.addEventListener('keydown', handleKeyDown);
  resetGame();
});

onUnmounted(() => {
  window.removeEventListener('keydown', handleKeyDown);
});
</script>

<style>
.mastermind-app {
    display: flex;
    justify-content: center;
    align-items: flex-start;
    padding: 20px;
    width: 100%;
}
.game-container {
    display: flex;
    flex-wrap: wrap;
    justify-content: center;
    gap: 30px;
}
.mastermind-board {
    font-family: sans-serif;
    display: flex;
    flex-direction: column;
    align-items: center;
    gap: 15px;
    background-color: #333;
    color: white;
    padding: 20px;
    border-radius: 8px;
    min-width: 320px;
    flex-grow: 1;
    box-sizing: border-box;
}
.mastermind-board h2 {
    margin-top: 0;
    margin-bottom: 10px;
}
.settings {
    display: flex;
    align-items: center;
    gap: 10px;
    background-color: #444;
    padding: 10px;
    border-radius: 8px;
    width: 100%;
    box-sizing: border-box;
}
.slider {
    width: 100%;
}
.secret-code-row, .guess-row, .guess {
    display: flex;
    gap: 10px;
}
.secret-slot, .guess-slot {
    width: 40px;
    height: 40px;
    border-radius: 50%;
    border: 2px solid #666;
    display: flex;
    justify-content: center;
    align-items: center;
    font-size: 24px;
    font-weight: bold;
    cursor: pointer;
}
.guess-slot.focused {
    border-color: white;
    box-shadow: 0 0 10px cyan;
}
.colors {
    display: flex;
    gap: 5px;
    flex-wrap: wrap;
    justify-content: center;
}
.colors button {
    width: 30px;
    height: 30px;
    border-radius: 50%;
    border: 2px solid transparent;
    cursor: pointer;
    transition: all 0.2s;
}
.colors button.selected {
    border-color: white;
    transform: scale(1.2);
}
button {
    padding: 10px 20px;
    font-size: 16px;
    border-radius: 5px;
    border: none;
    background-color: #4CAF50;
    color: white;
    cursor: pointer;
}
button:disabled {
    background-color: #555;
    cursor: not-allowed;
}
.history {
    display: flex;
    flex-direction: column;
    gap: 5px;
    width: 100%;
    min-height: 250px;
}
.player-history {
    max-height: 250px;
    overflow-y: auto;
    overflow-x: hidden;
    scrollbar-width: thin;
    scrollbar-color: #666 #333;
}
.knuth-history {
    justify-content: flex-start;
}
.placeholder-text {
    color: #888;
    align-self: center;
    margin-top: 20px;
}
.thinking-text {
    min-height: 24px;
}
.attempt {
    display: flex;
    align-items: center;
    justify-content: space-between;
    padding: 5px;
    border-radius: 4px;
    background-color: #2c2c2c;
}
.feedback-grid {
    display: grid;
    grid-template-columns: repeat(2, 1fr);
    gap: 4px;
}
.feedback-dot {
    width: 15px;
    height: 15px;
    border-radius: 50%;
    border: 1px solid #777;
}
.feedback-dot.black { background-color: black; }
.feedback-dot.white { background-color: white; }
.feedback-dot.empty { background-color: #555; }
.summary-box {
    margin-top: auto;
    padding: 10px;
    background-color: #2c2c2c;
    border: 1px solid #555;
    border-radius: 8px;
    text-align: center;
    width: 100%;
    box-sizing: border-box;
}
.summary-box p {
    margin: 5px 0;
}
.summary-phrase {
    font-weight: bold;
    font-size: 1.2em;
}
</style>
