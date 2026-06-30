# 技术架构 · ARCHITECTURE

## 概述

课程工作流启动包是一个**纯前端单文件 HTML 应用**，无构建步骤、无后端依赖、无外部库引用。所有逻辑（状态管理、渲染、文件处理、文本生成）均在浏览器本地执行。

## 设计原则

1. **零依赖**：不引入任何 npm 包、CDN 脚本或外部 CSS。整个应用是一个自包含的 `.html` 文件。
2. **零联网**：不发送任何数据到服务器。文件上传仅在浏览器内存中处理，生成预览后不持久化。
3. **跨 AI 平台**：产出物是纯文本（Prompt 协议包），可粘贴到任何 AI 对话窗口，不绑定特定 AI 服务。
4. **渐进增强**：剪贴板 API 优先使用现代 `navigator.clipboard`，降级到 `execCommand('copy')` 兼容 `file://` 协议。

## 核心模块

### 1. 状态管理 (`state`)

单一全局状态对象，所有 UI 渲染和数据生成都基于此状态：

```javascript
const state = {
  type: "lecture",        // 课程类型：lecture(A型) / interactive(B型)
  style: "expert",        // 讲师风格
  length: "half",         // 时长预设：half(半天) / full(全天) / custom(自定义)
  customMinutes: 150,     // 自定义时长（分钟）
  securityLevel: "standard", // 内容安全级别：standard / professional / government
  files: [],              // 上传的文件列表（仅内存，不上传）
  timeline: []            // 时间结构（可编辑）
};
```

### 2. 渲染引擎 (`render()`)

采用手写的响应式渲染模式：任何状态变更后调用 `render()`，重新渲染所有右侧预览面板。

渲染链路：
```
state 变更 → render() → renderWorkflow()    (工作流步骤)
                     → renderTimeline()    (时间结构编辑器)
                     → renderTotal()       (指标仪表盘)
                     → renderSourceTable() (来源标注表)
                     → renderVerification() (核验规则)
                     → renderQualityPills() (质检清单)
```

左侧控制面板的渲染独立于 `render()`，通过 `renderControls()` 在初始化和特定交互时调用。

### 3. 时间结构编辑器 (`renderTimeline()`)

可编辑的课程时间分配器，支持：
- 调整每段时长（分钟数输入）
- 增删环节（添加段落 / 休息 / 午休）
- 午休自动分隔上下午，不计入课程总时长
- 自定义总时长时按比例缩放各段时间

时间结构数据格式：
```javascript
{ type: "segment"|"break"|"lunch", label: "段落名", minutes: 30 }
```

### 4. 文件上传处理 (`setupUpload()` / `handleFiles()`)

- 支持点击选择和拖拽上传
- 文件仅在浏览器内存中读取（`FileReader`），不上传到任何服务器
- 显示文件名、大小、类型摘要
- 支持删除已添加的文件

### 5. 启动包文本生成器 (`buildKickstartText()`)

核心输出模块，将 `state` 对象转换为一段结构化的 AI 执行协议文本。生成内容包括：

```
┌─ 课程基本信息（名称、类型、风格、时长）
├─ 内容安全配置（级别、脱敏规则、出处标注规则）
├─ 时间结构（各段落时长分配）
├─ 工作流指令（Step 0 → Step 6 的完整流程）
├─ 5轮交互引导脚本（AI 收到后按脚本逐轮引导用户）
├─ 核验规则表
└─ 质量检查清单（15项）
```

### 6. 预览与输出

- **预览模态框** (`showPreview()`)：生成前全屏预览协议文本
- **复制剪贴板** (`copyKickstart()`)：优先 `navigator.clipboard`，降级 `execCommand('copy')`
- **下载文件** (`downloadKickstart()`)：通过 `Blob` + `URL.createObjectURL` 生成 `.txt` 下载

## 数据流

```
用户操作（选参数/上传文件/编辑时间线）
        ↓
    state 变更
        ↓
    render() → 右侧预览面板实时更新
        ↓
用户点击「生成课程启动包」
        ↓
    showPreview() → buildKickstartText() → 预览全文
        ↓
用户选择：复制剪贴板 / 下载 .txt
        ↓
粘贴到任意 AI 对话窗口 → AI 按5轮脚本引导 → 课程生产
```

## CSS 架构

使用 CSS 自定义属性（CSS Variables）定义设计令牌：

```css
:root {
  --ink: #202124;         /* 主文字色 */
  --blue: #002FA7;        /* 克莱因蓝(IKB)主色 */
  --paper: #fbfaf6;       /* 纸张色背景 */
  --panel: #ffffff;       /* 面板背景 */
  --line: #d8d5cc;        /* 分割线 */
  --shadow: 0 16px 40px rgba(32,33,36,.08);
}
```

布局采用 CSS Grid 双栏：左侧 `aside`（控制面板，sticky 固定）+ 右侧 `main`（预览区，可滚动）。

## 技术约束

| 约束 | 说明 |
|------|------|
| 单文件 | 所有 HTML/CSS/JS 在一个 `.html` 文件中，不拆分 |
| 无构建 | 不使用 webpack/vite/esbuild 等构建工具 |
| 无依赖 | 不引入 React/Vue/jQuery 等框架 |
| 无后端 | 不需要服务器、数据库、API |
| 浏览器兼容 | 支持 Chrome 90+ / Firefox 88+ / Safari 14+ / Edge 90+ |

## 部署

- **本地使用**：双击 `index.html`，通过 `file://` 协议运行
- **在线版**：GitHub Pages 自动从 `main` 分支部署，访问 `https://ai-nande.github.io/course-workflow-kickstart/`
