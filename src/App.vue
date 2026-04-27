<template>
  <div class="min-h-screen text-gray-800 font-sans relative z-0">
    <!-- 動態背景區塊 (漂浮光暈 + 點陣網格) -->
    <div class="fixed inset-0 z-[-1] bg-slate-50 overflow-hidden">
      <div class="absolute top-0 -left-10 w-[500px] h-[500px] bg-sky-300 rounded-full mix-blend-multiply filter blur-3xl opacity-70 animate-blob"></div>
      <div class="absolute top-0 -right-10 w-[500px] h-[500px] bg-indigo-300 rounded-full mix-blend-multiply filter blur-3xl opacity-70 animate-blob animation-delay-2000"></div>
      <div class="absolute -bottom-20 left-1/4 w-[500px] h-[500px] bg-purple-300 rounded-full mix-blend-multiply filter blur-3xl opacity-70 animate-blob animation-delay-4000"></div>
      <!-- 網格圖案背景 (增加層次感) -->
      <div class="absolute inset-0 bg-[linear-gradient(to_right,#80808012_1px,transparent_1px),linear-gradient(to_bottom,#80808012_1px,transparent_1px)] bg-[size:24px_24px]"></div>
    </div>

    <!-- 導覽列 -->
    <header class="fixed top-0 left-0 w-full bg-blue-400/95 backdrop-blur-sm text-white p-4 shadow-md z-30 transition-transform duration-300 ease-in-out transform -translate-y-full hover:translate-y-0 group">
      
      <!-- 觸發下拉的感應區塊 (預設顯示在螢幕最上方，hover 時導覽列滑出) -->
      <div class="absolute -bottom-5 left-0 w-full h-5 flex justify-center cursor-pointer">
        <div class="w-24 h-1.5 bg-blue-500/60 rounded-b-lg shadow-sm group-hover:opacity-0 transition-opacity duration-300 mt-0.5" title="碰觸展開選單"></div>
      </div>

      <!-- 全局音效開關 -->
      <button @click="isMuted = !isMuted" class="absolute top-4 right-4 md:right-8 bg-white/20 hover:bg-white/30 p-2 rounded-full backdrop-blur-sm transition-all duration-300 text-xl z-30" :title="isMuted ? '開啟音效' : '關閉音效'">
        <span v-if="!isMuted">🔊</span>
        <span v-else>🔇</span>
      </button>

      <h1 class="text-2xl font-bold text-center">數位教學輔助平台</h1>
      <nav class="mt-6 flex flex-wrap justify-center gap-6 md:gap-12 pb-2">
        <button 
          v-for="tab in tabs" 
          :key="tab.id"
          @click="currentTab = tab.id"
          :class="['px-4 py-2 rounded-md transition-colors', currentTab === tab.id ? 'bg-white text-blue-500 font-semibold' : 'bg-blue-500 hover:bg-blue-400']"
        >
          {{ tab.name }}
        </button>
      </nav>
    </header>

    <!-- 主要內容區 -->
    <main class="max-w-4xl mx-auto p-4 md:p-6 mt-24 bg-white rounded-xl shadow-lg relative z-10">
      
      <transition name="fade" mode="out-in">
        <div :key="currentTab">
      <!-- 1. 隨機抽籤機 -->
      <section v-if="currentTab === 'picker'" class="space-y-6">
        <h2 class="text-xl font-bold border-b pb-2">隨機抽籤機</h2>
        <div class="grid md:grid-cols-2 gap-6">
          <div>
            <label class="block mb-2 font-medium">匯入名單 (每行一個名字)</label>
            <textarea 
              v-model="rawNames" 
              class="w-full h-40 p-2 border rounded-md focus:ring-2 focus:ring-indigo-500 outline-none"
              placeholder="請輸入學生姓名或組別..."></textarea>
            <div class="mt-5 flex items-center gap-4">
              <input type="checkbox" id="exclude" v-model="excludePicked" class="w-4 h-4 text-indigo-600">
              <label for="exclude">抽中後排除 (不重複中獎)</label>
            </div>
            <button 
              @click="startPicking" 
              :disabled="isPicking || availableNames.length === 0"
              class="mt-8 w-full bg-indigo-600 text-white py-4 rounded-md hover:bg-indigo-700 disabled:opacity-50 font-bold text-lg tracking-wider"
            >
              {{ isPicking ? '抽籤中...' : '開始抽籤' }}
            </button>
          </div>
          <div class="flex flex-col items-center justify-center bg-gray-100 rounded-xl p-8 min-h-[200px]">
            <span class="text-sm text-gray-500 mb-2">抽中結果</span>
            <div 
              class="text-4xl font-bold text-indigo-700 transition-all duration-100"
              :class="{ 'scale-125 animate-pulse': isPicking }"
            >
              {{ currentResult || '?' }}
            </div>
            <div v-if="availableNames.length === 0 && rawNames.trim() !== ''" class="mt-4 text-red-500 font-medium">
              名單已抽完！
            </div>
          </div>
        </div>
      </section>

      <!-- 2. 小組競賽記分板 -->
      <section v-else-if="currentTab === 'scoreboard'" class="space-y-6">
        <h2 class="text-xl font-bold border-b pb-2">小組競賽記分板</h2>
        <div class="flex flex-wrap sm:flex-nowrap gap-6 mb-8">
          <input 
            v-model="newGroupName" 
            @keyup.enter="addGroup"
            class="flex-1 p-3 border rounded-md outline-none focus:ring-2 focus:ring-indigo-500 text-lg" 
            placeholder="輸入新小組名稱..."
          />
          <button @click="addGroup" class="bg-green-600 text-white px-8 py-3 rounded-md hover:bg-green-700 font-bold text-lg">新增</button>
        </div>
        
        <transition-group name="list" tag="div" class="space-y-3 overflow-hidden">
          <!-- 動態排序顯示與動畫 -->
          <div 
            v-for="(group, index) in sortedGroups" 
            :key="group.id"
            class="flex flex-col md:flex-row md:items-center justify-between p-4 bg-gray-50 border rounded-lg shadow-sm"
          >
            <div class="flex items-center gap-4 mb-2 md:mb-0">
              <span class="text-2xl font-bold" :class="index === 0 ? 'text-yellow-500' : 'text-gray-400'">
                #{{ index + 1 }}
              </span>
              <span class="text-lg font-semibold">{{ group.name }}</span>
            </div>
            <div class="flex items-center gap-4">
              <span class="text-2xl font-bold text-indigo-600 w-12 text-center">{{ group.score }}</span>
              <div class="flex flex-wrap gap-6">
                <button @click="updateScore(group.id, -1)" class="bg-red-100 text-red-700 px-5 py-2 rounded hover:bg-red-200 font-bold text-lg">-1</button>
                <button @click="updateScore(group.id, 1)" class="bg-blue-100 text-blue-700 px-5 py-2 rounded hover:bg-blue-200 font-bold text-lg">+1</button>
                <button @click="updateScore(group.id, 5)" class="bg-green-100 text-green-700 px-5 py-2 rounded hover:bg-green-200 font-bold text-lg">+5</button>
              </div>
            </div>
          </div>
        </transition-group>
      </section>

      <!-- 3. 小組報告計時器 -->
      <section v-else-if="currentTab === 'timer'" class="space-y-6 text-center">
        <h2 class="text-xl font-bold border-b pb-2 text-left">報告計時器</h2>
        
        <div class="flex flex-wrap justify-center gap-6 sm:gap-8 md:gap-12 mb-10">
          <button @click="setTimer(3)" class="bg-indigo-100 text-indigo-700 px-6 py-2 rounded-full font-medium hover:bg-indigo-200">3 分鐘</button>
          <button @click="setTimer(5)" class="bg-indigo-100 text-indigo-700 px-6 py-2 rounded-full font-medium hover:bg-indigo-200">5 分鐘</button>
          <button @click="setTimer(10)" class="bg-indigo-100 text-indigo-700 px-6 py-2 rounded-full font-medium hover:bg-indigo-200">10 分鐘</button>
        </div>

        <div 
          class="text-7xl md:text-9xl font-mono font-bold transition-colors duration-300"
          :class="{
            'text-red-600 animate-pulse': timeLeft > 0 && timeLeft <= 10,
            'text-gray-800': timeLeft > 10,
            'text-gray-300': timeLeft === 0 && !isTimerRunning
          }"
        >
          {{ formattedTime }}
        </div>

        <!-- 計時進度條 -->
        <div class="w-full max-w-md mx-auto bg-gray-200 rounded-full h-4 mt-6 overflow-hidden shadow-inner">
          <div class="h-full rounded-full transition-all duration-1000 ease-linear"
               :class="timeLeft <= 10 && timeLeft > 0 ? 'bg-red-500' : 'bg-indigo-500'"
               :style="{ width: `${(timeLeft / totalTime) * 100}%` }">
          </div>
        </div>

        <div class="flex flex-wrap justify-center gap-8 md:gap-16 mt-12">
          <button @click="toggleTimer" class="bg-indigo-600 text-white px-8 py-3 rounded-lg text-lg hover:bg-indigo-700">
            {{ isTimerRunning ? '暫停' : '開始' }}
          </button>
          <button @click="resetTimer" class="bg-gray-200 text-gray-700 px-8 py-3 rounded-lg text-lg hover:bg-gray-300">
            重設
          </button>
        </div>
      </section>

      <!-- 4. 數位測驗系統 -->
      <section v-else-if="currentTab === 'exam'" class="space-y-6">
        <h2 class="text-xl font-bold border-b pb-2">數位測驗系統 (隨機測驗)</h2>
        
        <div v-if="!examSubmitted" class="space-y-8">
          <div v-for="(q, qIndex) in questions" :key="q.id" class="bg-gray-50 p-5 rounded-lg border border-gray-100 shadow-sm">
            <h3 class="font-semibold text-lg mb-4">
              <span class="text-indigo-600 font-bold mr-1">
                [{{ q.type === 'single' ? '單選' : q.type === 'multiple' ? '多選' : '是非' }}]
              </span>
              {{ qIndex + 1 }}. {{ q.title }}
            </h3>
            
            <div class="space-y-5">
              <!-- 單選 / 是非 區塊 -->
              <template v-if="q.type === 'single' || q.type === 'tf'">
                <label 
                  v-for="(option, oIndex) in q.options" 
                  :key="oIndex"
                  class="flex items-center p-3 bg-white border rounded-lg cursor-pointer hover:bg-indigo-50 transition-colors"
                  :class="{ 'border-indigo-500 bg-indigo-50 ring-1 ring-indigo-500': userAnswers[q.id] === oIndex }"
                >
                  <input 
                    type="radio" 
                    :name="'q-' + q.id" 
                    :value="oIndex" 
                    v-model="userAnswers[q.id]"
                    class="w-5 h-5 text-indigo-600"
                  >
                  <span class="ml-3">{{ option }}</span>
                </label>
              </template>

              <!-- 多選 區塊 -->
              <template v-else-if="q.type === 'multiple'">
                <label 
                  v-for="(option, oIndex) in q.options" 
                  :key="oIndex"
                  class="flex items-center p-3 bg-white border rounded-lg cursor-pointer hover:bg-indigo-50 transition-colors"
                  :class="{ 'border-indigo-500 bg-indigo-50 ring-1 ring-indigo-500': userAnswers[q.id] && userAnswers[q.id].includes(oIndex) }"
                >
                  <input 
                    type="checkbox" 
                    :value="oIndex" 
                    v-model="userAnswers[q.id]"
                    class="w-5 h-5 text-indigo-600 rounded"
                  >
                  <span class="ml-3">{{ option }}</span>
                </label>
              </template>
            </div>
          </div>
          
          <button 
            @click="submitExam" 
            class="w-full bg-indigo-600 text-white py-4 rounded-lg text-lg font-bold hover:bg-indigo-700 shadow-md transition-transform active:scale-95"
          >
            交卷並看成績
          </button>
        </div>

        <!-- 測驗結果回饋 -->
        <div v-else class="space-y-6">
          <div class="text-center p-6 bg-indigo-50 rounded-lg shadow-inner">
            <h3 class="text-2xl font-bold text-indigo-800">測驗結果</h3>
            <p class="text-5xl font-extrabold mt-3 text-indigo-600">{{ examScore }} / 100</p>
          </div>

          <div v-for="(q, qIndex) in questions" :key="q.id" class="bg-gray-50 p-5 rounded-lg border-l-4 shadow-sm" :class="isAnswerCorrect(q) ? 'border-green-500' : 'border-red-500'">
            <h3 class="font-semibold text-lg mb-2">
              {{ qIndex + 1 }}. {{ q.title }}
              <span v-if="isAnswerCorrect(q)" class="text-green-600 text-sm ml-2 font-bold">✔ 答對</span>
              <span v-else class="text-red-600 text-sm ml-2 font-bold">✘ 答錯</span>
            </h3>
            
            <p class="text-gray-700 mb-3">你的答案：<span class="font-semibold">{{ getAnswerText(q, userAnswers[q.id]) }}</span></p>
            <div class="bg-white p-4 rounded border text-sm shadow-sm">
              <p class="font-medium text-green-700 mb-1">正確答案：{{ getAnswerText(q, q.correctAnswer) }}</p>
              <p class="text-gray-600">解析：{{ q.explanation }}</p>
            </div>
          </div>

          <button @click="resetExam" class="w-full bg-gray-800 text-white py-4 rounded-lg text-lg font-bold hover:bg-gray-700 shadow-md transition-transform active:scale-95">
            重新測驗 (隨機新題目)
          </button>
        </div>
      </section>
        </div>
      </transition>
    </main>
  </div>
</template>

<script setup>
import { ref, computed, onUnmounted } from 'vue';

// --- 音效系統 (Web Audio API) ---
const isMuted = ref(false); // 音效開關狀態
const AudioContext = window.AudioContext || window.webkitAudioContext;
const audioCtx = new AudioContext();

const playTone = (freq, type, duration, vol = 0.1) => {
  if (isMuted.value) return; // 若靜音則不播放
  
  if (audioCtx.state === 'suspended') audioCtx.resume();
  const osc = audioCtx.createOscillator();
  const gain = audioCtx.createGain();
  osc.type = type;
  osc.frequency.setValueAtTime(freq, audioCtx.currentTime);
  gain.gain.setValueAtTime(vol, audioCtx.currentTime);
  gain.gain.exponentialRampToValueAtTime(0.01, audioCtx.currentTime + duration);
  osc.connect(gain);
  gain.connect(audioCtx.destination);
  osc.start();
  osc.stop(audioCtx.currentTime + duration);
};

const playTick = () => playTone(600, 'sine', 0.05, 0.05);
const playTada = () => { playTone(523.25, 'triangle', 0.2, 0.1); setTimeout(() => playTone(659.25, 'triangle', 0.4, 0.15), 150); };
const playCoin = () => playTone(987.77, 'sine', 0.1, 0.05);
const playAlarm = () => { let i = 0; const intv = setInterval(() => { playTone(800, 'square', 0.2, 0.1); i++; if(i>=5) clearInterval(intv); }, 400); };

// --- 原生彩帶特效 (Confetti) ---
const triggerConfetti = () => {
  const colors = ['#fce18a', '#ff726d', '#b48def', '#f4306d', '#55efc4', '#81ecec'];
  for (let i = 0; i < 120; i++) {
    const confetti = document.createElement('div');
    confetti.className = 'fixed w-3 h-3 z-50 pointer-events-none rounded-sm';
    confetti.style.backgroundColor = colors[Math.floor(Math.random() * colors.length)];
    confetti.style.left = Math.random() * 100 + 'vw';
    confetti.style.top = '-5vh';
    document.body.appendChild(confetti);

    const fallDuration = Math.random() * 3 + 2;
    const xOffset = (Math.random() - 0.5) * 300;

    confetti.animate([
      { transform: `translate3d(0, 0, 0) rotate(0deg)`, opacity: 1 },
      { transform: `translate3d(${xOffset}px, 105vh, 0) rotate(${Math.random() * 720}deg)`, opacity: 0 }
    ], {
      duration: fallDuration * 1000,
      easing: 'cubic-bezier(.37,0,.63,1)'
    });

    setTimeout(() => confetti.remove(), fallDuration * 1000);
  }
};

// --- 導覽列狀態 ---
const tabs = [
  { id: 'picker', name: '隨機抽籤' },
  { id: 'scoreboard', name: '小組競賽' },
  { id: 'timer', name: '報告計時' },
  { id: 'exam', name: '數位測驗' }
];
const currentTab = ref('picker');

// ==========================================
// 1. 隨機抽籤機邏輯
// ==========================================
const rawNames = ref("第一組\n第二組\n第三組\n王小明\n陳大牛");
const currentResult = ref("");
const isPicking = ref(false);
const excludePicked = ref(true);
let pickingInterval = null;

const availableNames = computed(() => {
  return rawNames.value.split('\n').map(n => n.trim()).filter(n => n !== "");
});

const startPicking = () => {
  if (availableNames.value.length === 0) return;
  isPicking.value = true;
  
  // 動態跳轉視覺效果
  let counter = 0;
  pickingInterval = setInterval(() => {
    const randomIndex = Math.floor(Math.random() * availableNames.value.length);
    currentResult.value = availableNames.value[randomIndex];
    playTick(); // 抽籤跳動音效
    counter++;
    
    // 兩秒後停止並決定最終結果
    if (counter > 20) {
      clearInterval(pickingInterval);
      isPicking.value = false;
      playTada(); // 抽中歡呼音效
      
      // 若選擇排除，則將該名字從 rawNames 中移除
      if (excludePicked.value) {
        const finalName = currentResult.value;
        const updatedList = availableNames.value.filter(n => n !== finalName);
        rawNames.value = updatedList.join('\n');
      }
    }
  }, 100); // 每 100ms 跳轉一次
};

// ==========================================
// 2. 小組競賽記分板邏輯
// ==========================================
const newGroupName = ref("");
const groups = ref([
  { id: 1, name: "閃電組", score: 0 },
  { id: 2, name: "飛鷹組", score: 5 },
  { id: 3, name: "猛虎組", score: 2 }
]);

// 即時排序功能 (分數高到低)
const sortedGroups = computed(() => {
  return [...groups.value].sort((a, b) => b.score - a.score);
});

const addGroup = () => {
  if (newGroupName.value.trim() === "") return;
  groups.value.push({
    id: Date.now(),
    name: newGroupName.value.trim(),
    score: 0
  });
  newGroupName.value = "";
};

const updateScore = (id, points) => {
  const group = groups.value.find(g => g.id === id);
  if (group) {
    group.score += points;
    if (points > 0) playCoin(); // 加分音效
  }
};

// ==========================================
// 3. 小組報告計時器邏輯
// ==========================================
const timeLeft = ref(180); // 預設 3 分鐘 (秒)
const totalTime = ref(180); // 紀錄總時長供進度條使用
const isTimerRunning = ref(false);
let timerInterval = null;

const formattedTime = computed(() => {
  const m = Math.floor(timeLeft.value / 60).toString().padStart(2, '0');
  const s = (timeLeft.value % 60).toString().padStart(2, '0');
  return `${m}:${s}`;
});

const setTimer = (minutes) => {
  clearInterval(timerInterval);
  isTimerRunning.value = false;
  timeLeft.value = minutes * 60;
  totalTime.value = minutes * 60;
};

const toggleTimer = () => {
  if (isTimerRunning.value) {
    clearInterval(timerInterval);
    isTimerRunning.value = false;
  } else {
    if (timeLeft.value <= 0) return;
    isTimerRunning.value = true;
    timerInterval = setInterval(() => {
      if (timeLeft.value > 0) {
        timeLeft.value--;
      } else {
        clearInterval(timerInterval);
        isTimerRunning.value = false;
        playAlarm(); // 時間到音效
      }
    }, 1000);
  }
};

const resetTimer = () => {
  clearInterval(timerInterval);
  isTimerRunning.value = false;
  timeLeft.value = totalTime.value;
};

// 元件卸載時清除計時器
onUnmounted(() => {
  clearInterval(pickingInterval);
  clearInterval(timerInterval);
});

// ==========================================
// 4. 數位測驗系統邏輯
// ==========================================
const allQuestions = [
  {
    id: 1, type: 'single',
    title: "Vue.js 主要是用來建立什麼的框架？",
    options: ["後端伺服器", "資料庫管理", "使用者介面 (UI)", "作業系統"],
    correctAnswer: 2,
    explanation: "Vue.js 是一個用於構建使用者介面的漸進式 JavaScript 框架。"
  },
  {
    id: 2, type: 'single',
    title: "在 Vue 3 中，下列哪一個 API 被廣泛推崇用於邏輯重用？",
    options: ["Options API", "Composition API", "Mixins", "Filters"],
    correctAnswer: 1,
    explanation: "Vue 3 引入了 Composition API 來更好地組織與重用程式碼邏輯。"
  },
  {
    id: 3, type: 'multiple',
    title: "下列哪些屬於前端的 JavaScript 框架或函式庫？ (多選)",
    options: ["Vue.js", "Django", "React", "Spring Boot", "Angular"],
    correctAnswer: [0, 2, 4],
    explanation: "Django 和 Spring Boot 是後端框架，Vue, React, Angular 則是前端領域的三大主流。"
  },
  {
    id: 4, type: 'multiple',
    title: "在網頁設計中，CSS 主要負責哪些功能？ (多選)",
    options: ["網頁結構定義", "顏色與字體設定", "排版與定位", "資料庫連線"],
    correctAnswer: [1, 2],
    explanation: "網頁結構由 HTML 定義，資料庫由後端負責，CSS 專注於外觀、排版與動畫等視覺呈現。"
  },
  {
    id: 5, type: 'tf',
    title: "HTML 是縮寫自 HyperText Markup Language？ (是非)",
    options: ["⭕ 是", "❌ 否"],
    correctAnswer: 0,
    explanation: "HTML 全名為 HyperText Markup Language (超文本標記語言)。"
  },
  {
    id: 6, type: 'tf',
    title: "Tailwind CSS 是一個基於組件 (Component-based) 的 CSS 框架，如 Bootstrap 一樣提供現成的 UI 組件。 (是非)",
    options: ["⭕ 是", "❌ 否"],
    correctAnswer: 1,
    explanation: "Tailwind CSS 是一個功能類優先 (Utility-first) 的框架，不提供預設組件，而是提供基礎的 CSS class 讓你自由組合。"
  },
  {
    id: 7, type: 'single',
    title: "下列哪一種色彩格式支援透明度 (Alpha) 設定？",
    options: ["HEX (六碼)", "RGB", "RGBA", "CMYK"],
    correctAnswer: 2,
    explanation: "RGBA 中的 A 代表 Alpha，用來設定透明度。"
  }
];

const questions = ref([]);
const userAnswers = ref({});
const examSubmitted = ref(false);
const examScore = ref(0);

const initExam = () => {
  // 隨機打亂所有題目，並抽取前 5 題（若題庫少於 5 題則全取）
  const shuffled = [...allQuestions].sort(() => 0.5 - Math.random());
  questions.value = shuffled.slice(0, 5);
  
  userAnswers.value = {};
  // 多選題預設值必須為空陣列，否則 v-model 會出錯
  questions.value.forEach(q => {
    if (q.type === 'multiple') {
      userAnswers.value[q.id] = [];
    }
  });
  
  examSubmitted.value = false;
  examScore.value = 0;
};

// 驗證答案是否正確 (包含多選陣列的判斷)
const isAnswerCorrect = (q) => {
  const ans = userAnswers.value[q.id];
  if (q.type === 'multiple') {
    if (!Array.isArray(ans)) return false;
    // 將陣列排序後轉為字串比較
    const userAns = [...ans].sort();
    const correctAns = [...q.correctAnswer].sort();
    return JSON.stringify(userAns) === JSON.stringify(correctAns);
  }
  return ans === q.correctAnswer;
};

// 取得將索引轉為文字的答案 (供顯示作答結果用)
const getAnswerText = (q, answerData) => {
  if (answerData === undefined || answerData === null || (Array.isArray(answerData) && answerData.length === 0)) {
    return '未作答';
  }
  if (q.type === 'multiple') {
    return answerData.map(idx => q.options[idx]).join('、');
  }
  return q.options[answerData];
};

const submitExam = () => {
  let correctCount = 0;
  questions.value.forEach(q => {
    if (isAnswerCorrect(q)) {
      correctCount++;
    }
  });
  examScore.value = Math.round((correctCount / questions.value.length) * 100);
  examSubmitted.value = true;
  if (examScore.value >= 60) {
    playTada(); // 及格音效
    if (examScore.value === 100) {
      triggerConfetti(); // 滿分觸發彩帶
    }
  } else {
    playTone(300, 'sawtooth', 0.4, 0.1); // 不及格低沉音效
  }
};

const resetExam = () => {
  initExam(); // 重新抽題與初始化
};

// 系統載入時初始化測驗
initExam();
</script>
