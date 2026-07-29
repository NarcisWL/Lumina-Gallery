# macOS 每显示器窗口位置记忆 V2 实施计划

## 目标

修复主显示器在 MacBook 内置屏与外接显示器之间切换时，悬浮窗不能自动恢复各屏独立位置与尺寸的问题；完成 Release 打包并安全更新 `/Applications/LuviaGalleryWidget.app`。

## 已确认根因

- 旧实现使用 `CGDirectDisplayID` 数字作为跨重启持久化键，但该 ID 只适合在线会话；本机已为同一活动显示器累计近 30 个数字键。
- 旧实现持久化全局绝对 `NSWindow.frame`；切换主屏会重排全局坐标原点。
- 旧实现以 `window.screen ?? NSScreen.main` 选择恢复目标，不等同于菜单栏所在系统主屏。
- 屏幕变化恢复与 0.5 秒延迟保存、网格吸附没有互斥，可能把系统临时位置写回存档。

## 实施任务

### Task 1：可测试的显示器位置存储核心

- 新增纯逻辑位置模型与几何转换：
  - 内置屏使用 vendor/model，外接屏优先使用 vendor/model/serial；无数字序列号时以名称与物理尺寸构造最佳努力物理指纹。
  - 物理属性不足或两台连接屏指纹冲突时只使用进程内 session 键，禁止污染持久化档案。
  - 保存 `visibleFrame` 内的归一化横向位置、顶部位置以及窗口尺寸。
  - 恢复时按目标屏当前 `visibleFrame` 重建并完整夹紧。
- 测试先行，至少覆盖：
  - 主屏切换导致全局原点变化后，相对位置保持。
  - 分辨率、缩放、Dock/菜单栏造成 `visibleFrame` 变化。
  - 过大窗口与非法比例被安全夹紧。
  - 每台显示器的存档互不覆盖。

### Task 2：AppDelegate 状态机接入

- 系统主屏使用 `NSScreen.screens.first`，不得使用 `NSScreen.main` 推断。
- 启动时恢复当前系统主屏的 V2 位置。
- `didChangeScreenParametersNotification` 防抖后检测主屏 Resolver 键变化，并恢复该屏存档。
- 屏幕重配置和程序化恢复期间取消待执行保存与网格吸附，禁止 move/resize 回调污染存档。
- 手动跨屏移动只更新对应屏存档，不立即跳回。
- 所有隐藏入口在 `orderOut` 前统一取消延迟任务并保存稳定快照；重配置期间隐藏或退出不得写入临时坐标。
- 旧 `displayFrames/windowFrame` 数字键无法证明物理归属，完全失败关闭，不自动迁移。

### Task 3：验证与交付

- Swift 测试全绿。
- Xcode Debug 与 Release 构建成功。
- 运行 `macos-widget/scripts/package_release.sh`，验证 zip 和 `.app` 签名。
- 终止当前运行实例，把旧 `/Applications/LuviaGalleryWidget.app` 移至可恢复备份，再安装新包并启动。
- 核验进程、bundle ID、签名、版本、应用路径与运行日志。

## 验收标准

- 同一物理显示器不再因重启、拔插或主屏切换产生新的数字 ID 存档。
- 主屏在内置屏和外接屏之间切换时，自动使用对应显示器独立位置和尺寸。
- 全局屏幕原点、分辨率或 `visibleFrame` 变化不会造成窗口跳到错误屏或不可见区域。
- 位置锁定不阻止程序化恢复；恢复过程不污染另一台显示器档案。
- 新 Release 包已安装到 `/Applications/LuviaGalleryWidget.app`，旧包有可恢复备份。

## DIA 预判

- 用户可见窗口行为变化：需要更新 `macos-widget/README.md` 与 `release_notes.md`。
- 持久化结构变化：需要更新 `.agent/project_memory.md`。
- 完成后追加 HLG 记录并重建索引。
