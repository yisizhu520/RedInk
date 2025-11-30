# WARP.md

This file provides guidance to WARP (warp.dev) when working with code in this repository.

## 项目概述

RedInk（红墨）是一个小红书AI图文生成器，输入一句话和可选的参考图片，自动生成完整的小红书图文内容（封面+多页内容）。

## 常用命令

### 后端开发
```bash
# 安装依赖
uv sync

# 启动后端（端口 12398）
uv run python -m backend.app

# 运行测试
uv run pytest tests/
```

### 前端开发
```bash
cd frontend

# 安装依赖
pnpm install

# 启动开发服务器（端口 5173）
pnpm dev

# 构建生产版本
pnpm build
```

### Docker 部署
```bash
docker run -d -p 12398:12398 -v ./history:/app/history -v ./output:/app/output histonemax/redink:latest
```

## 技术架构

### 后端 (Python/Flask)
- **入口**: `backend/app.py` - Flask 应用，自动检测前端构建产物决定运行模式
- **路由层**: `backend/routes/` - 按功能模块化拆分
  - `outline_routes.py`: 大纲生成 API
  - `image_routes.py`: 图片生成/获取 API（SSE 流式响应）
  - `history_routes.py`: 历史记录 CRUD
  - `config_routes.py`: 配置管理
- **服务层**: `backend/services/`
  - `outline.py`: 调用文本 AI 生成大纲
  - `image.py`: 图片生成核心逻辑，支持并发/顺序两种模式
  - `history.py`: 历史记录持久化
- **生成器**: `backend/generators/` - 图片生成器工厂模式
  - `google_genai.py`: Google Gemini 图片生成
  - `openai_compatible.py`: OpenAI 兼容接口
  - `image_api.py`: 通用图片 API
- **配置**: `text_providers.yaml` 和 `image_providers.yaml`（从 `.example` 复制）

### 前端 (Vue 3/TypeScript/Vite)
- **状态管理**: `frontend/src/stores/generator.ts` - Pinia store，管理生成流程状态
- **API 层**: `frontend/src/api/index.ts` - 所有后端 API 调用封装
- **视图**: `frontend/src/views/`
  - `HomeView.vue`: 首页，输入主题
  - `OutlineView.vue`: 大纲编辑
  - `GenerateView.vue`: 图片生成进度
  - `ResultView.vue`: 结果展示
  - `HistoryView.vue`: 历史记录
  - `SettingsView.vue`: 系统设置

### 核心数据流
1. 用户输入主题 → `POST /api/outline` → AI 生成大纲（支持参考图片）
2. 用户确认大纲 → `POST /api/generate` → SSE 流式返回生成进度
3. 图片生成：先生成封面，再并发/顺序生成其他页面（使用封面作为风格参考）
4. 生成结果保存到 `history/{task_id}/` 目录

## 关键配置

- **高并发模式**: `image_providers.yaml` 中 `high_concurrency: true` 启用并行生成（最多 15 张）
- **短 Prompt 模式**: `short_prompt: true` 使用精简提示词模板
- **Prompt 模板**: `backend/prompts/` 目录下的 `.txt` 文件

## 注意事项

- 后端检测 `frontend/dist` 存在时自动托管静态文件（Docker 模式）
- 图片保存时同时生成缩略图（`thumb_` 前缀）
- API Key 等敏感配置不要提交到代码库
