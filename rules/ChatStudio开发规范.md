## 🤖 Assistant

# ChatStudio 开发规范

## 1. 技术栈

- **框架**：React + TypeScript + WXT
- **Chrome 插件**：Manifest V3
- **UI**：Tailwind CSS + shadcn/ui + Lucide Icons
- **状态管理**：Zustand
- **本地数据库**：IndexedDB + Dexie
- **轻量配置存储**：`chrome.storage.local`
- **表单与校验**：React Hook Form + Zod
- **Markdown**：`react-markdown` + `remark-gfm` + `rehype-highlight`

## 2. 项目结构

```text
src/
  app/              # 应用入口、路由、全局 Provider
  components/       # 通用 UI 组件
  features/
    chat/           # 对话、消息、流式渲染
    conversations/  # 会话列表、搜索、归档
    settings/       # 模型、密钥、偏好设置
  services/
    ai/              # 模型适配层
    search/          # 网络搜索适配层
    storage/         # IndexedDB、chrome.storage 封装
  stores/            # Zustand 状态
  types/             # 公共类型
  utils/             # 纯工具函数
```

按功能拆分模块。UI 组件不得直接调用模型 API 或 IndexedDB，统一经 `services` 层访问。

## 3. 插件架构

- 使用 Manifest V3。
- `sidepanel` 或 `popup` 提供主要对话 UI。
- `service worker` 处理后台请求、流式通信、菜单和生命周期事件。
- 内容脚本仅在需要读取当前页面内容、选中文本或注入页面能力时启用。
- 仅配置实际需要的 `permissions` 和 `host_permissions`。

```json
{
  "manifest_version": 3,
  "permissions": ["storage", "sidePanel"],
  "host_permissions": ["https://your-api.example.com/*"],
  "background": {
    "service_worker": "src/background.ts",
    "type": "module"
  }
}
```

## 4. 数据存储

- `chrome.storage.local`：偏好设置、当前会话 ID、模型配置、轻量缓存。
- IndexedDB：会话、消息、摘要、来源、附件元数据。
- 使用 Dexie 管理版本、索引和数据迁移。
- 每条消息独立存储，禁止整体反复写入全部 `messages`。
- 附件保存 URL、Blob 或引用，不将图片转 Base64 放入消息正文。

```ts
type Conversation = {
  id: string;
  title: string;
  createdAt: number;
  updatedAt: number;
  messageCount: number;
  summary?: string;
};

type Message = {
  id: string;
  conversationId: string;
  role: "system" | "user" | "assistant" | "tool";
  content: string;
  createdAt: number;
  model?: string;
};
```

`messages` 建立 `conversationId` 和 `[conversationId+createdAt]` 索引。

## 5. AI 模型接入

模型调用必须经过统一适配层，页面不直接依赖 OpenAI、deepseek、智谱 等厂商 SDK。

```ts
type ChatOptions = {
  model: string;
  reasoningLevel?: "auto" | "fast" | "balanced" | "deep";
  webSearch?: boolean;
  stream?: boolean;
};

interface AIProvider {
  chat(messages: Message[], options: ChatOptions): AsyncIterable<AIEvent>;
  getCapabilities(model: string): ModelCapabilities;
}
```

适配层负责：

- 鉴权、请求、流式响应、取消请求。
- 模型能力检测。
- 将统一的推理强度映射到厂商参数。
- 统一错误、限流和重试策略。
- 规范化工具调用、引用和响应格式。

不展示或承诺展示完整思维链。界面提供“推理强度”和可选“步骤摘要”。

## 6. 上下文与搜索

请求上下文顺序：

```text
系统提示词 + 会话摘要 + 最近消息 + 当前问题 + 搜索结果
```

- 按 token 预算截断上下文。
- 较早消息生成摘要，但保留原始历史。
- 首次只加载最近约 50 条消息，历史消息滚动分页加载。
- 联网搜索优先使用模型原生能力。
- 搜索来源以标题、摘要、URL 结构化保存，并在回答中展示引用。

## 7. 安全要求

- 禁止在插件包内写入共享模型或搜索 API Key。
- 产品统一密钥必须使用后端代理。
- 用户自填 Key 不写日志、不上传到无关服务。
- 所有模型与网页内容均视为不可信数据，禁止直接插入 HTML。
- 删除、清空、导入和导出操作必须二次确认。

## 8. 基础功能

- 新建、重命名、归档、删除、搜索会话
- 模型选择、推理强度、联网搜索开关
- 自定义模型的API Key和API 地址
- 支持模型的请求响应检测，以判断是否连接成功
- 流式输出、停止生成、重新生成
- Markdown、代码块、复制、引用来源
- 会话导出与本地数据清理
- 清晰处理网络失败、鉴权失败、额度不足和限流错误


## 9. 注意事项

- 流式渲染：如何处理 AI 的流式输出 (SSE)，保证 UI 不卡顿、不闪烁。
- 自动滚动：消息列表如何智能滚动，既能在生成时跟随最新内容，又不妨碍用户向上查看历史记录。
- Markdown 渲染：如何安全地渲染 AI 回复中的 Markdown 格式（标题、列表、代码块等）。
- 上下文管理：用户可能几个月甚至更久都不清数据，一直用同个会话来对话，避免每次和ai对话的上下文一直累积，导致输入token巨大
