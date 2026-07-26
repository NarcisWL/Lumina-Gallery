# WebUI 可恢复导航升级实施计划

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [x]`) syntax for tracking.

**Goal:** 为 WebUI 建立统一、可恢复的浏览器式导航，使目录、收藏夹、媒体查看器和三种图库布局在后退、前进、上一级及刷新后保持正确位置。

**Architecture:** 浏览器 History 作为唯一历史事实源，`GalleryLocation` 描述可分享地址状态，`ViewportSnapshot` 描述会话内浏览锚点，按完整查询键隔离分页缓存。`VirtualGallery` 通过统一位置协议适配网格、时间线和瀑布流，`App.tsx` 只负责装配导航控制器、查询缓存和页面组件。

**Tech Stack:** React 19、TypeScript 5、Vite 5、react-window 1.8、TanStack Query 5、Vitest 1.6、Testing Library、jsdom 24。

## Global Constraints

- Web 端优先；本轮不修改 React Native、macOS Widget 或后端 API。
- 保持现有 `#folder=` 深链兼容，不在本轮暴露新的服务器绝对路径格式。
- History、内存缓存和 `sessionStorage` 只保存查询键、锚点与分页元数据，不保存媒体数组。
- 位置恢复优先使用稳定项目 ID，像素值只作兜底。
- 本地模式与服务器模式必须共享导航语义。
- 所有新增文档、注释和用户可见文案使用简体中文。
- 不回滚或覆盖任务开始前已有改动。

---

### Task 1: 导航领域模型与纯函数

**Files:**
- Create: `navigation/types.ts`
- Create: `navigation/location.ts`
- Create: `navigation/history-state.ts`
- Test: `test/navigation-location.test.ts`

**Interfaces:**
- Produces: `GalleryLocation`、`ViewportSnapshot`、`GalleryHistoryState`、`createLocationKey()`、`parseGalleryUrl()`、`serializeGalleryUrl()`、`getParentFolderPath()`、`createHistoryState()`。
- Consumes: `ViewMode`、`GridLayout`、`SortOption`、`FilterOption` from `types.ts`。

- [x] **Step 1: 编写失败测试**

覆盖 Hash 深链解析、查询参数往返、Windows/Unix 路径父级计算、不同搜索/排序生成不同 `locationKey`、历史状态序列化不包含媒体数组。

- [x] **Step 2: 运行测试确认 RED**

Run: `npx vitest run test/navigation-location.test.ts`

Expected: FAIL，原因是 `navigation/location.ts` 等模块尚不存在。

- [x] **Step 3: 实现最小领域模型**

```ts
export interface GalleryLocation {
  key: string;
  view: ViewMode;
  folderPath: string;
  search: string;
  sort: SortOption;
  filter: FilterOption;
  layout: GridLayout;
  randomSeed?: string;
  mediaId?: string;
}

export interface ViewportSnapshot {
  locationKey: string;
  anchorItemId?: string;
  anchorIndex?: number;
  offsetWithinItem: number;
  fallbackScrollTop: number;
  loadedOffset: number;
  capturedAt: number;
}
```

- [x] **Step 4: 运行测试确认 GREEN**

Run: `npx vitest run test/navigation-location.test.ts`

Expected: PASS。

### Task 2: History 控制器与快照存储

**Files:**
- Create: `navigation/navigation-controller.ts`
- Create: `navigation/snapshot-store.ts`
- Create: `hooks/useGalleryNavigation.ts`
- Test: `test/navigation-controller.test.ts`

**Interfaces:**
- Consumes: Task 1 的 `GalleryLocation`、`GalleryHistoryState`、`ViewportSnapshot`。
- Produces: `GalleryNavigationController`、`createSnapshotStore()`、`useGalleryNavigation()`。

- [x] **Step 1: 编写失败测试**

覆盖 `push`、`replace`、`back`、`forward`、`up`、`popstate`、每个顶层视图最后位置、最多 40 个 `sessionStorage` 快照和失效目录降级。

- [x] **Step 2: 运行测试确认 RED**

Run: `npx vitest run test/navigation-controller.test.ts`

Expected: FAIL，原因是控制器尚不存在。

- [x] **Step 3: 实现 History 唯一事实源**

进入目录、提交搜索和打开媒体使用 `pushState`；排序、筛选和布局使用 `replaceState`；上一级优先回退到真实父目录历史条目，否则创建父目录条目。

- [x] **Step 4: 运行测试确认 GREEN**

Run: `npx vitest run test/navigation-controller.test.ts`

Expected: PASS。

### Task 3: 按查询键隔离服务端分页

**Files:**
- Create: `hooks/useGalleryQuery.ts`
- Create: `navigation/query-key.ts`
- Test: `test/gallery-query-key.test.ts`
- Modify: `index.tsx`
- Modify: `package.json`
- Modify: `package-lock.json`

**Interfaces:**
- Consumes: `GalleryLocation`。
- Produces: `createGalleryQueryKey()`、`useGalleryQuery()`、全局 `QueryClientProvider`。

- [x] **Step 1: 安装已核验兼容依赖**

Run: `npm install @tanstack/react-query@5.101.4`

- [x] **Step 2: 安装测试依赖**

Run: `npm install -D vitest@1.6.1 jsdom@24.1.3 @testing-library/react@16.3.2 @testing-library/dom@10.4.1 @testing-library/user-event@14.6.1`

- [x] **Step 3: 编写查询键失败测试**

验证用户、视图、目录、搜索、排序、筛选和随机种子任一变化都会产生不同查询键，布局变化只影响视口快照而不重复请求媒体数据。

- [x] **Step 4: 实现分页查询**

使用 TanStack Query 的无限查询缓存当前目录分页；请求绑定 `AbortSignal`，旧导航请求不得覆盖新位置；返回时复用已加载页面和 `nextOffset`。

- [x] **Step 5: 运行测试**

Run: `npx vitest run test/gallery-query-key.test.ts`

Expected: PASS。

### Task 4: 三种布局统一位置协议

**Files:**
- Create: `components/gallery/viewport-types.ts`
- Create: `components/gallery/GridViewport.tsx`
- Create: `components/gallery/TimelineViewport.tsx`
- Create: `components/gallery/MasonryViewport.tsx`
- Modify: `components/VirtualGallery.tsx`
- Test: `test/viewport-anchor.test.ts`

**Interfaces:**
- Consumes: `ViewportSnapshot`、`MediaItem[]`。
- Produces: `GalleryViewportProps`，包含 `viewKey`、`restoreSnapshot`、`onSnapshotChange`、`onRestoreComplete`。

- [x] **Step 1: 编写锚点换算失败测试**

覆盖项目 ID 优先、索引兜底、响应式列数变化、目标项目尚未加载和失效锚点降级。

- [x] **Step 2: 实现统一适配器**

网格使用 `FixedSizeGrid.scrollToItem()`；时间线使用 `VariableSizeList.scrollToItem()`；瀑布流使用首个可见项目锚点与容器偏移；所有布局节流上报快照。

- [x] **Step 3: 验证三种布局**

Run: `npx vitest run test/viewport-anchor.test.ts`

Expected: PASS。

### Task 5: 浏览器式导航控件

**Files:**
- Create: `components/navigation/GalleryNavigationBar.tsx`
- Create: `components/navigation/Breadcrumbs.tsx`
- Modify: `components/Navigation.tsx`
- Modify: `components/navigation/MobileHeader.tsx`
- Modify: `contexts/LanguageContext.tsx`
- Test: `test/gallery-navigation-bar.test.tsx`

**Interfaces:**
- Consumes: `useGalleryNavigation()` 返回的 `canGoBack`、`canGoForward`、`back()`、`forward()`、`up()`、`navigate()`。
- Produces: 桌面和移动 Web 共用的后退、前进、上一级、面包屑和回到顶部入口。

- [x] **Step 1: 编写交互失败测试**

覆盖按钮禁用状态、面包屑跳转、移动标题、键盘快捷键和回到顶部。

- [x] **Step 2: 实现控件**

桌面显示后退、前进、上一级和面包屑；移动端显示返回、当前目录和可横向滚动面包屑；保留现有视觉令牌和响应式布局。

- [x] **Step 3: 运行组件测试**

Run: `npx vitest run test/gallery-navigation-bar.test.tsx`

Expected: PASS。

### Task 6: App 集成与媒体查看器历史

**Files:**
- Modify: `App.tsx`
- Modify: `components/ImageViewer.tsx`
- Modify: `types.ts`
- Test: `test/app-navigation-flow.test.tsx`

**Interfaces:**
- Consumes: Tasks 1-5 全部接口。
- Produces: 单一导航状态装配；旧 `currentPath`、重复初始化和全局分页单例退出导航事实源。

- [x] **Step 1: 编写关键流程失败测试**

覆盖父目录第 700 项返回、收藏夹返回、媒体查看器返回、A/B/C 后退前进、快速目录请求竞态和刷新深链。

- [x] **Step 2: 集成导航与查询**

离开位置前同步捕获快照；恢复时先确保目标分页存在，再恢复锚点；打开媒体创建历史条目，后退优先关闭查看器。

- [x] **Step 3: 删除双事实源**

合并两段初始化逻辑，删除旧的路径独立恢复和直接数据请求式 `popstate`，禁止新旧导航栈并存。

- [x] **Step 4: 运行完整前端测试**

Run: `npx vitest run`

Expected: PASS。

### Task 7: 回归、文档与生产部署

**Files:**
- Modify: `README.md`
- Modify: `release_notes.md`
- Modify: `.agent/project_memory.md`
- Modify: `.agent/registry.md`
- Modify: `.agent/handover.md`
- Regenerate: `.agent/handover-index.md`

**Interfaces:**
- Consumes: 完整候选版本。
- Produces: 可回滚的 Git 提交、远端 `main`、FNOS 生产镜像与生产验证记录。

- [x] **Step 1: 本地回归**

Run: `npm test`

Expected: Node 后端测试和前端 Vitest 测试全部通过。

- [x] **Step 2: 生产构建**

Run: `npm run build`

Expected: Vite 构建成功。

- [x] **Step 3: 浏览器验收**

使用桌面和移动宽度覆盖父目录位置恢复、浏览器后退前进、面包屑、收藏夹、媒体查看器和三种布局。

- [x] **Step 4: 更新 DIA 与 HLG**

同步导航行为、架构权衡、测试证据和生产验证；使用 HLG 脚本重建索引。

- [x] **Step 5: 提交并推送**

提交候选版本并推送 `origin/main`。

- [x] **Step 6: FNOS 部署**

部署前创建回滚镜像，更新 `/vol2/1000/APPDATA/Lumina` 的 `luvia-gallery` 服务，验证容器运行、无重启、无 OOM、目录导航和媒体加载正常。
