# MeetingApp 会议助手模板

> 基于 BaseApp 通用基座派生的「会议助手」专用模板。
> 位置：`AppTemplate/MeetingApp/`
> 适用场景：把平台里的「会议助手 App」一键导出成独立的会议记录应用。

## 与 BaseApp 的差异

MeetingApp 在 BaseApp 基础上**只新增/修改了 4 个文件**，业务资产全部复用：

| 文件 | 类型 | 说明 |
|------|------|------|
| `src/views/MeetingView.vue` | **新增** | 会议主页：整合「实时录制」+「上传音频」两个入口，生成会议纪要 |
| `src/router/index.ts` | 修改 | 根路径 `/` 重定向到 `/meeting`（非 `/chat`）；登录后默认进会议页 |
| `src/components/layout/SideBar.vue` | 修改 | 顶部加「开始会议」主入口按钮；设置返回路径改为 `/meeting` |
| `TEMPLATE_README.md` | 新增 | 本文件 |

> BaseApp 已自带的会议资产（直接复用，未改动）：
> - `components/common/MeetingTranscribe.vue`（上传转写组件，MeetingView 复用）
> - `components/common/VoiceRecorder.vue`
> - `composables/useMeeting.ts`（离线转写：文件→分块上传→摘要）
> - `composables/useVoice.ts`（实时 ASR：WebSocket /Chat/FunASR）
> - `stores/voice.ts`、`services/voiceApi.ts`、`types/meeting.d.ts`、`types/voice.d.ts`
> - `src-tauri/src/commands/audio.rs`（cpal 麦克风采集，16kHz/单声道/int16）

继承 BaseApp 的全部能力：占位符换皮、AppID 锁定（`VITE_LOCKED_APP_ID`）。

## 两个会议场景

### 场景一：实时录制（MeetingView 默认 tab）
```
点击麦克风 → useVoice.startRecording()
  → Tauri audio.rs 采集(cpal, 16k mono int16) → emit('audio-pcm-data')
  → useVoice 监听 → WebSocket 推送到 /Chat/FunASR(FunASR 2pass)
  → 返回文本 → voiceStore 累积 → 实时显示
点击「结束并生成纪要」→ 停止录音 → 把全文文本提交给平台会议 App
  → POST /Chat/completions (stream:false) → LargeModel 节点按会议纪要 prompt 输出
  → Markdown 渲染展示，可复制/下载
```

### 场景二：上传音频（MeetingView 切换 tab / 复用 MeetingTranscribe 组件）
```
拖入音频文件 → useMeeting.transcribeFile()
  → platform.audio.convertToWav → 分块 POST /Voice/Transcribe
  → 返回带说话人/时间戳的 segments → POST /Voice/PostProcess 做摘要+待办
  → 展示结构化纪要，可导出 txt/md
```

## 平台后端依赖

MeetingApp 调用的后端端点（须由 `AgentBrook.API` 提供）：

| 端点 | 协议 | 用途 | 场景 |
|------|------|------|------|
| `/Chat/FunASR` | WebSocket | 实时流式语音识别 | 实时录制 |
| `/Voice/Transcribe` | HTTP POST (octet-stream) | 音频分块转写 | 上传音频 |
| `/Voice/PostProcess` | HTTP POST JSON | 摘要 + 待办抽取 | 上传音频 |
| `/Chat/completions` | HTTP POST | 提交转写文本生成会议纪要 | 实时录制→纪要 |

**实时录制→纪要的链路**要求平台侧已编排好「会议助手 App」：
工作流 `Start → LargeModel(会议纪要 prompt) → End`，AppID 锁定到该 App。
会议纪要 prompt 模板见 `doc/APP_FACTORY_DEPLOY.md` 第四节。

## 占位符与锁定

完全继承 BaseApp 的 8 个占位符 + AppID 锁定机制（见 `BaseApp/TEMPLATE_README.md`）。
构建时 `AppCustomizer` 会把 `.env.production` 的 `VITE_LOCKED_APP_ID` 替换为会议助手 AppID，
登录后自动锁定到该 App。

## 本地验证

```bash
cd AppTemplate/MeetingApp
npm install

# 1. 开发态验证（占位符原样可见，会议页为首屏）
npm run tauri:dev
#    登录后应直接进入会议页（非聊天页）
#    左上角应看到「开始会议」按钮
#    实时录制 tab：点麦克风开始（需后端 /Chat/FunASR 可用）
#    上传 tab：拖入音频（需后端 /Voice/Transcribe 可用）

# 2. 模拟构建期注入(验证锁定 + 纪要链路)
#    编辑 .env.production：
#      VITE_LOCKED_APP_ID=<会议助手App的AppID>
#      VITE_API_BASE_URL=<后端地址>
#      VITE_APP_ID/VITE_APP_SECRET=<平台App凭据>
npm run tauri:build
```

## 已知限制（二期处理）

1. **实时转写无说话人分离**：`/Chat/FunASR` 当前只返回纯文本，不返回 `sentence_info.spk`。
   实时模式下的纪要因此没有发言人标注。二期需升级 useVoice 解析 spk 字段（参照服务端
   `SpeakerLabelNormalizer` 逻辑），让实时转写也带说话人标注。
2. **Tauri 端音频格式转换空实现**：`platform.audio.convertToWav` 在 Tauri adapter 里直接返回
   原始字节（未真正转 WAV 头）。上传模式的 mp3/mp4 等非 WAV 文件可能转写失败，需补真实解码。
3. **纪要生成为非流式**：为复用 http 实例的 AES 加密+鉴权，纪要生成走 `stream:false` 同步等待，
   长文本时前端会阻塞直到全部返回。如需流式逐字显示，需把 token 注入裸 fetch 的 SSE。
4. **单设备采集**：Tauri audio.rs 只采集 `default_input_device()`（单麦克风），无法区分会议
   多设备/多说话人分轨。多说话人识别依赖服务端 diarization（仅离线链路 `/Voice/Transcribe` 支持）。
5. **Agent 模式未锁定**：与 BaseApp 一致，仅 Chat 模式锁定 AppID。
