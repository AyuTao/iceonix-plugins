---
name: iceonix
description: 冰爪（Iceonix）本机创作与发布执行器的使用说明：剪辑冰爪里打开的视频并交给达芬奇精剪，字幕与配音，公众号长文排版与存草稿，多平台发布与扫码登录，模板做视频，数字人口播与 AI 视频，口播稿开拍，热点素材，微信加好友。用户提到冰爪、当前打开的文稿或剪辑，或要用冰爪做这些事时使用。
---

# 冰爪（Iceonix）

冰爪是用户 Mac 上的创作与分发执行器：录屏剪辑、字幕配音、公众号排版和多平台发布都由冰爪 App 在本机执行。你负责理解需求、写内容、调用冰爪 MCP 工具并汇报结果。冰爪 App 要保持运行。

## 先读对应章节

动手前先读与任务对应的章节。章节在本技能目录的 `references/` 下，用你读文件的工具打开；Hermes 用 `skill_view(name="iceonix", file_path="references/<文件名>")`。任务跨章节时都要读，例如「剪完发到视频号」读剪辑和发布两章。

| 用户要做的事 | 章节 |
|---|---|
| 剪冰爪里正在剪的视频：删口误和停顿、推近、改画面样式、导出成片、交给达芬奇精剪 | [剪辑与达芬奇](references/editing.md) |
| 转文字、生成或修改字幕、配音、克隆声音、分说话人 | [字幕与配音](references/speech.md) |
| 写文章、公众号长文、排版主题、封面、存公众号草稿、博客 | [公众号与长文](references/article.md) |
| 发布视频或图片帖到各平台、检查和扫码登录、重发和重试 | [发布与登录](references/publish.md) |
| 用文案、截图和音乐做宣传片、资讯或知识视频，把推文或热点做成视频 | [做视频](references/video.md) |
| 数字人口播、照片做成说话视频、对口型、AI 生成产品或情景镜头 | [数字人与 AI 视频](references/digital-human.md) |
| 把定稿的口播脚本交给用户开拍（提词器），或发到手机 UP100 | [口播稿与开拍](references/narration.md) |
| 搜热门视频、对标博主、下载爆款素材和字幕 | [热点与素材](references/trends.md) |
| 在 Mac 微信里按手机号或微信号加好友 | [微信加好友](references/wechat-contacts.md) |

## 共同规则

1. **只用冰爪 MCP。** 平台网页、Safari 和微信 App 只能由冰爪内部驱动。不得用浏览器、Computer Use、终端或 osascript 操作它们，不得打开、重启或结束 Safari、微信、冰爪进程，不得手改 `iceonix.json`、`edit.json`、`README.md` 等冰爪管理的文件。查资料可以用你自己的只读能力。
2. **报错就原样汇报。** MCP 报错时不改用 shell 或 CLI 重投。只有运行环境里根本没有冰爪 MCP 时，才按各章附的 `iceonix` CLI 做；CLI 同样只是交给运行中的冰爪 App 执行。
3. **结果以工具返回为准。** 不猜测成功。超时、结果不明或仍在等待时不重新提交；同一次请求重试复用原 `requestId`，有意重做才换新的。
4. **公众号只存草稿，绝不群发。** 原创声明由用户决定，不擅自打开。
5. **发布授权。** 用户本轮已给出素材、平台和最终文案，就视为授权，不重复确认。关键内容由你生成或改动时，展示一次最终内容，只确认一次。用户说「直接发布」「生成后直接发」「无需确认」即为预授权。只授权了预演时，预演通过也不能真实发布。`iceonix.json` 的 `status=ready` 和已配置的定时任务属于任务授权，不得扩大平台或改写文案。
6. **让用户只说目标。** 工具名、路径、批次、requestId 和安全约束由你在内部处理，不要让用户复述。
7. **聊天通道少说话。** 在微信、飞书等通道（如 Hermes）里，消息有频率配额：中间过程不逐条播报，只在需要确认和给最终结果时发消息。

## 当前打开的文稿和剪辑

用户说「这篇」「当前这篇」「我正在剪的这条」时，先调用 `get_open_document`：

- 有 `documentId`：冰爪里打开的是项目文稿，按[公众号与长文](references/article.md)修改。写入后冰爪里的文稿和公众号预览会实时刷新，用户可以边看边让你改。
- `editor.open=true`：冰爪里打开的是剪辑器。剪辑工具不传 `projectPath` 就作用于它，按[剪辑与达芬奇](references/editing.md)做。
- 都没有：请用户先在冰爪里打开要处理的文稿或视频，不要猜。

## 项目基础

- 冰爪项目是一个目录，文稿、素材、作品和发布记录都在里面。读已有项目先 `list_projects` 再 `get_project`；新任务调用一次 `create_project`。
- 项目级修改把最新 `revision` 传为 `expectedRevision`；发布工具用 `projectPath`（绝对路径）。`projectId` 不是路径，不要混用。
- 文稿用 `create_project_document`、`get_project_document`、`update_project_document`，按 `documentRevision` 防冲突。冲突时重新读取再合并，不要盲目覆盖。
- 素材先用 `add_project_asset` 加进项目，作品用 `create_project_work` 创建（`assetIds` 只收 `asset.id`）。
- 发布工具都要带 `projectPath`：已有项目传绝对路径，项目外的独立发布传 `new`。

## 生图

- 一组不同画面只调用一次 `generate_image(prompts=[...], requestId=...)`；同一画面要多张用 `prompt` 加 `n`。
- 返回 `terminal=false` 时保留 `taskId`，只用 `get_image_generation` 查询原任务。部分完成、超时或 `unknown` 都不得重新提交，以免重复扣费。
- 直接使用返回的 `data.imagePaths`，不要自己拼路径。
