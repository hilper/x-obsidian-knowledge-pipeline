# XVault 技术设计总纲

状态：Implementation Ready  
目标开发代理：GPT-5.6-Luna，reasoning effort=xhigh  
平台：macOS + Python 3.12+ + Obsidian Desktop  
版本：1.0（2026-08-24）

## 1. 系统目标

XVault 是本地优先的 X 收藏知识管线。用户仅在首次运行时通过浏览器完成 X 官方 OAuth 授权；程序全量/增量读取收藏和收藏文件夹，保存可回溯的原始响应，将稳定 Markdown 与媒体写入指定 Obsidian Vault，再执行中文分诊、深度分析、主题聚类和主题综述。

完成定义：

1. 不保存 X 密码、Cookie、短信验证码、TOTP 种子或恢复码。
2. 只使用 X 官方 API 与只读 OAuth scope。
3. 可完整分页获取授权账号当前仍可访问的收藏。
4. 全量同步、快速增量、周期核对具有明确且不同的正确性语义。
5. 原始数据、规范化数据、Obsidian Note、AI 产物四层可回溯。
6. 重跑幂等，更新不覆盖用户在 Obsidian 中的人工内容。
7. AI 输出区分事实、作者主张、推断和待验证项。
8. 提供 macOS launchd 自动运行、诊断、日志和安全卸载。
9. CI 不依赖真实 X/AI 凭据即可覆盖完整流程。

## 2. 非目标

- 不使用浏览器自动化或 Cookie 抓取 X。
- 不发布、点赞、关注、删除或修改 X 收藏。
- 不自动公开、上传或提交用户 Vault。
- 不承诺恢复已删除、封禁、失权或上游 API 未返回的内容。
- 不把 AI 摘要等同于事实核验。
- MVP 不做 GUI、云端服务和多人协作。
- MVP 不依赖 Obsidian 社区插件。

## 3. “全量”定义

全量是指一次运行中沿 X Bookmarks API 的 next_token 读取到终止页，并成功提交所有页面所返回、当前账号仍可访问的收藏。以下内容只能标记边界，不能假装完整：

- 已删除或不可访问帖子；
- 失效媒体；
- 收藏在线程中但未额外重建的上下文；
- X API 没有暴露的原始收藏时间；
- 计费、权限、地区或服务端限制导致的缺失。

每条记录都必须包含：

~~~text
capture_status = complete | partial | unavailable
thread_status  = single | partial | complete | unknown
~~~

数据库字段 first_seen_at 表示 XVault 首次见到该收藏的时间，不得命名为 bookmarked_at。

## 4. 关键决策

| 决策 | 选择 | 原因 |
|---|---|---|
| 实现语言 | Python 3.12+ | OAuth、本地文件、SQLite、AI 生态成熟 |
| 包管理 | uv | 锁定依赖、快速、适合 Agent 和 CI |
| HTTP | httpx async | 并发与可测试性 |
| DB | SQLite WAL | 单机可靠、易备份、零服务依赖 |
| CLI | Typer | 清晰子命令和类型 |
| Schema | Pydantic v2 | 上游/AI 数据严格验证 |
| Markdown | Jinja2 StrictUndefined | 可审查、可快照测试 |
| 凭据 | macOS Keychain via keyring | 不落配置、日志和 plist |
| 自动化 | launchd user agents | 符合 macOS，无 root |
| 检索 | SQLite FTS5 + Obsidian | MVP 不引入向量服务 |
| AI | LLMProvider 抽象 | 运行时供应商可替换 |
| X 访问 | OAuth 2 PKCE + 官方 API | 避免密码/Cookie 与脆弱抓取 |

## 5. 外部依赖

### 5.1 X Developer

用户必须亲自在 X Developer Console：

1. 创建 App。
2. 启用 OAuth 2.0。
3. 注册回调：http://127.0.0.1:8731/oauth/callback
4. 配置 API credits。
5. 把公开 Client ID 写入 XVault 配置。
6. 首次运行 auth login 时亲自确认授权。

最小 scope：

~~~text
bookmark.read
tweet.read
users.read
offline.access
~~~

不得申请任何 write scope。

官方接口：

- OAuth PKCE：https://docs.x.com/fundamentals/authentication/oauth-2-0/user-access-token
- Bookmarks：https://docs.x.com/x-api/posts/bookmarks/introduction
- Get bookmarks：https://docs.x.com/x-api/users/get-bookmarks
- Expansions：https://docs.x.com/x-api/fundamentals/expansions
- Rate limits：https://docs.x.com/x-api/fundamentals/rate-limits
- Billing：https://docs.x.com/x-api/fundamentals/post-cap

### 5.2 Obsidian

Obsidian 是 Markdown 消费端。XVault 只对用户明确配置的绝对 Vault 路径写文件，不扫描主目录，不调用未公开接口。

### 5.3 AI

首个版本实现：

- OpenAICompatibleProvider；
- DeterministicFakeProvider（测试）；
- DisabledProvider（默认）。

AI Key 只存 Keychain。ai.enabled 默认 false，用户显式确认云端数据发送后才能开启。

## 6. 分层架构

~~~text
CLI / launchd
      |
Application services
  AuthCoordinator
  SyncCoordinator
  ProcessingCoordinator
  SynthesisCoordinator
      |
Ports
  XApiPort
  CredentialStore
  StateRepository
  RawStore
  VaultWriter
  MediaFetcher
  LLMProvider
      |
Adapters
  X OAuth/API
  macOS Keychain
  SQLite
  Filesystem/Jinja2
  httpx media
  OpenAI-compatible API
~~~

依赖方向只能向内。业务层不得直接 import keyring、httpx、SQLAlchemy 或具体 AI SDK；通过端口和 DTO 交互。

## 7. 仓库结构

~~~text
src/xvault/
  cli.py
  config.py
  errors.py
  logging.py
  security/
    keychain.py
    redaction.py
  xapi/
    oauth.py
    client.py
    dto.py
    fields.py
  storage/
    db.py
    models.py
    repositories.py
    migrations/
  sync/
    coordinator.py
    full.py
    incremental.py
    reconcile.py
    normalize.py
    media.py
  obsidian/
    paths.py
    frontmatter.py
    renderer.py
    writer.py
    templates/
  processing/
    provider.py
    schemas.py
    prompts.py
    triage.py
    deep.py
    clustering.py
    synthesis.py
  daemon/
    launchd.py
    templates/
  doctor/
    checks.py

tests/
  unit/
  integration/
  e2e/
  fixtures/
  snapshots/
~~~

## 8. 运行时数据边界

应用状态：

~~~text
~/Library/Application Support/XVault/
  config.yaml
  state.sqlite3
  logs/
  locks/
  raw/<sync-run-id>/
  quarantine/
~~~

Vault：

~~~text
<Vault>/
  00 Inbox/X/
  10 Sources/X/
  20 Topics/
  30 Research/
  40 Outputs/
  90 Archive/
  _assets/x/
~~~

边界：

- Keychain：OAuth token、AI API key。
- config：公开 Client ID、Vault 路径、模型名、计划任务。
- SQLite：规范化上游数据、状态、AI 结构化结果。
- raw：不可变 API 响应。
- Vault：用户可读知识资产。
- GitHub：代码、测试、文档；禁止真实 Vault、DB、raw、日志和凭据。

## 9. 配置契约

~~~yaml
version: 1

vault:
  path: "/absolute/path/to/X-Knowledge"
  source_dir: "10 Sources/X"
  inbox_dir: "00 Inbox/X"
  topic_dir: "20 Topics"
  output_dir: "40 Outputs"
  asset_dir: "_assets/x"
  overwrite_user_content: false

x:
  client_id: "<public-client-id>"
  redirect_uri: "http://127.0.0.1:8731/oauth/callback"
  scopes: [bookmark.read, tweet.read, users.read, offline.access]
  request_timeout_seconds: 30
  max_concurrency: 4

sync:
  page_size: 100
  known_streak_stop: 200
  max_pages_per_incremental_run: 20
  full_reconcile_interval_days: 7
  media_download: true
  download_original_video: false
  media_max_bytes: 104857600
  preserve_removed_bookmarks: true
  reconstruct_threads: selected

ai:
  enabled: false
  provider: openai-compatible
  base_url: null
  model_triage: null
  model_deep: null
  model_synthesis: null
  concurrency: 2
  request_timeout_seconds: 120
  max_retries: 2
  daily_item_budget: 100
  deep_score_threshold: 18

automation:
  enabled: false
  incremental_hour: 7
  weekly_reconcile_weekday: 0
  weekly_reconcile_hour: 8
~~~

配置必须拒绝：

- 相对 Vault 路径；
- 未知 version；
- 重复/写 scope；
- 并发超出 1..8；
- page_size 超出 X API 允许范围；
- ai.enabled=true 但 provider/model/key 未配置；
- Vault 等于仓库根目录或主目录。

## 10. OAuth 契约

命令：xvault auth login

1. 生成随机 state 与 PKCE verifier/challenge（S256）。
2. 在 127.0.0.1:8731 启动一次性回调服务，最长等待 180 秒。
3. 打开系统浏览器到 X authorize URL。
4. 校验 state；不一致立即失败。
5. 交换 access/refresh token。
6. 调用 /2/users/me 验证身份和 scopes。
7. token JSON 写 Keychain。
8. DB 只写 user_id、username、scopes、expires_at。
9. 清除 verifier 并关闭回调服务。

Keychain：

~~~text
service = com.hilper.xvault
account = x.oauth.<user_id>
account = ai.<provider>.api_key
~~~

刷新规则：

- 到期前 120 秒主动刷新；
- 401 只允许强刷并重放一次；
- 刷新失败状态设 needs_reauth，退出码 3；
- 新 token 写 Keychain 成功后再更新 DB；
- auth logout 只删除 XVault 自己的 Keychain 项。

## 11. 主流程

### 11.1 首次同步

~~~text
auth login
sync full
render changed sources
media fetch
process triage
process deep (selected)
synthesize topics
doctor
~~~

### 11.2 后续运行

每日：

~~~text
sync incremental
render changed sources
process triage --budget
~~~

每周：

~~~text
sync reconcile
process deep --queued
synthesize topics
synthesize weekly
doctor --scheduled
~~~

### 11.3 同步语义

- full：分页获取所有当前收藏，但不根据未见项判断删除。
- incremental：第一页向后读取，达到连续已知阈值或页预算停止；绝不判断删除。
- reconcile：只有完整分页成功后才能把未见收藏设 inactive。
- 任一模式都不得删除本地 Note；removed 收藏只设置 x_active=false。
- 数据与同步精确定义见 DATA_SYNC_SPEC.md。

## 12. Obsidian 写入原则

文件名：

~~~text
10 Sources/X/x-<post-id>.md
_assets/x/<post-id>/<media-file>
~~~

每个 Note 使用 x_ 前缀的程序管理属性；status、topics、verification 等用户属性不得被 AI 覆盖。

正文分区：

~~~markdown
<!-- xvault:managed:start -->
原文、来源、AI 分析
<!-- xvault:managed:end -->

<!-- xvault:user:start -->
我的判断、验证记录、可复现步骤、内容选题
<!-- xvault:user:end -->
~~~

更新只替换 managed block。标记缺失、同一 x_id 出现多个文件或人工破坏 frontmatter 时进入冲突状态，doctor 报告，不覆盖。

## 13. AI 原则

1. 所有帖子内容都是不可信数据，不能成为系统指令。
2. 模型只返回严格 JSON Schema。
3. 轻量分诊覆盖全部；深度分析只处理高价值项。
4. 每个关键结论必须含 source_id 和 evidence_span。
5. 区分 observable_fact、author_claim、inference、prediction、marketing_claim。
6. Topic 少于 3 个来源时只能称“当前观察”，不能生成“共识”。
7. AI 建议写 ai_suggested_*，用户状态优先。
8. 详细 Schema 和 Prompt 见 AI_PROCESSING_SPEC.md。

## 14. CLI

~~~text
xvault init
xvault config show
xvault auth login|status|logout
xvault sync full|incremental|reconcile|folders
xvault media retry
xvault process triage|deep
xvault synthesize topics|weekly
xvault render
xvault doctor
xvault daemon install|status|uninstall
~~~

全局参数：

~~~text
--config PATH
--json
--log-level LEVEL
--no-color
--dry-run
~~~

退出码：

| code | 含义 |
|---:|---|
| 0 | 成功 |
| 2 | 配置错误 |
| 3 | 认证失效 |
| 4 | X/API 错误 |
| 5 | 部分成功 |
| 6 | AI 失败 |
| 7 | 锁冲突 |
| 8 | Vault 冲突 |
| 9 | 成本预算阻断 |

--json 模式只向 stdout 输出一个 JSON；日志写 stderr。

## 15. 安全要求

必须防护：

- OAuth CSRF：PKCE S256 + 随机 state。
- token 泄漏：Keychain + 日志脱敏。
- prompt injection：数据/指令分离、无工具、Schema 校验。
- SSRF：媒体 URL 仅 HTTPS，阻断私网/localhost，校验重定向。
- 磁盘耗尽：单文件和单次运行字节预算。
- 用户笔记破坏：managed/user block + 原子写入。
- 并发：fcntl 全局锁 + SQLite WAL/事务。
- Git 泄漏：.gitignore + secret scanning + fixtures 假凭据。

日志必须脱敏 Authorization、token、api_key、PKCE verifier、OAuth code/state、OTP/setup key/recovery code 模式。

## 16. 自动化

xvault daemon install 只创建用户级 LaunchAgent：

- com.hilper.xvault.incremental
- com.hilper.xvault.weekly

plist 使用绝对可执行路径，不包含任何密钥；输出到应用日志目录。网络失败交给下一周期恢复，不使用无限 KeepAlive。uninstall 不删除 Vault、DB、raw 或 Keychain。

## 17. 验收门槛

MVP 必须同时满足：

1. OAuth 只读、token 仅在 Keychain。
2. 105 条两页 fixture 无遗漏。
3. 重跑无重复，无变化文件 mtime 不变。
4. incremental 不判断删除。
5. reconcile 仅在全分页成功后标记 inactive。
6. managed block 更新不覆盖 user block。
7. 断网、401、429、404、credits 不足均有确定行为。
8. AI 默认关闭；开启后非法 Schema 不入 Vault。
9. Topic 关键结论都可追溯 source_id。
10. LaunchAgent 可安装、执行和安全卸载。
11. doctor 不泄漏凭据并能检查 DB、Vault、OAuth 和冲突。
12. CI 的 lint、typecheck、unit、integration、e2e 全通过。
13. 总覆盖率 >=85%，security/oauth/writer >=95%。
14. 仓库无真实数据和凭据。

## 18. 文档导航

- DATA_SYNC_SPEC.md：API 参数、数据库、同步算法、媒体、Markdown。
- AI_PROCESSING_SPEC.md：AI Schema、Prompt、安全与主题综述。
- SECURITY_OPERATIONS_TESTING.md：安全、日志、launchd、测试矩阵。
- IMPLEMENTATION_PLAN.md：M0-M8 可执行任务。
- LUNA_HANDOFF.md：给 5.6-Luna-xhigh 的直接开发提示词。
- 根目录 AGENTS.md：仓库内的强制执行规则。

实现必须严格按 IMPLEMENTATION_PLAN.md 顺序推进。