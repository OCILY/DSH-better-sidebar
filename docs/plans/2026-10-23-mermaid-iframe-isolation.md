# Markdown 预览 Mermaid 图表：布局测量隔离到沙箱 iframe

- 日期：2026-10-23
- 影响面：`src/client/mermaid.tsx`、`src/client/mermaid-frame.ts`（新增）、
  `scripts/build-mermaid-frame-asset.mjs`（新增）、`src/client/sidebar.module.css`
- 关联：v0.13.1 引入的 Mermaid 预览（#164）、#341（`suppressErrorRendering`）

## 1. 现象

在 DSH Desktop（Windows，DSH 0.2.1-rc.2，插件 0.28.0）中打开一份约 90 KB、含 31 个
mermaid 围栏的 Markdown（`SYSTEM_ARCHITECTURE.md`）时页面卡死：桌面壳日志出现

```
renderer surface watchdog: page did not respond
terminating unresponsive renderer
unresponsive renderer replaced (reason: crashed, exitCode: 2)
```

后渲染器被杀、页面重载；侧栏恢复上次标签页后再次卡死，形成崩溃循环。
同尺寸、把 mermaid 围栏换成普通代码围栏的文档打开完全正常 → 问题锁定在 Mermaid
预览渲染链路，而不是文档大小或 Markdown 解析。

## 2. 根因（真机实测）

Mermaid 用 `SVGElement.getBBox()` 给每个标签测量文字尺寸（`calculateTextDimensions`
把测点 `<svg>` 追加到 `document.body`）。该测量与**整个宿主文档的布局**共享一个布局树：

| 测量环境 | 一次 `getBBox` | 说明 |
|---|---|---|
| 真实 DSH 窗口（15,418 个 DOM 节点、复杂样式） | **59–75 ms** | 每次测量触发一次文档级样式/布局刷新 |
| 空 iframe 文档 | **1 ms** | 独立、极小的布局树 |

一张图要做几十次测量，因此单图 **1.5–7.3 秒**；旧实现把文档里所有图在挂载时同时
发起渲染（每块一个 React root + `mermaid.render` 并行），31 张图累计让主线程连续
无响应 > 20 秒，超过桌面壳看门狗（5s 探测 / 10s 超时 / 连续 2 次）阈值。

对照实验（同页面内探针，2026-10-23）：

- `#root { contain: layout style }` / `contain: strict`：**无改善**（59/62 ms）；
- `#root { content-visibility: hidden }`：1 ms，但会隐藏整个界面，不可接受；
- 同源空 iframe 内测量：**1 ms**。

结论：唯一既能保持界面可见、又能去掉测量成本的隔离边界是**独立的 iframe 文档**。

## 3. 方案

### 3.1 自包含的沙箱 iframe（渲染器）

`src/client/mermaid-frame.ts`：

- 首次需要渲染图表时懒创建隐藏 iframe：
  `sandbox="allow-scripts"`（**不带 allow-same-origin**，opaque origin，无法访问 GUI
  文档），`position:fixed` 移到视口外、`visibility:hidden`；
- `src` 为 `blob:` URL，内容由 `MERMAID_FRAME_HTML` 提供（见 3.2）；
- postMessage 协议：
  - 父 → 子：`{ type: 'mermaid-render', id, code, theme }`
  - 子 → 父：`{ type: 'mermaid-result', id, svg }` / `{ type: 'mermaid-error', id, message }`
  - 子就绪：`{ type: 'mermaid-frame-ready' }`
- 单例复用、单请求 30s 超时、就绪 10s 超时；失败自动重建一次，连续失败后该页面
  停止重试并让对应图走「错误 + 源码」兜底（仅故障时，不用于正常路径）；
- 父侧仍在 `sanitizeSvg()` 之后注入 SVG，安全链路不变。

iframe 内的 iframe 协议（`FRAME_API`）负责 `mermaid.initialize()` 与 `render()`：
`securityLevel: 'strict'`、`htmlLabels: false`、`suppressErrorRendering: true`，
主题随请求传入（`'dark' | 'default'`）。

### 3.2 自包含资产（无 host 路由）

`scripts/build-mermaid-frame-asset.mjs` 用插件自己的 `mermaid` 依赖
（`mermaid/dist/mermaid.min.js`，UMD）与 `FRAME_API` 拼成一个完整 HTML，把两个
`<script>` 内联（`</script` → `<\/script`，其余源码逐字保留），生成
`src/client/mermaid-frame-asset.generated.ts`（`export const MERMAID_FRAME_HTML: string`，
显式 `: string` 注解避免 3.5 MB 字面量进入 `.d.ts`）。

生成文件加入 `.gitignore`，由 `prepare` / `build` / `bundle` / `watch` 在 tsc/vitest/
tsdown 之前生成；git 源安装走 `prepare`，因此同样可用。

选择内联而不是新增 host 路由：

- 插件客户端 chunk 体系（`/sidebar/bundle`）只服务构建期 chunk；
- 新增 frame HTML 路由需要 host 半改动 + 路由测试，且 frame 需要反代前缀安全的 URL；
- 内联后客户端 chunk 是自包含的：**3.69 MB（原 client-mermaid.js 7.04 MB）**，
  首次预览仍只加载这一个懒 chunk，没有额外请求；
- 代价：生成/构建多一步；mermaid 版本升级由 `pnpm install` 后的 `prepare` 自动重建。

### 3.3 渲染调度：可见优先 + 空闲补齐 + 占位

`src/client/mermaid.tsx`：

- 删除 chunk 内直接 `mermaid.render` 的旧路径，改为 `renderMermaidDiagram(code, theme)`；
- 模块级共享队列（`requestIdleCallback`，400ms 上限）一次只渲染一张图，两次之间
  `setTimeout(16)` 让出主线程（输入、布局、看门狗探针都能插进来）；
- 优先级：挂载时在视口附近（或 `IntersectionObserver`，rootMargin 800px 内）的图优先；
  不可见/隐藏 tab 的图走空闲队列，**最终全部渲染成图**（不再退化为代码块）；
- 卸载、主题切换、围栏变化用 generation 计数取消旧任务；
- 占位骨架 `.mermaidPending` / `.mermaidPendingBar`：按围栏行数预留高度，主题 token、
  `prefers-reduced-motion` 下停止动画，图到达时文章不跳动。

## 4. 实测结果（同机、同文档、31 图）

| 指标 | 修复前 | 修复后 |
|---|---|---|
| 单图渲染 | 1,500–7,300 ms | **30–104 ms** |
| 31 图总耗时 | 主线程阻塞 > 20 s，渲染器被杀 | **约 5 s 内空闲队列逐张完成** |
| 首次打开交互 | 卡死/崩溃循环 | 首屏立即可读，可见图先出 |
| watchdog 崩溃 | 必现 | **0 次新增** |
| 客户端 chunk | 7.04 MB | **3.69 MB** |
| 图表显示 | 卡死 / 受旧实现影响 | 全部成图 |

`pnpm typecheck`、`pnpm lint`、`pnpm build`、相关 spec（含新增占位/失败/资产用例）
全绿。

## 5. 已知取舍与后续

- **如果 iframe 加载失败**（极端 CSP/环境限制）：对应图显示错误 + 源码，不会退回
  主线程慢渲染，也不会静默丢内容；正常路径下图表全部渲染。
- **资产生成是新增构建步骤**：生成文件不提交；`prepare`/`build` 自动生成，缺失时
  脚本给出明确报错（"run pnpm install first"）。
- **可进一步压缩**：把 UMD 换成本项目 rolldown 构建的 minify chunk 可去重依赖
  （当前内联 3.4 MB minified UMD，已比原 7.04 MB 非压缩 chunk 小）；但需新增
  chunk 名与 5 处镜像，超出本修复范围。
- **仍受主线程调度影响**：图多时后台渲染会占用若干秒 CPU，但每次让出主线程，不触发
  看门狗；滚动时新进入视口的图会插队。
- **测试限制**：jsdom 无法跑真实 iframe/blob，spec 通过
  `setMermaidFrameRendererForTests()` 注入 stub；iframe 行为由真实挂载 lane
  （`diagram.md` 冒烟）覆盖。

## 6. 验证清单

- [x] 真机复现（修复前必现）与修复后回归（同一文档，无新增 watchdog）
- [x] 单图/总耗时测量（30–104 ms / ~5 s）
- [x] 占位 → 图替换、失败 → 源码兜底、多围栏全量渲染、跨围栏引用语义
- [x] 生成资产结构（两段内联脚本、严格 safe 转义、strict 配置）
- [x] `pnpm typecheck` / `pnpm lint` / `pnpm build` / `pnpm test` / `check:consumer-types`
