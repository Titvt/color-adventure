<script setup>
import chroma from "chroma-js";
import { onMounted, onUnmounted, ref } from "vue";

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
let isPreviewMode = ref(false);
let previewMainColor = ref("");
let previewDiffColor = ref("");
let previewDiffIndex = ref(null);
let shareUrl = ref("");
let isShareCopied = ref(false);
let isShareCopyFailed = ref(false);

onMounted(() => {
  checkShareParam();
});

onUnmounted(() => {
  if (timerInterval) {
    clearInterval(timerInterval);
  }
});

let checkShareParam = () => {
  let urlParams = new URLSearchParams(window.location.search);
  let shareParam = urlParams.get("share");

  if (shareParam) {
    try {
      let decodedData = JSON.parse(atob(shareParam));

      if (
        decodedData &&
        decodedData.mainColor &&
        decodedData.diffColor &&
        decodedData.diffIndex !== undefined
      ) {
        isPreviewMode.value = true;
        previewMainColor.value = decodedData.mainColor;
        previewDiffColor.value = decodedData.diffColor;
        previewDiffIndex.value = decodedData.diffIndex;
        showPreview();
      } else {
        exitPreviewMode();
      }
    } catch (e) {
      exitPreviewMode();
    }
  }
};

let showPreview = () => {
  grid.value = [];

  for (let i = 0; i < 16; i++) {
    grid.value.push({
      index: i,
      color:
        i === previewDiffIndex.value
          ? previewDiffColor.value
          : previewMainColor.value,
    });
  }
};

let generateShareLink = () => {
  if (!grid.value || grid.value.length === 0 || diffCellIndex.value === null) {
    return;
  }

  let mainColor = grid.value.find(
    (cell) => cell.index !== diffCellIndex.value
  ).color;
  let diffColor = grid.value.find(
    (cell) => cell.index === diffCellIndex.value
  ).color;
  let shareData = {
    mainColor,
    diffColor,
    diffIndex: diffCellIndex.value,
  };
  let base64Data = btoa(JSON.stringify(shareData));
  let url = new URL(window.location.href);
  url.searchParams.set("share", base64Data);
  shareUrl.value = url.toString();
  navigator.clipboard
    .writeText(shareUrl.value)
    .then(() => {
      isShareCopied.value = true;
      setTimeout(() => {
        isShareCopied.value = false;
      }, 3000);
    })
    .catch(() => {
      isShareCopyFailed.value = true;
      setTimeout(() => {
        isShareCopyFailed.value = false;
      }, 3000);
    });
};

let exitPreviewMode = () => {
  isPreviewMode.value = false;
  let url = new URL(window.location.href);
  url.searchParams.delete("share");
  window.history.replaceState({}, document.title, url.toString());
};

let generateColorPair = (score) => {
  let targetDeltaE = Math.max(Math.exp(-score / 10) * 20, 0.1);
  let color1, color2;
  let deltaE = 0;

  while (deltaE <= 20) {
    color1 = chroma.random();
    color2 = chroma.random();
    deltaE = chroma.deltaE(color1, color2);
  }

  let colorScale = chroma.scale([color1, color2]);
  let low = 0;
  let high = 1;
  let mid, midColor, currentDeltaE;
  let bestMid = 0.5;
  let bestDiff = Infinity;

  for (let i = 0; i < 100; i++) {
    mid = (low + high) / 2;
    midColor = colorScale(mid);
    currentDeltaE = chroma.deltaE(color1, midColor);
    let diff = Math.abs(currentDeltaE - targetDeltaE);

    if (diff < bestDiff) {
      bestDiff = diff;
      bestMid = mid;
    }

    if (bestDiff < 0.1) {
      break;
    }

    if (currentDeltaE > targetDeltaE) {
      high = mid;
    } else {
      low = mid;
    }
  }

  let diffColor = colorScale(bestMid);
  return {
    mainColor: color1.css(),
    diffColor: diffColor.css(),
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
</script>

<template>
  <div class="game-container">
    <h1 class="rainbow-title">Color Adventure</h1>
    <div v-if="isPreviewMode" class="preview-mode">
      <h2>观战模式</h2>
      <div class="game-grid">
        <div
          v-for="cell in grid"
          :key="cell.index"
          :style="{ backgroundColor: cell.color }"
          class="grid-cell"
        ></div>
      </div>
      <button @click="exitPreviewMode" class="exit-preview-button">
        结束观战
      </button>
    </div>
    <div v-else>
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
        <button @click="generateShareLink" class="share-button">
          分享当前关卡
        </button>
        <div v-if="isShareCopied" class="share-copied-message">
          分享链接已复制到剪贴板！
        </div>
        <div v-if="isShareCopyFailed" class="share-copy-failed-message">
          分享链接复制到剪贴板失败！
        </div>
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
  background-color: white;
}

.rainbow-title {
  background: linear-gradient(
    to right,
    #ff0000,
    #ff7f00,
    #ffff00,
    #00ff00,
    #0000ff,
    #4b0082,
    #8b00ff
  );
  -webkit-background-clip: text;
  background-clip: text;
  color: transparent;
  font-weight: bold;
  text-shadow: 2px 2px 4px rgba(0, 0, 0, 0.3);
  animation: rainbow-animation 6s linear infinite;
  font-size: 2.5rem;
}

@keyframes rainbow-animation {
  0% {
    background-position: 0% 50%;
  }
  50% {
    background-position: 100% 50%;
  }
  100% {
    background-position: 0% 50%;
  }
}

.game-container {
  max-width: 440px;
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

.preview-mode {
  margin-top: 20px;
}

.exit-preview-button {
  margin: 20px;
  padding: 12px 24px;
  font-size: 18px;
  background-color: #e74c3c;
  color: white;
  border: none;
  border-radius: 4px;
  cursor: pointer;
  transition: background-color 0.3s;
}

.exit-preview-button:hover {
  background-color: #c0392b;
}

.share-button {
  margin: 20px;
  padding: 12px 24px;
  font-size: 18px;
  background-color: #3498db;
  color: white;
  border: none;
  border-radius: 4px;
  cursor: pointer;
  transition: background-color 0.3s;
}

.share-button:hover {
  background-color: #2980b9;
}

.share-copied-message {
  font-size: 16px;
  color: #2ecc71;
  margin-top: 10px;
}

.share-copy-failed-message {
  font-size: 16px;
  color: #e74c3c;
  margin-top: 10px;
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
