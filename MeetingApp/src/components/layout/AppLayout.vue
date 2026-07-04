<template>
  <div class="app-layout" :class="{ 'sidebar-collapsed': appStore.sidebarCollapsed }">
    <TitleBar v-if="isTauri()" />

    <div class="app-body">
      <aside
        class="sidebar"
        :style="{ width: appStore.sidebarCollapsed ? '0px' : `${appStore.sidebarWidth}px` }"
      >
        <SideBar />
      </aside>

      <div
        v-show="!appStore.sidebarCollapsed"
        class="resize-handle"
        @mousedown="onResizeStart"
      />

      <main class="main-content">
        <StatusBar @toggle-sidebar="appStore.sidebarCollapsed = !appStore.sidebarCollapsed" />
        <div class="content-area">
          <router-view />
        </div>
      </main>
    </div>

    <FloatingToolbar />
  </div>
</template>

<script setup lang="ts">
import { computed, onMounted } from 'vue'
import { isTauri, platform } from '@/platform'
import { useAppStore } from '@/stores/app'
import { useChatStore } from '@/stores/chat'
import TitleBar from './TitleBar.vue'
import SideBar from './SideBar.vue'
import StatusBar from './StatusBar.vue'
import FloatingToolbar from '@/components/common/FloatingToolbar.vue'
import { useConnection } from '@/composables/useConnection'
import { useWindowState } from '@/composables/useWindowState'
import { useGlobalShortcut } from '@/composables/useGlobalShortcut'

const appStore = useAppStore()
const chatStore = useChatStore()

// ====== 应用工厂：AppID 锁定 ======
// 构建期由 AutoPublishJob 注入 .env.production 的 VITE_LOCKED_APP_ID。
// 命中则此独立应用只服务单一 App：启动自动选中、隐藏 App 选择器、新建会话直接复用。
// 留空(本地开发)则保持主客户端原行为——拉取全部 App 并弹窗让用户选。
const LOCKED_APP_ID = (import.meta.env.VITE_LOCKED_APP_ID || '').trim()
const isAppLocked = computed(() => LOCKED_APP_ID.length > 0)

let resizing = false

function onResizeStart(e: MouseEvent) {
  e.preventDefault()
  resizing = true
  const startX = e.clientX
  const startWidth = appStore.sidebarWidth
  document.body.style.cursor = 'col-resize'
  document.body.style.userSelect = 'none'

  const onMove = (ev: MouseEvent) => {
    if (!resizing) return
    const delta = ev.clientX - startX
    appStore.setSidebarWidth(startWidth + delta)
  }
  const onUp = () => {
    resizing = false
    document.body.style.cursor = ''
    document.body.style.userSelect = ''
    document.removeEventListener('mousemove', onMove)
    document.removeEventListener('mouseup', onUp)
  }
  document.addEventListener('mousemove', onMove)
  document.addEventListener('mouseup', onUp)
}

useConnection()
useWindowState()
useGlobalShortcut()

onMounted(async () => {
  chatStore.fetchSessions()
  await chatStore.fetchApps()

  if (isAppLocked.value) {
    // 锁定模式：直接选中目标 App，不弹选择器
    const target = chatStore.apps.find(a => a.AppID === LOCKED_APP_ID)
    if (target) {
      chatStore.resetToApp(LOCKED_APP_ID)
    } else {
      console.warn(`[AppFactory] 锁定的 AppID=${LOCKED_APP_ID} 不在可用应用列表中`)
    }
  }

  // 注册通知点击回调，点击通知时切换到对应会话
  platform.notification.onNotificationClick?.((sessionId: string) => {
    if (sessionId) {
      chatStore.selectSession(sessionId)
    }
  })
})
</script>

<style lang="scss" scoped>
.app-layout {
  display: flex;
  flex-direction: column;
  height: 100vh;
  overflow: hidden;
}

.app-body {
  display: flex;
  flex: 1;
  overflow: hidden;
}

.sidebar {
  flex-shrink: 0;
  overflow: hidden;
  border-right: 1px solid var(--border-color, #e4e7ed);
  background: var(--bg-sidebar, #f8f9fa);

  &:not([style*="width: 0px"]) {
    transition: none;
  }
}

.resize-handle {
  width: 4px;
  cursor: col-resize;
  flex-shrink: 0;
  background: transparent;
  transition: background 0.2s;
  position: relative;
  z-index: 10;

  &:hover {
    background: var(--el-color-primary-light-7);
  }
}

.main-content {
  flex: 1;
  display: flex;
  flex-direction: column;
  overflow: hidden;
}

.content-area {
  flex: 1;
  overflow: hidden;
}

.app-list {
  display: flex;
  flex-direction: column;
  gap: 8px;
}

.app-item {
  padding: 12px;
  border: 1px solid var(--border-color, #e4e7ed);
  border-radius: 8px;
  cursor: pointer;
  transition: all 0.2s;

  &:hover {
    border-color: var(--el-color-primary);
    background: var(--el-color-primary-light-9);
  }
}

.app-name {
  font-weight: 500;
  display: block;
}

.app-desc {
  font-size: 12px;
  color: var(--text-secondary, #909399);
  margin-top: 4px;
  display: block;
}
</style>
