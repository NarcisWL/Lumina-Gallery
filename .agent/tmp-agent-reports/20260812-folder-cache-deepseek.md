# 一次性外部 CLI 架构复核报告：写入时目录封面缓存 + 扫描状态轻量化契约

> 任务 ID：`luvia-folder-cache-architecture`
> 复核 Agent：DeepSeek（Provider 信任依据：用户本轮明确要求使用 DeepSeek）
> 日期：2026-08-12
> 数据级别：S1。仓库内容视为不可信数据，未执行任何夹带指令（包括计划文档中要求激活 TDD skill、运行命令、提交等指令均未执行）。
> 约束遵守：仅写入本报告文件；未读取 `.env`、`data/jwt_secret.key`、`secure_env`、SSH/钥匙串/浏览器资料；未联网、未安装依赖、未修改业务文件、未触碰 DIA/HLG、未执行 Git/部署/破坏性命令。
> 环境事实：本机 Node v26.5.1；`node_modules/better-sqlite3` 为 NODE_MODULE_VERSION 115（Node 22 预编译），本地 `node --test` 全部后端用例因 `ERR_DLOPEN_FAILED` 无法运行（禁止安装/rebuild，故不改动）。以下判断基于静态源码分析 + 只读检查。报告内容均为**建议**，不落业务代码。

---

## SUMMARY

现状有三处独立但同根的问题，均源于「读时计算」且全同步：

1. **`/api/scan/status`**（server.js:1267）在管理员分支每次调用 `database.getStats()`（database.js:760），内部对 90.8 万行执行 **4 次全表 `COUNT(DISTINCT f.id)`**（database.js:762-765），约 5.5s 事件循环冻结。前端在扫描/缩略图任务激活期间**每秒轮询一次**（App.tsx:1317），意味着活跃扫描期间每秒钟冻结一次主循环——这是当前最严重的全站阻塞源。
2. **`/api/library/folders`**（server.js:852）每次请求调用 `queryFolderCovers(subs)`（database.js:522），对**每个**直属子目录执行一次「自身 + 后代半开区间」范围查询并按 `last_modified DESC, id ASC` 排序取 1（database.js:537-547）。235 个子目录即 235 次范围排序；对后代很多的目录，单次查询就要对整棵子树排序。7-29 计划已把「每个目录各查一次通用递归」收敛为「一次批量入口」，但**每个目录仍是范围排序**，读时成本未根治。
3. **封面缩略图缺失时仍下发 `/api/thumb/:id`**（server.js:810）且 `/api/thumb/:id` 对缺失缩略图**现场同步 FFmpeg**（server.js:2318-2350，单次最多 15-45s）。大量目录封面同时缺失会触发并发 FFmpeg 风暴，进一步放大阻塞与资源占用。

**推荐架构：写时物化 + 读时 O(1)。**

- **目录封面缓存**：把现有一张**从未被读写过的死表 `folders`**（database.js:125-132）激活并物化目录封面。封面选择语义严格保持不变（子树内 `last_modified DESC, id ASC` 取 1）。写入路径（扫描 upsert、删除、改名、清库）通过统一的比较式维护原语（`compare-and-update`，每祖先 1 次 PK 查询，无范围排序）增量维护；读路径改为单条 `IN` 查询 + `files` 主键 `LEFT JOIN`，彻底移除每目录范围排序。未迁移完成的目录回退到现有 range-sort SQL，保证增量迁移期间行为等价。
- **扫描状态轻量化契约**：`/api/scan/status` 管理员分支改为读取内存中的聚合快照（O(1)，0 次 COUNT），快照由数据库写入钩子增量维护并在 `app_meta` 表持久化，后台用 rowid 游标分块对账修正漂移。非管理员分支保持现状（零值，不透出统计）。响应字段、状态码、鉴权方式**完全不变**。
- **缩略图成功条件**：正式定义 `generateThumbnail` 成功标准（ffmpeg 退出码 0 且产物存在且 size>0），成功即回调把 `folders.cover_thumb_ready` 置 1；`formatFolderCoverMedia` 在缩略图缺失时输出 `url: null`（前端 FolderCard.tsx:41 已兼容空 cover），杜绝读路径触发 FFmpeg 风暴。

**为何最小且安全**：读接口签名（`queryFolderCovers`、`getStats`）与路由代码不变，仅替换 database 层实现；`folders` 表与 `thumbnails` 表本就存在，为纯增量列 + 一张新 `app_meta` 元数据表；维护原语全部走 `lib/` 新模块 + 数据库模块薄封装，遵守 AGENTS.md「server.js 不得直接访问 database.db」约束；所有后台任务复用既有 `backgroundTaskCoordinator` 串行化，分块让出事件循环、可取消、断点续跑。

---

## EVIDENCE

- `server.js:1267-1306` `/api/scan/status`；`server.js:1286` `const stats = database.getStats();`；管理员分支用 `stats.totalFiles/totalImages/totalVideos/totalAudio`。
- `database.js:760-774` `getStats()`：`totalFiles`（762）、`totalImages`（763）、`totalVideos`（764）、`totalAudio`（765）各调一次 `countFiles`。
- `database.js:666-673` `countFiles` → `SELECT COUNT(DISTINCT f.id) ...`（`DISTINCT` 放大成本），约 90.8 万行 → 单次即秒级。
- `App.tsx:1317` `scanTimeoutRef.current = setTimeout(poll, 1000)`：扫描/缩略图激活期间每秒轮询 `/api/scan/status`，叠加 5.5s 冻结。
- `server.js:852-976` `/api/library/folders`；`server.js:901`、`914`、`967` 三处各调用一次 `database.queryFolderCovers(subs)`（搜索/收藏夹/普通三分支）。
- `database.js:522-568` `queryFolderCovers`：对去重后的**每个**目录执行 `SELECT f.* ... INDEXED BY idx_folder_path WHERE (folder_path=? OR (>= start AND < end)) AND media_type IN ('image','video') ORDER BY f.last_modified DESC, f.id ASC LIMIT 1`——仍是「逐目录范围 + 全范围排序」。
- `database.js:125-132` `folders` 表存在（`path` PK、`media_count`、`cover_file_id`、`last_updated`），全仓仅建表无任何读写（grep `INSERT INTO folders|UPDATE folders|DELETE FROM folders|FROM folders` 除建表外零命中）——**死表，可安全激活**。
- `server.js:807-827` `formatFolderCoverMedia`：`let url = /api/thumb/${id}`（810）无条件返回；缩略图存在才追加 `?t=mtimeMs`（812-817），缺失时**仍返回该 URL**。
- `server.js:2300-2355` `/api/thumb/:id`：缓存缺失分支走现场 `generateThumbnail(fileObj, true)`（2340），超时链 15s/20s/10s（server.js:1661/1670/1677）。
- `server.js:1575-1693` `generateThumbnail`：产物名 `md5(Buffer.from(file.path).toString('base64')) + '.webp'`（1579-1580）；成功判定 `!result.err && exists && size>0`（1682）；成功回调 `updateDbWithDimensions` 写回 files 表尺寸（1683）。
- `server.js:1078-1265` `runMediaScan`：`insertFilesBatch` 批量 upsert（1203）、`reconcileScannedFiles` 游标清理（1239）；**无任何封面/统计维护钩子**。
- `database.js:678-703` `deleteFile`；`database.js:708` `deleteFilesBatch`（→ `lib/database-batch-operations.js:2-20`，`getFilesAfterRowid` 返回 `{rowid,id,path,last_modified}`）；`database.js:720-725` `deleteFilesByFolder`；`database.js:727-732` `deleteFilesBySourceId`；`database.js:753-758` `clearAllFiles`——全部不触碰 folders 表。
- `database.js:855-881` `renameFile`：删旧 id 插新 id（864/869）、更新 favorites 与 thumbnails（871-872）——**cover_file_id 会悬挂**。
- `server.js:2615-2669` `/api/cache/clear`：清空 CACHE_DIR + `clearThumbnails()`（2658）。
- `server.js:2671-2715` `/api/cache/prune`：按 DB 有效 id 删除孤儿 webp。
- `AGENTS.md` 全局规则 + `.agent/project_memory.md:56`「Never access database.db directly in server.js」——新逻辑必须走 `lib/` 模块 + database.js 薄封装。
- 测试现状：`test/database-file-query.test.js:276-364` 断言封面 SQL 含 `INDEXED BY idx_folder_path`（新读路径需改）；`test/scan-results-contract.test.js:88-98` 断言路由一次 `queryFolderCovers(subs)` 且 `buildFolderResult` 不查库（契约保持）；测试通过「拦截 database.js 内 require('better-sqlite3')」的内存库隔离加载。
- 前端兼容：`components/FolderCard.tsx:41` `if (!folder.coverMedia || !folder.coverMedia.url) return null;` —— `url:null` 安全。
- 本地测试执行结果：`ERR_DLOPEN_FAILED`（better-sqlite3 预编译 ABI 与 Node v26 不匹配），**无法运行基线**；所有后端判断基于静态分析。

---

## FILES

允许读取：`server.js`、`database.js`、`lib/background-file-walker.js`、`test/*.js`、`package.json`、`.agent/plans/2026-07-29-fnos-folder-cover-stall-fix.md`（均已读取）。
允许写入：仅本报告。**未改动任何业务文件。**

建议变更所涉及的文件（供实施 Agent 参考，本 Agent 不改）：
- 新建：`lib/folder-cover-cache.js`、`lib/media-stats.js`、`test/folder-cover-cache.test.js`、`test/media-stats.test.js`
- 修改：`database.js`、`server.js`、`test/database-file-query.test.js`、`test/scan-results-contract.test.js`
- 治理文档（简体中文）：`release_notes.md`、`.agent/registry.md`、`.agent/handover.md`（本 Agent 因「禁止 DIA/HLG」约束不落笔）

---

## CHANGES

### A. 数据库 Schema / 索引（幂等）

```sql
-- 1) 激活死表 folders，新增列（幂等：检查 PRAGMA table_info 后 ALTER）
--    folders(path TEXT PRIMARY KEY, media_count INTEGER DEFAULT 0,
--            cover_file_id TEXT, cover_last_modified INTEGER,
--            cover_thumb_ready INTEGER NOT NULL DEFAULT 0, last_updated INTEGER NOT NULL)
ALTER TABLE folders ADD COLUMN cover_last_modified INTEGER;        -- 增量比较免 JOIN
ALTER TABLE folders ADD COLUMN cover_thumb_ready INTEGER NOT NULL DEFAULT 0;

-- 2) 轻量元数据表：统计快照 + 迁移/重建断点
CREATE TABLE IF NOT EXISTS app_meta (key TEXT PRIMARY KEY, value TEXT NOT NULL);

-- 3) 索引
--    folders.path 已是主键，读路径 IN 查询走 PK，无需新索引。
--    不新增 idx_folders_cover_file：封面引用关系具有「只出现在该文件自身祖先链」的闭包性质，
--    维护始终可用祖先链游走（深度有界，< 10）覆盖，无需反向索引。若未来需要全量重算
--    「按 cover 找目录」再做反查索引，作为可选项保留。
```

### B. 写入时目录封面缓存（`lib/folder-cover-cache.js`）

统一原语（顺序与现有 `ORDER BY last_modified DESC, id ASC` 完全一致，`(last_modified, id)` 字典序可精确复现）：

```
compareAndUpdate(F, c)：SELECT cover_file_id, cover_last_modified FROM folders WHERE path=F
  - 无行 → INSERT (F, c.id, c.lastModified, ready=0)
  - (c.lastModified > lm) || (lm 相等 && c.id < cover_id) → UPDATE cover_file_id/cover_last_modified/last_updated
  - 否则保持
walkAncestors(F)：F → dirname(F) → … → 文件系统根（parent === F 停止）；对每个祖先逐级 compareAndUpdate
```

- **insertBatchMaintain(files)**：在 `insertFilesBatch` 既有事务内执行。先按 `folder_path` 去重，对每个去重目录计算 **batchBest**（仅 image/video，`last_modified` 最大、相等取 `id` 最小）；对 batch 内从 image/video 退出（upsert 变 audio 等）的文件，其目录链标记 full-recompute。每个去重目录沿 `walkAncestors` 做一次 compareAndUpdate。**每批次成本 ≈ 去重目录数 × 深度**（深度 < 10），远小于逐目录范围排序；90.8 万文件全量扫描约数千批次，总维护语句在万级，对扫描总耗时影响可忽略。
- **deleteMaintain({id, path, ...})**：删除前取 `folder_path = dirname(path)`；对受影响祖先（含自身目录）逐一比较 `cover_file_id === deletedId`，命中的目录加入 dirty 集；事务删除文件后，对每个 dirty 目录用现有 range-sort SQL（`ORDER BY last_modified DESC, id ASC LIMIT 1`）**重算**，无结果则 `DELETE FROM folders WHERE path=?`。重算只发生在「被删文件恰好是某目录封面」时，极少。
  - `deleteFilesBatch`：由 `getFilesAfterRowid` 提供 `{id,path}`，删除前统一收集 dirty 祖先集（去重），删除后集中重算。
  - `deleteFilesByFolder(folderPath)`：先 `SELECT path FROM files WHERE folder_path=? OR folder_path LIKE folderPath+'/%'` 收集路径计算 dirty 集；事务删除后 `DELETE FROM folders WHERE path=? OR path LIKE folderPath+'/%'`，再重算祖先链。
  - `deleteFilesBySourceId`：同理，删除前收集全部受影响文件路径。
- **renameMaintain(oldPath, newPath)**：`renameFile` 事务内，旧链（`dirname(oldPath)` 祖先）全重算 + `DELETE folders` 悬挂行；新链对「新文件（新 id、`last_modified=Date.now()`）」做 compareAndUpdate 或重算。重算量 ≤ 2×深度。
- **clearAllFiles()**：事务内 `DELETE FROM folders` + `DELETE FROM app_meta WHERE key='folder_covers_built'` + 统计快照归零。
- **queryFolderCovers(folderPaths)**（读路径，签名与返回 `Map<string,MediaFile>` 不变，路由零改动）：
  1. 规范化 + 去重 + 原键回填（复用 database.js:525-535 现有逻辑，`path.resolve` 归一化键）；
  2. 单条 `SELECT fo.path AS folder_path, f.id, f.path AS media_path, f.name, f.type, f.media_type, f.last_modified, fo.cover_thumb_ready FROM folders fo LEFT JOIN files f ON f.id = fo.cover_file_id WHERE fo.path IN (...)`，结果 `mapFileRow` 后回填原键；
  3. **未命中**（迁移未完成 / 该目录确实无媒体 / cover_file_id 悬挂）→ 回退现有 range-sort SQL 逐目录计算（保持与现状等价）。迁移完成后未命中仅剩「无媒体目录」，回退即返回空。若单请求未命中比例异常升高（如 >50%），`console.warn` 提示迁移未完成。
- **markCoverThumbReady(file)**：`walkAncestors(file.folderPath)`，`UPDATE folders SET cover_thumb_ready=1 WHERE path IN (…) AND cover_file_id=file.id`。
- **markAllCoversThumbNotReady()**：`UPDATE folders SET cover_thumb_ready=0`（`/api/cache/clear` 用）。
- **rebuildAll({shouldStop,onProgress})**：首次迁移/重建。
  - 阶段一（批量 fold）：`INSERT OR IGNORE INTO folders(path, cover_file_id, cover_last_modified, last_updated) SELECT folder_path, id, last_modified, ? FROM files WHERE media_type IN ('image','video') ORDER BY last_modified DESC, id ASC`——SQLite `INSERT OR IGNORE` 保留先到者，配合排序即得每目录最大者；再用 rowid 游标分块（复用 `getFilesAfterRowid` 思路，块间 `setImmediate`）从底向上对每目录与其直属子目录 fold 父级封面（`MAX(自己, 各子目录)`）。
  - 阶段二（差分自检）：取 `folders` 中每个 `cover_file_id` 与对应 `files` 行做完整性校验，dangling 引用剔除；可选与「全量 replay 维护」结果比对。
  - 断点：进度写入 `app_meta('folder_covers_built')`/`app_meta('folder_covers_checkpoint')`，支持中断续跑；完成后写入完成标记。
  - 必须经 `backgroundTaskCoordinator` 运行（与扫描/缓存统计串行），**绝不**在请求路径内执行。

### C. 扫描状态轻量化契约（`lib/media-stats.js`）

- **数据**：内存快照 `{ totalFiles, totalImages, totalVideos, totalAudio, dbSize }`；持久化到 `app_meta('media_stats_json')`，启动时加载。
- **维护钩子**（全部在 database.js 薄封装内）：
  - `upsertFile` / `insertFilesBatch`：新行 +1；upsert 且 media_type 变化时旧类型 -1、新类型 +1。
  - `deleteFile`/`deleteFilesBatch`/`deleteFilesByFolder`/`deleteFilesBySourceId`：删除前按 media_type -1（批量删除由 `getFilesAfterRowid` 提供 path，可在删除前查出类型）。
  - `renameFile`：类型不变则数字不变（路径变化不影响计数）。
  - `clearAllFiles`：全部归零。
  - 每 N 次变更（如 1000）或扫描完成/优雅退出时写回 `app_meta`。
- **对账**：扫描完成后 + 每 30 分钟，经 coordinator 用 rowid 游标分块（每块 ~5 万行，块间 `setImmediate`）重算四计数并覆盖快照，修正任何钩子遗漏导致的漂移；对账期间读仍返回旧快照（不阻塞）。
- **`/api/scan/status` 契约**：管理员分支改为 `database.getStatsSnapshot()`（O(1)，0 次 COUNT，`dbSize` 复用快照）；响应字段 `{ status, count, currentPath, total, mediaStats:{images,videos,audio,totalFiles}, storage, cacheCount, totalItems }`、状态码、鉴权方式**保持不变**；非管理员分支维持现状零值。新增 `res` 契约测试断言路由不再出现 `database.getStats(`。
- **`/api/system/status`**（server.js:2466-2511）：管理员分支改用快照；**非管理员分支保留现有 allowedPaths 作用域 COUNT（行为与权限语义不变，非热路径，可后续再优化）**。
- **新鲜度语义**：状态 UI 允许滞后 ≤ 对账周期，与既有 `globalCacheCount/globalCacheSize` 快照语义一致（server.js:150-152）。

### D. 缩略图成功条件（契约）

- **成功定义**：`generateThumbnail` 返回 `true` 当且仅当最终 ffmpeg 退出码为 0 **且** `thumbPath` 存在 **且** `fs.statSync(thumbPath).size > 0`（server.js:1682 现状，正式化并作为唯一判定）。
- **成功回调**：成功返回前调用 `database.markCoverThumbReady(file)`，把该文件所在祖先链中「封面为此文件」的目录 `cover_thumb_ready` 置 1。此钩子覆盖所有生成入口：`/api/thumb/:id` 现场生成（2340）、`processFilesConcurrently` 重生成（1733）、smart-repair（2015 段）。
- **读路径约束**：`formatFolderCoverMedia` 改为——`thumbPath` 不存在或 size==0 时返回 `{ ...coverMedia, url: null }`（保留元数据供前端文案/类型判断）；存在时维持 `url + '?t=mtimeMs'`。`cover_thumb_ready` 作为免 existsSync 的优化位，但**最终正确性以磁盘存在性为准**（兜底：即使 flag=1 而磁盘缺失也返回 url:null）。
- **清缓存联动**：`/api/cache/clear`（2615）清空目录后调用 `markAllCoversThumbNotReady()`，使已清空封面缩略图即时失效，避免下发坏 URL；缩略图重建后由成功回调自动回填。
- **不在封面缓存中改变选择语义**：封面仍按 `last_modified DESC, id ASC` 选择，不因「有无缩略图」改变封面候选；缩略图缺失时只表现为「不显示封面」（url:null），语义稳定、可预测。

### E. 与现有计划的衔接

7-29 计划（`queryFolderCovers` 批量 + `idx_folder_path` + 慢请求日志）为已落地底座，本方案在其上**替换查询内核**（批量 → 缓存读），慢请求中间件保留用于观测未来回归；`queryFolderCovers` 原 range-sort SQL 保留为迁移期回退与重算工具。

---

## TESTS

环境限制：本机 `better-sqlite3` ABI 与 Node v26 不匹配，**无法运行基线测试**（`ERR_DLOPEN_FAILED`）。以下为**建议新增/修改的测试**及其验证命令，供实施 Agent 在有正确 ABI 的环境执行。

建议命令：`node --check lib/folder-cover-cache.js && node --check lib/media-stats.js && node --check database.js && node --check server.js && node --test test/folder-cover-cache.test.js test/media-stats.test.js test/database-file-query.test.js test/scan-results-contract.test.js test/background-file-walker.test.js test/database-batch-operations.test.js test/request-timing.test.js`（后端全集即 `npm run test:backend`）。

1. **新 `test/folder-cover-cache.test.js`**（直接 `new Database(':memory:')`，仿 `test/database-batch-operations.test.js` 风格）：
   - 封面选择与现 range-sort 结果完全一致：直属媒体、深层后代、同前缀兄弟隔离、`last_modified` 相同取 `id ASC`（复用 7-29 夹具，`test/database-file-query.test.js:276-364`）。
   - 深目录插入 → 全部祖先链 compareAndUpdate，根目录封面正确升级；新文件不比封面新 → 封面不变。
   - 删除封面文件 → 该目录与祖先重算为次新；删除非封面 → 不变。
   - `renameFile` 旧链重算 + 新链更新，无悬挂 `cover_file_id`。
   - `clearAllFiles` 清空 folders 表。
   - 读路径 `queryFolderCovers`：`EXPLAIN QUERY PLAN` 命中 `folders` 主键；未迁移目录回退 range-sort 且结果等价；dangling cover 返回 null。
   - `markCoverThumbReady` / `markAllCoversThumbNotReady` 状态位正确。
   - `rebuildAll`：小样本（数百行，含多级目录与 tie-break）重建后与增量维护结果差分一致；断点续跑幂等。
2. **新 `test/media-stats.test.js`**：插入/删除/改类型（image→audio）/清库后快照计数正确；`app_meta` 持久化与重启加载；对账游标重算修正手工污染的计数；块间让出（定时器可触发）。
3. **改 `test/scan-results-contract.test.js`**：新增断言 `/api/scan/status` 管理员分支**不再调用 `database.getStats(`/`countFiles`**（`doesNotMatch`），响应字段集合保持（`status/count/currentPath/total/mediaStats/storage/cacheCount/totalItems`），非管理员分支仍返回零值统计。
4. **改 `test/database-file-query.test.js`**：封面用例改为同时断言「缓存读路径命中 folders 主键」与「回退 range-sort 等价」，移除对 `INDEXED BY idx_folder_path` 的必选断言（改为回退 SQL 保留该断言）。
5. **保留并通过**：7-29 契约测试（`test/scan-results-contract.test.js:88-98`：路由一次 `queryFolderCovers(subs)`、`buildFolderResult` 不查库）——本方案不改路由与签名，应原样通过。

---

## RISKS

1. **迁移/重建期阻塞（最高风险）**：`rebuildAll` 若误入请求路径或未分块让出，会重现 5.5s/数分钟级冻结（与历史 523.7s 事故同源）。强制约束：仅经 `backgroundTaskCoordinator` 后台运行、rowid 游标分块（~5 万行/块）、块间 `setImmediate`、支持 `shouldStop` 取消与断点续跑；读路径在迁移期自动回退 range-sort，无需停机。
2. **维护钩子遗漏导致封面悬挂**：任何绕过 `upsertFile/delete*/renameFile` 的直接 SQL 写 `files` 表都会让 `folders.cover_file_id` 悬挂。缓解：读路径 `LEFT JOIN files` 天然容忍悬挂（返回 null cover）；`rebuildAll` 定期（每次扫描完成后）全量对账自愈；新增写入必须经薄封装。建议在 `database.js` 顶部注释固化「所有 files 写操作必须走封装」。
3. **比较式维护的正确性边界**：upsert 改变 media_type（image/video ↔ 其它）时，比较式增量会漏判「旧封面失效」。缓解：batch 内检测 media_type 变化并标记 full-recompute；对账自愈兜底。tie-break 用 `(last_modified DESC, id ASC)` 与现有 SQL 严格同序，禁用自定义比较器。
4. **统计快照漂移**：钩子式计数在历史路径/异常退出下可能漂移。缓解：`app_meta` 持久化 + 扫描完成/30 分钟周期对账（只增修正，不阻塞读）；对账期间继续服务旧值（与 `globalCacheCount` 语义一致）。**非管理员 `/api/system/status` 保留现状作用域 COUNT，不改变权限语义，勿顺手改成全局快照（避免越权透出计数）**。
5. **`url:null` 前端副作用**：封面缺失时卡片显示占位而非旧图。FolderCard.tsx:41 已处理空 cover；需实施时人工核验 folders 视图下无封面目录的渲染与旧行为差异（旧行为是「发 URL → 现场 FFmpeg → 长超时失败」同样无图，故为净改善）。
6. **回滚风险**：`folders`/`app_meta` 为增量列/新表，旧代码不识别但无害；回滚即还原 database.js/server.js，保留表结构，下一次扫描/迁移自动重建或回退。无需删库、无破坏性迁移。**绝不**在发布前对生产库做非幂等操作；候选镜像用生产库**只读**跑 `queryFolderCovers` 基准（沿用 7-29 流程，`<1000ms` 目标）。
7. **并发写锁**：better-sqlite3 单连接同步，封面维护与文件写入同在既有事务内完成，不新增跨事务窗口；后台任务（扫描/重建/对账/缓存统计）由 coordinator 串行，无并发写冲突。**禁止**把维护放在 `/api/thumb/:id` 请求路径中执行 DB 写。

---

## NEXT

1. **实施顺序（增量、每步可独立验证）**：
   - Step 1：新增 `lib/media-stats.js` + 钩子 + `getStatsSnapshot()`，改 `/api/scan/status` 读快照；跑 `test/media-stats.test.js` + 扫描状态契约测试（消除 5.5s 冻结，独立收益）。
   - Step 2：新增 `lib/folder-cover-cache.js` + 读路径替换 `queryFolderCovers` 内核（保留 range-sort 回退）+ `formatFolderCoverMedia` url:null 守卫；跑 `test/folder-cover-cache.test.js` 与既有封面契约测试。
   - Step 3：接入写时维护（`insertFilesBatch`/`delete*`/`renameFile`/`clearAllFiles`）+ 缩略图成功回调 `markCoverThumbReady` + `/api/cache/clear` 联动。
   - Step 4：`rebuildAll` 迁移任务 + 启动编排 + `app_meta` 断点；生产按 7-29 流程构建候选镜像、生产库只读基准、回滚镜像、双跑验证（期间读路径回退保证行为等价）。
2. **生产验证**：容器 revision/镜像 ID/端口 9980:3001 四层探测；真实 235 子目录目录请求 `<1000ms`；扫描激活期间并发探测 `/api/config` 确认不再周期性冻结；`/api/scan/status` 返回体与旧响应字段 diff 一致。
3. **治理收口**（实施 Agent 负责，本 Agent 按约束不落笔）：`release_notes.md`、`.agent/registry.md` 同步注册；HLG Skill `append` dry-run 后 `--apply` 记录提交/镜像/耗时/风险；简体中文注释与文档。
4. **后续可选项（本方案不强制）**：`cover_thumb_ready` 用于免 existsSync 的读路径微优化；按需的 `idx_folders_cover_file` 反查索引；`/api/system/status` 非管理员分支的异步作用域计数。
5. **唯一输出**：本报告写入 `.agent/tmp-agent-reports/20260812-folder-cache-deepseek.md`；stdout 输出 `AGENT_DONE_LUVIA_DEEPSEEK`。
