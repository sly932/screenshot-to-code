# 核心流程解构文档

本文按端到端时序梳理“截图 → 前端代码”生成链路，并注明关键实现位置与提示词内容，便于快速追踪或扩展。

## 1. 前端如何发起生成
- 入口组件 `frontend/src/App.tsx:182-320`：
  - `doGenerateCode` 将用户输入与持久化设置合并，创建新的 commit 记录，并注册 WebSocket 回调。
  - `doCreate` / `doCreateFromText` / `doUpdate` 分别处理初次生成（图片、视频、文本）与迭代更新；更新模式会调用 `extractHistory` 构建 prompt 对话并附带最近一次 HTML `frontend/src/App.tsx:282-320`。
  - 提交管理由 `createCommit` 和 Zustand store 协同实现：`frontend/src/components/commits/utils.ts:1-27`、`frontend/src/store/project-store.ts:47-200`。
- 前端状态更新：
  - 变体数量、流式 token、状态栏输出、完成/失败等通过回调实时写入 store，并驱动 UI（例如 `Variants`、`PreviewPane`）。

## 2. WebSocket 客户端通道
- `frontend/src/generateCode.ts:1-94`：
  - 连接地址 `WS_BACKEND_URL`，`open` 事件发送 `FullGenerationSettings`（由生成参数与设置组合，定义见 `frontend/src/types.ts:21-38`）。
  - 监听消息类型 `chunk/status/setCode/error/variantComplete/variantError/variantCount`（对应后端枚举），逐类调用回调。
  - 处理关闭码（用户取消、服务端异常、正常完成），并在浏览器 Toast 中反馈。

## 3. FastAPI 入口与管线骨架
- 后端应用在 `backend/main.py:1-26` 注册 `/generate-code` WebSocket 路由。
- `backend/routes/generate_code.py:68-1120` 构建分层中间件管线（Pipeline）：
  1. **WebSocketSetupMiddleware** 接入并最终负责关闭连接 `backend/routes/generate_code.py:781-796`。
  2. **ParameterExtractionMiddleware** 接收 JSON 参数，校验 stack、inputMode、API Key、历史等 `backend/routes/generate_code.py:808-848`。
  3. **StatusBroadcastMiddleware** 先告知前端 `NUM_VARIANTS` 并下发初始状态 `backend/routes/generate_code.py:822-838`。
  4. **PromptCreationMiddleware** 调用 `PromptCreationStage` 组装 prompt `backend/routes/generate_code.py:840-855`。
  5. **CodeGenerationMiddleware** 选择模型、并行生成、处理视频或 mock 模式 `backend/routes/generate_code.py:857-942`。
  6. **PostProcessingMiddleware** 在所有变体完成后写日志等后置操作 `backend/routes/generate_code.py:944-976`。

## 4. 模型与并行变体
- `NUM_VARIANTS` 默认 4，可通过 `backend/config.py:6` 调整；变体计数会回传给前端以动态扩容界面。
- 模型分配策略 `ModelSelectionStage`：根据可用的 OpenAI/Anthropic/Gemini Key、生成类型与输入模式循环取模 `backend/routes/generate_code.py:314-401`。
- `ParallelGenerationStage` 负责：
  - 为每个模型创建异步任务并流式回调 `chunk` 消息 `backend/routes/generate_code.py:537-775`。
  - 针对 OpenAI 调用 `_stream_openai_with_error_handling` 附带认证/配额异常捕获 `backend/routes/generate_code.py:642-697`。
  - 变体完成后发送 `setCode`、`variantComplete`，并在需要时触发图像替换 `backend/routes/generate_code.py:731-763`。
- 图像生成（可选）：
  - `generate_images` 会扫描 HTML 中的占位图，将 alt 文本作为 prompt 调用 DALL·E 或 Flux `backend/image_generation/core.py:96-158`。
  - 更新后的 URL 会被写回 HTML，再回传给前端。

## 5. Prompt 组装机制
- 总入口 `backend/prompts/__init__.py:22-66`：根据 `inputMode`、`generationType`、是否导入代码决定调用以下方法，并在 `update` 场景中交替添加历史消息，保持 user/assistant 角色交错。
- **截图 / 图像模式**：
  - System prompt 来源：`backend/prompts/screenshot_system_prompts.py:1-198`（Tailwind、React、Vue、Bootstrap、Ionic、SVG 各自一套，强调“完全还原 UI、禁止省略注释、使用 placehold.co”）。
  - User prompt：通用文案 “Generate code for a web page that looks exactly like this.” 定义于 `backend/prompts/__init__.py:13-19`，对 SVG 使用专门文本。
- **文本模式**：
  - System prompt 来自 `backend/prompts/text_prompts.py:1-87`，添加现代设计、库引用等指导。
  - User 层拼接成 “Generate UI for <用户描述>” `backend/prompts/__init__.py:167-179`。
- **导入现有代码**：
  - 直接把原 HTML 或 SVG 嵌入 system prompt，文件 `backend/prompts/imported_code_prompts.py:1-144`，强调在原基础上继续完整输出。
- **视频模式**：
  - 使用 `assemble_claude_prompt_video` 加载序列帧，并配合 `VIDEO_PROMPT` 指导模型先写 `<thinking>` 再输出 `<html>` `backend/prompts/claude_prompts.py:9-41`。

## 6. 关键消息与前端呈现
- 消息枚举 `MessageType` 定义在 `backend/routes/generate_code.py:49-58`，前端对应枚举体 `WebSocketResponse` `frontend/src/generateCode.ts:14-25`。
- 状态更新流向：
  - `chunk` → `appendCommitCode` 累积代码 `frontend/src/App.tsx:220-224`。
  - `setCode` → 将完整版本写入某个变体 `frontend/src/App.tsx:224-226`。
  - `variantComplete`/`variantError` → 调整 Zustand 中的 `VariantStatus` `frontend/src/App.tsx:228-235`，并由 `Variants` 组件展示加载/成功/失败指示 `frontend/src/components/variants/Variants.tsx:5-142`。
  - `status` 文本会写入执行控制台供调试 `frontend/src/App.tsx:227-228`。
  - `variantCount` 调整前端数组尺寸，适配任意并发数 `frontend/src/App.tsx:236-238`。

## 7. 扩展与排错提示
- Mock 模式：设置环境变量 `MOCK=true` 时，`CodeGenerationMiddleware` 会改用本地录制的响应，便于调试 `backend/routes/generate_code.py:859-869`。
- 错误处理：
  - OpenAI 认证/模型/限额异常在后端被捕获并转化为 `variantError` 提示 `backend/routes/generate_code.py:658-697`。
  - 若所有变体失败，管线会通过 `throw_error` 关闭 WebSocket `backend/routes/generate_code.py:917-936`。
- 日志：成功的首个变体会写入 `fs_logging` 目录，便于复盘 `backend/routes/generate_code.py:518-533`。

以上内容涵盖了从前端事件到提示词组装、再到并行生成与结果回传的完整链路，可作为后续开发或文档化的基础材料。
