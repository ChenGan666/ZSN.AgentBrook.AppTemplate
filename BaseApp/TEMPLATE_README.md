# BaseApp 通用基座模板

> 这是「应用工厂」的通用基座模板，从 `ZSN.AgentBrook.Client/client-app` 派生。
> 用于把平台里任意 App 一键导出成独立品牌应用。
>
> 模板位置：`AppTemplate/BaseApp/`
> 来源基线：`ZSN.AgentBrook.Client/client-app/`（复制时排除了 node_modules/dist/target/gen）

## 与主 client-app 的差异

本模板**只做了三处定制**，业务代码与主 client-app 完全一致：

1. **占位符化**：配置文件里的品牌信息/连接信息改成 `__XXX__` 占位符，构建时由 `AutoPublishJob` 的 `AppCustomizer` 替换。
2. **AppID 锁定**：新增 `VITE_LOCKED_APP_ID` 支持，命中后启动自动选中单一 App、隐藏 App 切换 UI。
3. **清理**：移除 node_modules / dist / target / package-lock.json / 临时文件。

## 占位符清单（AppCustomizer 会替换这些）

| 占位符 | 出现位置 | 替换为 |
|--------|----------|--------|
| `__APP_PRODUCT_NAME__` | tauri.conf.json, index.html, lib.rs, TitleBar.vue | PublishConfig.brand.productName |
| `__APP_IDENTIFIER__` | tauri.conf.json | PublishConfig.brand.identifier |
| `__APP_VERSION__` | tauri.conf.json | PublishConfig.brand.version |
| `__APP_TITLE__` | .env.production, .env.development, TitleBar.vue | PublishConfig.brand.appTitle |
| `__API_BASE_URL__` | .env.production | PublishConfig.connection.apiBaseUrl |
| `__APP_ID__` | .env.production, .env.development | PublishConfig.connection.appId |
| `__APP_SECRET__` | .env.production, .env.development | PublishConfig.connection.appSecret |
| `__LOCKED_APP_ID__` | .env.production | PublishConfig.lockApp.appId |

> `AppCustomizer.PlaceholderTargets` 已硬编码这 6 个文件路径。若移动文件需同步更新。

## AppID 锁定逻辑（VITE_LOCKED_APP_ID）

两个改动点（搜索 `isAppLocked`）：

1. **`src/components/layout/AppLayout.vue`**：
   - `onMounted`：fetchApps 后若锁定，直接 `resetToApp(LOCKED_APP_ID)`，不弹 App 选择器。
   - `handleNewChat`：锁定时直接复用锁定 App，不弹选择器。

2. **`src/components/chat/ChatInput.vue`**：
   - template：`v-if="!isAppLocked"` 包住 App 切换按钮和对话框。

**开发环境**（`.env.development`）默认 `VITE_LOCKED_APP_ID=`（空），保持主客户端原行为——拉全部 App 让用户选，便于本地联调。

## 本地验证（模板开发者）

```bash
cd AppTemplate/BaseApp
npm install                          # 首次安装依赖

# 1. 验证占位符原样可见（开发模式，未替换）
npm run tauri:dev                    # 应看到标题/品牌为 __APP_TITLE__ 等

# 2. 验证锁定逻辑（手动模拟构建期注入）
#    临时编辑 .env.production，把占位符换成真实值：
#      VITE_LOCKED_APP_ID=某个已发布的AppID
#      VITE_API_BASE_URL=http://你的后端:5003/api
#      VITE_APP_ID/VITE_APP_SECRET=平台App凭据
npm run tauri:build                  # 产物在 src-tauri/target/release/bundle/
```

## 结构

```
BaseApp/
├── .env.production          ← 构建期被 AppCustomizer 替换的占位符
├── .env.development         ← 开发环境(留空锁定,本地可改)
├── index.html               ← title 占位符
├── package.json
├── vite.config.ts
├── tsconfig*.json
├── src/                     ← Vue3 前端(与主 client-app 一致 + 锁定逻辑)
│   ├── components/layout/AppLayout.vue   ← 锁定逻辑(启动选中/新建复用)
│   ├── components/layout/TitleBar.vue    ← 占位符
│   └── components/chat/ChatInput.vue     ← 锁定逻辑(隐藏切换入口)
├── src-tauri/               ← Tauri 桌面壳(占位符化的 tauri.conf.json + lib.rs)
│   ├── tauri.conf.json
│   ├── src/lib.rs
│   ├── Cargo.toml
│   ├── capabilities/
│   └── icons/               ← 图标(AppCustomizer 可按 PublishConfig.icons 覆盖)
└── TEMPLATE_README.md       ← 本文件
```

## 与会议助手模板的关系

`AppTemplate/MeetingApp`（待建）将从本模板再派生：
- 继承占位符 + 锁定逻辑
- 叠加会议 UI（MeetingTranscribeView）、实时转写、音频采集等会议能力

## 已知限制（二期处理）

- **品牌文案**：locales（zh-CN.ts/en-US.ts）里的"AgentBrook"等品牌词未占位符化（首版聚焦配置文件换皮）。如需全文案定制，扩展 AppCustomizer 或用 `branding.i18nOverrides`。
- **Agent 模式**：AgentView/useAgentOrchestrator 调用 App 作为工具的路径未做锁定，首版仅 Chat 模式锁定。
- **图标/logo**：默认沿用主 client-app 的图标；`PublishConfig.icons`/`branding` 提供路径时 AppCustomizer 会覆盖。
