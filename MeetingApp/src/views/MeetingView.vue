<template>
  <div class="meeting-view">
    <!-- 顶部模式切换 -->
    <div class="mode-tabs">
      <div
        class="mode-tab"
        :class="{ active: mode === 'live' }"
        @click="mode = 'live'"
      >
        <el-icon><Microphone /></el-icon>
        <span>实时录制</span>
      </div>
      <div
        class="mode-tab"
        :class="{ active: mode === 'upload' }"
        @click="mode = 'upload'"
      >
        <el-icon><Upload /></el-icon>
        <span>上传音频</span>
      </div>
    </div>

    <!-- 实时录制模式 -->
    <div v-if="mode === 'live'" class="live-section">
      <div class="live-controls">
        <el-button
          :type="isRecording ? 'danger' : 'primary'"
          size="large"
          circle
          @click="toggleRecording"
        >
          <el-icon :size="28">
            <Microphone v-if="!isRecording" />
            <VideoPause v-else />
          </el-icon>
        </el-button>
        <div class="live-status">
          <span v-if="voiceStore.state === 'connecting'" class="status-connecting">
            <span class="pulse-dot" /> 连接识别服务...
          </span>
          <span v-else-if="voiceStore.state === 'recording'" class="status-recording">
            <span class="pulse-dot" /> 录音中 · {{ liveDurationText }}
          </span>
          <span v-else-if="voiceStore.state === 'error'" class="status-error">
            识别失败，请重试
          </span>
          <span v-else>点击开始录制会议</span>
        </div>
        <el-select v-model="voiceStore.language" size="small" class="lang-select">
          <el-option label="中文" value="zh" />
          <el-option label="英文" value="en" />
          <el-option label="中英混合" value="mixed" />
        </el-select>
      </div>

      <!-- 实时转写文本区 -->
      <div class="live-transcript">
        <div v-if="!liveFullText && !voiceStore.interimText" class="empty-tip">
          录音过程中的语音会实时转写到这里
        </div>
        <div class="confirmed-text">{{ liveFullText }}</div>
        <div v-if="voiceStore.interimText" class="interim-text">{{ voiceStore.interimText }}</div>
      </div>

      <!-- 实时模式操作 -->
      <div class="live-actions">
        <el-button
          v-if="isRecording"
          type="warning"
          @click="finishLiveMeeting"
        >
          结束并生成纪要
        </el-button>
        <el-button
          v-if="!isRecording && liveFullText"
          :disabled="generatingMinutes"
          :loading="generatingMinutes"
          type="primary"
          @click="generateMinutes(liveFullText)"
        >
          {{ generatingMinutes ? '正在生成纪要...' : '生成会议纪要' }}
        </el-button>
        <el-button v-if="!isRecording && liveFullText" @click="clearLiveText">清空</el-button>
      </div>
    </div>

    <!-- 上传模式：复用现有离线转写组件 -->
    <div v-else class="upload-section">
      <MeetingTranscribe />
    </div>

    <!-- 生成的会议纪要展示(两种模式共用) -->
    <div v-if="minutes" class="minutes-result">
      <el-divider content-position="left">会议纪要</el-divider>
      <div class="minutes-content" v-html="renderedMinutes"></div>
      <div class="minutes-actions">
        <el-button size="small" @click="copyMinutes">复制</el-button>
        <el-button size="small" @click="downloadMinutes">下载 Markdown</el-button>
      </div>
    </div>
  </div>
</template>

<script setup lang="ts">
import { ref, computed, onUnmounted } from 'vue'
import { Microphone, VideoPause, Upload } from '@element-plus/icons-vue'
import { ElMessage } from 'element-plus'
import { useVoiceStore } from '@/stores/voice'
import { useVoice } from '@/composables/useVoice'
import MeetingTranscribe from '@/components/common/MeetingTranscribe.vue'
import { marked } from 'marked'
import DOMPurify from 'dompurify'
import http from '@/services/http'
import { platform } from '@/platform'

type Mode = 'live' | 'upload'
const mode = ref<Mode>('live')

// ====== 实时录制 ======
const voiceStore = useVoiceStore()
const { startRecording, stopRecording, getFullText } = useVoice()

const isRecording = computed(
  () => voiceStore.state === 'recording' || voiceStore.state === 'connecting',
)
const liveFullText = computed(() => getFullText())

// 录音时长计时
const recordStartTs = ref(0)
const nowTs = ref(Date.now())
let durationTimer: ReturnType<typeof setInterval> | null = null
const liveDurationText = computed(() => {
  if (!recordStartTs.value) return '00:00'
  const sec = Math.floor((nowTs.value - recordStartTs.value) / 1000)
  const m = String(Math.floor(sec / 60)).padStart(2, '0')
  const s = String(sec % 60).padStart(2, '0')
  return `${m}:${s}`
})

async function toggleRecording() {
  if (isRecording.value) {
    await stopRecording()
    if (durationTimer) { clearInterval(durationTimer); durationTimer = null }
  } else {
    clearLiveText()
    await startRecording()
    recordStartTs.value = Date.now()
    nowTs.value = Date.now()
    durationTimer = setInterval(() => { nowTs.value = Date.now() }, 1000)
  }
}

// 结束实时会议：停止录音并立即生成纪要
async function finishLiveMeeting() {
  await stopRecording()
  if (durationTimer) { clearInterval(durationTimer); durationTimer = null }
  if (!liveFullText.value) {
    ElMessage.warning('没有识别到语音内容')
    return
  }
  await generateMinutes(liveFullText.value)
}

function clearLiveText() {
  voiceStore.clearText()
  recordStartTs.value = 0
}

// ====== 会议纪要生成(提交给平台会议 App 的工作流) ======
const minutes = ref<string>('')
const generatingMinutes = ref(false)
const renderedMinutes = computed(() =>
  DOMPurify.sanitize(marked.parse(minutes.value || '', { async: false }) as string),
)

/**
 * 把转写文本提交给锁定的会议 App(通过 /Chat/completions)，由平台工作流生成结构化纪要。
 * 平台会议 App 的工作流：Start → LargeModel(会议纪要prompt) → End。
 *
 * 复用 http 实例(自带 AES 加密 + bearer/memberbearer 鉴权)，走 stream:false 非流式聚合返回，
 * 避免裸 fetch 绕过平台的请求加密机制。
 */
async function generateMinutes(transcriptText: string) {
  if (!transcriptText.trim()) {
    ElMessage.warning('转写内容为空')
    return
  }
  generatingMinutes.value = true
  minutes.value = ''
  try {
    const lockedAppId = (import.meta.env.VITE_LOCKED_APP_ID || '').trim()
    const { data } = await http.post('/Chat/completions', {
      appid: lockedAppId,
      messages: [{ role: 'user', content: transcriptText }],
      stream: false,
    })
    // 平台非流式返回结构：聚合的文本在 data.Data 内
    const text = extractMinutesText(data)
    if (!text) throw new Error('未返回纪要内容')
    minutes.value = text
    ElMessage.success('纪要已生成')
  } catch (e: any) {
    ElMessage.error(e.message || '纪要生成失败')
  } finally {
    generatingMinutes.value = false
  }
}

// 兼容多种可能的返回结构
function extractMinutesText(data: any): string {
  if (!data) return ''
  if (typeof data === 'string') return data
  const d = data.Data ?? data.data
  if (typeof d === 'string') return d
  if (d?.content) return d.content
  if (d?.text) return d.text
  if (d?.choices?.[0]?.message?.content) return d.choices[0].message.content
  if (Array.isArray(d)) {
    // 可能是消息数组，取 assistant 内容
    const ai = d.find((m: any) => m.Role === 'assistant' || m.role === 'assistant')
    if (ai?.Content) return ai.Content
    if (ai?.content) return ai.content
  }
  return ''
}

function copyMinutes() {
  navigator.clipboard.writeText(minutes.value).then(() => ElMessage.success('已复制'))
}

async function downloadMinutes() {
  const blob = new Blob([minutes.value], { type: 'text/markdown;charset=utf-8' })
  await platform.file.save(blob, `会议纪要_${new Date().toISOString().slice(0, 10)}.md`)
}

onUnmounted(() => {
  if (durationTimer) clearInterval(durationTimer)
  if (isRecording.value) stopRecording()
})
</script>

<style lang="scss" scoped>
.meeting-view {
  height: 100%;
  display: flex;
  flex-direction: column;
  padding: 16px;
  gap: 16px;
  overflow-y: auto;
}

.mode-tabs {
  display: flex;
  gap: 8px;
  flex-shrink: 0;
}

.mode-tab {
  display: flex;
  align-items: center;
  gap: 6px;
  padding: 8px 16px;
  border-radius: 8px;
  cursor: pointer;
  border: 1px solid var(--border-color, #e4e7ed);
  transition: all 0.2s;
  font-size: 14px;

  &:hover {
    border-color: var(--el-color-primary);
  }

  &.active {
    background: var(--el-color-primary);
    color: #fff;
    border-color: var(--el-color-primary);
  }
}

.live-section {
  display: flex;
  flex-direction: column;
  gap: 16px;
}

.live-controls {
  display: flex;
  align-items: center;
  gap: 16px;
  padding: 16px;
  background: var(--el-fill-color-light, #f5f7fa);
  border-radius: 8px;
}

.live-status {
  flex: 1;
  font-size: 14px;
}

.status-connecting { color: var(--el-color-warning); }
.status-recording { color: var(--el-color-danger); display: flex; align-items: center; gap: 6px; }
.status-error { color: var(--el-color-danger); }

.pulse-dot {
  display: inline-block;
  width: 8px;
  height: 8px;
  border-radius: 50%;
  background: currentColor;
  animation: pulse 1.5s infinite;
}

@keyframes pulse {
  0%, 100% { opacity: 1; }
  50% { opacity: 0.3; }
}

.lang-select {
  width: 120px;
}

.live-transcript {
  min-height: 200px;
  max-height: 320px;
  overflow-y: auto;
  padding: 12px;
  border: 1px solid var(--border-color, #e4e7ed);
  border-radius: 8px;
  line-height: 1.8;
  font-size: 14px;
  background: var(--el-bg-color, #fff);
}

.empty-tip {
  color: var(--text-secondary, #909399);
  text-align: center;
  margin-top: 60px;
}

.confirmed-text {
  color: var(--el-text-color-primary, #303133);
}

.interim-text {
  color: var(--el-text-color-secondary, #909399);
  font-style: italic;
}

.live-actions {
  display: flex;
  gap: 8px;
  justify-content: center;
}

.minutes-result {
  padding: 12px;
  border-top: 1px solid var(--border-color, #e4e7ed);
}

.minutes-content {
  padding: 12px;
  background: var(--el-fill-color-light, #f5f7fa);
  border-radius: 8px;
  line-height: 1.8;
  min-height: 60px;

  :deep(h1), :deep(h2), :deep(h3) { margin: 8px 0; }
  :deep(ul), :deep(ol) { padding-left: 20px; }
}

.minutes-actions {
  display: flex;
  gap: 8px;
  justify-content: flex-end;
  margin-top: 8px;
}
</style>
