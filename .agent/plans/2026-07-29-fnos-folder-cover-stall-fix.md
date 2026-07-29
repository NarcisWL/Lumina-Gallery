# FNOS 目录封面查询冻结修复实施计划

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:test-driven-development to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**目标：** 消除 `/api/library/folders` 为大量直属子目录逐个执行全表级递归封面查询造成的数分钟 Node.js 事件循环冻结，同时保持目录封面选择、权限、响应结构和排序语义不变。

**架构：** 在数据库层新增单入口批量封面查询 `queryFolderCovers(folderPaths)`，对目录路径去重后复用同一个 prepared statement，并强制使用 `idx_folder_path` 对“目录自身或真实后代范围”做索引查询。服务端一次取得全部封面映射，再构造目录响应；另加入可测试的慢请求计时中间件，确保未来阻塞能留下完成或断开耗时证据。

**技术栈：** Node.js CommonJS、Express、better-sqlite3、`node:test`、Docker、Docker Compose。

## 全局约束

- 不改变 `/api/library/folders` 的路径、参数、鉴权方式、状态码和 `{ folders }` 响应结构。
- 目录封面仍选择目录自身或任意深度后代中 `last_modified` 最新的 image/video；时间相同时按 `id ASC` 稳定选择。
- 路径范围必须排除同前缀兄弟，例如 `/album/a` 不得匹配 `/album/ab`，并按运行平台分隔符工作。
- 服务端每次目录请求只能调用一次 `database.queryFolderCovers(subs)`；禁止在 `subs.map(...)` 内调用 `database.queryFiles()`。
- 批量查询内部必须复用 prepared statement，并使用 `idx_folder_path` 的等值与后代半开区间；不得恢复递归文件系统封面扫描。
- 慢请求日志不得输出 Authorization、Cookie、查询字符串或响应正文；默认阈值为 1000ms。
- 不引入第三方运行时依赖，不修改生产端口、卷映射、GPU、内存限制或代理超时。
- 所有新增文档和注释使用简体中文。
- 生产发布必须先构建隔离候选镜像并旁路冒烟，再保留当前生产镜像为回滚标签，最后才由 Compose 强制重建。

---

### Task 1: 批量索引封面查询与路由接入

**Files:**
- Modify: `database.js`
- Modify: `server.js`
- Modify: `test/database-file-query.test.js`
- Modify: `test/scan-results-contract.test.js`

**Interfaces:**
- Produces: `database.queryFolderCovers(folderPaths)`，返回 `Map<string, MediaFile>`。
- Consumes: `buildFolderResult(folderPath, coverMedia, isFavorite)`，只格式化文件系统统计和已取得的封面记录。

- [ ] **Step 1: 写失败测试**

在 `test/database-file-query.test.js` 增加真实内存 SQLite 用例：

```js
test('批量目录封面只匹配自身或真实后代，并为每个目录选择最新媒体', () => {
    // 固定夹具覆盖：直属媒体、深层后代、同前缀兄弟、audio 排除、
    // last_modified 相同时 id ASC，以及重复输入路径。
    const covers = database.queryFolderCovers(['/album/a', '/album/b', '/album/a']);
    assert.equal(covers.get('/album/a').id, 'a-newest');
    assert.equal(covers.get('/album/b').id, 'b-stable-first');
    assert.equal(covers.has('/album/ab'), false);
});
```

同时记录真实 prepared SQL，并对该 SQL 运行 `EXPLAIN QUERY PLAN`，断言计划包含 `idx_folder_path`，从而捕获再次退化为 `idx_media_type + TEMP B-TREE` 全库扫描的回归。更新路由契约测试，要求目录路由一次调用 `queryFolderCovers(subs)`，且 `buildFolderResult` 不再直接查询数据库。

- [ ] **Step 2: 验证测试正确失败**

Run:

```bash
node --test test/database-file-query.test.js test/scan-results-contract.test.js
```

Expected: FAIL，原因是 `database.queryFolderCovers` 尚不存在，且路由仍在每个目录中调用递归 `queryFiles`。

- [ ] **Step 3: 实现最小数据库批量接口**

新增路径范围构造：目录自身使用 `f.folder_path = ?`，后代使用 `[folder + path.sep, incrementLastCodeUnit(folder + path.sep))` 半开区间。复用以下 prepared statement：

```sql
SELECT f.*
FROM files f INDEXED BY idx_folder_path
WHERE (
    f.folder_path = ?
    OR (f.folder_path >= ? AND f.folder_path < ?)
)
AND f.media_type IN ('image', 'video')
ORDER BY f.last_modified DESC, f.id ASC
LIMIT 1
```

对去重后的有效绝对路径执行查询并返回 `Map`；空数组返回空 `Map`。将现有行映射抽取为内部复用函数，保持 `queryFiles` 返回字段不变。

- [ ] **Step 4: 一次批量取得封面后构造目录响应**

在搜索、收藏夹和普通目录三条分支中统一调用：

```js
const coverByFolder = database.queryFolderCovers(subs);
const folders = subs.map(folderPath =>
    buildFolderResult(
        folderPath,
        coverByFolder.get(folderPath) || null,
        favoriteFolders.has(folderPath)
    )
);
```

`buildFolderResult` 只执行当前目录的 `statSync/readdirSync` 和封面 URL 格式化，不执行数据库查询。

- [ ] **Step 5: 验证转绿**

Run:

```bash
node --test test/database-file-query.test.js test/scan-results-contract.test.js
```

Expected: PASS。

### Task 2: 慢请求可观测性

**Files:**
- Create: `lib/request-timing.js`
- Create: `test/request-timing.test.js`
- Modify: `server.js`

**Interfaces:**
- Produces: `createRequestTimingMiddleware({ thresholdMs, now, warn })`。
- Consumes: Express `req/res/next`；只对超过阈值的 `finish` 或未完成 `close` 记录一次日志。

- [ ] **Step 1: 写失败测试**

使用真实 `EventEmitter` 响应对象覆盖：

```js
test('慢请求只记录路径、状态、耗时和结果，不泄露查询字符串', () => {
    // 1500ms finish 应写一条；100ms finish 不写；
    // 未 finish 的 close 写 outcome=close；finish 后 close 不重复。
});
```

- [ ] **Step 2: 验证测试正确失败**

Run:

```bash
node --test test/request-timing.test.js
```

Expected: FAIL，原因是 `lib/request-timing.js` 尚不存在。

- [ ] **Step 3: 实现并接入**

默认阈值 `1000`，日志格式固定为：

```text
[HTTP] Slow request method=GET path=/api/library/folders status=200 duration=1500ms outcome=finish
```

使用 `req.path`，不得使用包含查询字符串的 `req.originalUrl`；在 CORS/JSON 中间件之后、鉴权之前接入。

- [ ] **Step 4: 验证转绿**

Run:

```bash
node --test test/request-timing.test.js
node --check lib/request-timing.js
node --check server.js
```

Expected: PASS。

### Task 3: 回归、文档、提交与生产发布

**Files:**
- Modify: `release_notes.md`
- Modify: `.agent/project_memory.md`
- Modify: `.agent/registry.md`
- Append via HLG Skill: `.agent/handover.md`
- Generated via HLG Skill: `.agent/handover-index.md`

**Interfaces:**
- Produces: 可审计提交、候选镜像、回滚镜像和生产验证记录。

- [ ] **Step 1: 完整本地验证**

Run:

```bash
npm run test:backend
npm run test:frontend
npm run build
node --check database.js
node --check server.js
node --check lib/request-timing.js
git diff --check
```

Expected: 新增后端测试与既有后端测试全部通过；前端若仍只有已登记的 `localStorage` 桩清理故障，必须单独报告且不得误称全绿；生产构建和语法检查通过。

- [ ] **Step 2: 提交并推送**

提交源码、测试、计划和 DIA 文档，推送 `origin/main`，确认本地 HEAD 与 `origin/main` 相同。

- [ ] **Step 3: 构建隔离候选**

在 FNOS 从精确提交归档构建 `promenarleng/luvia-gallery:candidate-<shortsha>`；不得直接覆盖生产 `latest`。在临时候选容器中运行后端测试、首页/API 冒烟，并用生产数据库只读运行 `queryFolderCovers` 对 235 子目录基准，目标总耗时小于 1000ms。

- [ ] **Step 4: 建立回滚点并切换**

把当前生产镜像标记为 `promenarleng/luvia-gallery:rollback-<shortsha>-pre`，再把已验证候选标记为 `latest`，使用现有 `/vol2/1000/APPDATA/Lumina/docker-compose.yml` 强制重建 `luvia-gallery`。不得修改 Compose 内容。

- [ ] **Step 5: 生产验证**

验证容器 revision、镜像 ID、restart/OOM、端口 `9980:3001`、Mac/宿主/容器 3001/3002 四层 `/` 与 `/api/config`。使用容器内临时签发且不输出的 JWT 请求真实问题目录，确认状态 200、返回 235 个目录、总耗时小于 1000ms；并在请求期间并发探测 `/api/config`，确认没有全站阻塞。

- [ ] **Step 6: DIA/HLG 收口**

更新发布状态，使用 HLG Skill 的 `append` 先 dry-run 再 `--apply`，记录提交、候选/生产/回滚镜像、真实目录耗时、四层探测和风险；重新提交并推送治理记录。
