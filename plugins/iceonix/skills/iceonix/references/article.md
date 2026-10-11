# 公众号与长文

把项目主题、语音转写、素材或草稿整理成一篇文章。正文只维护一份，公众号和博客是它的输出渠道。

- 本章处理：有完整正文的文章（Markdown、摘要、正文配图），以及用户要求的公众号封面、排版主题、存草稿，或博客输出。
- 视频、1～9 张图片加短文案的平台图片帖（包括公众号贴图草稿）、平台登录，按[发布与登录](publish.md)做。
- 用户只说「发公众号」时，先看素材：只有图片和短文案走发布一章；需要正文、封面或排版走本章；还不清楚就问。

## 两种文章来源

**项目文稿（首选）**：冰爪项目里的一份 Markdown 文稿，正文只此一份。

- 用户说「当前这篇」「冰爪里打开的文章」时，先 `get_open_document` 取 `projectPath` 和 `documentId`。没有 `documentId` 就请用户先在冰爪里打开那份文稿，不要猜。
- 找文稿：`get_project` 读 `documents[]`；没有合适的就 `create_project_document(title=文章标题, role="master")` 新建。**文稿标题就是公众号标题**，MCP 暂时改不了文稿标题，新建时就要传对；`create_project` 带 `body` 生成的文稿标题是「初始文稿」，不适合直接当文章。
- 改正文：`get_project_document` 取 `documentRevision`，再 `update_project_document` 整篇替换。冲突时重新读取并合并。
- 公众号设置：`update_document_wechat_output` 写主题、作者、摘要、原创声明和封面，只传要改的字段。封面先用 `add_project_asset` 加进项目，再传返回的 `asset.path`。
- 发布：`publish_article(projectPath=项目绝对路径, documentId=文稿 ID)`。不要再调用 `save_article_draft`，否则会出现两份正文。
- 实时预览：写入后，冰爪里打开的这份文稿和它的公众号预览立即刷新。用户可以在冰爪里看排版效果再告诉你改哪里。

**独立文章**（不属于任何项目）：用 `save_article_draft` 写出一份 `.md`，以返回的 `articlePath` 发布。

输出范围：

- 用户只要求写作、改稿或讨论时，写到文稿即停止：不生成封面、不碰公众号后台、不发博客。
- 用户明确要求公众号输出时，才走下面的公众号流程。主题、封面、作者和原创声明是渠道设置，不是另一篇正文。
- 用户要求博客时，复用同一份文稿，由冰爪关联当前 Quartz 绑定的 Obsidian 文件并重新编译预览。不要用 `write_file` 在别处复制文章。
- 已有文章就继续改同一份；用户没要求新文章时不要新建第二份。

## 公众号链路只走冰爪

用户说「用冰爪发公众号」「写入公众号草稿箱」「一条龙」时，整条链路只用冰爪工具：

- 只存草稿，绝不点群发。Safari 里的公众号后台要已登录，冰爪 App 要在运行。
- 用户已明确要求「一条龙」「写入公众号」或「存草稿箱」时，这一句就授权了完整的私有草稿链路，中途不再确认；只要求写文章、改稿或预览时不得写入公众号后台。
- 失败后不要「自己修浏览器」：不结束或打开 Safari、不读网页、不手工粘贴正文。只按下面的失败恢复做，仍失败就停下如实报告，把文稿位置或 `articlePath` 留给用户。

## 工具

| 作用 | 工具 | 关键结果 |
|---|---|---|
| 定位当前文稿 | `get_open_document` | 冰爪当前打开的项目、文稿和公众号预览主题 |
| 读取项目与文稿 | `get_project` / `get_project_document` | 文稿 `id`、正文与 `documentRevision` |
| 新建、修改项目文稿 | `create_project_document` / `update_project_document` | 正文按 `documentRevision` 防冲突 |
| 写文稿的公众号设置 | `update_document_wechat_output` | 主题、作者、摘要、原创、封面；预览实时刷新 |
| 正文配图 | `generate_image` / `get_image_generation` | `data.imagePaths`；长任务保存 `taskId` |
| 公众号封面 | `generate_article_cover` | `data.coverPath`，精确 2.35:1，最终 1645×700 |
| 列出排版主题 | `list_article_themes` | `id`、`name`、`isDefault`、`isBuiltin` |
| 读取主题 CSS | `get_article_theme` | `css` 与 `isBuiltin` |
| 保存排版主题 | `save_article_theme` | 以返回的 `name` 为准；同名默认拒绝 |
| 预览截图 | `render_article_preview` | 手机预览截图，每次最多 3 屏，可临时指定主题 |
| 保存独立文章 | `save_article_draft` | 仅独立文章：`articlePath` |
| 写入公众号草稿 | `publish_article` | 项目文稿传 `projectPath` + `documentId`；独立文章传 `articlePath` |
| 登录 | `check_login` / `start_login` / `await_login` | `platform="wechatMP"`；二维码以图片返回 |

## 标准流程

1. **收集素材**：至少要有一个主题、一段语音转写或一份草稿。不够就问主题、目标读者或核心观点。
2. **读资料**：读项目文稿和用户本轮明确 `@` 的资料。人设和文风沿用你自己的设定和用户本轮要求。
3. **写文章**：
   - 不超过 64 字的标题，与正文分开；正文不重复一级标题，从导语或 `##` 开始。
   - 完整 Markdown 正文。
   - 基于全文的 60～90 字摘要，用于转发卡片和会话展示。
   - 正文配图的数量、位置和提示词，按段落语义规划，不随机插图。
   - 一个结合文章的封面方向；用户没指定风格时由主题推导。
4. **判断范围**：用户要一条龙或存草稿时，继续做配图、封面、排版和草稿，中途不确认，完成后统一汇报。只要写稿时到此为止。原创声明没有明确值时先问；作者没指定就不传，冰爪会用用户设置的默认作者。
5. **配图和封面**（两者可并行）：正文配图按[总说明](../SKILL.md)的生图规则调用一次 `generate_image`；封面单独调用 `generate_article_cover`，不要先生成方图再裁。
6. **把图片放进正文**：项目文稿先用 `add_project_asset` 加进项目，正文引用返回的项目内相对路径（如 `![](Attachments/图1.png)`），项目搬家也不丢图。独立文章用 URL 编码的 `file://` 绝对地址，如 `![](file:///Users/example/Pictures/body%20image.png)`，并把这些路径传给 `save_article_draft.imagePaths`。
7. **写好公众号设置**：项目文稿用 `update_project_document` 写最终正文，再用 `update_document_wechat_output` 写主题、作者、摘要、原创和封面，每次用上一步返回的最新 revision。独立文章调用一次 `save_article_draft`，把标题、正文、摘要、主题、作者、原创、配图和封面一起写入。不要把正文直接交给 `publish_article.caption`。
8. **预演再存草稿**：首次使用、排版刚改过或页面状态不确定时，先 `dryRun=true` 预演；返回 `status=success` 且 `themeName` 正确后，再用同一正文来源 `dryRun=false` 存草稿。预演和正式保存必须串行。
9. **汇报**：成功时说「📝 已存进公众号草稿箱，请到公众号后台终审后群发」，并说明配图数、是否带封面。没有封面也能存，但要提醒用户补。预演不能说成已存草稿。

```text
# 项目文稿
publish_article(platform="wechatMP", projectPath="项目绝对路径", documentId="文稿 ID", dryRun=true)
# 独立文章
publish_article(platform="wechatMP", articlePath="save_article_draft 返回的路径", projectPath="new", dryRun=true)
```

## 重复调用与失败恢复

冰爪按内容防止重复草稿：

- 同一内容预演或失败过一次后，不带 `requestId` 再调用只会返回上次的结果，Safari 不会动。不要把这种秒回当成重新跑过。
- 预演和正式保存的内容指纹不同，不能共用一个 `requestId`。
- 明确失败后要重跑：请用户在冰爪任务中心点重试；或确认草稿确实没存上后，换一个新的 `requestId` 再调用。

错误明确是「等待登录」时：`start_login(platform="wechatMP")`，把二维码**图片**发给用户，再 `await_login`。登录后冰爪会续发原任务，不要再调用 `publish_article`。普通发布前不要调 `check_login`，公众号驱动会在发布页即时检测。

其它预演失败：停下报告，不自动重复，也不继续正式保存。冰爪会在打开 Safari 前把 Markdown 转成逐元素内联样式，并在页面内校验标题结构和残留的 Markdown 标记，任何一步失败都不会降级粘贴原文。

## 排版主题

每次确认和发布前调用 `list_article_themes`，按文章语气和题材选一个，例如科技产品选科技类主题，人文叙事选温润类主题。把返回的精确 `name` 或 `id` 传给 `update_document_wechat_output.themeName`（项目文稿）或 `save_article_draft.themeName`（独立文章）。

在现有主题上优化，或用户要新风格时：

1. `get_article_theme(name=...)` 读出完整 CSS，按用户意见修改，遵守 `save_article_theme` 工具描述里的 CSS 规则。
2. 内置主题（`isBuiltin=true`）只能用新名称另存；自定义主题可以另存，也可在用户同意后传 `overwrite=true` 覆盖。覆盖会让所有用这个主题的文章一起变。
3. 先用 `render_article_preview(themeName=新主题)` 自己看截图：标题层级、段落留白、引用块和图片，以及文字和底色的对比度。有问题改了再看，长文按 `nextPage` 看完后面几屏。
4. 满意后以返回的 `name` 写回文稿设置。冰爪里正在看这个主题的预览会立即刷新。

用户说「把冰爪里当前这篇重新设计排版」时：`get_open_document` 定位文稿 → `render_article_preview` 看现状 → 写新 CSS 并 `save_article_theme` → `render_article_preview(themeName=新主题)` 自查 → `update_document_wechat_output` 套用。这几步不碰 Safari，不需要中途确认。

- 不得省略主题，也不得自己编造或拼接主题名称。
- 不要把大段 CSS 传给 `themeCSS`，它只用于用户明确提供自定义 CSS 时。
- 主题不存在或渲染失败会明确报错，不会静默换主题。

## 封面

```text
generate_article_cover(title="文章标题", body="Markdown 正文", style="用户指定风格，可留空", includeTitle=true, titleText="封面上的标题")
```

- 实际出图由冰爪用用户配置的 AI 生图模型完成。
- `includeTitle` 默认 `true`，标题直接画进封面，不要再让冰爪叠加同一标题。用户要无字底图或想自己排字时传 `false`，生成后引导他在冰爪封面设计器里编辑文字。
- 封面比例只有 2.35:1（容差 ±0.002），成图无裁切缩放到 1645×700。用户自己的图片比例不符时，请他在冰爪封面设计器里手动裁切，或改用 `generate_article_cover`；工具不会替用户裁图。
- 不要把 AI 封面当正文配图，也不要把正文图片当封面。

## 参数要点

- `save_article_draft` 的 `title`、`body`、`themeName` 必填；`summary` 最多 120 字。
- `author` 省略时用冰爪设置的默认作者，传值逐篇覆盖，传空字符串表示不署名。公众号作者最多 8 个字。声明原创时作者必填：既没传作者、冰爪也没设默认作者时会被拒绝，先问用户署名。
- `declareOriginal` 由用户决定，不得默认改为 `true`。
- `publish_article.platform` 取 `wechatMP`（公众号草稿，缺省）或 `x`（把同一篇文章发成 X 推文：标题加正文转纯文本，封面在前、正文插图补足，最多 4 张）。`x` 会直接公开发推，属于真实发布，要按共同规则取得授权，首次先预演；普通账号限 280 加权字，长文需要 Premium。用户要「公众号和 X 都发」时，对同一正文来源分别调用两次，结果分开汇报。
- `articlePath` 必须来自本次 `save_article_draft` 的结果，`documentId` 必须来自 `get_project`。
- 独立文章的 `.md` 会出现在冰爪公众号编辑页左侧草稿列表，预演和正式保存复用同一个 `articlePath`。

## 正文规范

- 人设、语气、事实边界服从你自己的设定、项目文稿和用户本轮要求；不编造资料不支持的事实。
- 标题具体、有钩子，避免「浅谈」「随笔」之类的空标题。前两段直接给出冲突、利益点或核心判断。
- 用 `##` 组织段落，每段 1～3 句，适当留白；多用实例、数字和细节。
- `**加粗**`、列表、引用和公式保持标准 Markdown。
- 结尾自然收束，给出符合人设的关注、留言或「在看」引导。

## CLI 兜底

只有运行环境里根本没有冰爪 MCP、但装了 `iceonix` CLI 时才用。CLI 同样交给运行中的冰爪 App 执行，不得用 `osascript`、curl 或其它命令代替。

```bash
iceonix article-cover --title "文章标题" --body-file "/绝对路径/article.md" --style "封面风格"

iceonix publish-article --title "文章标题" --body-file "/绝对路径/article.md" \
  --cover "/绝对路径/cover.png" --images "/绝对路径/body-1.png,/绝对路径/body-2.png" \
  --theme "主题名称" --author "署名" --dry-run
```

预演成功后去掉 `--dry-run` 保存草稿。

## 禁止事项

- 不得用视频或图片帖发布工具处理公众号长文章。
- 不得自动群发、模拟点击群发，或暗示已群发。
- 用户只要求写文章、改稿或预览时，不得调用 `publish_article`。
- 不得擅自声明原创。
- 不得修改 `Inspirations/Plans` 下的灵感计划文件。
