<template>
  <div class="sidebar-container">
    <div class="sidebar-header">
      <!-- 会议助手：会议为唯一主入口 -->
      <el-button type="primary" class="meeting-btn" @click="goMeeting">
        <el-icon><Microphone /></el-icon>
        <span>开始会议</span>
      </el-button>
    </div>

    <!-- 导航列表(仅会议/设置) -->
    <nav class="sidebar-nav">
      <div
        class="nav-item"
        :class="{ active: route.path === '/meeting' }"
        @click="goMeeting"
      >
        <el-icon><Microphone /></el-icon>
        <span>会议</span>
      </div>
      <div
        class="nav-item"
        :class="{ active: route.path === '/settings' }"
        @click="toggleSettings"
      >
        <el-icon><Setting /></el-icon>
        <span>设置</span>
      </div>
    </nav>

    <div class="sidebar-footer">
      <div class="user-info" v-if="userStore.userInfo" @click="toggleSettings">
        <el-avatar :size="32" :src="userStore.userInfo.avatar">
          {{ userStore.userInfo.name?.charAt(0) || '?' }}
        </el-avatar>
        <span class="user-name">{{ userStore.userInfo.name }}</span>
      </div>
      <el-button text @click="toggleSettings">
        <el-icon><Setting /></el-icon>
      </el-button>
    </div>
  </div>
</template>

<script setup lang="ts">
import { Setting, Microphone } from '@element-plus/icons-vue'
import { useRouter, useRoute } from 'vue-router'
import { useUserStore } from '@/stores/user'

const router = useRouter()
const route = useRoute()
const userStore = useUserStore()

/** 会议助手主入口：跳转会议页。 */
function goMeeting() {
  router.push('/meeting')
}

function toggleSettings() {
  if (route.path === '/settings') {
    router.push('/meeting')
  } else {
    router.push('/settings')
  }
}
</script>

<style lang="scss" scoped>
.sidebar-container {
  display: flex;
  flex-direction: column;
  height: 100%;
  padding: 12px;
}

.sidebar-header {
  margin-bottom: 12px;
}

/* 会议助手主入口按钮 */
.meeting-btn {
  width: 100%;
  height: 40px;
  font-size: 14px;
  font-weight: 500;
}

/* 导航列表 */
.sidebar-nav {
  flex: 1;
  display: flex;
  flex-direction: column;
  gap: 4px;
  overflow-y: auto;
}

.nav-item {
  display: flex;
  align-items: center;
  gap: 8px;
  padding: 9px 12px;
  border-radius: 8px;
  cursor: pointer;
  font-size: 14px;
  color: var(--text-secondary, #909399);
  transition: all 0.2s;

  &:hover {
    background: var(--el-fill-color-light, #f5f7fa);
    color: var(--text-primary, #303133);
  }

  &.active {
    background: var(--el-color-primary-light-9, #ecf5ff);
    color: var(--el-color-primary, #409eff);
    font-weight: 500;
  }

  .el-icon {
    font-size: 16px;
  }
}

.sidebar-footer {
  display: flex;
  align-items: center;
  justify-content: space-between;
  padding-top: 12px;
  border-top: 1px solid var(--border-color, #e4e7ed);
}

.user-info {
  display: flex;
  align-items: center;
  gap: 8px;
  cursor: pointer;
  border-radius: 6px;
  padding: 4px;
  transition: background 0.2s;

  &:hover {
    background: var(--el-fill-color-light);
  }
}

.user-name {
  font-size: 14px;
  color: var(--text-primary, #303133);
}
</style>
