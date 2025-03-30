<script setup lang="ts">
import { ref } from 'vue'

const endpoint = ref('http://127.0.0.1:9997')
const model = ref('FishSpeech-1.5')
const prompt = ref('')
const isStreaming = ref(false)
const audioUrl = ref('')
const isLoading = ref(false)
const error = ref('')
const wordCount = ref(0)
const referenceAudio = ref<File | null>(null)
const referenceText = ref('')
const speed = ref(1)

const updateWordCount = (text: string) => {
  wordCount.value = text.length
}

const handleFileUpload = (event: Event) => {
  const target = event.target as HTMLInputElement
  if (target.files && target.files.length > 0) {
    referenceAudio.value = target.files[0]
  }
}

const generateSpeech = async () => {
  if (!prompt.value.trim()) {
    error.value = 'Please enter some text'
    return
  }

  isLoading.value = true
  error.value = ''

  try {
    let formData: FormData | null = null

    if (referenceAudio.value) {
      formData = new FormData()
      formData.append('prompt_speech', referenceAudio.value)
      formData.append('model', model.value)
      formData.append('input', prompt.value)
      formData.append('voice', 'echo')
      formData.append('stream', String(isStreaming.value))
      formData.append('speed', String(speed.value))
      if (referenceText.value) {
        formData.append('kwargs', JSON.stringify({ prompt_text: referenceText.value }))
      }
    }

    const response = await fetch(`${endpoint.value}/v1/audio/speech`, {
      method: 'POST',
      headers: formData
        ? undefined
        : {
            accept: 'application/json',
            'Content-Type': 'application/json',
          },
      body:
        formData ||
        JSON.stringify({
          model: model.value,
          input: prompt.value,
          voice: 'echo',
          stream: isStreaming.value,
          speed: speed.value,
        }),
    })

    if (!response.ok) {
      throw new Error('Failed to generate speech')
    }

    const blob = await response.blob()
    audioUrl.value = URL.createObjectURL(blob)
  } catch (err) {
    error.value = err instanceof Error ? err.message : 'An error occurred'
  } finally {
    isLoading.value = false
  }
}

const clearAudio = () => {
  if (audioUrl.value) {
    URL.revokeObjectURL(audioUrl.value)
    audioUrl.value = ''
  }
}

const clearReference = () => {
  referenceAudio.value = null
  referenceText.value = ''
}
</script>

<template>
  <div class="container">
    <div class="layout">
      <!-- 左侧输入区域 -->
      <div class="input-panel">
        <div class="header-section">
          <h1>文本转语音</h1>
          <a
            href="https://inference.readthedocs.io/en/latest/models/model_abilities/audio.html#audio-speech"
            target="_blank"
            class="reference-link"
          >
            参考文档
          </a>
        </div>

        <div class="config-section">
          <div class="config-item">
            <label>API 地址:</label>
            <input
              v-model="endpoint"
              type="text"
              class="config-input"
              placeholder="http://127.0.0.1:9997"
            />
          </div>
          <div class="config-item">
            <label>模型:</label>
            <input v-model="model" type="text" class="config-input" placeholder="FishSpeech-1.5" />
          </div>
        </div>

        <div class="info-text">
          <span>感谢用户对字数限制为 5,000 UTF-8 字节。</span>
        </div>

        <div class="text-input-section">
          <textarea
            v-model="prompt"
            placeholder="请输入要转换的文本..."
            class="text-input"
            @input="updateWordCount(prompt)"
          ></textarea>
          <div class="word-count">{{ wordCount }}/500</div>
        </div>

        <div class="reference-section">
          <div class="reference-header">参考音频（可选）</div>
          <div class="reference-upload">
            <input
              type="file"
              accept="audio/mp3,audio/wav"
              @change="handleFileUpload"
              class="file-input"
              id="audio-upload"
            />
            <label for="audio-upload" class="upload-label">
              {{ referenceAudio ? referenceAudio.name : '选择音频文件' }}
            </label>
            <button v-if="referenceAudio" @click="clearReference" class="clear-reference-btn">
              清除
            </button>
          </div>
          <textarea
            v-if="referenceAudio"
            v-model="referenceText"
            placeholder="请输入参考音频中说的文字..."
            class="reference-text"
            rows="2"
          ></textarea>
        </div>

        <div class="controls-section">
          <div class="voice-controls">
            <div class="control-group">
              <label class="stream-toggle">
                <input type="checkbox" v-model="isStreaming" />
                <span>启用流式输出</span>
              </label>
            </div>
            <div class="control-group">
              <label class="speed-control">
                <span>语速: {{ (speed * 100).toFixed(0) }}%</span>
                <input
                  type="range"
                  v-model="speed"
                  min="0.1"
                  max="1"
                  step="0.1"
                  class="speed-slider"
                />
              </label>
            </div>
          </div>

          <button @click="generateSpeech" :disabled="isLoading" class="generate-btn">
            {{ isLoading ? '生成中...' : '创建' }}
          </button>
        </div>

        <div v-if="error" class="error-message">
          {{ error }}
        </div>
      </div>

      <!-- 右侧音频区域 -->
      <div class="output-panel">
        <div class="audio-section" :class="{ 'no-audio': !audioUrl }">
          <div v-if="audioUrl" class="audio-card">
            <div class="audio-info">
              <div class="audio-title">生成的音频</div>
            </div>
            <audio controls :src="audioUrl" class="audio-player"></audio>
            <button @click="clearAudio" class="clear-btn">清除</button>
          </div>
          <div v-else class="placeholder">
            <div class="placeholder-text">音频将在这里显示</div>
          </div>
        </div>
      </div>
    </div>
  </div>
</template>

<style scoped>
.container {
  max-width: 1400px;
  margin: 0 auto;
  padding: 2rem;
  height: calc(100vh - 4rem);
}

.layout {
  display: grid;
  grid-template-columns: 3fr 2fr;
  gap: 2rem;
  height: 100%;
}

.input-panel {
  background: white;
  border-radius: 8px;
  padding: 2rem;
  box-shadow: 0 2px 8px rgba(0, 0, 0, 0.05);
  display: flex;
  flex-direction: column;
}

.header-section {
  display: flex;
  justify-content: space-between;
  align-items: center;
  margin-bottom: 1.5rem;
}

.reference-link {
  color: #2563eb;
  text-decoration: none;
  font-size: 0.9rem;
  display: flex;
  align-items: center;
  gap: 0.5rem;
}

.reference-link:hover {
  text-decoration: underline;
}

.config-section {
  background: #f9fafb;
  border-radius: 8px;
  padding: 1rem;
  margin-bottom: 1.5rem;
}

.config-item {
  display: flex;
  align-items: center;
  gap: 0.5rem;
  margin-bottom: 0.5rem;
}

.config-item:last-child {
  margin-bottom: 0;
}

.config-item label {
  color: #666;
  font-size: 0.9rem;
  min-width: 60px;
}

.config-input {
  flex: 1;
  padding: 0.5rem;
  border: 1px solid #e5e7eb;
  border-radius: 6px;
  font-size: 0.9rem;
}

.config-input:focus {
  outline: none;
  border-color: #2563eb;
  box-shadow: 0 0 0 2px rgba(37, 99, 235, 0.1);
}

.output-panel {
  display: flex;
  flex-direction: column;
}

h1 {
  font-size: 1.5rem;
  color: #333;
  margin-bottom: 1.5rem;
  font-weight: 500;
}

.info-text {
  color: #666;
  font-size: 0.9rem;
  margin-bottom: 1rem;
}

.text-input-section {
  position: relative;
  flex: 1;
  display: flex;
  flex-direction: column;
  min-height: 0;
  margin-bottom: 1rem;
}

.text-input {
  flex: 1;
  padding: 1rem;
  border: 1px solid #e5e7eb;
  border-radius: 8px;
  font-size: 1rem;
  resize: none;
  background: #f9fafb;
  min-height: 200px;
}

.text-input:focus {
  outline: none;
  border-color: #2563eb;
  box-shadow: 0 0 0 2px rgba(37, 99, 235, 0.1);
}

.word-count {
  position: absolute;
  right: 1rem;
  bottom: 1rem;
  color: #666;
  font-size: 0.875rem;
}

.reference-section {
  background: #f9fafb;
  border-radius: 8px;
  padding: 1rem;
  margin-bottom: 1rem;
}

.reference-header {
  font-size: 0.9rem;
  color: #666;
  margin-bottom: 0.5rem;
}

.reference-upload {
  display: flex;
  gap: 0.5rem;
  align-items: center;
  margin-bottom: 0.5rem;
}

.file-input {
  display: none;
}

.upload-label {
  flex: 1;
  padding: 0.5rem 1rem;
  background: white;
  border: 1px dashed #e5e7eb;
  border-radius: 6px;
  cursor: pointer;
  color: #666;
  font-size: 0.9rem;
  white-space: nowrap;
  overflow: hidden;
  text-overflow: ellipsis;
}

.upload-label:hover {
  border-color: #2563eb;
  color: #2563eb;
}

.clear-reference-btn {
  padding: 0.5rem 1rem;
  background: #ef4444;
  color: white;
  border: none;
  border-radius: 6px;
  cursor: pointer;
  font-size: 0.9rem;
}

.clear-reference-btn:hover {
  background: #dc2626;
}

.reference-text {
  width: 100%;
  padding: 0.5rem;
  border: 1px solid #e5e7eb;
  border-radius: 6px;
  font-size: 0.9rem;
  resize: none;
  margin-top: 0.5rem;
}

.controls-section {
  display: flex;
  justify-content: space-between;
  align-items: center;
  margin-top: 1rem;
}

.voice-controls {
  display: flex;
  flex-direction: column;
  gap: 1rem;
}

.control-group {
  display: flex;
  align-items: center;
  gap: 1rem;
}

.stream-toggle {
  display: flex;
  align-items: center;
  gap: 0.5rem;
  color: #666;
  cursor: pointer;
}

.stream-toggle input {
  width: 1rem;
  height: 1rem;
}

.speed-control {
  display: flex;
  align-items: center;
  gap: 1rem;
  color: #666;
  min-width: 200px;
}

.speed-slider {
  flex: 1;
  height: 4px;
  -webkit-appearance: none;
  background: #e5e7eb;
  border-radius: 2px;
  outline: none;
}

.speed-slider::-webkit-slider-thumb {
  -webkit-appearance: none;
  width: 16px;
  height: 16px;
  background: #2563eb;
  border-radius: 50%;
  cursor: pointer;
  transition: background-color 0.2s;
}

.speed-slider::-webkit-slider-thumb:hover {
  background: #1d4ed8;
}

.generate-btn {
  background: #2563eb;
  color: white;
  border: none;
  padding: 0.75rem 2rem;
  border-radius: 6px;
  font-size: 1rem;
  cursor: pointer;
  transition: background-color 0.2s;
}

.generate-btn:hover {
  background: #1d4ed8;
}

.generate-btn:disabled {
  background: #93c5fd;
  cursor: not-allowed;
}

.error-message {
  margin-top: 1rem;
  padding: 0.75rem;
  color: #dc2626;
  background: #fee2e2;
  border-radius: 6px;
}

.audio-section {
  background: white;
  border-radius: 8px;
  height: 100%;
  box-shadow: 0 2px 8px rgba(0, 0, 0, 0.05);
}

.audio-card {
  height: 100%;
  padding: 1.5rem;
  display: flex;
  flex-direction: column;
}

.audio-info {
  display: flex;
  align-items: center;
  margin-bottom: 1rem;
}

.audio-title {
  font-size: 1.1rem;
  color: #333;
  font-weight: 500;
}

.audio-player {
  width: 100%;
  margin-bottom: 1rem;
}

.clear-btn {
  background: #ef4444;
  color: white;
  border: none;
  padding: 0.5rem 1rem;
  border-radius: 6px;
  font-size: 0.9rem;
  cursor: pointer;
  transition: background-color 0.2s;
  align-self: flex-start;
}

.clear-btn:hover {
  background: #dc2626;
}

.placeholder {
  height: 100%;
  display: flex;
  align-items: center;
  justify-content: center;
  color: #9ca3af;
}

.placeholder-text {
  font-size: 1.1rem;
}
</style>
