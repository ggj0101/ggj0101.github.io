<script setup lang="ts">
import { ref, onMounted, onUnmounted, computed, nextTick } from 'vue'

interface Cell {
  id: number
  row: number
  col: number
  isActive: boolean
  timeLeft: number
  isAnimating: boolean
}

// 遊戲狀態
const level = ref(1)
const score = ref(0)
const totalSpawned = ref(0)
const gameBoard = ref<Cell[]>([])
const gameInterval = ref<number | null>(null)
const updateInterval = ref<number | null>(null)
const gameTimer = ref<number | null>(null)
const gameTimeLeft = ref(60) // 遊戲時間 60 秒
const isGameRunning = ref(false)
const isFullscreen = ref(false)
const cellSize = ref(25) // 動態格子大小

// 遊戲配置
const BOARD_ROWS = 6
const BOARD_COLS = 12
const CELL_SIZE = 25

// 根據難度計算的遊戲參數
const whiteDuration = computed(() => {
  // 難度1: 5秒, 難度5: 1秒
  return Math.max(1, 6 - level.value)
})

const spawnRate = computed(() => {
  // 難度1: 每5秒1個, 難度5: 每5秒10個
  return Math.min(10, level.value * 2)
})

const spawnInterval = computed(() => {
  // 根據生成速率計算間隔時間（毫秒）
  return Math.max(500, 5000 / spawnRate.value)
})

// 初始化遊戲棋盤
const initBoard = () => {
  gameBoard.value = []
  for (let row = 0; row < BOARD_ROWS; row++) {
    for (let col = 0; col < BOARD_COLS; col++) {
      gameBoard.value.push({
        id: row * BOARD_COLS + col,
        row,
        col,
        isActive: false,
        timeLeft: 0,
        isAnimating: false
      })
    }
  }
}

// 隨機生成白方塊
const spawnWhiteBlock = () => {
  const availableCells = gameBoard.value.filter(cell => !cell.isActive)
  if (availableCells.length === 0) return

  const randomCell = availableCells[Math.floor(Math.random() * availableCells.length)]
  randomCell.isActive = true
  randomCell.timeLeft = whiteDuration.value * 1000 // 轉換為毫秒
  totalSpawned.value++
}

// 點擊方塊
const clickCell = async (cell: Cell) => {
  if (!cell.isActive || !isGameRunning.value) return

  // 觸發消除特效
  cell.isAnimating = true
  score.value++

  // 等待動畫完成後移除方塊
  setTimeout(() => {
    cell.isActive = false
    cell.isAnimating = false
    cell.timeLeft = 0
  }, 200)
}

// 更新遊戲狀態
const updateGame = () => {
  gameBoard.value.forEach(cell => {
    if (cell.isActive && !cell.isAnimating) {
      cell.timeLeft -= 100
      if (cell.timeLeft <= 0) {
        cell.isActive = false
        cell.timeLeft = 0
      }
    }
  })
}

// 開始遊戲
const startGame = () => {
  stopGame()
  initBoard()
  score.value = 0
  totalSpawned.value = 0
  gameTimeLeft.value = 60
  isGameRunning.value = true

  // 進入全螢幕並調整棋盤大小
  if (!isFullscreen.value) {
    toggleFullscreen()
  } else {
    calculateBoardSize()
  }

  // 設定方塊生成間隔
  gameInterval.value = setInterval(() => {
    if (isGameRunning.value) {
      spawnWhiteBlock()
    }
  }, spawnInterval.value)

  // 設定遊戲狀態更新間隔
  updateInterval.value = setInterval(() => {
    updateGame()
  }, 100)

  // 設定遊戲計時器
  gameTimer.value = setInterval(() => {
    gameTimeLeft.value--
    if (gameTimeLeft.value <= 0) {
      endGame()
    }
  }, 1000)
}

// 停止遊戲
const stopGame = () => {
  isGameRunning.value = false
  if (gameInterval.value) {
    clearInterval(gameInterval.value)
    gameInterval.value = null
  }
  if (updateInterval.value) {
    clearInterval(updateInterval.value)
    updateInterval.value = null
  }
  if (gameTimer.value) {
    clearInterval(gameTimer.value)
    gameTimer.value = null
  }
}

// 重置遊戲
const resetGame = () => {
  stopGame()
  initBoard()
  score.value = 0
  totalSpawned.value = 0
  gameTimeLeft.value = 60
  isGameRunning.value = false
}

// 改變難度
const changeLevel = (newLevel: number) => {
  level.value = newLevel
  if (isGameRunning.value) {
    startGame() // 重新開始遊戲以應用新難度
  }
}

// 計算命中率
const hitRate = computed(() => {
  if (totalSpawned.value === 0) return 0
  return Math.round((score.value / totalSpawned.value) * 100)
})

// 格式化時間顯示
const formatTime = computed(() => {
  const minutes = Math.floor(gameTimeLeft.value / 60)
  const seconds = gameTimeLeft.value % 60
  return `${minutes}:${seconds.toString().padStart(2, '0')}`
})

// 計算棋盤大小
const calculateBoardSize = () => {
  const windowWidth = window.innerWidth
  const windowHeight = window.innerHeight

  // 預留空間給界面元素
  const reservedHeight = isFullscreen.value ? 150 : 250
  const availableWidth = windowWidth - 40 // 左右邊距
  const availableHeight = windowHeight - reservedHeight

  // 計算最適合的格子大小
  const maxCellWidth = Math.floor(availableWidth / BOARD_COLS)
  const maxCellHeight = Math.floor(availableHeight / BOARD_ROWS)

  // 取較小值確保棋盤完全顯示
  cellSize.value = Math.min(maxCellWidth, maxCellHeight, 50) // 最大50px
  cellSize.value = Math.max(cellSize.value, 15) // 最小15px
}

// 全螢幕功能
const toggleFullscreen = async () => {
  try {
    if (!isFullscreen.value) {
      await document.documentElement.requestFullscreen()
      isFullscreen.value = true
    } else {
      await document.exitFullscreen()
      isFullscreen.value = false
    }
    // 延遲計算確保螢幕模式切換完成
    setTimeout(() => {
      calculateBoardSize()
    }, 100)
  } catch (error) {
    console.error('全螢幕切換失敗:', error)
  }
}

// 監聽全螢幕狀態變化
const handleFullscreenChange = () => {
  isFullscreen.value = !!document.fullscreenElement
  calculateBoardSize()
}

// 監聽視窗大小變化
const handleResize = () => {
  calculateBoardSize()
}

// 遊戲結束處理
const endGame = () => {
  stopGame()
  isGameRunning.value = false

  // 顯示結果並退出全螢幕
  setTimeout(() => {
    alert(`遊戲結束！\n最終分數: ${score.value}\n總出現數量: ${totalSpawned.value}\n命中率: ${hitRate.value}%`)
    if (isFullscreen.value) {
      document.exitFullscreen().catch(() => {})
    }
  }, 100)
}

// 生命週期
onMounted(() => {
  initBoard()
  calculateBoardSize()

  // 監聽全螢幕狀態變化
  document.addEventListener('fullscreenchange', handleFullscreenChange)
  // 監聽視窗大小變化
  window.addEventListener('resize', handleResize)
})

onUnmounted(() => {
  stopGame()

  // 清理事件監聽器
  document.removeEventListener('fullscreenchange', handleFullscreenChange)
  window.removeEventListener('resize', handleResize)

  // 確保退出全螢幕
  if (isFullscreen.value) {
    document.exitFullscreen().catch(() => {})
  }
})
</script>

<template>
  <div class="game-container">
    <!-- 頂部控制區域 -->
    <div class="game-header">
      <!-- 左上角難度選擇 -->
      <div class="level-controls">
        <span class="label">難度:</span>
        <button
          v-for="lvl in 5"
          :key="lvl"
          :class="['level-btn', { active: level === lvl }]"
          @click="changeLevel(lvl)"
        >
          {{ lvl }}
        </button>
      </div>

      <!-- 中間控制按鈕和時間顯示 -->
      <div class="game-controls">
        <div class="time-display">
          <span class="label">時間:</span>
          <span :class="['time-value', { warning: gameTimeLeft <= 10, danger: gameTimeLeft <= 5 }]">
            {{ formatTime }}
          </span>
        </div>
        <div class="control-buttons">
          <button class="control-btn start" @click="startGame" :disabled="isGameRunning">開始遊戲</button>
          <button class="control-btn stop" @click="stopGame" :disabled="!isGameRunning">停止遊戲</button>
          <button class="control-btn reset" @click="resetGame">重置遊戲</button>
          <button class="control-btn fullscreen" @click="toggleFullscreen" :disabled="isGameRunning">
            {{ isFullscreen ? '退出全螢幕' : '全螢幕' }}
          </button>
        </div>
      </div>

      <!-- 右上角分數統計 -->
      <div class="score-display">
        <div class="score-item">
          <span class="label">分數:</span>
          <span class="value">{{ score }}</span>
        </div>
        <div class="score-item">
          <span class="label">總數:</span>
          <span class="value">{{ totalSpawned }}</span>
        </div>
        <div class="score-item">
          <span class="label">命中率:</span>
          <span class="value">{{ hitRate }}%</span>
        </div>
      </div>
    </div>

    <!-- 遊戲棋盤 -->
    <div
      class="game-board"
      :style="{
        gridTemplateColumns: `repeat(12, ${cellSize}px)`,
        gridTemplateRows: `repeat(6, ${cellSize}px)`
      }"
    >
      <div
        v-for="cell in gameBoard"
        :key="cell.id"
        :class="[
          'cell',
          {
            'active': cell.isActive,
            'animating': cell.isAnimating
          }
        ]"
        :style="{
          width: `${cellSize}px`,
          height: `${cellSize}px`
        }"
        @click="clickCell(cell)"
      >
        <div v-if="cell.isActive" class="white-block">
          <div class="timer-bar" :style="{ width: `${(cell.timeLeft / (whiteDuration * 1000)) * 100}%` }"></div>
        </div>
      </div>
    </div>
  </div>
</template>

<style scoped>
.game-container {
  display: flex;
  flex-direction: column;
  align-items: center;
  min-height: 100vh;
  background-color: #1a1a1a;
  color: white;
  padding: 20px;
  font-family: 'Courier New', monospace;
  transition: all 0.3s ease;
}

.game-container:fullscreen {
  padding: 10px;
}

.game-header {
  display: flex;
  justify-content: space-between;
  align-items: center;
  width: 100%;
  max-width: 1200px;
  margin-bottom: 30px;
  padding: 0 20px;
}

.level-controls {
  display: flex;
  align-items: center;
  gap: 10px;
}

.label {
  font-weight: bold;
  margin-right: 10px;
  color: #00ff00;
}

.level-btn {
  width: 40px;
  height: 40px;
  border: 2px solid #00ff00;
  background: transparent;
  color: #00ff00;
  cursor: pointer;
  transition: all 0.3s ease;
  font-weight: bold;
}

.level-btn:hover {
  background: rgba(0, 255, 0, 0.2);
}

.level-btn.active {
  background: #00ff00;
  color: #1a1a1a;
}

.game-controls {
  display: flex;
  flex-direction: column;
  align-items: center;
  gap: 15px;
}

.time-display {
  display: flex;
  align-items: center;
  gap: 10px;
  font-size: 1.5em;
  font-weight: bold;
}

.time-value {
  color: #00ff00;
  font-family: 'Courier New', monospace;
  font-size: 1.2em;
  padding: 5px 15px;
  border: 2px solid #00ff00;
  border-radius: 5px;
  background: rgba(0, 255, 0, 0.1);
  min-width: 60px;
  text-align: center;
}

.time-value.warning {
  color: #ffd93d;
  border-color: #ffd93d;
  background: rgba(255, 217, 61, 0.1);
  animation: pulse-warning 1s infinite;
}

.time-value.danger {
  color: #ff6b6b;
  border-color: #ff6b6b;
  background: rgba(255, 107, 107, 0.1);
  animation: pulse-danger 0.5s infinite;
}

.control-buttons {
  display: flex;
  gap: 15px;
}

.control-btn {
  padding: 10px 20px;
  border: 2px solid #00ff00;
  background: transparent;
  color: #00ff00;
  cursor: pointer;
  transition: all 0.3s ease;
  font-weight: bold;
}

.control-btn:hover {
  background: rgba(0, 255, 0, 0.2);
}

.control-btn:disabled {
  opacity: 0.5;
  cursor: not-allowed;
  background: rgba(128, 128, 128, 0.2);
  border-color: #666;
  color: #666;
}

.control-btn:disabled:hover {
  background: rgba(128, 128, 128, 0.2);
}

.control-btn.start:hover {
  border-color: #00ff00;
  color: #00ff00;
}

.control-btn.stop:hover {
  border-color: #ff6b6b;
  color: #ff6b6b;
}

.control-btn.reset:hover {
  border-color: #ffd93d;
  color: #ffd93d;
}

.control-btn.fullscreen:hover {
  border-color: #6c5ce7;
  color: #6c5ce7;
}

.score-display {
  display: flex;
  flex-direction: column;
  gap: 5px;
  text-align: right;
}

.score-item {
  display: flex;
  justify-content: space-between;
  gap: 15px;
}

.value {
  color: #00ff00;
  font-weight: bold;
  min-width: 50px;
}

.game-board {
  display: grid;
  gap: 1px;
  background-color: #333;
  padding: 10px;
  border: 3px solid #00ff00;
  border-radius: 5px;
  margin: 0 auto;
  transition: all 0.3s ease;
}

.cell {
  background-color: #000;
  border: 1px solid #444;
  cursor: pointer;
  position: relative;
  transition: all 0.1s ease;
}

.cell:hover {
  border-color: #666;
}

.white-block {
  width: 100%;
  height: 100%;
  background-color: white;
  position: relative;
  border-radius: 2px;
  transition: all 0.2s ease;
}

.cell.active .white-block {
  animation: spawn 0.2s ease-out;
}

.cell.animating .white-block {
  animation: destroy 0.2s ease-in forwards;
}

.timer-bar {
  position: absolute;
  bottom: 0;
  left: 0;
  height: 3px;
  background: linear-gradient(90deg, #ff0000, #ffff00, #00ff00);
  transition: width 0.1s linear;
}

@keyframes spawn {
  0% {
    transform: scale(0);
    opacity: 0;
  }
  50% {
    transform: scale(1.2);
  }
  100% {
    transform: scale(1);
    opacity: 1;
  }
}

@keyframes destroy {
  0% {
    transform: scale(1);
    opacity: 1;
  }
  50% {
    transform: scale(1.3);
    background-color: #ffff00;
  }
  100% {
    transform: scale(0);
    opacity: 0;
    background-color: #ff0000;
  }
}

@keyframes pulse-warning {
  0%, 100% {
    opacity: 1;
  }
  50% {
    opacity: 0.7;
  }
}

@keyframes pulse-danger {
  0%, 100% {
    opacity: 1;
    transform: scale(1);
  }
  50% {
    opacity: 0.8;
    transform: scale(1.05);
  }
}

/* 響應式設計 */
@media (max-width: 768px) {
  .game-header {
    flex-direction: column;
    gap: 15px;
    text-align: center;
  }

  .game-controls {
    align-items: center;
  }

  .control-buttons {
    flex-wrap: wrap;
    justify-content: center;
    gap: 8px;
  }

  .time-display {
    font-size: 1.2em;
  }

  .control-btn {
    padding: 8px 12px;
    font-size: 0.9em;
  }
}

/* 全螢幕模式樣式 */
:fullscreen .game-header {
  margin-bottom: 15px;
}

:fullscreen .game-controls {
  gap: 10px;
}

:fullscreen .time-display {
  font-size: 1.3em;
}

:fullscreen .control-buttons {
  gap: 10px;
}
</style>
