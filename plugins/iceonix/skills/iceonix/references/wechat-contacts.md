# 微信加好友

仅在用户明确要求时使用。冰爪在已登录的 Mac 微信里处理用户给出的手机号或完整微信号；你只调用冰爪工具，不自己操作微信。默认只搜索预演，不提交申请。昵称或备注名不能代替微信号。

## 号码格式

- 以 `1` 开头的 11 位纯数字一律按国内手机号搜索。
- 海外号码必须由用户提供带 `+` 国际区号的完整号码，`+` 后共 7～15 位数字（示例为虚构号码 `+12025550123`）。不要根据号码或地区猜国家码。
- 微信号：6～64 位，以英文字母开头，其余只含英文字母、数字、下划线或短横线，保留原有大小写。这是冰爪当前接受的范围，不代表微信官方规则。
- 纯数字按手机号判断，以 `+` 开头按国际手机号判断，不猜用户想搜哪类账号。海外号码和微信号的真实搜索尚未完成跨账号验收，遇到无法确认的结果要停下核验。

冰爪 App 的小工具可以从 CSV、TSV 或 XLSX 导入号码（XLSX 有多个含内容的工作表时，请先另存为单工作表或 CSV）。手机号可含空格、短横线和括号，`+1 2025550123` 会统一为 `+12025550123`，有效的 `+86` 号码会转成 11 位形式。MCP 与 CLI 只接收账号列表，不接收表格路径。导入后先核对有效账号、重复项和人数，再启动任务。

## 调用

| MCP 工具 | CLI（仅没有 MCP 时） | 用途 |
| --- | --- | --- |
| `start_wechat_contacts` | `iceonix wechat-contacts start` | 启动批次，默认仅预演 |
| `get_wechat_contacts` | `iceonix wechat-contacts get` | 查询原批次状态 |
| `continue_wechat_contacts` | `iceonix wechat-contacts continue` | 核对上一条后处理下一人 |
| `stop_wechat_contacts` | `iceonix wechat-contacts stop` | 停止批次；已提交的申请无法撤回 |

1. 每批生成一个 UUID 作 `requestId`，向 `start_wechat_contacts` 传去重后的 `accounts`（CLI `--request-id` 和可重复的 `--account`）。省略 `sendRequests` 或设为 `false` 只预演。
2. 只有用户明确授权了目标账号、申请附言和每个账号的申请次数，才传 `sendRequests=true` 和非空 `greeting`（CLI `--send --greeting`）。附言 1～50 字，不含换行。`maxSubmissionAttempts` 为每账号最多申请 1～3 次，默认 1；设为 2 或 3 可能产生重复申请。
3. `maxBatchCount` 设置本批最多处理 1～500 人，默认 10。冰爪的每日申请人数上限默认 10，可由用户调整；`maxDailyContacts` 只能调低本批上限，不能超过冰爪设置。当天对同一账号再次申请只计一人，预演不计人数。CLI 对应 `--max-batch-count`、`--max-daily-contacts`、`--max-submission-attempts`。
4. 启动后用原 `requestId` 调 `get_wechat_contacts`。进入 `awaiting_review` 时核对上一条结果；结果明确且整批已获授权，才用 `continue_wechat_contacts` 处理下一条。结果不明时先请用户亲自核对微信，确认后才传 `reviewConfirmed=true`（CLI `--review-confirmed`）。
5. 连接超时、响应丢失或结果不明时，只用原 `requestId` 查询。不要换 ID 重新启动，也不要把预演授权当成真实申请授权。同一 ID 的参数不可变。

## 结果判断

| 状态码 | 含义与下一步 |
| --- | --- |
| `already_friend` | 已是好友，不再申请 |
| `request_pending` | 等待对方验证，不再申请 |
| `not_found` | 找不到该用户：核对完整微信号或手机号，海外号码还要核对国际区号 |
| `account_status_abnormal` | 微信提示被搜账号状态异常、无法显示（不是找不到）。停下由用户核验，不自动重搜或申请 |
| `dry_run_verified` | 预演看到可添加，没有提交 |
| `applied` | 已提交并显示等待验证，还不是好友 |
| `submission_attempted` | 已尝试提交但结果未确认，先由用户核验 |
| `rate_limited`、`submission_rate_limited` | 遇到频控，停止本批；后者保留已尝试的申请次数 |
| `daily_limit_reached`、`daily_quota_unavailable` | 今日上限已满或额度记录不可用，停止真实申请 |
| `needs_review`、`failed` | 查看原因和微信当前状态，不自动重投 |

目标已是好友、等待验证或遇到频控时，冰爪会停止后续申请。未经确认的提交结果只能记为待核验。
