<script setup>
import { ref, onUnmounted } from "vue";

let isGameStarted = ref(false);
let isGameOver = ref(false);
let score = ref(0);
let grid = ref([]);
let diffCellIndex = ref(null);
let timeLeft = ref(10);
let timerInterval = null;

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

let startGame = () => {
  isGameStarted.value = true;
  isGameOver.value = false;
  score.value = 0;
  generateGrid();
  resetTimer();
};

let resetTimer = () => {
  if (timerInterval) {
    clearInterval(timerInterval);
  }

  timeLeft.value = 15;
  timerInterval = setInterval(() => {
    timeLeft.value -= 1;

    if (timeLeft.value <= 0) {
      clearInterval(timerInterval);
      isGameOver.value = true;
    }
  }, 1000);
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
    resetTimer();
  } else {
    isGameOver.value = true;
    clearInterval(timerInterval);
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
      <button @click="startGame" class="start-button">
        {{ isGameOver ? "再玩一次" : "开始游戏" }}
      </button>
    </div>
    <div v-else class="game-play">
      <div class="score-display">当前得分：{{ score }}</div>
      <div class="timer-display">剩余时间：{{ timeLeft }}秒</div>
      <div class="game-instructions">找出不同颜色的格子</div>
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

.timer-display {
  font-size: 20px;
  font-weight: bold;
  margin-bottom: 10px;
  color: #e74c3c;
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
