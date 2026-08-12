# 外部 DeepSeek 最终独立审阅报告

> 范围：仅当前工作树中与「WebUI 大媒体库性能与加载体验」相关的未提交 diff。
> 依据：最终复审时的 `git status` / `git diff` 以及相关源码与测试源码的只读核验；DIA/HLG 收口产生的后续文档增量由主控单独复核。
> 主控已在 Node 20 跑通后端 54/54、前端 157/157 与 Vite 生产构建，本审阅不重复执行测试，仅面向测试未覆盖的运行时缺陷与契约一致性做静态核验。
> 审阅时间：2026-08-13。对照基线：HEAD `d36d66c`。
> 复审更新（2026-08-13）：原 P2-2「首包与骨架渲染并发重取竞争窗口」已按 P1 修复并经本轮只读复核注销；当前未处理 P0=0、P1=0，残余 P2=5 项。详见「复审结论」一节。

## 结论

未发现本轮新增或放大的 **P0**（0 个），未发现需要立即修复的 **P1**（0 个）。

计划书中的五项核心机制在代码与契约测试中均一致成立，参数绑定顺序、索引与 ORDER BY 键序、迁移事务语义、权限指纹缓存失效、首载与失败恢复路径均核验通过。存在若干 **P2** 级回归/退化与已声明的残余风险，详见下文。其中「深滚动逐页追赶」放大（500→120）与「运行期删除-重入库的历史 path 收藏不可见」是相对 HEAD 的真实行为收窄，但前者是计划白纸黑字接受的 offset 深页风险，后者被计划显式声明为「后续专项处理」，均不构成阻断缺陷。

## 复审结论（2026-08-13，P2-2 修复注销）

原 P2-2 整改后，分页并发竞态已从「加载路径两侧」闭合，核心签名与各场景逐一复核通过：

- **门禁签名**：`canLoadNextGalleryPage`（`App.tsx:225-252`）要求 `serverOffset > 0` 且 `readyDatasetIdentity === currentDatasetIdentity` 且 `!isInitialLoading` 且 `!isInitialSkeletonCovering` 且 `!isFetching`。
- **首包锁路径**：数据集 effect 进入非缓存分支时先将 `galleryReadyDatasetIdentity` 置 `''`（`App.tsx:1478`）、`serverOffset` 归零（`App.tsx:1515`），首包须 `files+folders` 双双成功后经 `resolveReadyGalleryDatasetIdentity` 才写入就绪身份（`App.tsx:1533-1540`）；任一侧失败则不在 `results.some(false)` 分支写入身份并置 `galleryLoadError`，`canLoadNextGalleryPage` 双因子（offset=0 与身份空/不匹配）均拒绝续页。
- **缓存命中路径**：命中即恢复 offset/total/hasMore 并同步写入就绪身份（`App.tsx:1480-1492`），续页可直接恢复预取；`cacheCurrentGallery` 已被 `shouldCacheCurrentGallery`（禁 initial/skeleton/error 期写入，`App.tsx:1449`）约束，不会以「offset 0 + hasMore」的半成品状态落缓存。
- **数据集切换旧 offset**：`handleGalleryLocationChange`/`handleSetViewMode`/`handleFolderClick` 统一走 `advanceGalleryDatasetNavigation`（身份不变则只缓存不推进纪元）；切换瞬间旧 `serverOffset` 与旧就绪身份仍在门禁内被「身份不匹配」拦截（`readyDatasetIdentity` 与基于新 location 的 `currentDatasetIdentity` 不等），effect 随后归零 offset 并重置身份，双保险成立。
- **Grid/Masonry 初始守卫与初始结束恢复预取**：Grid `onItemsRendered` 追加 `!isInitialLoading` 短路（`GridViewport.tsx:249`），Masonry `requestNextPage` 以 `loadState.isInitialLoading` 短路且 IO 观察器依赖 `isInitialLoading` 重建（`MasonryViewport.tsx:345-375`），首载结束（initial 置 false、身份就绪、offset>0）后自动恢复近底预取；`loadMoreServerFiles` 内部仍持 `endReachedLockRef`（`App.tsx:1572-1574`）互斥续页。
- **测试契约**：`canLoadNextGalleryPage` 各拒绝分支、`resolveReadyGalleryDatasetIdentity([true,false])=>''`、favorites 等待期 initial 门禁、数据集切换旧 offset 拒绝、Grid/Masonry 首载不触发与恢复预取均已由 `test/app-navigation-flow.test.tsx`（566-616、650-679）与 `test/viewport-anchor.test.ts`（598-627、676-718）锁定。

当前未处理 P0=0，未处理 P1=0；残余 P2=5（P2-1、P2-3、P2-4、P2-5、P2-6）。

## P0

无（0 个）。

## P1

无（0 个）。

## P2（本轮相关，非阻断；复核后残余 5 项）

> 原 P2-2 已于 2026-08-13 按 P1 修复并注销，见「复审结论」。

| # | 问题 | 位置 | 说明与影响 | 参考 |
| --- | :-: | --- | --- | --- |
| P2-1 | 深滚动/深媒体位恢复需按 120 条/页顺序追赶，往返次数约放大 4 倍 | `components/gallery/GridViewport.tsx:244-268`、`App.tsx:1264` | HEAD 分页批量 500 条，本轮收敛为 120 条；GridViewport 的 `onItemsRendered nearEnd` 阈值基于 `items.length`，用户把滚动条拖到深位置或恢复深媒体位时，需连续顺序请求直至 `items.length` 追上可见索引（百万库下可上千次顺序往返）。恢复期间可见区域长时间为骨架。属于计划「深页 offset 仍随偏移增长」已接受风险，但相对 HEAD 的体验放大是真实的本轮退化。 | 计划「风险与后续」第 2 条 |
| P2-3 | 运行期「删除文件-重新入库」后，旧 path 收藏在下次重启前不可见 | `database.js:521-527`、`database.js:1122-1154` | 收藏列表改为 ID-only 连接，历史 path 收藏仅在**启动时**迁移为 ID；若某文件在运行期被按 path 删除（`deleteFile` 无 id 分支不清理 favorites）后重新入库，其遗留 path 收藏在下次启动迁移前不参与收藏列表/计数。HEAD 的 `OR fav.item_id = f.path` 可覆盖该运行期场景。计划已显式声明为后续专项；重启后自动恢复。 | 计划「风险与后续」第 3 条 |
| P2-4 | admin 默认全库 `totalExact` 依赖 `library_stats` 缓存行 | `server.js:1459-1479`、`database.js:953-965`；触发线 `database.js:242-266` | 若老库升级后计数与 `files` 表失配（低概率），管理员精确 total 可能偏差；极端情况（缓存为 0 而文件存在）会导致默认全库网格 `serverTotal=0` 而出现空白。`library_stats` 由插入/删除/类型触发器增量维护，并在启动用 `COUNT(*)` 兜底 seed，多数路径一致；残余为升级一致性，未在真实百万库上验证。 | 计划「风险与后续」第 1 条 |
| P2-5 | 受限账号侧边栏总数退化与系统状态显示未知 | `App.tsx:1348`、`App.tsx:2815`、`components/settings/SystemTab.tsx:15-17` | 非管理员默认全库请求 `totalExact=false`，`setLibraryTotalCount` 不再更新，侧边栏 `totalPhotos={libraryTotalCount || files.length}` 退化为「已加载条数」（首个 120），不再显示 scoped 精确总数；系统状态页以「—」占位（明确设计选择）。属有意识权衡，但侧边栏计数的展示是相对 HEAD 的可见退化。 | 计划「受限账号系统状态」条款 |
| P2-6 | 启动建组合索引一次性阻塞成本 | `database.js:211-218` | `ensurePerformanceCaches` 在已有上百万行的库上首次启动时创建 7 个索引，SQLite 同步执行期间阻塞服务请求。计划已声明；非缺陷。 | 计划「风险与后续」第 1 条 |

## 已核验范围（对应任务指定五项）

### 1. 分页 totalExact / hasMore — 通过
- `database.js:649-658` `queryFilesPage` 以 `limit+1` 判 `hasMore`，`slice(0, limit)` 返回页数据；`queryFiles` 尾部 `LIMIT ? OFFSET ?` 参数拼接顺序与投影/WHERE 占位符严格一致（`database.js:622-641`，投影 EXISTS 的 `?` 在前、join/where 参数居中、limit/offset 殿后）。
- `server.js:91-99` `resolveScanResultsPageMetadata`：`totalExact=false` 时 `total = offset + 返回数 + (hasMore?1:0)`（已知下界 + 哨兵），末页升级为精确；`totalExact = hasExactTotal || !hasMore`。
- `server.js:79-86` + `1459-1479`：仅**管理员+无筛选+全库**走 `getCachedStats().totalFiles` 精确值，其余作用域不阻塞精确计数；hashMore 始终来自 limit+1（不依赖计数精度），因此分页正确性不受统计精度影响。契约测试 `test/scan-results-contract.test.js`（limit+1、sentinel 语义、shouldUseCachedLibraryTotal 白名单等）已同步覆盖。
- 边界：正则校验 offset/limit（非法返回 400，`server.js:62-77`）；`limit` 上限 500、下限 1，超界拒绝。

### 2. 收藏 ID-only 与迁移 — 通过
- 列表/计数连接改为 `favorite_filter.item_id = f.id` 且仅 `INNER JOIN`（`database.js:521-527`），投影 `CASE WHEN EXISTS(...)` 逐行判 is_fav（`database.js:608-621`），无 `OR path`、无 `SELECT DISTINCT`。
- `toggleFavorite` 严格按 ID（base64 of path）存取；`/api/favorites/ids` 返回 item_id 集合（`database.js:1168-1177`）；`deleteFile`/`deleteFilesBatch` 均按 ID 清理 favorites（`database.js:1004-1006`）。
- `migrateFavorites`（`database.js:1122-1154`）事务内先删除「同一用户同一媒体的 path 行（存在 ID 行时）」再统一 UPDATE 映射；`files.path` 有唯一索引，JOIN/子查询走索引；迁移可重复执行（测试断言两次调用幂等）。契约测试覆盖冲突去重、幂等、ID-only SQL 断言。

### 3. sizeDesc 组合索引 — 通过
- ORDER BY `f.size DESC, f.id ASC`（`database.js:632`）与索引 `(size DESC, id ASC)` 键序严格匹配；媒体类型/文件夹组合索引左前缀可与查询列序对齐（`database.js:79-84`、`211-218`）。
- `EXPLAIN QUERY PLAN` 断言使用 `idx_files_media_size_desc_id` / `idx_files_folder_size_desc_id` 且无 `USE TEMP B-TREE`；其余 date/name 排序沿用既有索引与方向。索引在 `createSchema` 与 `ensurePerformanceCaches` 幂等创建，新旧库均覆盖。

### 4. 权限作用域缓存 — 通过
- react-query 缓存键改为包含 `filter`（`navigation/query-key.ts:40-51`），并绑定 `currentGalleryUserScope` 指纹（`App.tsx:1384-1393`）；作用域指纹 = 用户名 + 角色 + 权限路径排序哈希（`App.tsx:61-78`），`/api/config` 现向本人返回 `allowedPaths`（`server.js:791-800`），登录即注入 `User.allowedPaths`。
- 首页持久缓存升级 v2：`readGalleryHomeCache` 强校验版本 + 重算指纹（`App.tsx:96-121`），指纹不匹配即忽略；登出清除 `CACHE_HOME_KEY`（`App.tsx:885`）；`galleryFiles` 查询在指纹变化时整批移除（`App.tsx:1425-1427`），`libraryTotalCount` 按作用域归零/重置（`App.tsx:691-699`）。权限变化必然触发缓存 miss（保守方向）。

### 5. 首载骨架与分页失败重试 — 通过
- 网格：初始骨架约覆盖两个视口（`GridViewport.tsx:11-15`、`126-128`），续页补 `+2 列` 骨架（`getGridEffectiveItemCount`）；瀑布流：初始骨架按 1.02 平均高估算两视口（`MasonryViewport.tsx:11-23`），续页按列补 3 个（`LOAD_MORE_SKELETONS_PER_COLUMN=3`）。
- 卡片几何：优先服务端 `aspectRatio/width/height`，缺失按媒体 ID 稳定哈希回退，且加载完成后不再改 aspect（`PhotoCard.tsx:44-69`、`MediaCard` style aspectRatio）；首行两排 `imagePriority=true` 走 eager/high、其余 lazy（`GridViewport.tsx:334`、`MasonryViewport.tsx` imagePriority）。
- 预取：瀑布流 `IntersectionObserver` 底部 rootMargin 1.5 视口 + `requestAnimationFrame` 滚动节流（`MasonryViewport.tsx:139-160`、`336-405`），失败经 `loadStateRef` 面板锁恢复且滚动内仍会按阈值重试；网格基于 `onItemsRendered`，失败同样在锁释放后于下一次滚动画格重试。
- 首载骨架/旧内容/空态的覆盖逻辑（`shouldPreserveGalleryHydratedFiles`、`shouldCoverGalleryWithInitialSkeleton`、`shouldShowServerEmptyLibrary`、`shouldShowFavoritesEmptyState`）在 `App.tsx` 各渲染分支引用一致，详见 P2-2（已修复）/P2-5 的边角退化记录。

## 本轮回归 vs HEAD 既有风险

**本轮回归/行为收窄（相对 HEAD，均为 P2，不阻断）：**
- 首载骨架与分页 120 收敛带来的深滚动逐页追赶放大（P2-1）。
- 收藏 ID-only + 仅启动迁移导致的运行期「删除-重入库」path 收藏可见性收窄（P2-3），以及 `queryFavoriteFiles`/`countFavoriteFiles`（`database.js:1179-1211`）成为不再被 server 引用的死代码（HEAD 亦未在热路径使用，整体无害）。
- 受限账号侧边栏/系统状态精确计数退化（P2-5）。
- **原 P2-2 已修复**：首包并发重取窗口由「就绪数据集身份 + offset>0 + initial 门禁」三重闭合，复核注销。

**HEAD 既有风险（本轮未引入、未处理）：**
- `ORDER BY RANDOM()` 深页 offset 跳变与全库随机扫描；瀑布流动态窗口化缺失导致的持续滚动 DOM 增长；`server.js:2021` 的 `database.countFiles({recursive:true})` 仍为无过滤全表 COUNT；`server.js` 既有媒体调试日志记录完整 `req.url`（handover 已注明）。上述均属既有问题，不在本轮范围。

## 残余风险（审阅无法在只读环境下淘汰的项）

1. 未对接近百万行的真实/合成库离线执行 `EXPLAIN QUERY PLAN` 与启动建索引耗时、深页 offset 分位数实测（计划验收第 2 条依赖主控的其他通道）；本报告仅完成结构核验。
2. `library_stats` 仅在启动与触发器路径下保证与 `files` 一致，未在真实升级库上验证（P2-4）。
3. `MasonryViewport` 的 `isInitialLoading && items.length===0` 骨架分支与 `loadMoreSentinelRef` 在未知浏览器（无 IntersectionObserver）下无按钮兜底（旧版浏览器退化）。
4. 首页持久缓存仅绑定「用户名/角色/权限路径」指纹，未绑定 serverId/扫描代次；同一真机切换同源不同服务器后端时，启动帧可能短暂展示旧库首页数据（随后被后续请求覆盖）。

## 复审边界（2026-08-13 P2-2 修复复审）

- 只读核验当前工作树 `App.tsx`、`components/gallery/GridViewport.tsx`、`components/gallery/MasonryViewport.tsx`、`components/gallery/viewport-types.ts`、`navigation/query-key.ts` 相对 HEAD `d36d66c` 的增量，以及 `test/app-navigation-flow.test.tsx`、`test/viewport-anchor.test.ts` 对应契约用例。
- 重点复核：`galleryReadyDatasetIdentity`/`canLoadNextGalleryPage`、缓存命中续页、files+folders 双成功就绪、首包失败禁续页、数据集切换旧 offset 拒绝、Grid/Masonry initial 守卫与 initial 结束恢复预取。
- 未修改任何业务/测试/文档文件；未新增写入除本报告外的任何路径；未运行测试套件（主控已单独执行 Node20 后端 54/54、前端 157/157、Vite build）；未安装依赖、访问网络或生产；未提交/推送/发布。

## 审阅边界

- 只读 `git diff`、相关源码与测试；未修改任何业务/测试/文档文件；未运行测试套件（主控已执行）；未安装依赖、访问网络或生产；未提交/推送/部署。
- `git diff --check` 通过（无空白错误）。
- 本轮唯一写入：本报告文件及 P2-2 修复复核更新。
