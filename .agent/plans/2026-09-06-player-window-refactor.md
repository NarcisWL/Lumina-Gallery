# 播放器弹窗化重构（阶段二点五：视觉与交互解耦）实施计划

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** 把播放器从"全屏遮罩模态"重构为"非模态悬浮弹窗"：点开媒体出现可拖动的自适应比例浮窗，浏览不受打断；浮窗可收成小窗与 FAB 圆钮；点击全屏才进入独占全屏。

**Architecture:** `PlayerState` 扩展 `displayMode: 'window' | 'mini' | 'fab' | 'fullscreen'`（窗口与形态偏好持久化到 localStorage）；新组件 `PlayerWindow` 取代现 `MediaPlayer` 的全屏遮罩形态（非模态、无遮罩层、framer-motion 形态切换动画、全站 `glass-1 gallery-toolbar-glass` 浮岛材质）；独占全屏复用现有全屏布局并挂 Fullscreen API。信息面板已在壳层（hotfix-1），收藏实时回写 `patchItem` 已就绪。

**Tech Stack:** 现有栈（React 19 + framer-motion + Tailwind + vitest/jsdom），零新增依赖。

## Global Constraints

- 新增注释简体中文；零新增依赖；不动 `mobile/`。
- 非模态：浮窗打开时不得渲染全屏遮罩层，页面其余区域保持可交互（画廊可继续点选、滚动）。
- 浮窗材质必须复用全站浮岛类：`glass-1 gallery-toolbar-glass rounded-2xl border border-white/5 shadow-2xl`（与 GalleryNavigationBar 一致）。
- 浮窗打开/形态切换/关闭必须有过渡动画（framer-motion；AnimatePresence + layout）。
- 视频/音频在 window/mini/fab 形态下持续播放（形态切换不重载 video 元素：VideoPane 保持在 DOM，仅容器变形）。
- 图片的滚轮缩放/双击缩放/拖拽平移在浮窗内保留（contain 显示于窗口内容区）。
- 每任务 TDD；跳过 git commit（主控统一处理）。
- 既有 3 个 Node 26 localStorage 环境失败不计；`utils/animation.ts` tsc 基线不计。

---

## 文件结构

```
components/player/
  player-state.ts     # +displayMode、SET_MODE/PATCH 动作、open 默认 window、偏好持久化纯函数
  PlayerProvider.tsx  # +setMode(mode)、displayMode 派生；窗口位置/尺寸偏好存取
  PlayerWindow.tsx    # 新：非模态浮窗壳（拖动/三形态/动画/材质/控制栏）
  PlayerFullscreen.tsx# 新：独占全屏形态（Fullscreen API + 现有全屏布局迁移）
  MediaPlayer.tsx     # 调度器：isOpen 时按 displayMode 渲染 Window 或 Fullscreen（信息面板/收藏逻辑随壳层）
  ImageViewPane.tsx   # 适配窗口 contain 显示（现有缩放逻辑保留）
  VideoPane.tsx       # 不变（容器内 contain）
```

## 关键设计（执行者必读）

- **状态机**：`open` 默认 `displayMode:'window'`；`SET_MODE` 在 window/mini/fab 间自由切换；`fullscreen` 进入时请求 `document.documentElement` 或容器 `requestFullscreen()`，`fullscreenchange` 退出时回 `window`；`close` 从任意形态生效。
- **位置/尺寸偏好**：`components/player/window-prefs.ts` 纯函数 `loadWindowPrefs()/saveWindowPrefs()`（localStorage key `luvia.playerWindow.v1`，字段 `{x,y,width,mode}`，带 try/catch 与字段校验——测试环境无 localStorage 必须不抛错）。窗口尺寸 = 记忆宽度 clamp(320, 视口宽*0.42, 560) × 媒体比例高度 clamp(视口高*0.24, 视口高*0.72)；比例用 `thumb_width/thumb_height` 或 16:9 兜底；视频在 `onLoadedMetadata` 后可校正。
- **拖动**：头部区域 pointer 事件（pointerdown 捕获起点 → pointermove 更新 → pointerup 落库），位置 clamp 在视口内。不使用 framer-motion drag（与内部图片拖拽手势冲突），自实现 pointer 逻辑以便测试。
- **fab 形态**：56px 圆钮，右下角（16px 边距）固定（fab 不拖动），背景显示当前项缩略图（`getAuthUrl(item.thumbnailUrl)`，无缩略图用图标），视频显示播放角标。点击 fab→window。
- **非模态与 z-index**：浮窗容器 `fixed z-40`（低于现有模态弹窗体系），`pointer-events-auto` 仅容器本身；不渲染任何 `inset-0` 遮罩。Esc 键：fullscreen 先退全屏；其余形态关闭播放器。
- **全屏**：`PlayerFullscreen` 迁移现 `MediaPlayer` 的全屏遮罩布局（黑底/控制栏/左右导航/信息面板），进入时 `containerRef.requestFullscreen()`，监听 `fullscreenchange` 同步状态；video 在 window→fullscreen 间不跨容器搬移：fullscreen 形态下重挂 VideoPane（可接受短暂重载，阶段三再优化为 keep-alive）。

---

### Task A: displayMode 状态机与偏好持久化（纯函数）

**Files:** Modify `components/player/player-state.ts`、`components/player/types.ts`、Create `components/player/window-prefs.ts`；Test `test/player-state.test.ts`（追加）+ `test/window-prefs.test.ts`

**Interfaces（后续任务按名消费）:**
- `PlayerState` 增加 `displayMode: 'window'|'mini'|'fab'|'fullscreen'`；`open` 后默认 `'window'`；`SET_MODE {mode}`（fab/mini/window 互转自由，fullscreen 由宿主 effect 驱动）。
- `PlayerProvider` 增 `setMode(mode)`；`useMediaPlayer()` 返回值新增 `displayMode`。
- `loadWindowPrefs(): {x,y,width}|null`、`saveWindowPrefs(prefs)`（localStorage 缺失/损坏返回 null，不抛错）。

- [ ] Step 1 写失败测试（displayMode 默认值、SET_MODE 转换、非法 mode 拒绝、prefs 往返与损坏容错）
- [ ] Step 2 跑红 `npx vitest run test/player-state.test.ts test/window-prefs.test.ts`
- [ ] Step 3 实现类型与 reducer 分支、prefs 纯函数
- [ ] Step 4 跑绿；Step 5 报告（无 commit）

### Task B: PlayerWindow 浮窗壳（非模态/拖动/比例自适应/材质/动画）

**Files:** Create `components/player/PlayerWindow.tsx`；Modify `components/player/MediaPlayer.tsx`（isOpen 时渲染 PlayerWindow，替代现全屏 motion.div 布局；信息面板/收藏/控制栏逻辑迁入或由其承载）；Modify `components/player/ImageViewPane.tsx`（内容区 contain 适配）；Test `test/player-window.test.tsx`

**Interfaces:** `<PlayerWindow onToggleFavorite? />`；从 `useMediaPlayer()` 取 `state/currentItem/close/next/prev/setMode`；拖动用头部抓手 `data-testid="player-window-handle"`；窗口容器 `data-testid="player-window"`。

关键行为（逐条对应断言）：
- 打开时无 `inset-0` 遮罩兄弟节点；容器为 `fixed` 浮窗，默认出现在视口右下区域（右 24px 下 24px，clamp 后）。
- pointer 拖动 handle 移动窗口位置；不拖动时点击内容区正常（图片缩放等不受干扰）。
- 关闭按钮 `close()`；Esc 关闭。
- 动画：`AnimatePresence` + `initial/animate/exit`（scale 0.85→1 + fade）；exit 存活 250ms 足够动画。
- 材质类含 `gallery-toolbar-glass`。

- [ ] Step 1 写失败测试（渲染位置特征、无遮罩、拖动指针事件模拟、Esc、动画类存在、媒体比例反映在容器 style aspect-ratio）
- [ ] Step 2 跑红；Step 3 实现（含 ImageViewPane contain 适配：根容器改为 `absolute inset-0` 于窗口内容区，保留缩放）；Step 4 跑绿；Step 5 报告

### Task C: mini 与 FAB 形态及切换动画

**Files:** Modify `components/player/PlayerWindow.tsx`（三形态分支）+ `PlayerProvider`（形态切换经由 setMode）；Test `test/player-window.test.tsx`（追加）

关键行为：
- mini：固定 240px 宽小窗（仍可拖动），内容区只保留媒体与最小控制（关闭/展开/播放暂停对视频沿用原生控件）。
- fab：56px 圆钮 `data-testid="player-fab"`，右下 16px，缩略图背景 `getAuthUrl(item.thumbnailUrl)`（onError 兜底 Icons.Image），点击回 window；fab 存在时视频暂停由用户手动（不自动暂停，音乐场景继续）。
- 切换动画用 framer-motion `layout` + AnimatePresence mode="popLayout"（窗口↔fab 形变）。
- 形态偏好落 localStorage（Task A prefs 扩展 mode 字段）。

- [ ] Step 1 写失败测试（fab 渲染特征/点击回 window/mini 特征/切模式动画容器存在）；Step 2 红；Step 3 实现；Step 4 绿；Step 5 报告

### Task D: 独占全屏形态

**Files:** Create `components/player/PlayerFullscreen.tsx`；Modify `MediaPlayer.tsx`（displayMode==='fullscreen' 渲染分支）、`PlayerWindow.tsx`（控制栏加全屏按钮）；Test `test/player-fullscreen.test.tsx`

关键行为：
- 控制栏全屏按钮：`setMode('fullscreen')` + 宿主 effect `requestFullscreen()`（容器 ref）；`fullscreenchange` 退出（含用户 Esc）→ `setMode('window')`。
- 全屏布局迁移现 MediaPlayer 的黑底全屏样式（`fixed inset-0 z-50 bg-black/95` + 控制栏 + 左右导航 + 信息面板）。
- jsdom 不支持 requestFullscreen：实现里对 `requestFullscreen?.()` 可选调用并 catch；测试断言 `setMode('fullscreen')` 后渲染全屏分支与退出回调。

- [ ] Step 1-5 同前（TDD）

### Task E: 接线验证与全量回归

- [ ] App/接线核对：`GalleryApp` 内 `patchItem` 仍生效（收藏回写）、信息面板按钮在 window/fullscreen 两形态均可用；`handleOpenMedia` 不变。
- [ ] `npx vitest run` 全量（除 3 既有失败全绿）；`npm run build`；`npx tsc --noEmit` 零新增。
- [ ] 手工冒烟清单（主控/用户）：打开浮窗继续浏览、拖动、mini/fab、全屏进出、视频窗口内持续播放。

### Task F: 文档与部署（主控执行）

- release_notes v1.3.1 小节；提交推送；FNOS 构建/备份/候选/切换/验证；HLG append。

## 后续（不在本计划）

- 窗口自由 resize；多窗口；PiP API；视频全屏 keep-alive。
