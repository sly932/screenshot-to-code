# 项目知识整理

## 1. 项目定位
- **核心目标**：将截图、设计稿、Figma 导出乃至屏幕录制转换为可运行的前端代码。
- **支持栈**：HTML+Tailwind、HTML+CSS、React+Tailwind、Vue+Tailwind、Bootstrap、Ionic+Tailwind、SVG。
- **主要模型**：Claude 3.7 Sonnet、GPT-4o；图像生成可选 DALL·E 3 或 Flux Schnell。

## 2. 系统架构概览
- **前端**：基于 React + Vite，使用 Zustand 管理项目/应用状态，Radix UI 与 Tailwind 负责界面组件与样式。
- **后端**：FastAPI 应用，提供 WebSocket 代码生成、截图服务、评测等接口；采用分层中间件流水线处理每次生成请求。
- **通信机制**：前端通过 WebSocket 发送生成指令，后端流式返回 `chunk`、`status`、`setCode`、`variantComplete`、`variantError` 等消息，并支持多变体并行。

## 3. 核心流程详解
1. **参数提取**：后端解析前端传入的生成栈、输入模式、API Key、生成类型（create/update）、历史上下文等。
2. **模型选择**：根据可用的 OpenAI/Anthropic/Gemini Key 以及输入模式，循环分配到 `NUM_VARIANTS` 个变体模型。
3. **提示词构建**：按栈和模式拼装 prompt，必要时附带历史提交或导入代码信息。
4. **并行生成**：为每个变体创建异步任务，流式回传代码片段与状态；可选执行图像替换（占位图 → AI 生成图）。
5. **前端展示**：Zustand store 记录 commit/variant 状态，`Variants` 组件提供切换与状态提示，`PreviewPane` 支持桌面/移动/源码视图与下载。

## 4. 主要代码入口
- `backend/main.py`：注册各类路由、设置 CORS。
- `backend/routes/generate_code.py`：WebSocket 管线、模型分配、并行生成与错误处理。
- `backend/routes/screenshot.py`：截图代理服务，接入 ScreenshotOne。
- `backend/image_generation/core.py`：占位图识别与 DALL·E / Flux 生成逻辑。
- `frontend/src/App.tsx`：前端主控组件，负责触发生成、管理历史、协调 UI。
- `frontend/src/generateCode.ts`：统一管理 WebSocket 生命周期及回调。
- `frontend/src/store/project-store.ts`：定义提交树与变体状态。
- `design-docs/variant-system.md`、`plan.md`：描述现有多变体系统与非阻塞化改造计划。

## 5. 本地开发与运行
- **后端**：使用 Poetry；常用命令
  ```bash
  cd backend
  poetry install
  poetry run uvicorn main:app --reload --port 7001
  poetry run pyright   # 类型检查
  poetry run pytest    # 单元测试
  ```
- **前端**：使用 Yarn + Vite；
  ```bash
  cd frontend
  yarn
  yarn dev
  yarn test
  ```
- **Docker**：根目录提供 `docker-compose.yml`，可快速启动端到端服务（开发时不推荐因为热重载受限）。

## 6. 功能扩展与路线
- **多变体改造**：`plan.md` 拟定的“非阻塞流式”方案，让首个变体完成即允许交互，支持中途取消其余任务。
- **视频转代码**：`backend/video_to_app.py` 使用 Claude 原生流式接口，将屏幕录制拆帧并生成交互原型。
- **截图集成**：前端可配置 ScreenshotOne API Key，实现网页 URL → 截图 → 设计输入流程。

## 7. 建议的初始步骤
1. 准备 `.env` 并配置 OpenAI/Anthropic（及可选 Replicate）API Key。
2. 启动后端与前端，验证图片/视频/文本三种模式的生成链路。
3. 按 `plan.md` 的阶段计划评估是否优先落地非阻塞多变体重构，并补充相应测试。
