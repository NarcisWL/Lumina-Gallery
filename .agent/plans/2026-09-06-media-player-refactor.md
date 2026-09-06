# 独立媒体播放器：核心架构与队列解耦实施计划

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** 把 `ImageViewer` 从与浏览位置耦合的单体查看模态重构为全局独立的媒体播放器层——任何视图以"队列 + 起始项"打开，统一图片/视频查看与播放，`mediaId` 移出导航位置。

**Architecture:** 新建 `components/player/` 播放器层：纯函数队列状态机（reducer）→ `PlayerProvider`（React Context）→ `MediaPlayer` UI 壳（overlay/键盘/控制栏）+ `ImageViewPane`/`VideoPane` 面板。App 各点击入口改为 `openPlayer({ items, startIndex })`；`GalleryLocation` 删除 `mediaId`，导航层序列化/解析同步清理。旧 `ImageViewer` 与死代码 `ViewerControls` 删除。

**Tech Stack:** React 19 + TypeScript + vitest/jsdom + @testing-library/react（现有栈，零新增依赖）。

## Global Constraints

- 所有新增注释与文档使用简体中文（项目 AGENTS.md）。
- 零新增 npm 依赖；不引入状态库，播放器状态用 `useReducer` + Context。
- 音频播放继续走现有 `AudioPlayer`（`components/AudioPlayer.tsx`），本轮不改其链路；播放器对 `mediaType === 'audio'` 的队列项跳过打开（调用点已过滤，双保险）。
- 移动端 `mobile/` 目录不在范围内。
- 视频仍用原生 `<video>`；`VideoPane` 的 src 必须经 `resolveVideoSource(item)` 解析，为后续转码计划预留唯一替换点。
- 每个任务结束运行对应测试命令；Task 7 后运行全量前端测试 + `npm run build`。
- 受限账号不做前端权限判断（现状即由后端 API 兜底），播放器不新增任何网络请求。

---

## 文件结构

```
components/player/
  types.ts            # PlayerMediaSource、PlayerState、PlayerAction、PlayerCallbacks
  player-state.ts     # 纯函数：createInitialPlayerState / playerReducer / selectCurrentItem
  PlayerProvider.tsx  # Context + useMediaPlayer()；App 顶层挂载
  MediaPlayer.tsx     # UI 壳：overlay、键盘、控制栏、面板调度
  ImageViewPane.tsx   # 图片面板：缩放/平移/幻灯片/信息面板（自 ImageViewer 迁移）
  VideoPane.tsx       # 视频面板：原生 video + resolveVideoSource + 降级界面
navigation/
  types.ts            # 删 mediaId（Task 5）
  location.ts         # 序列化/解析/key 哈希清理 mediaId（Task 5）
删除:
  components/ImageViewer.tsx
  components/viewer/ViewerControls.tsx   # 无引用死代码
```

## 背景事实（执行者必读）

- 现状打开链路：`handleOpenMedia`（`App.tsx:2381`）`setSelectedItem(item)` + `updateLocation({ mediaId }, 'push')`；关闭链路 `handleCloseMedia`（`App.tsx:2418`）优先 `galleryNavigation.back()`；同步 effect（`App.tsx:791-796`）监听 `location.mediaId` 设置/清空 `selectedItem`。三者一并移除。
- `GalleryLocation.mediaId?: string`（`navigation/types.ts:25`）；URL 参数 `media=`（`location.ts:174` 写、`location.ts:235-237` 读）；`mediaId` 参与 location key 哈希（`location.ts:105`）。**key 哈希中移除该行会改变所有新生成 key 的值**——key 只在会话内用于缓存/快照匹配，历史条目里的旧 key 随旧 state 存活，跨格式不比对，无迁移需求（已验证：`restoreAuthoritative` 按 key 精确匹配，旧条目恢复时 key 来自旧 state 自身，不重算）。
- 队列来源：图片/视频点击入口 `handleMediaGalleryItemClick`（`App.tsx:2405`，作用于 `processedFiles`，`App.tsx:2553` 定义）与 `handleFolderGalleryItemClick`（`App.tsx:2390`，作用于 `mixedItems`，含 `mediaType === 'folder'` 卡片需过滤）。
- `ImageViewer.tsx` 内部结构：控制栏 385-479、信息面板 481-590、视频 600-631、音频 632-648（不迁移，音频入口在调用点已分流）、图片 649-672、左右导航按钮 675-692、键盘 191-220、缩放 224-330。
- 测试环境注意：本机 Node 26 的 vitest 无 `localStorage` 全局，组件测试须 `vi.stubGlobal('localStorage', ...)`（参考 `test/image-viewer-jump.test.tsx` 的 beforeEach）。

---

### Task 1: 播放器队列状态机（纯函数）

**Files:**
- Create: `components/player/types.ts`
- Create: `components/player/player-state.ts`
- Test: `test/player-state.test.ts`

**Interfaces:**
- Produces: `PlayerMediaSource { items: MediaItem[]; startIndex: number; sourceLabel?: string }`、`PlayerState { isOpen: boolean; items: MediaItem[]; index: number; sourceLabel?: string }`、`PlayerAction = { type: 'open'; source } | { type: 'close' } | { type: 'next' } | { type: 'prev' } | { type: 'select'; index }`、`createInitialPlayerState(): PlayerState`、`playerReducer(state, action): PlayerState`、`selectCurrentItem(state): MediaItem | null`

- [ ] **Step 1: 写失败测试**

```ts
// test/player-state.test.ts
import { describe, expect, it } from 'vitest';
import { createInitialPlayerState, playerReducer, selectCurrentItem } from '../components/player/player-state';
import type { MediaItem } from '../types';

const item = (id: string): MediaItem => ({
  id, url: `/api/file/${id}`, name: `${id}.jpg`, path: `/media/${id}.jpg`,
  folderPath: '/media', size: 1, type: 'image/jpeg', lastModified: 0,
  mediaType: 'image', sourceId: 'local',
});

const source = (count: number, startIndex = 0) => ({
  items: Array.from({ length: count }, (_, i) => item(`id${i}`)),
  startIndex,
});

describe('播放器队列状态机', () => {
  it('空队列或越界起点时 open 被拒绝', () => {
    expect(playerReducer(createInitialPlayerState(), { type: 'open', source: source(0) })).toEqual(createInitialPlayerState());
    expect(playerReducer(createInitialPlayerState(), { type: 'open', source: source(2, 5) })).toEqual(createInitialPlayerState());
  });

  it('open 设置队列、起点与打开状态', () => {
    const next = playerReducer(createInitialPlayerState(), { type: 'open', source: source(3, 1) });
    expect(next.isOpen).toBe(true);
    expect(next.index).toBe(1);
    expect(selectCurrentItem(next)?.id).toBe('id1');
  });

  it('同队列内 next/prev 移动索引，到达边界后保持不动', () => {
    let state = playerReducer(createInitialPlayerState(), { type: 'open', source: source(2) });
    state = playerReducer(state, { type: 'next' });
    expect(selectCurrentItem(state)?.id).toBe('id1');
    state = playerReducer(state, { type: 'next' });
    expect(state.index).toBe(1);
    state = playerReducer(state, { type: 'prev' });
    expect(state.index).toBe(0);
  });

  it('close 只改开关不清队列（保留队列供动画期间渲染）', () => {
    let state = playerReducer(createInitialPlayerState(), { type: 'open', source: source(2) });
    state = playerReducer(state, { type: 'close' });
    expect(state.isOpen).toBe(false);
    expect(state.items).toHaveLength(2);
  });
});
```

- [ ] **Step 2: 运行确认失败**

Run: `npx vitest run test/player-state.test.ts`
Expected: FAIL（模块不存在）

- [ ] **Step 3: 实现**

```ts
// components/player/types.ts
import type { MediaItem } from '../../types';

export interface PlayerMediaSource {
  items: MediaItem[];
  startIndex: number;
  sourceLabel?: string;
}

export interface PlayerState {
  isOpen: boolean;
  items: MediaItem[];
  index: number;
  sourceLabel?: string;
}

export type PlayerAction =
  | { type: 'open'; source: PlayerMediaSource }
  | { type: 'close' }
  | { type: 'next' }
  | { type: 'prev' }
  | { type: 'select'; index: number };
```

```ts
// components/player/player-state.ts
import type { MediaItem } from '../../types';
import type { PlayerAction, PlayerState } from './types';

export const createInitialPlayerState = (): PlayerState => ({ isOpen: false, items: [], index: 0 });

const canOpen = (source: { items: MediaItem[]; startIndex: number }) =>
  source.items.length > 0 && source.startIndex >= 0 && source.startIndex < source.items.length;

export const playerReducer = (state: PlayerState, action: PlayerAction): PlayerState => {
  switch (action.type) {
    case 'open':
      if (!canOpen(action.source)) return state;
      return {
        isOpen: true,
        items: action.source.items,
        index: action.source.startIndex,
        sourceLabel: action.source.sourceLabel,
      };
    case 'close':
      // 保留 items/index 供关闭动画期间渲染，下次 open 会整体覆盖。
      return { ...state, isOpen: false };
    case 'next':
      return state.index < state.items.length - 1 ? { ...state, index: state.index + 1 } : state;
    case 'prev':
      return state.index > 0 ? { ...state, index: state.index - 1 } : state;
    case 'select':
      return action.index >= 0 && action.index < state.items.length ? { ...state, index: action.index } : state;
    default:
      return state;
  }
};

export const selectCurrentItem = (state: PlayerState): MediaItem | null =>
  state.isOpen ? state.items[state.index] ?? null : null;
```

- [ ] **Step 4: 运行确认通过**

Run: `npx vitest run test/player-state.test.ts`
Expected: PASS (4 tests)

- [ ] **Step 5: Commit**

```bash
git add components/player/types.ts components/player/player-state.ts test/player-state.test.ts
git commit -m "feat: 播放器队列状态机（纯函数）"
```

---

### Task 2: PlayerProvider（React 接入层）

**Files:**
- Create: `components/player/PlayerProvider.tsx`
- Modify: `App.tsx`（仅临时挂载 Provider，包裹现有 `<ImageViewer>`；Task 5 替换）
- Test: `test/player-provider.test.tsx`

**Interfaces:**
- Consumes: Task 1 的 reducer 三件套。
- Produces: `useMediaPlayer(): { open(source: PlayerMediaSource): void; close(): void; next(): void; prev(): void; state: PlayerState; currentItem: MediaItem | null }`。`<PlayerProvider>` 无 props。

- [ ] **Step 1: 写失败测试**

```tsx
// test/player-provider.test.tsx
import React from 'react';
import { act } from 'react';
import { createRoot } from 'react-dom/client';
import { cleanup, render, screen } from '@testing-library/react';
import { afterEach, beforeEach, describe, expect, it, vi } from 'vitest';
import { PlayerProvider, useMediaPlayer } from '../components/player/PlayerProvider';
import type { MediaItem } from '../types';

beforeEach(() => {
  (globalThis as Record<string, unknown>).IS_REACT_ACT_ENVIRONMENT = true;
  vi.stubGlobal('localStorage', { getItem: vi.fn(() => null), setItem: vi.fn(), removeItem: vi.fn(), clear: vi.fn() });
});
afterEach(() => { cleanup(); vi.unstubAllGlobals(); });

const item = (id: string): MediaItem => ({
  id, url: `/api/file/${id}`, name: `${id}.jpg`, path: `/media/${id}.jpg`,
  folderPath: '/media', size: 1, type: 'image/jpeg', lastModified: 0,
  mediaType: 'image', sourceId: 'local',
});

const Probe = ({ onReady }: { onReady: (api: ReturnType<typeof useMediaPlayer>) => void }) => {
  const api = useMediaPlayer();
  onReady(api);
  return <span data-testid="current">{api.currentItem?.name ?? 'none'}</span>;
};

describe('PlayerProvider', () => {
  it('open 后 currentItem 指向起点，close 后 currentItem 为 null', () => {
    let api: ReturnType<typeof useMediaPlayer> | undefined;
    const container = document.createElement('div');
    document.body.appendChild(container);
    const root = createRoot(container);
    act(() => {
      root.render(
        <PlayerProvider>
          <Probe onReady={(value) => { api = value; }} />
        </PlayerProvider>
      );
    });
    const items = [item('a'), item('b')];
    act(() => api!.open({ items, startIndex: 1 }));
    expect(screen.getByTestId('current').textContent).toBe('b.jpg');
    act(() => api!.close());
    expect(screen.getByTestId('current').textContent).toBe('none');
    root.unmount();
  });

  it('空队列 open 是 no-op', () => {
    let api: ReturnType<typeof useMediaPlayer> | undefined;
    const container = document.createElement('div');
    document.body.appendChild(container);
    const root = createRoot(container);
    act(() => {
      root.render(
        <PlayerProvider>
          <Probe onReady={(value) => { api = value; }} />
        </PlayerProvider>
      );
    });
    act(() => api!.open({ items: [], startIndex: 0 }));
    expect(screen.getByTestId('current').textContent).toBe('none');
    root.unmount();
  });
});
```

- [ ] **Step 2: 运行确认失败**

Run: `npx vitest run test/player-provider.test.tsx`
Expected: FAIL（模块不存在）

- [ ] **Step 3: 实现**

```tsx
// components/player/PlayerProvider.tsx
import React, { createContext, useContext, useMemo, useReducer } from 'react';
import { createInitialPlayerState, playerReducer, selectCurrentItem } from './player-state';
import type { PlayerMediaSource, PlayerState } from './types';

interface PlayerContextValue {
  state: PlayerState;
  currentItem: ReturnType<typeof selectCurrentItem>;
  open: (source: PlayerMediaSource) => void;
  close: () => void;
  next: () => void;
  prev: () => void;
}

const PlayerContext = createContext<PlayerContextValue | undefined>(undefined);

export const PlayerProvider: React.FC<{ children: React.ReactNode }> = ({ children }) => {
  const [state, dispatch] = useReducer(playerReducer, undefined, createInitialPlayerState);
  const value = useMemo<PlayerContextValue>(() => ({
    state,
    currentItem: selectCurrentItem(state),
    open: (source) => dispatch({ type: 'open', source }),
    close: () => dispatch({ type: 'close' }),
    next: () => dispatch({ type: 'next' }),
    prev: () => dispatch({ type: 'prev' }),
  }), [state]);
  return <PlayerContext.Provider value={value}>{children}</PlayerContext.Provider>;
};

export const useMediaPlayer = (): PlayerContextValue => {
  const value = useContext(PlayerContext);
  if (!value) throw new Error('useMediaPlayer 必须在 PlayerProvider 内使用');
  return value;
};
```

- [ ] **Step 4: App.tsx 临时挂载**

`App.tsx` 组件树最外层（`<ImageViewer>` 所在层级的上级容器内）包一层 `<PlayerProvider>`；`ImageViewer` 暂保留，Task 5 移除。挂载后 `npx vitest run test/app-navigation-flow.test.tsx` 确认 43 passed（除 3 个既有 localStorage 环境失败）。

- [ ] **Step 5: 运行确认通过**

Run: `npx vitest run test/player-provider.test.tsx`
Expected: PASS (2 tests)

- [ ] **Step 6: Commit**

```bash
git add components/player/PlayerProvider.tsx test/player-provider.test.tsx App.tsx
git commit -m "feat: PlayerProvider 全局播放器上下文"
```

---

### Task 3: MediaPlayer 壳与图片面板（自 ImageViewer 迁移）

**Files:**
- Create: `components/player/MediaPlayer.tsx`
- Create: `components/player/ImageViewPane.tsx`
- Test: `test/media-player.test.tsx`

**Interfaces:**
- Consumes: `useMediaPlayer()`。
- Produces: `<MediaPlayer />`（无 props，渲染 null 当 `state.isOpen === false`）；`<ImageViewPane item={MediaItem} callbacks... />`；`resolveVideoSource(item: MediaItem): { kind: 'native'; url: string }`（本任务先在 `components/player/video-source.ts` 定义并导出，Task 4 消费，阶段四转码只改此文件）。

**迁移映射（源 `components/ImageViewer.tsx` → 目标）：**

| 源行号 | 内容 | 去向 |
|---|---|---|
| 31-104 | transform/slideshow/exif/fullscreen state | `ImageViewPane.tsx`（exif/fullscreen 相关随面板走） |
| 191-220 | 键盘导航 effect | `MediaPlayer.tsx`（←→/空格回调来自 `useMediaPlayer`，Esc 调 `close()`；删除 `onCloseRef` 闭包链） |
| 374-383 | overlay motion.div + 背景点击关闭 | `MediaPlayer.tsx`（`onClick={close}`） |
| 385-479 | 控制栏 | `MediaPlayer.tsx` 内联（删除重命名/收藏/删除按钮——这些操作留在画廊卡片层，播放器只读查看；收藏按钮保留，调 `onToggleFavorite`，从 App 透传） |
| 481-590 | 信息面板（含 EXIF） | `ImageViewPane.tsx` |
| 592-673 | 图片缩放/拖拽/双击/滚轮/触摸 | `ImageViewPane.tsx` |
| 675-692 | 左右导航按钮 | `MediaPlayer.tsx`（`onNext`/`onPrev` 来自 useMediaPlayer） |
| 345-358 | handleDeleteClick/handleJumpClick | **删除**（跳转按钮移除：播放器已不绑定路径，跳转语义由画廊"所在文件夹"视图承接；删除操作同样不进播放器） |

- [ ] **Step 1: 写失败测试**

```tsx
// test/media-player.test.tsx
import React from 'react';
import { act } from 'react';
import { createRoot } from 'react-dom/client';
import { cleanup, fireEvent, render, screen } from '@testing-library/react';
import { afterEach, beforeEach, describe, expect, it, vi } from 'vitest';
import { PlayerProvider, useMediaPlayer } from '../components/player/PlayerProvider';
import { MediaPlayer } from '../components/player/MediaPlayer';
import type { MediaItem } from '../types';

// localStorage stub 与 test/player-provider.test.tsx 相同（Node 26 环境）
beforeEach(() => {
  (globalThis as Record<string, unknown>).IS_REACT_ACT_ENVIRONMENT = true;
  vi.stubGlobal('localStorage', { getItem: vi.fn(() => null), setItem: vi.fn(), removeItem: vi.fn(), clear: vi.fn() });
});
afterEach(() => { cleanup(); vi.unstubAllGlobals(); });

const item = (id: string, mediaType: 'image' | 'video' = 'image'): MediaItem => ({
  id, url: `/api/file/${id}`, name: `${id}.jpg`, path: `/media/${id}.jpg`,
  folderPath: '/media', size: 1,
  type: mediaType === 'image' ? 'image/jpeg' : 'video/mp4',
  lastModified: 0, mediaType, sourceId: 'local',
});

const Harness = ({ items }: { items: MediaItem[] }) => {
  const player = useMediaPlayer();
  return (
    <>
      <button onClick={() => player.open({ items, startIndex: 0 })}>open</button>
      <MediaPlayer />
    </>
  );
};

const setup = (items: MediaItem[]) => {
  const container = document.createElement('div');
  document.body.appendChild(container);
  const root = createRoot(container);
  act(() => root.render(<PlayerProvider><Harness items={items} /></PlayerProvider>));
  return { container, root };
};

describe('MediaPlayer 壳', () => {
  it('关闭状态下不渲染任何内容', () => {
    const { root, container } = setup([item('a')]);
    expect(container.querySelector('[data-testid="media-player"]')).toBeNull();
    root.unmount();
  });

  it('点击背景关闭', () => {
    const { root } = setup([item('a')]);
    fireEvent.click(screen.getByText('open'));
    const overlay = screen.getByTestId('media-player');
    fireEvent.click(overlay); // 背景点击 → 关闭
    expect(screen.queryByTestId('media-player')).toBeNull();
    root.unmount();
  });

  it('图片与视频项分别渲染对应面板，队列切换更新面板', () => {
    const { root } = setup([item('a'), item('b', 'video')]);
    fireEvent.click(screen.getByText('open'));
    expect(screen.getByAltText('a.jpg')).toBeTruthy();
    fireEvent.keyDown(window, { key: 'ArrowRight' });
    expect(screen.getByTestId('video-pane')).toBeTruthy();
    root.unmount();
  });

  it('音频项不渲染任何音频面板（音频仍由 AudioPlayer 承接）', () => {
    const audio = { ...item('c'), mediaType: 'audio' as const, type: 'audio/mpeg' };
    const { root, container } = setup([audio]);
    fireEvent.click(screen.getByText('open'));
    // 壳层兜底：audio 项没有对应面板，不出现 audio 元素（调用点另有过滤）
    expect(container.querySelector('audio')).toBeNull();
    root.unmount();
  });
});
```

- [ ] **Step 2: 运行确认失败**

Run: `npx vitest run test/media-player.test.tsx`
Expected: FAIL（模块不存在）

- [ ] **Step 3: 实现 MediaPlayer.tsx**

```tsx
// components/player/MediaPlayer.tsx（骨架，控制栏按迁移映射自 ImageViewer.tsx:385-479 内联，保留收藏/信息/全屏/关闭按钮，删除重命名/删除/跳转按钮）
import React, { useEffect, useState } from 'react';
import { AnimatePresence, motion } from 'framer-motion';
import { useMediaPlayer } from './PlayerProvider';
import { ImageViewPane } from './ImageViewPane';
import { VideoPane } from './VideoPane';
import { Icons } from '../ui/Icon';
import { useLanguage } from '../../contexts/LanguageContext';

export const MediaPlayer: React.FC = () => {
  const { state, currentItem, close, next, prev } = useMediaPlayer();
  const [isFullScreen, setIsFullScreen] = useState(false);
  const { t } = useLanguage();

  useEffect(() => {
    const onKeyDown = (e: KeyboardEvent) => {
      if (!state.isOpen) return;
      if (e.key === 'Escape') close();
      if (e.key === 'ArrowRight') next();
      if (e.key === 'ArrowLeft') prev();
    };
    window.addEventListener('keydown', onKeyDown);
    return () => window.removeEventListener('keydown', onKeyDown);
  }, [state.isOpen, close, next, prev]);

  if (!currentItem) return null;

  return (
    <AnimatePresence>
      <motion.div
        data-testid="media-player"
        initial={{ opacity: 0 }} animate={{ opacity: 1 }} exit={{ opacity: 0 }}
        className="fixed inset-0 z-50 bg-black/95 flex items-center justify-center overflow-hidden"
        onClick={close}
      >
        {/* 控制栏：自 ImageViewer.tsx:385-479 迁移。保留：收藏(onToggleFavorite 由 App 经 props 注入 MediaPlayer)、
            信息、全屏、关闭。删除：重命名、删除、跳转文件夹。位置指示 items.length/index 显示队列进度。 */}
        {/* 面板调度：click 冒泡由各面板自行 stopPropagation（迁移代码已带） */}
        {currentItem.mediaType === 'video'
          ? <VideoPane item={currentItem} />
          : currentItem.mediaType === 'image'
            ? <ImageViewPane item={currentItem} />
            : null}
        {/* 左右导航按钮：自 ImageViewer.tsx:675-692 迁移，onClick 改为 { stopPropagation; prev()/next() }，
            边界禁用：index===0 隐藏左、index===items.length-1 隐藏右 */}
      </motion.div>
    </AnimatePresence>
  );
};
```

收藏回调注入：`MediaPlayer` 增加 `props { onToggleFavorite?: (item: MediaItem, type: 'file') => void }`，App 挂载处传 `handleToggleFavorite`。

- [ ] **Step 4: 实现 ImageViewPane.tsx 与 video-source.ts**

`ImageViewPane.tsx`：自 `ImageViewer.tsx` 迁移 31-104（transform/slideshow/缩放约束）、224-330（wheel/drag/touch/zoom）、481-590（信息面板）、592-673 的图片分支（650-671 motion.img）。修改点仅三处：`item` 来自 props；`onToggleInfo` 本地 state 保留；删除 `onNext`/`onPrev`/`onClose`/`onJumpToFolder`/`onDelete`/`onRename` 相关 props 与逻辑（幻灯片定时器保留，调 props 传 `onSlideNext`）。

```ts
// components/player/video-source.ts
import type { MediaItem } from '../../types';
import { getAuthUrl } from '../../utils/fileUtils';

export type VideoSource = { kind: 'native'; url: string };

/** 视频播放源唯一解析点：阶段四服务端转码只需扩展此函数（探测+转码 URL 回退）。 */
export const resolveVideoSource = (item: MediaItem): VideoSource => ({
  kind: 'native',
  url: getAuthUrl(item.url),
});
```

- [ ] **Step 5: 运行确认通过**

Run: `npx vitest run test/media-player.test.tsx`
Expected: PASS (4 tests)

- [ ] **Step 6: Commit**

```bash
git add components/player/ test/media-player.test.tsx
git commit -m "feat: MediaPlayer 播放器壳与图片/视频面板迁移"
```

---

### Task 4: 视频面板迁移

**Files:**
- Create: `components/player/VideoPane.tsx`
- Test: `test/video-pane.test.tsx`

**Interfaces:**
- Consumes: Task 3 的 `resolveVideoSource`。
- Produces: `<VideoPane item={MediaItem} />`；渲染 `data-testid="video-pane"` 容器；解码失败渲染 `data-testid="video-fallback"` 降级界面（含下载链接）。

- [ ] **Step 1: 写失败测试**

```tsx
// test/video-pane.test.tsx
import React from 'react';
import { act } from 'react';
import { createRoot } from 'react-dom/client';
import { cleanup, fireEvent, render, screen } from '@testing-library/react';
import { afterEach, beforeEach, describe, expect, it, vi } from 'vitest';
import { VideoPane } from '../components/player/VideoPane';
import type { MediaItem } from '../types';

beforeEach(() => {
  (globalThis as Record<string, unknown>).IS_REACT_ACT_ENVIRONMENT = true;
  vi.stubGlobal('localStorage', { getItem: vi.fn(() => null), setItem: vi.fn(), removeItem: vi.fn(), clear: vi.fn() });
});
afterEach(() => { cleanup(); vi.unstubAllGlobals(); });

const videoItem: MediaItem = {
  id: 'v1', url: '/api/file/v1', name: 'clip.mov', path: '/media/clip.mov',
  folderPath: '/media', size: 1, type: 'video/quicktime', lastModified: 0,
  mediaType: 'video', sourceId: 'local',
};

describe('VideoPane', () => {
  it('默认渲染原生 video 且 src 来自 resolveVideoSource', () => {
    const container = document.createElement('div');
    document.body.appendChild(container);
    const root = createRoot(container);
    act(() => root.render(<VideoPane item={videoItem} />));
    const video = container.querySelector('video');
    expect(video).toBeTruthy();
    expect(video?.getAttribute('src')).toContain('/api/file/v1');
    root.unmount();
  });

  it('video error 事件后渲染降级界面并提供下载链接', () => {
    const container = document.createElement('div');
    document.body.appendChild(container);
    const root = createRoot(container);
    act(() => root.render(<VideoPane item={videoItem} />));
    fireEvent.error(container.querySelector('video')!);
    expect(screen.getByTestId('video-fallback')).toBeTruthy();
    expect(screen.getByTestId('video-fallback').querySelector('a[download]')).toBeTruthy();
    root.unmount();
  });
});
```

- [ ] **Step 2: 运行确认失败**

Run: `npx vitest run test/video-pane.test.tsx`
Expected: FAIL（模块不存在）

- [ ] **Step 3: 实现**

```tsx
// components/player/VideoPane.tsx（自 ImageViewer.tsx:600-631 迁移，结构不变）
import React, { useEffect, useRef, useState } from 'react';
import { Icons } from '../ui/Icon';
import { getAuthUrl } from '../../utils/fileUtils';
import { resolveVideoSource } from './video-source';
import type { MediaItem } from '../../types';

export const VideoPane: React.FC<{ item: MediaItem }> = ({ item }) => {
  const [videoError, setVideoError] = useState(false);
  const videoRef = useRef<HTMLVideoElement>(null);
  const source = resolveVideoSource(item);
  useEffect(() => setVideoError(false), [item.id]);

  if (videoError) {
    return (
      <div data-testid="video-fallback" className="flex flex-col items-center justify-center p-8 bg-gray-900 rounded-xl border border-gray-700 text-center max-w-md" onClick={(e) => e.stopPropagation()}>
        <Icons.AlertTriangle size={48} className="text-yellow-500 mb-4" />
        <h3 className="text-xl font-bold text-white mb-2">Playback Failed</h3>
        <p className="text-gray-400 text-sm mb-6">
          The video format <span className="font-mono bg-black/30 px-1 rounded">{item.type}</span> might not be supported by your browser.
        </p>
        <a href={getAuthUrl(item.url)} download className="bg-white text-gray-900 hover:bg-gray-200 px-6 py-2 rounded-full font-bold transition-colors flex items-center gap-1">
          <Icons.Download size={18} /> Download Video
        </a>
      </div>
    );
  }
  return (
    <div data-testid="video-pane" className="w-full h-full flex flex-col items-center justify-center relative group" onClick={(e) => e.stopPropagation()}>
      <video
        ref={videoRef}
        src={source.url}
        controls autoPlay
        onError={() => setVideoError(true)}
        className="max-w-full max-h-full shadow-2xl rounded-sm focus:outline-none"
      />
    </div>
  );
};
```

- [ ] **Step 4: 运行确认通过**

Run: `npx vitest run test/video-pane.test.tsx`
Expected: PASS (2 tests)

- [ ] **Step 5: Commit**

```bash
git add components/player/VideoPane.tsx test/video-pane.test.tsx
git commit -m "feat: VideoPane 视频面板与统一播放源解析点"
```

---

### Task 5: App 接线与 mediaId 导航解耦

**Files:**
- Modify: `App.tsx`（mediaId 相关 21 处引用清理）
- Modify: `navigation/types.ts:25`（删 `mediaId?: string`）
- Modify: `navigation/location.ts:105`（key 哈希删 `media=` 行）、`location.ts:174`（序列化删）、`location.ts:235-237`（解析删——旧链接中的 `media=` 参数被静默忽略，深链降级为纯浏览位置）
- Modify: `test/app-navigation-flow.test.tsx`、`test/navigation-location.test.ts`、`test/navigation-controller.test.ts`、`test/gallery-query-key.test.ts`（mediaId 相关断言更新）
- Delete: `components/ImageViewer.tsx`、`components/viewer/ViewerControls.tsx`

**Interfaces:**
- Consumes: Task 2 的 `useMediaPlayer()`——App 中解构命名 `const { open: openPlayer, close: closePlayer } = useMediaPlayer()`；Task 3 的 `<MediaPlayer onToggleFavorite={handleToggleFavorite} />`。
- Produces: `handleOpenMedia(item: MediaItem)` 改为：

```tsx
const handleOpenMedia = (item: MediaItem) => {
    const snapshot = galleryViewportRef.current?.captureSnapshot();
    if (snapshot?.locationKey === galleryNavigation.location.key) {
        galleryNavigation.captureImmediateSnapshot({ ...snapshot, loadedOffset: serverOffset });
    }
    openPlayer({ items: processedFiles.filter(f => f.mediaType === 'image' || f.mediaType === 'video'), startIndex: Math.max(0, processedFiles.findIndex(f => f.id === item.id)) });
};
```

**导航位置与播放器联动（新增行为）：** `mediaId` 移出导航后，播放器打开不再产生历史条目；浏览器后退会回到**上一个浏览位置**。为保证"后退总能回到浏览流"，App 新增联动 effect：导航位置 key 变化时自动关闭播放器（跳过首次渲染）——后退、前进或任何视图切换都会收起播放器；同位置内的关闭仍由 Esc/背景点击/关闭按钮完成。

```tsx
const locationKey = galleryNavigation.location.key;
const isFirstLocationRef = useRef(true);
useEffect(() => {
    if (isFirstLocationRef.current) { isFirstLocationRef.current = false; return; }
    closePlayer();
}, [locationKey, closePlayer]);
```

- [ ] **Step 1: 更新导航层测试（红）**

`test/navigation-location.test.ts`：删除/改写 `media=` 序列化与解析断言为新契约——序列化不输出 `media=`；解析含 `media=` 的输入时忽略该参数。`test/gallery-query-key.test.ts`：key 哈希不再含 media 维度。

Run: `npx vitest run test/navigation-location.test.ts test/gallery-query-key.test.ts`
Expected: FAIL

- [ ] **Step 2: 实现导航层清理**

- `navigation/types.ts`：删除 `mediaId?: string;`（第 25 行）。
- `navigation/location.ts`：删除 105 行 `media=${location.mediaId ?? ''},`；删除 174 行 `if (location.mediaId) params.set(...)`；删除 235-237 行解析块。
- Run: `npx tsc --noEmit -p tsconfig.json 2>&1 | head -30`（App.tsx 将报 mediaId 类型错误，属预期，Step 3 清理）

- [ ] **Step 3: App.tsx 清理（21 处）**

删除/改写清单（行号为当前值，±3 容差）：
- 791-796 同步 effect：删除 `mediaId` 分支，effect 只保留 view/folderPath/layout/sort/filter/searchDraft 同步与 `setSelectedItem` 移除（`selectedItem` state 整个删除——播放器接管）。
- 1522-1553 主加载 effect：无 mediaId 依赖，不动。
- 2381-2423：`handleOpenMedia` 按上文 Interfaces 改写；`handleCloseMedia` 整个删除；`handleFolderGalleryItemClick`（2390）音频分支保留、图片/视频分支改调新 `handleOpenMedia`；`handleMediaGalleryItemClick`（2405）同。
- 3132-3157：`<ImageViewer .../>` 整块替换为 `<MediaPlayer onToggleFavorite={handleToggleFavorite} />`。
- 3141/3150（onNext/onPrev 内 updateLocation mediaId replace）：删除（播放器内部 next/prev）。
- `createTopLevelViewLocationUpdate`（329-339）：返回类型与实现删 `mediaId: undefined`；`UnifiedGalleryToolbar` onSearch/onSortChange/onLayoutChange/onFilterChange（432-446）的 `mediaId: undefined` 字段全部移除。
- 其余 grep `mediaId` 到零引用：`grep -n "mediaId" App.tsx` 输出为空。

- [ ] **Step 4: 全量前端测试**

Run: `npx vitest run 2>&1 | tail -5`
Expected: 除 3 个既有 localStorage 环境失败外全部 PASS；app-navigation-flow.test.tsx 中 mediaId 相关用例按新契约更新（打开媒体不再产生导航 push——改为断言 `player.open` 效果或删除对应断言）。

- [ ] **Step 5: 构建验证**

Run: `npm run build 2>&1 | tail -3`
Expected: `✓ built`

- [ ] **Step 6: Commit**

```bash
git add -A
git commit -m "refactor: 媒体播放器解耦导航 mediaId，替代 ImageViewer"
```

---

### Task 6: 行为验收与文档

- [ ] **Step 1: 手工验收清单（本地 `npm run dev`）**

1. 全库/搜索/收藏/文件夹视图点开图片 → 播放器以该视图列表为队列，←→ 在队列内移动，末尾不再前进。
2. 点开视频 → 原生播放；构造一个浏览器不支持的编码文件（如 HEVC mov）→ 显示降级界面与下载。
3. 播放器打开时浏览器后退 → 播放器关闭回原视图，滚动位置保持。
4. 收藏切换、信息面板、全屏、幻灯片可用；音频点击仍走 AudioPlayer。
5. 旧格式链接（含 `media=` 参数）打开不报错，落在本位置。

- [ ] **Step 2: release_notes.md 追加 v1.3.0 本地候选小节**（行为变化：播放器全局化、后退不再精确回到"正在看的某张图"、跳转按钮移除）

- [ ] **Step 3: 最终全量验证**

Run: `npx vitest run 2>&1 | tail -4 && npm run build 2>&1 | tail -3`
Expected: 既有 3 个环境失败之外全绿；构建通过。

---

## 后续独立计划（本计划不含）

1. **显示与交互打磨**（阶段三）：缩放边界/手势细节、控制栏自动隐藏、容器级全屏、比例适配、可访问性。依赖本计划落地后做设计审计（`design-taste-frontend`）再立计划。
2. **视频转码**（阶段四）：后端 ffprobe 探测接口 + 按需 FFmpeg 转码（流式 fMP4）+ 缓存治理 + `resolveVideoSource` 扩展。独立后端计划，含并发上限实测。
