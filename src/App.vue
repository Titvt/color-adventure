<script setup>
import { ref, onUnmounted } from "vue";

let isGameStarted = ref(false);
let isGameOver = ref(false);
let score = ref(0);
let grid = ref([]);
let diffCellIndex = ref(null);
let timeLeft = ref(15);
let timerInterval = null;
let gameMode = ref(null);
let lives = ref(3);
let lifeAnimation = ref(false);
let startTime = ref(0);
let totalGameTime = ref(0);
let gameModeName = ref("");

let generateColorPair = (score) => {
  let baseR = Math.floor(Math.random() * 256);
  let baseG = Math.floor(Math.random() * 256);
  let baseB = Math.floor(Math.random() * 256);
  let diff =
    Math.max(Math.floor(Math.exp(-score / 10) * 128), 1) *
    (Math.random() < 0.5 ? 1 : -1);
  let channelToChange = Math.floor(Math.random() * 3);
  let secondR = baseR;
  let secondG = baseG;
  let secondB = baseB;

  if (channelToChange === 0) {
    if (baseR + diff < 0 || baseR + diff > 255) {
      diff = -diff;
    }

    secondR = baseR + diff;
  } else if (channelToChange === 1) {
    if (baseG + diff < 0 || baseG + diff > 255) {
      diff = -diff;
    }

    secondG = baseG + diff;
  } else {
    if (baseB + diff < 0 || baseB + diff > 255) {
      diff = -diff;
    }

    secondB = baseB + diff;
  }

  return {
    mainColor: `rgb(${baseR}, ${baseG}, ${baseB})`,
    diffColor: `rgb(${secondR}, ${secondG}, ${secondB})`,
  };
};

let startGame = (mode) => {
  isGameStarted.value = true;
  isGameOver.value = false;
  score.value = 0;
  gameMode.value = mode;
  lives.value = gameMode.value === "challenge" ? 3 : Infinity;
  generateGrid();
  startTime.value = Date.now();
  gameModeName.value = mode === "challenge" ? "限时挑战模式" : "勇往直前模式";

  if (timerInterval) {
    clearInterval(timerInterval);
    timerInterval = null;
  }

  if (gameMode.value === "challenge") {
    timeLeft.value = 15;
    startTimer();
  }
};

let startTimer = () => {
  if (timerInterval) {
    clearInterval(timerInterval);
  }

  timeLeft.value = 15;
  timerInterval = setInterval(() => {
    timeLeft.value -= 1;

    if (timeLeft.value <= 0) {
      clearInterval(timerInterval);
      loseLife();
    }
  }, 1000);
};

let loseLife = () => {
  lives.value -= 1;
  lifeAnimation.value = true;
  setTimeout(() => {
    lifeAnimation.value = false;
  }, 500);

  if (lives.value <= 0) {
    gameOver();
  } else if (gameMode.value === "challenge") {
    startTimer();
  }
};

let gameOver = () => {
  isGameOver.value = true;

  if (timerInterval) {
    clearInterval(timerInterval);
    timerInterval = null;
  }

  totalGameTime.value = Math.floor((Date.now() - startTime.value) / 1000);
};

let generateGrid = () => {
  grid.value = [];
  let colors = generateColorPair(score.value);
  diffCellIndex.value = Math.floor(Math.random() * 16);

  for (let i = 0; i < 16; i++) {
    grid.value.push({
      index: i,
      color: i === diffCellIndex.value ? colors.diffColor : colors.mainColor,
    });
  }
};

let selectCell = (index) => {
  if (!isGameStarted.value || isGameOver.value) {
    return;
  }

  if (index === diffCellIndex.value) {
    score.value++;
    generateGrid();

    if (gameMode.value === "challenge") {
      startTimer();
    }
  } else {
    if (gameMode.value === "endless") {
      gameOver();
    } else {
      loseLife();
    }
  }
};

onUnmounted(() => {
  if (timerInterval) {
    clearInterval(timerInterval);
  }
});
</script>

<template>
  <div class="game-container">
    <h1>Color Adventure</h1>
    <div v-if="!isGameStarted || isGameOver" class="start-screen">
      <h2 v-if="isGameOver">游戏结束！你的得分：{{ score }}</h2>
      <h3 v-if="isGameOver">
        游戏模式：{{ gameModeName }}，总耗时：{{ totalGameTime }}秒
      </h3>
      <div class="mode-selection">
        <button @click="startGame('challenge')" class="mode-button">
          限时挑战模式
        </button>
        <button @click="startGame('endless')" class="mode-button">
          勇往直前模式
        </button>
      </div>
    </div>
    <div v-else class="game-play">
      <div class="score-display">当前得分：{{ score }}</div>
      <div class="mode-display">当前模式：{{ gameModeName }}</div>
      <div v-if="gameMode === 'challenge'" class="timer-display">
        剩余时间：{{ timeLeft }}秒
      </div>
      <div
        v-if="gameMode === 'challenge'"
        class="lives-display"
        :class="{ 'animate-life': lifeAnimation }"
      >
        剩余生命值：{{ lives }}
      </div>
      <div class="game-instructions">找出并点击颜色与众不同的格子</div>
      <div class="game-grid">
        <div
          v-for="cell in grid"
          :key="cell.index"
          :style="{ backgroundColor: cell.color }"
          @click="selectCell(cell.index)"
          class="grid-cell"
        ></div>
      </div>
    </div>
  </div>
</template>

<style scoped>
html {
  cursor: default;
}

body {
  margin: 0;
}

.game-container {
  max-width: 640px;
  margin: auto;
  padding: 20px;
  text-align: center;
}

.start-screen {
  display: flex;
  flex-direction: column;
  align-items: center;
  justify-content: center;
  margin-top: 50px;
}

.mode-selection {
  display: flex;
  flex-direction: row;
  justify-content: center;
  gap: 20px;
  margin-bottom: 20px;
}

.mode-button {
  padding: 12px 24px;
  font-size: 18px;
  background-color: #4caf50;
  color: white;
  border: none;
  border-radius: 4px;
  cursor: pointer;
  transition: background-color 0.3s;
}

.mode-button:hover {
  background-color: #45a049;
}

.start-button {
  padding: 12px 24px;
  font-size: 18px;
  background-color: #4caf50;
  color: white;
  border: none;
  border-radius: 4px;
  cursor: pointer;
  transition: background-color 0.3s;
}

.start-button:hover {
  background-color: #45a049;
}

.game-play {
  margin-top: 20px;
}

.score-display {
  font-size: 24px;
  font-weight: bold;
  margin-bottom: 10px;
}

.mode-display {
  font-size: 20px;
  font-weight: bold;
  margin-bottom: 10px;
  color: #2ecc71;
}

.timer-display {
  font-size: 20px;
  font-weight: bold;
  margin-bottom: 10px;
  color: #e74c3c;
}

.lives-display {
  font-size: 20px;
  font-weight: bold;
  margin-bottom: 10px;
  color: #3498db;
  transition: transform 0.3s, font-size 0.3s;
}

.animate-life {
  animation: pulse 0.5s ease-in-out;
}

@keyframes pulse {
  0% {
    transform: scale(1);
  }
  50% {
    transform: scale(1.5);
    color: #e74c3c;
  }
  100% {
    transform: scale(1);
  }
}

.game-instructions {
  font-size: 18px;
  margin-bottom: 20px;
}

.game-grid {
  display: flex;
  flex-wrap: wrap;
  width: 100%;
  aspect-ratio: 1;
  margin: auto auto;
  background-color: #f0f0f0;
  border: 2px solid black;
}

.grid-cell {
  display: inline-block;
  width: 25%;
  height: 25%;
  box-sizing: border-box;
  border: 2px solid black;
  cursor: pointer;
}
</style>
