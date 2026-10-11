# 剪辑与达芬奇

冰爪剪辑器剪的是冰爪自己录制的工程（`.iceonixrec`）：屏幕、摄像头、麦克风分轨，带鼠标轨道，所以能自动追焦、换光标、换背景。导入的普通视频文件不能在冰爪里剪；用户要剪这类视频时，请他用达芬奇等剪辑软件。

推荐分工是**冰爪初剪，达芬奇精剪**：删口误、删停顿、推近、字幕、画面样式在冰爪里完成；B-roll、配乐、音效、调色交给达芬奇。

## 工具

| 作用 | 工具 |
|---|---|
| 定位冰爪里打开的剪辑器 | `get_open_document`（看 `editor`） |
| 读工程：词级转写、提词稿、已有追焦、鼠标事件、画面样式 | `get_editor_project` |
| 删段、恢复、字幕、追焦、摄像头全屏、音量、画面样式 | `apply_editor_edit` |
| 撤销某一批改动 | `undo_editor_edit` |
| 写入中断后恢复 | `recover_editor_edit`（仅在 `get_editor_project` 明确报告中断时） |
| 按原素材秒数截剪辑画面 | `render_editor_frames` |
| 在冰爪里打开剪辑区给用户看 | `open_editor_preview` |
| 导出成片、查导出状态 | `export_editor_video` / `get_editor_export` |
| 剪后最终字幕 | `generate_final_subtitles` 等，见[字幕与配音](speech.md) |
| 交给达芬奇、精剪成片收回项目 | `send_to_davinci` / `attach_external_render` |

## 剪「当前这条」

1. `get_open_document`，看 `editor.open`。为 `false` 时请用户先在冰爪里打开要剪的视频，不要猜工程。
2. `get_editor_project`、`apply_editor_edit`、`undo_editor_edit`、`render_editor_frames`、`open_editor_preview` 不传 `projectPath` 时，就作用于冰爪当前打开的剪辑器。导出和最终字幕是后台任务，必须显式传 `projectPath`（用 `editor.recordingPath`）。
3. 先读工程再动手：按词级转写和提词稿（`referenceScript`）决定删哪些口误和停顿，按鼠标点击决定在哪里推近。剪辑、追焦都用原素材秒数，焦点是左上原点的归一化坐标。不要猜切点。
4. 拿不准画面时用 `render_editor_frames(times=[原素材秒数])` 截图看：这段画面有没有内容、推近后主体是否被裁掉、字幕是否挡住关键信息。被删掉的时间点会返回 `deleted=true`。
5. `apply_editor_edit` 默认只预演（`dryRun=true`），先把方案告诉用户；用户说过「直接剪」就传 `dryRun=false` 一步写入。写入后冰爪里打开的剪辑器立即同步，用户能直接看到。
6. 改完在关键位置再截几张图自查，再用一两句话告诉用户改了什么、在哪几秒。每批改动都能用 `undo_editor_edit` 单独撤销；`revision` 冲突时重新读工程，不要用旧方案硬写。

## 画面样式

`apply_editor_edit` 里 `kind="style"` 的操作一次改多项画面设置：背景（none / solid / gradient / image）、留白、圆角、阴影、比例（original / vertical / square）、光标样式与大小、快捷键提示、字幕显示与烧录、点击音效、背景音乐、摄像头位置与形状。

- 先看 `get_editor_project` 返回的 `style`，只传要改的项。字段名与 `style` 相同，取值范围见工具描述，超出范围会被拒绝。
- 背景音乐和背景图必须在冰爪工作目录内，可先用 `add_project_asset` 加进项目。`bgmPath` 传空字符串表示去掉背景音乐。
- 没有摄像头的工程不能改摄像头设置。
- 改完用 `render_editor_frames` 截图看效果，不满意就继续调或撤销。

## 导出成片

1. 生成一个 UUID 作 `requestId`，`export_editor_video(projectPath, requestId, expectedRevision)` 冻结当前版本后台导出，成片保留追焦、光标、字幕和包装，并登记为项目的可发布版本。
2. 只用 `get_editor_export` 查询同一个 `requestId`。`completed` 后才能发布；`failed` 或 `unknown` 不自动重投。
3. 发布按[发布与登录](publish.md)做，真实发布前按共同规则取得授权。

## 冰爪初剪 → 达芬奇精剪

用户要在达芬奇里加 B-roll、配乐、音效、调色或精修节奏时：

1. 先在冰爪完成初剪（删段、推近、字幕）。需要剪后字幕时先按[字幕与配音](speech.md)生成最终字幕。
2. 选交接方式，并把差别告诉用户：
   - **带效果成片**：适合录屏教程，或冰爪的效果已经满意。先 `export_editor_video(purpose="davinci")`，等 `get_editor_export` 返回 `completed`，再 `send_to_davinci(mode="rendered", exportRequestId=导出的 requestId)`。成片保留追焦、光标、背景和摄像头，不混背景音乐、不烧字幕、不加片头封面，人声和系统声分轨。
   - **原片时间线**：适合口播，或要在达芬奇里大改节奏。`send_to_davinci(mode="timeline")`。原片没有追焦和背景，录屏原片里没有鼠标光标，摄像头是单独一轨，要在达芬奇里重摆。
3. 两种方式都进同一个达芬奇项目，每次新建一条时间线，不覆盖已有时间线。
4. **字幕不会随时间线进入达芬奇。** 把返回的 `subtitlePath`（SRT）告诉用户，请他在达芬奇里用「文件 → 导入 → 字幕」导入，会按时间码自动对齐。不要用脚本往字幕轨追加：会错位，实测还会让达芬奇崩溃。
5. 精剪：如果你接入了达芬奇自带的 MCP（`davinci-resolve`），可以在这条新时间线上放 B-roll、配乐和音效；没有接入就交给用户手动精剪。
6. 用户在达芬奇里把成片渲染到返回的 `renderDirectory` 后，用 `attach_external_render(projectId, filePath)` 把成片收回项目，成为新版本，再按授权发布。只收冰爪工作目录里的文件；项目正在发布时会被拒绝。

返回状态：

- `imported`：已导入达芬奇。
- `manualImport`：达芬奇已打开时间线文件，需要用户在达芬奇里点完导入。
- 失败并提示达芬奇没有响应：多半有弹窗挡着（例如软件更新），请用户先关掉再重试。

同步是单向的：达芬奇里的改动不会回写冰爪工程。
