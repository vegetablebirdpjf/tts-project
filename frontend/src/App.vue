<script setup lang="ts">
  import { ref, computed } from 'vue'

  interface Voice {
    id: string
    name: string
    lang: string
    tag: string
    color: string
  }

  const voices: Voice[] = [
    { id: 'narrator', name: '旁白者', lang: 'zh-CN', tag: '沉稳', color: '#4f8ef7' },
    { id: 'female1', name: '晓晓', lang: 'zh-CN', tag: '温柔', color: '#f76fa8' },
    { id: 'male1', name: '云扬', lang: 'zh-CN', tag: '磁性', color: '#43d9a2' },
    { id: 'female2', name: '艾米', lang: 'en-US', tag: '活泼', color: '#f7c948' },
    { id: 'male2', name: '阿伦', lang: 'en-US', tag: '深沉', color: '#a68af7' },
    { id: 'narrator2', name: '新闻播报', lang: 'zh-CN', tag: '专业', color: '#f77a43' },
  ]

  const selectedVoice = ref<Voice | null>(null)
  const inputText = ref('')
  const isGenerating = ref(false)
  const isPlaying = ref(false)
  const audioUrl = ref<string | null>(null)
  const audioElement = ref<HTMLAudioElement | null>(null)
  const errorMsg = ref('')
  const bars = Array.from({ length: 28 }, (_, i) => i)

  const charCount = computed(() => inputText.value.length)
  const canGenerate = computed(() => selectedVoice.value && inputText.value.trim().length > 0 && !isGenerating.value)

  function selectVoice(v: Voice) {
    selectedVoice.value = v
    audioUrl.value = null
    errorMsg.value = ''
  }

  function getSpeechVoice(lang: string): SpeechSynthesisVoice | undefined {
    const all = window.speechSynthesis.getVoices()
    return all.find(v => v.lang.startsWith(lang.split('-')[0])) ||
      all.find(v => v.lang.includes(lang.split('-')[0])) ||
      all[0]
  }

  async function generate() {
    if (!canGenerate.value || !selectedVoice.value) return
    errorMsg.value = ''
    isGenerating.value = true
    audioUrl.value = null

    try {
      // Use Web Speech API to synthesize and capture audio
      await new Promise<void>((resolve, reject) => {
        const utterance = new SpeechSynthesisUtterance(inputText.value)
        const voices = window.speechSynthesis.getVoices()
        const voice = getSpeechVoice(selectedVoice.value!.lang)
        if (voice) utterance.voice = voice
        utterance.lang = selectedVoice.value!.lang
        utterance.rate = 0.95
        utterance.pitch = selectedVoice.value!.id.includes('female') ? 1.2 : 0.9

        utterance.onend = () => resolve()
        utterance.onerror = (e) => reject(new Error(e.error))

        window.speechSynthesis.cancel()
        window.speechSynthesis.speak(utterance)

        // Simulate generation delay for UX feedback
        setTimeout(() => {
          isGenerating.value = false
          // Mark as "generated" — direct playback via speech API
          audioUrl.value = 'speech-api'
          isPlaying.value = true
        }, 800)

        utterance.onend = () => {
          isPlaying.value = false
          resolve()
        }
      })
    } catch (e: any) {
      errorMsg.value = '生成失败：' + (e.message || '未知错误')
      isGenerating.value = false
    }
  }

  function togglePlay() {
    if (isPlaying.value) {
      window.speechSynthesis.cancel()
      isPlaying.value = false
    } else {
      generate()
    }
  }
</script>

<template>
  <div class="app">
    <!-- Background grain -->
    <div class="noise"></div>
    <div class="glow glow-1"></div>
    <div class="glow glow-2"></div>

    <main class="container">
      <!-- Header -->
      <header class="header">
        <div class="logo-mark">
          <span class="logo-icon">◈</span>
        </div>
        <h1 class="title">VOICE <em>STUDIO</em></h1>
        <p class="subtitle">声音克隆 · 文字转语音</p>
      </header>

      <!-- Voice Selector -->
      <section class="section">
        <label class="section-label">
          <span class="label-num">01</span> 选择声音人物
        </label>
        <div class="voice-grid">
          <button
            v-for="v in voices"
            :key="v.id"
            class="voice-card"
            :class="{ active: selectedVoice?.id === v.id }"
            :style="{ '--accent': v.color }"
            @click="selectVoice(v)"
          >
            <div class="voice-avatar">
              {{ v.name.charAt(0) }}
            </div>
            <div class="voice-info">
              <span class="voice-name">{{ v.name }}</span>
              <span class="voice-tag">{{ v.tag }}</span>
            </div>
            <div class="voice-lang">{{ v.lang }}</div>
            <div class="voice-check">✓</div>
          </button>
        </div>
      </section>

      <!-- Text Input -->
      <section class="section">
        <label class="section-label">
          <span class="label-num">02</span> 输入文本内容
        </label>
        <div class="textarea-wrap">
          <textarea
            v-model="inputText"
            class="textarea"
            placeholder="请在此输入需要转换为语音的文本内容..."
            maxlength="500"
            rows="5"
          ></textarea>
          <div class="char-count" :class="{ warn: charCount > 450 }">
            {{ charCount }} / 500
          </div>
        </div>
      </section>

      <!-- Generate Button -->
      <section class="section generate-section">
        <button
          class="gen-btn"
          :class="{ generating: isGenerating, playing: isPlaying, disabled: !canGenerate && !isPlaying }"
          @click="audioUrl ? togglePlay() : generate()"
          :disabled="!canGenerate && !isPlaying"
        >
          <span class="btn-icon">
            <span v-if="isGenerating" class="spinner"></span>
            <span v-else-if="isPlaying">■</span>
            <span v-else-if="audioUrl">▶</span>
            <span v-else>▶</span>
          </span>
          <span class="btn-text">
            {{ isGenerating ? '合成中...' : isPlaying ? '停止播放' : audioUrl ? '重新播放' : '生成语音' }}
          </span>

          <!-- Wave bars inside button -->
          <span class="btn-waves" :class="{ active: isPlaying }">
            <span v-for="b in bars" :key="b" class="bar" :style="{ animationDelay: `${b * 0.05}s` }"></span>
          </span>
        </button>

        <p v-if="errorMsg" class="error-msg">⚠ {{ errorMsg }}</p>

        <p v-if="audioUrl && !isPlaying" class="success-msg">
          ✓ 语音已生成完毕
        </p>
      </section>

      <!-- Status bar -->
      <footer class="status-bar">
        <span>{{ selectedVoice ? `人物：${selectedVoice.name}` : '未选择声音' }}</span>
        <span class="dot">·</span>
        <span>{{ charCount > 0 ? `${charCount} 字` : '等待输入' }}</span>
        <span class="dot">·</span>
        <span :class="{ active: isPlaying }">{{ isPlaying ? '● 播放中' : '○ 待机' }}</span>
      </footer>
    </main>
  </div>
</template>

<style scoped>
  @import url('https://fonts.googleapis.com/css2?family=Space+Mono:ital,wght@0,400;0,700;1,400&family=Noto+Serif+SC:wght@400;700&display=swap');

  *, *::before, *::after { box-sizing: border-box; margin: 0; padding: 0; }

  .app {
    min-height: 100vh;
    background: #0a0a0f;
    color: #e8e4dc;
    font-family: 'Space Mono', monospace;
    position: relative;
    overflow-x: hidden;
  }

  .noise {
    position: fixed; inset: 0; pointer-events: none; z-index: 0;
    opacity: 0.03;
    background-image: url("data:image/svg+xml,%3Csvg viewBox='0 0 256 256' xmlns='http://www.w3.org/2000/svg'%3E%3Cfilter id='n'%3E%3CfeTurbulence type='fractalNoise' baseFrequency='0.9' numOctaves='4' stitchTiles='stitch'/%3E%3C/filter%3E%3Crect width='100%25' height='100%25' filter='url(%23n)'/%3E%3C/svg%3E");
    background-size: 200px;
  }

  .glow { position: fixed; border-radius: 50%; filter: blur(120px); pointer-events: none; z-index: 0; }
  .glow-1 { width: 500px; height: 500px; top: -150px; left: -100px; background: rgba(79,142,247,0.12); }
  .glow-2 { width: 400px; height: 400px; bottom: -100px; right: -80px; background: rgba(166,138,247,0.1); }

  .container {
    position: relative; z-index: 1;
    max-width: 780px;
    margin: 0 auto;
    padding: 48px 24px 80px;
  }

  /* Header */
  .header { text-align: center; margin-bottom: 60px; }
  .logo-mark { margin-bottom: 16px; }
  .logo-icon { font-size: 32px; color: #4f8ef7; text-shadow: 0 0 20px rgba(79,142,247,0.6); }
  .title {
    font-family: 'Space Mono', monospace;
    font-size: clamp(32px, 6vw, 52px);
    font-weight: 700;
    letter-spacing: 0.15em;
    color: #f0ece4;
    line-height: 1;
  }
  .title em { font-style: normal; color: #4f8ef7; }
  .subtitle {
    margin-top: 12px;
    font-size: 12px;
    letter-spacing: 0.3em;
    color: #5a5750;
    text-transform: uppercase;
  }

  /* Section */
  .section { margin-bottom: 40px; }
  .section-label {
    display: flex; align-items: center; gap: 12px;
    font-size: 11px;
    letter-spacing: 0.25em;
    text-transform: uppercase;
    color: #5a5750;
    margin-bottom: 16px;
  }
  .label-num {
    font-size: 10px; color: #4f8ef7;
    border: 1px solid rgba(79,142,247,0.3);
    padding: 2px 6px; border-radius: 3px;
  }

  /* Voice Grid */
  .voice-grid {
    display: grid;
    grid-template-columns: repeat(auto-fill, minmax(220px, 1fr));
    gap: 12px;
  }

  .voice-card {
    position: relative;
    display: flex; align-items: center; gap: 12px;
    background: rgba(255,255,255,0.03);
    border: 1px solid rgba(255,255,255,0.07);
    border-radius: 12px;
    padding: 14px 16px;
    cursor: pointer;
    transition: all 0.2s;
    text-align: left;
    color: inherit;
    font-family: inherit;
    overflow: hidden;
  }
  .voice-card::before {
    content: ''; position: absolute; inset: 0;
    background: linear-gradient(135deg, var(--accent), transparent);
    opacity: 0; transition: opacity 0.3s;
    border-radius: inherit;
  }
  .voice-card:hover { border-color: rgba(255,255,255,0.15); transform: translateY(-1px); }
  .voice-card:hover::before { opacity: 0.06; }
  .voice-card.active { border-color: var(--accent); box-shadow: 0 0 20px rgba(79,142,247,0.15); }
  .voice-card.active::before { opacity: 0.1; }

  .voice-avatar {
    width: 40px; height: 40px; border-radius: 10px;
    background: var(--accent);
    display: flex; align-items: center; justify-content: center;
    font-size: 18px; font-weight: 700; color: #0a0a0f;
    flex-shrink: 0;
    font-family: 'Noto Serif SC', serif;
    position: relative; z-index: 1;
  }
  .voice-info { flex: 1; position: relative; z-index: 1; }
  .voice-name { display: block; font-size: 14px; font-weight: 700; color: #e8e4dc; font-family: 'Noto Serif SC', serif; }
  .voice-tag { display: block; font-size: 10px; color: var(--accent); letter-spacing: 0.1em; margin-top: 2px; }
  .voice-lang { font-size: 9px; color: #3a3730; letter-spacing: 0.05em; position: relative; z-index: 1; }
  .voice-check {
    position: absolute; top: 8px; right: 10px;
    font-size: 11px; color: var(--accent);
    opacity: 0; transition: opacity 0.2s;
  }
  .voice-card.active .voice-check { opacity: 1; }

  /* Textarea */
  .textarea-wrap { position: relative; }
  .textarea {
    width: 100%;
    background: rgba(255,255,255,0.03);
    border: 1px solid rgba(255,255,255,0.08);
    border-radius: 12px;
    padding: 16px;
    color: #e8e4dc;
    font-family: 'Noto Serif SC', serif;
    font-size: 15px;
    line-height: 1.8;
    resize: vertical;
    transition: border-color 0.2s;
    min-height: 130px;
  }
  .textarea::placeholder { color: #3a3730; }
  .textarea:focus { outline: none; border-color: rgba(79,142,247,0.4); }
  .char-count {
    position: absolute; bottom: 12px; right: 14px;
    font-size: 10px; color: #3a3730; letter-spacing: 0.05em;
    transition: color 0.2s;
  }
  .char-count.warn { color: #f77a43; }

  /* Generate Button */
  .generate-section { display: flex; flex-direction: column; align-items: center; gap: 16px; }

  .gen-btn {
    position: relative;
    display: flex; align-items: center; justify-content: center; gap: 12px;
    background: #0a0a0f;
    border: 1px solid rgba(79,142,247,0.5);
    color: #4f8ef7;
    font-family: 'Space Mono', monospace;
    font-size: 13px;
    letter-spacing: 0.2em;
    text-transform: uppercase;
    padding: 16px 40px;
    border-radius: 100px;
    cursor: pointer;
    transition: all 0.3s;
    min-width: 240px;
    overflow: hidden;
  }
  .gen-btn::after {
    content: ''; position: absolute; inset: 0; border-radius: inherit;
    background: radial-gradient(circle at center, rgba(79,142,247,0.15), transparent 70%);
    opacity: 0; transition: opacity 0.3s;
  }
  .gen-btn:hover:not(.disabled)::after { opacity: 1; }
  .gen-btn:hover:not(.disabled) { border-color: #4f8ef7; box-shadow: 0 0 30px rgba(79,142,247,0.2); }
  .gen-btn.disabled { opacity: 0.3; cursor: not-allowed; }
  .gen-btn.playing { border-color: #43d9a2; color: #43d9a2; }
  .gen-btn.generating { border-color: #f7c948; color: #f7c948; }

  .btn-icon { font-size: 14px; position: relative; z-index: 1; }
  .btn-text { position: relative; z-index: 1; }

  /* Wave bars in button */
  .btn-waves {
    position: absolute; inset: 0;
    display: flex; align-items: center; justify-content: center; gap: 2px;
    opacity: 0; transition: opacity 0.3s;
    pointer-events: none;
  }
  .btn-waves.active { opacity: 0.15; }
  .bar {
    width: 2px;
    height: 8px;
    background: #43d9a2;
    border-radius: 2px;
    animation: wavebar 0.8s ease-in-out infinite alternate;
  }
  @keyframes wavebar {
    0% { height: 4px; }
    100% { height: 24px; }
  }

  /* Spinner */
  .spinner {
    display: inline-block;
    width: 12px; height: 12px;
    border: 2px solid rgba(247,201,72,0.3);
    border-top-color: #f7c948;
    border-radius: 50%;
    animation: spin 0.7s linear infinite;
  }
  @keyframes spin { to { transform: rotate(360deg); } }

  .error-msg { font-size: 12px; color: #f77a43; letter-spacing: 0.05em; }
  .success-msg { font-size: 12px; color: #43d9a2; letter-spacing: 0.05em; }

  /* Status bar */
  .status-bar {
    display: flex; align-items: center; justify-content: center; gap: 12px;
    margin-top: 60px;
    padding-top: 24px;
    border-top: 1px solid rgba(255,255,255,0.06);
    font-size: 10px;
    letter-spacing: 0.15em;
    color: #3a3730;
    text-transform: uppercase;
  }
  .dot { color: #2a2720; }
  .active { color: #43d9a2; }
</style>
