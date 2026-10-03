---
name: iceonix-workflows
description: 连接 Iceonix（冰爪）本地项目库、Markdown/图文创作、平台登录和跨平台发布。用户要求查看或编辑冰爪项目、录制剪辑、整理文章或图文草稿、检查账号登录、执行 dry-run、确认发布、查询发布状态或只重试失败平台时使用。
---

# 冰爪创作与发布

通过 Iceonix MCP 操作本机项目。项目库是事实源；App 是唯一的 Safari、微信和发布驱动者。

## 项目创作

1. 读取已有项目先调用 `list_projects`，再调用 `get_project`。新任务调用一次 `create_project`。
2. `create_project` / `get_project` 返回 `projectId`、`revision` 与 `projectPath`：项目级修改把最新 `revision` 传为 `expectedRevision`，发布工具使用 `projectPath`，不要混用。
3. 创建 Markdown 使用 `create_project_document`。它返回 `projectRevision`、`document.id` 与 `documentRevision`：下一次项目级修改使用 `projectRevision`，更新正文使用 `documentRevision`。
4. 素材先用 `add_project_asset` 加入项目；每次都把返回的 `project.revision` 传给下一次项目级修改。创建作品使用 `create_project_work`，其中 `assetIds` 只接收 `asset.id`。
5. 编辑图文短文案使用 `update_document_publish_output`；不要用短文案覆盖 Markdown 长文章。revision 冲突时重新读取并合并，禁止盲目覆盖。

## 生图恢复

1. 一组不同画面只调用一次 `generate_image(prompts=[...], requestId=稳定幂等键)`；不要拆成多次生图。
2. 快速完成时读取 `data.imagePaths`。返回 `state=queued/running` 或 `terminal=false` 时，保留 `taskId`，只调用 `get_image_generation(taskId=...)` 查询原任务，不重新提交。
3. 首次响应丢失但已经传过 `requestId` 时，用同一个 `requestId` 调 `get_image_generation` 恢复。相同 ID 的参数冲突、部分成功、`unknown` 或客户端超时都不得换 ID 重投，以免重复生成和扣费。
4. 只有 `state=succeeded` 才把完整路径继续交给素材和发布链路；`partial`、`failed` 或 `unknown` 要如实展示已完成路径与错误，由用户决定是否创建新的生图任务。

## 项目归属

- `publish_video`、`publish_image_carousel`、`batch_publish`、`batch_publish_image_carousel` 与 `publish_article` 都必须携带 `projectPath`。
- 发布已有项目时使用 `create_project` / `get_project` 返回的绝对 `projectPath`。在冰爪项目助手中该参数由连接自动绑定，不向用户索要，也不得改成 `new`。
- 项目外的独立发布明确传 `projectPath="new"`。视频与图文会建立标准发布项目，公众号长文章仍只进入私有草稿库。不要省略，也不要把 `projectId` 当路径。

## 选择界面入口

- 支持 MCP Apps 的宿主（如 Claude 桌面版）直接调用 `show_project_review`、`show_platform_login` 或 `show_publish_status`，面板会显示在对话里：用户可以在面板里改正文和图文草稿、扫码登录、确认后重试未成功的平台。
- Codex 桌面端优先调用 `open_iceonix_workspace`：项目确认传 `view=projectReview`，平台登录传 `view=platformLogin`，发布状态传 `view=publishStatus` 与 `batchId`。返回 `browserHandoff.required=true` 后用应用内 Browser 打开 `browserHandoff.url`，不要改写链接。
- Claude Code 等终端宿主同样调用 `open_iceonix_workspace`，把 `browserHandoff.url` 原样作为可点击链接交给用户在浏览器打开；链接只在本机有效，不要改写或转发给别人。
- 面板里的操作会以 `[冰爪面板] …` 同步到上下文：看到后以冰爪里的最新内容为准，不要再用旧内容覆盖。面板里的提示词按钮会以用户身份发来消息，照常处理即可。

## 登录

1. 只有用户明确要求检查或登录时才调用 `check_login` / `show_platform_login`；默认读本机验证缓存，用户明确要求实时刷新时传 `force_refresh=true`。普通发布前不要预检，发布驱动复用免检窗口并优先处理目标页的明确登录墙。
2. 需要登录时调用一次 `start_login`，再调用一次 `await_login`；二维码 PNG 作为图片展示，不索取或保存密码。
3. 登录超时或发布进入等待登录时，不重新提交发布工具；冰爪会续接原任务。

## 安全发布

1. 调用 `list_platforms` 确认平台与 `supportedAssetTypes`。单平台视频用 `publish_video`，多平台视频用 `batch_publish`；单平台图文用 `publish_image_carousel`，多平台图文用一次 `batch_publish_image_carousel`。
2. 公众号长文章先 `list_article_themes` → `save_article_draft`，再把返回的 `articlePath` 与正确的 `projectPath` 交给 `publish_article`。它只存私有草稿，不群发。
3. 参数名是 `dryRun`。`dryRun=true` 会上传和填写但跳过最终发布或保存，不构成真实发布授权；不得写成 `dry_run`。真实发布（含重发、重试）必须同时传 `confirmPublish=true`，否则冰爪拒绝执行；`publish_article` 只存草稿，不需要。
4. 真实公开发布前检查最终素材顺序、平台、标题、正文、标签与声明。用户已明确给出最终内容，或明确预授权“生成后直接发布/无需确认”时不要重复确认，直接传 `confirmPublish=true`；否则 Agent 生成或修改关键内容后只确认一次，确认后再传。
5. `batch_publish_image_carousel` 立即返回 `batchId`，随后只调用一次 `await_publish_batch`。超时只查询原批次，不重新提交；只有批次明确为 `failed`、`partial` 或 `blocked`，并且用户同意后，才调用 `retry_publish_batch`。
6. `batch_publish` 视频会同步返回逐平台结果，不产生供 `await_publish_batch` 使用的 `batchId`。

## 输出结果

- 区分“已写入项目”“已进入队列”“dry-run 通过”“已发布成功”“已存公众号草稿”和“结果未知”。
- 展示结构化逐平台结果，不能用一个平台失败覆盖其它平台成功。
- MCP 失败后禁止用 CLI、shell 或浏览器重投，也不要手工修改 `iceonix.json`。

## 模板视频

没有拍摄素材的宣传、资讯、知识视频用冰爪内置模板出片，用户不用装任何工具；完整分镜格式见 `iceonix-video` 技能。

1. 先和用户确认旁白稿，再 `analyze_music` 拿鼓点与段落，`synthesize_speech` 逐句配音（同一音色）。
2. 写 `storyboard.json`，`render_video(quality=preview)` 出低清预览；未完成用 `get_video_render(renderId)` 等待，先处理 `warnings`。
3. 按用户在对话里的意见改分镜、重新预览；确认后 `quality=final` 出正片。需要进项目时 `add_project_asset` + `create_project_work`，发布前照常让用户确认。

## 录制剪辑

1. 用 `get_editor_project` 获取当前录制工程、剪辑 revision、转写与鼠标事件。剪辑 revision 与项目文稿 revision 不通用。所有操作时间为原素材秒数，追焦焦点是左上原点的 0～1 坐标。
2. `apply_editor_edit` 默认 dry-run。按用户意图提交批量删段、字幕、追焦、摄像头布局或音量操作；实际应用传 `dryRun=false`，沿用稳定 requestId。用户明确授权自动剪辑可直接应用，否则先展示方案。不要直接写工程文件。
3. 需要检查画面时调用 `open_editor_preview`，在冰爪主窗口定位项目内剪辑区，左侧可显示该项目的 Chat；后台编辑无需先打开预览。撤销用原批次 requestId 和最新 revision。若有后续修改，先读状态，不强行覆盖。写入中断按错误提示调用 recover_editor_edit，不猜测旧 revision。字幕改字后旧词表失效，不按旧词表继续删词。
4. 导出调用一次 `export_editor_video`，之后仅用 `get_editor_export` 查询原 requestId。只有 completed 的成片才能进入发布；unknown/failed 不自动重投。发布沿用当前用户授权，明确要求自动剪辑并直接发布时不重复要求确认。
5. 首版使用冰爪原生渲染保留鼠标和追焦；不承诺达芬奇 Fusion 转换、共享工程直接编辑或双向同步。
