# 发布与登录

把视频或一组图片发布到各平台，检查登录、扫码登录、重发和重试。发布动作由冰爪的确定性代码执行（状态机、按作品去重、防重发），你只负责理解意图、写文案、协调登录和汇报结果，不替冰爪管理发布状态。

- 本章处理：已经进入冰爪项目的视频、1～9 张图片组成的平台图片帖、公众号贴图草稿，以及平台登录。
- 公众号 Markdown 长文章（正文、配图、封面、排版）按[公众号与长文](article.md)做，不得用图片帖工具代替。
- 用户只说「发公众号」又没说形态时，先问是「图片加短文案的贴图草稿」还是「带正文和排版的长文章」。

## 工具

| 作用 | MCP 工具 | CLI（仅没有 MCP 时） |
|---|---|---|
| 列出平台和各自支持的素材类型 | `list_platforms` | `iceonix platforms` |
| 读登录验证缓存 | `check_login` | `iceonix check-login` |
| 实时诊断登录（用户明确要求时） | `check_login(force_refresh=true)` | `iceonix check-login --force-refresh` |
| 开始登录（扫码平台返回二维码图片） | `start_login` | — |
| 等待登录完成 | `await_login` | — |
| 视频号合集预设 | `get_series_presets` | — |
| 视频发到一个平台 | `publish_video` | `iceonix publish` |
| 视频发到多个平台 | `batch_publish` | `iceonix batch-publish` |
| 图片帖发到一个平台 | `publish_image_carousel` | `iceonix publish --image ...` |
| 图片帖发到多个平台 | `batch_publish_image_carousel` | `iceonix batch-publish --image ...` |
| 查询 / 等待图片帖批次 | `get_publish_batch` / `await_publish_batch` | — |
| 重试失败平台 / 有意重发 | `retry_publish_batch` / `republish_batch` | `iceonix republish` |
| 只写发布文案、先不发 | `update_project_publish_copy` | — |

- **dryRun（预演）**：四个发布工具都接受 `dryRun`（CLI `--dry-run`）。为 true 时真实打开平台、上传素材、填写文案，只跳过最后的发布点击，成功返回「测试通过（dry-run，未真正发布）」。
- **confirmPublish**：真实发布、重发和重试必须传 `confirmPublish=true`（CLI `--confirm`），只有按共同规则取得授权后才能传。漏传会被冰爪拒绝并提示「真实发布需要用户确认」，这时先问用户，不要自己补传。公众号长文章只存草稿，不需要它。

## 平台 ID（大小写敏感）

| 平台 | ID | 说明 |
|---|---|---|
| 抖音 | `douyin` | |
| 微信视频号 | `wechat` | |
| 微信朋友圈 | `wechatMoments` | 仅图片帖，用本机微信 App |
| 小红书 | `xhs` | |
| B站 | `bilibili` | |
| 快手 | `kuaishou` | |
| YouTube | `youtube` | 仅视频 |
| Instagram | `instagram` | 图片帖与视频；没有二维码，掉登录时冰爪自动点已保存账号的「继续」，要密码时由用户在 Safari 完成；不支持自定义封面、原创声明和位置，批量时冰爪自动不传 |
| Facebook | `facebook` | 图片帖与视频，只发个人时间线，不代发公共主页或群组；不支持封面、原创声明、位置和定时 |
| X（推特） | `x` | 1～4 张图、视频或文字推文；没有二维码，由用户在 Safari 手动登录；按账号类型限字（普通 280，Premium 25,000，加权计字），普通账号视频最长 140 秒；不支持封面、原创声明、位置和定时 |
| 公众号 | `wechatMP` | 仅图片帖，保存私有贴图草稿，绝不群发 |

公众号是驼峰 `wechatMP`，传小写会报「不支持的平台」。

**Facebook 受众**：默认公开（`public`）。只有用户明确要求仅好友可见时，才在发布工具传 `facebookAudience: "friends"`（CLI `--facebook-audience friends`），其它平台不要传。冰爪会点选分享对象并在发帖前回读，对不上就停止。`title` 会并入帖子正文。九张图时后四张的顺序冰爪无法逐张核验，汇报时不要说成已逐张核对。

**展开平台清单：**

- 视频号 `wechat`、朋友圈 `wechatMoments`、公众号 `wechatMP` 是三个独立目标，不得合并、去重或互相代替。
- 用户说「全部平台」「所有图文平台」时，先调 `list_platforms`：发视频取 `supportedAssetTypes` 含 `video` 的平台，发图片帖取含 `imageCarousel` 的平台，不要凭记忆手写。用户在冰爪设置里关掉的平台会被跳过，如实报告即可。
- 用户列出了平台子集或排除了某个平台，就严格照办，不擅自扩大。
- 平台清单只在内部展开，不向用户复述内部 ID。

## 素材和文案

**视频路径**：发布工具要视频的绝对路径。优先用附件给出的路径；冰爪项目里的成片用 `get_project` 或导出结果返回的路径。

聊天通道（如 Hermes 的微信、飞书）里，用户发的视频可能只显示成占位符，文件其实已下载到本地缓存，自己找最新的：

```bash
find ~/.hermes/cache/videos ~/.hermes/cache/documents -type f \
  \( -iname "*.mov" -o -iname "*.mp4" \) 2>/dev/null \
  -exec stat -f '%Sm %z %N' -t '%H:%M:%S' {} \; | sort -r | head -5
```

用之前把文件名、时间和大小报给用户确认，例如「我收到 video_xxx.mov（10:52，1.8MB），就用它发布，对吗？」；时间相近的有多个就列出来让他选。视频和发布指令在聊天里常分两条到达，顺序不定：凑齐「视频绝对路径」和「目标平台」再执行，缺哪个问哪个，不要猜。

**图片帖**：`imagePaths` 的数组顺序就是发布顺序。缓存里找图（`~/.hermes/cache/images`）时同样要把文件名、时间、大小和最终顺序发给用户确认；不要整目录全选，不要混入临时图片，不要改动用户确认过的顺序。在项目里发布时，只用本轮附件、生成工具返回的路径和项目里明确的素材，不扫描其它项目或目录。

**图片帖约束：**

- 去重后 1～9 张（X 最多 4 张）。超过上限时先把实际保留的顺序发给用户确认，不得静默丢图。
- 图片帖不是文章：不要创建或改写项目 `README.md` 或其它 Markdown 文档。标题、短正文和标签只作为发布参数，由冰爪存到图片帖作品里。
- **由你生成或改写的图片帖，正文加标签合计不超过 200 个字符**（中英文、数字、标点、空格、换行、`#` 都算，Emoji 算一个）。先定标签，再用剩余字数写正文。用户自己给的成稿不受这条限制，超过时原样展示让他确认，或主动给一个 200 字内的精简版，不得静默删改。

| 平台 | 标题上限 | 正文加标签上限 |
|---|---:|---:|
| 视频号 | 16 | 1000 |
| 朋友圈 | 无标题 | 1000 |
| 抖音图文 | 20 | 1000 |
| 小红书 | 20 | 1000 |
| B站图文 | 20 | 1000 |
| 快手 | 无标题 | 500（标题并入正文） |
| Instagram | 无标题 | 2200（标题并入配文） |
| X | 无标题 | 加权 280（Premium 25,000，超限拒绝不截断） |
| 公众号贴图 | 20 | 1000 |

**文案：**

- 用户只给了想法时，为各平台写有差异的标题、正文和最多 5 个标签：抖音简短有节奏，小红书生活化可适当用 emoji，视频号正式有深度，B站接地气可稍长，快手真实直白。
- 标签不带 `#`，最多 5 个；冰爪会为 `#` 和分隔空格预留字数。朋友圈没有标题框，优先用 `caption`，正文为空时才用 `title` 兜底。
- 标题只用文字、数字、空格、连字符和下划线；书名号、引号、冒号、感叹号、问号、斜杠、括号和 Emoji 改成空格。冰爪执行前还会再清洗一次。
- 只写文案先不发：用 `get_editor_project` 读字幕（没有就 `transcribe_video`），再 `update_project_publish_copy(projectId, expectedRevision, title, caption, tags)` 写回项目，告诉用户去冰爪项目的发布页审核。

## 发布

1. 收齐素材、平台和最终文案后直接调用发布工具。**普通发布前不要调 `check_login`**：冰爪会复用登录验证缓存（抖音、快手、小红书、B站 24 小时，视频号、公众号 6 小时），目标页出现登录墙时自动转入「等待登录」。
2. **两个及以上平台必须批量**：视频只调用一次 `batch_publish`，图片帖只调用一次 `batch_publish_image_carousel`，严禁拆成多次单平台调用或脚本循环。只有一个平台时才用 `publish_video` / `publish_image_carousel`。
3. 视频批量用 `captions` 传各平台专属文案（键为平台 ID），`caption` 作通用兜底；图片帖批量用 `titles` / `captions` 覆盖平台文案。
4. 每个发布工具都带 `projectPath`：已有项目用 `get_project` 返回的绝对路径，项目外的独立发布传 `new`。
5. **视频号默认声明原创**：视频发到视频号时传 `declareOriginal: true`，只有用户说明是转载或搬运时才传 `false`。
6. 内容声明按需填（见附录）。AI 生成的内容如实声明 `ai_generated`。
7. 视频号合集：先 `get_series_presets` 取 `key`，再给视频号视频传 `seriesKey` 和 `seriesEpisode`。其它平台不传，批量时冰爪只交给视频号。
8. 用户指定未来时间时，立刻调用发布工具并传 `platformScheduleAt`（如 `2026-08-17 20:35` 或 ISO8601，北京时间）。这是平台原生定时，冰爪马上提交，到点由平台发表；不要自己等到点再调用。支持视频号（10 分钟后）、抖音（2 小时后）、小红书（1 小时后）、B站视频与图文（5 分钟后）和 YouTube 视频（15 分钟后），超过 7 天按第 7 天；其它平台不传。
9. **图片帖批次**：`batch_publish_image_carousel` 返回 `batchId` 后只调用一次 `await_publish_batch` 等终态。一个任务只提交一次：无论返回参数错误、超时还是连接失败，都不得再次提交；没拿到 `batchId` 就停下报告原错误。只有需要实时展示单次快照时才用 `get_publish_batch`，不要循环查询。`batch_publish` 视频直接返回逐平台结果，不要调用批次等待工具。

**什么时候预演**：用户明确要求，或有说得出的具体风险（新账号、素材或页面异常）时才预演。用户已授权真实发布且没有具体风险时，直接真实发布。预演会真实上传一轮，之后的真实发布还要再上传，选择预演前告诉用户原因。

## 登录

**用户要查或要登录**（最常见）：

1. 调用一次 `check_login(force_refresh=true, with_qr=true)`，不传平台即查全部。用户只问缓存状态时可以不传 `force_refresh`。
2. 回一条消息：简述状态，把结果里的二维码作为**图片**一起发出，附一句「用对应 App 扫码即可」。宿主只拿到 `/tmp/iceonix_qr_xxx.png` 这类路径时，把文件当图片发，不要把路径当文字发。
3. 到此为止：不要再 `await_login`，不要傻等，已登录的平台不打扰。

**发布结果是「等待登录」**：

1. 对该平台 `start_login`，扫码平台把二维码图片发给用户；Instagram、Facebook、X 返回 `awaiting_manual_login` 没有二维码，提示用户在已打开的 Safari 页面里登录。
2. `await_login(platform, timeout_seconds=180)`。登录后冰爪会续发原任务，不要再次调用发布工具；同批其它平台不受影响。
3. 超时只告诉用户任务仍在等待登录，不重新投递。超时不代表平台拒绝登录，Safari 标签会保留，用户仍可继续登录。

登录验证缓存不是定时保活，冰爪不会在后台定时打开 Safari。

## 汇报

- 最终消息只写总体成功、失败数量，并列出失败、待处理或结果不明的平台和简短原因。成功平台由冰爪的结果卡展示，不再逐项复述标题、正文和图片。
- 只照录工具返回的逐平台状态，网页上的按钮或页面变化不能算成功。dry-run 通过不能说成已发布。
- 结果异常或互相矛盾时，先只读核验原批次和原项目；核实不了就停下如实报告。已成功、待核验或结果未知的平台不得自动重投。明确失败也只报告，用户同意后用 `retry_publish_batch` 在原项目重试（成功平台会跳过），不得用 `projectPath: "new"` 为同一作品另建项目补发。

**有意重发**：只有用户明确说「再发一次」「强制重发」时，才用 `republish_batch(batchId, platforms, requestId)` 重发原批次里选定的平台，再用 `await_publish_batch` 等新批次。重新提供完整素材时，也可在发布工具传 `force=true` 加新 `requestId`。不得因等待、网络错误或结果未知自动 `force`；原素材已删、运行中或待核验时不得绕过。

## 红线

- 不做矩阵群发、不批量刷屏，一次一条。
- 你不进发布回路：不维护发布状态、不做去重，那是冰爪的职责。
- 朋友圈和公众号只发图片帖，YouTube 只发视频。公众号贴图只存私有草稿；朋友圈真实发表不可撤回。
- 项目里发布失败时，不得改用 CLI、拆成单平台、手写 `iceonix.json`，也不得用终端或界面工具操作、重启 Safari、微信或冰爪。只报告原始错误，等修复后复用原批次恢复。

## 附录：内容声明字段

只对对应平台生效。除 B站默认 `no_label` 外，其它平台默认 `none`。AI 生成的内容统一选 `ai_generated`。

| 字段 | 平台 | 可选值 |
|---|---|---|
| `wechatDeclaration` | 视频号 | `none` / `ai_generated` / `fiction`（虚构剧情）/ `personal_opinion` / `marketing` / `self_shot`（自行拍摄）/ `reprint` |
| `bilibiliDeclaration` | B站 | `no_label` / `ai_generated` / `fiction` / `marketing` / `personal_opinion` / `reprint` |
| `douyinDeclaration` | 抖音 | `none` / `ai_generated` / `personal_opinion` / `reprint` / `marketing` / `fiction` |
| `xhsDeclaration` | 小红书 | `none` / `ai_generated` / `fiction` / `marketing`（利益相关）/ `source_self`（自制）/ `source_repost`（转载） |
| `kuaishouDeclaration` | 快手 | `none` / `ai_generated` / `fiction` / `personal_opinion` / `repost` |

- `declareOriginal`（布尔）是否声明原创，与内容声明是两回事；视频号视频默认 `true`。
- `location`：默认建议 `"none"`（不显示位置，清除视频号等平台自动读取的城市）；`"auto"` 保留平台默认定位；也可传具体地点。视频号单独覆盖用 `wechatLocation`。

## 附录：CLI

只在运行环境里没有冰爪 MCP 时使用，输出为 JSON 或文本。

```bash
iceonix platforms
iceonix check-login [--force-refresh]

# 单平台真实发布（必须已有授权）
iceonix publish --platform douyin --video "/绝对路径/demo.mp4" \
  --title "标题" --caption "正文" --tags "标签1,标签2" --xhs-declaration ai_generated --confirm
# 可选：--declare-original --wechat-declaration --location --bilibili-declaration --douyin-declaration
#      --kuaishou-declaration --series-key <key> --series-episode <n>
#      --platform-schedule-at "2026-08-17 20:35" --facebook-audience friends

# 多平台（必须用批量）
iceonix batch-publish --platforms wechat,douyin,xhs,bilibili,youtube \
  --video "/绝对路径/demo.mp4" --cover "/绝对路径/cover.jpg" \
  --title "标题" --caption "正文" --tags "标签1,标签2" --project-path "/绝对路径/项目" --confirm

# 图片帖预演（--image 可重复 1～9 次，顺序即发布顺序）
iceonix publish --platform wechatMoments --image "/绝对路径/01.png" --image "/绝对路径/02.png" \
  --caption "朋友圈正文" --tags "冰爪" --dry-run

# 有意重发原批次
iceonix republish --batch-id <UUID> --platforms <IDs> --force --confirm [--wait]
```

CLI 不支持扫码登录，请用 MCP `start_login`，或让用户提前在冰爪里登录。
