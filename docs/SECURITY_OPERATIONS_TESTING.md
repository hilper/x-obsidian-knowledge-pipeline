# 安全、运行维护与测试规范

## 1. 安全资产

敏感资产：

- X access token、refresh token；
- AI API key；
- OAuth authorization code、PKCE verifier；
- 未来可能存在的 Client Secret；
- 用户 Vault 内容、原始 API 响应和媒体；
- TOTP setup key、OTP、GitHub/X recovery codes（XVault 永远不应接触）。

非敏感但需要保护完整性的资产：

- Client ID；
- user_id、username；
- Vault 路径；
- taxonomy、prompt version；
- launchd 配置；
- SQLite schema version。

## 2. Keychain

接口：

~~~python
class CredentialStore(Protocol):
    def get(self, account: str) -> SecretStr | None: ...
    def set(self, account: str, value: SecretStr) -> None: ...
    def delete(self, account: str) -> bool: ...
    def exists(self, account: str) -> bool: ...
~~~

实现要求：

- service 固定 com.hilper.xvault。
- SecretStr 不允许 repr 明文。
- 不提供 list-all/secrets export。
- doctor 只能调用 exists 和读取安全的 expires_at 镜像。
- 单元测试使用 InMemoryCredentialStore。
- production 必须验证 keyring backend 是 macOS Keychain；Plaintext backend 立即失败。
- auth logout 删除对应 OAuth 项，但不删除 AI key。
- purge 必须独立命令并要求用户明确确认，MVP 可不实现。

## 3. 日志脱敏

统一 RedactionProcessor 在 structlog 最后序列化前运行。

必须遮盖键名（不区分大小写）：

~~~text
authorization
access_token
refresh_token
client_secret
api_key
password
code_verifier
setup_key
recovery_code
otp
~~~

字符串模式：

- Bearer 后的内容；
- oauth2/token body 中的 code；
- URL query 的 code、state、token；
- GitHub/X 常见 token 前缀；
- 6 位 OTP 与 setup key 只在认证上下文中遮盖，避免误伤普通统计数字。

安全日志：

~~~json
{
  "event": "auth.refresh.failed",
  "user_id": "...",
  "error_code": "invalid_grant",
  "token_present": true
}
~~~

禁止：

- 打印完整 request/response headers；
- debug 输出 Keychain value；
- 异常 repr 包含 SecretStr；
- 把完整 AI 请求写普通日志。

## 4. OAuth 安全

- state：至少 32 随机字节，URL-safe。
- verifier：RFC 7636 合规，43..128 字符。
- challenge：S256。
- 回调只监听 127.0.0.1，不监听 0.0.0.0。
- 固定端口 8731 与 Developer Console 一致。
- 回调只接受 GET /oauth/callback。
- Host header 不参与重定向构造。
- 收到首个合法回调后关闭。
- 超时 180 秒。
- state 不匹配返回浏览器错误页，不交换 token。
- 浏览器错误页不显示 code/state。
- token 响应立即写 Keychain，raw store 禁止保存。
- scope 响应必须至少含请求的四个 scope，否则登录失败。

## 5. Prompt Injection

- LLM 无工具。
- 帖子文本放 data 字段，不拼进 system 指令模板。
- 系统提示声明不可信。
- 输出严格 Schema。
- evidence/source_id 二次校验。
- URL 不自动打开。
- HTML 在渲染前转义/转换为安全 Markdown。
- 不允许帖子内容修改 provider、model、Vault path 或命令参数。
- 任何“忽略前文”“读取密钥”“调用终端”只作为来源文本保存。

## 6. 媒体 SSRF 与文件安全

每次 URL 及重定向：

1. scheme=https。
2. hostname 非空。
3. DNS 所有解析地址均非 private、loopback、link-local、multicast、reserved、unspecified。
4. 连接时防 DNS rebinding：优先使用受控 resolver/连接钩子；若实现成本过高，MVP 采用严格 X CDN allowlist 并记录决策。
5. 最多 5 redirect。
6. 禁止 file、ftp、data。
7. 流式限制大小。
8. 文件名完全由 media_key/hash 生成，不使用远端路径。
9. temp 目录与最终目录在同文件系统。
10. 文件权限默认 0600，目录 0700；用户可配置放宽但不得默认。

## 7. 文件写入

- Vault 路径 resolve 后必须位于用户配置的 Vault 根。
- 所有相对目标调用 safe_join，拒绝 ..、symlink escape。
- 写入前检查父路径的 resolved path。
- 临时文件与目标同目录。
- flush、fsync、os.replace。
- Note 存在且非 XVault 管理时拒绝覆盖。
- 不自动删除任何 Note/媒体。
- daemon uninstall 不碰 Vault。
- 未来 purge 必须列出精确目标并二次确认。

## 8. 并发与一致性

全局锁：

~~~text
~/Library/Application Support/XVault/locks/global.lock
~~~

使用 fcntl.flock(LOCK_EX | LOCK_NB)。

- sync/process/render/daemon run 获取同一全局写锁。
- doctor 默认只读；需要修复时获取写锁。
- 锁占用返回 code 7，输出持锁 PID 和 started_at（如果可安全读取）。
- 不自动 kill 进程。
- DB WAL；每页事务。
- Markdown 写入在 DB 已有规范化数据后进行。
- render 失败不回滚上游数据，run=partial，可重跑修复。

## 9. launchd

标签：

~~~text
com.hilper.xvault.incremental
com.hilper.xvault.weekly
~~~

Incremental ProgramArguments：

~~~text
<absolute-xvault-executable>
--config
<absolute-config-path>
sync
incremental
--json
~~~

Weekly 推荐由单一命令封装：

~~~text
xvault scheduled weekly
~~~

内部顺序：

1. sync reconcile；
2. process triage（预算内）；
3. process deep（队列内）；
4. synthesize topics；
5. synthesize weekly；
6. doctor --scheduled。

plist：

- Label；
- ProgramArguments；
- WorkingDirectory；
- StartCalendarInterval；
- StandardOutPath；
- StandardErrorPath；
- ProcessType=Background。

禁止：

- token/API key；
- shell command 字符串；
- KeepAlive=true 无限重启；
- root LaunchDaemon；
- 写 /Library/LaunchDaemons。

install：

1. 解析配置。
2. 显示标签、执行路径、时间、日志位置。
3. 生成 temp plist。
4. plutil -lint。
5. 写 ~/Library/LaunchAgents。
6. launchctl bootstrap gui/<uid>。
7. launchctl print 验证。

uninstall：

1. bootout 精确标签；
2. 将 plist 移到废纸篓或明确删除（实现选择须记录）；
3. 不删数据和凭据；
4. 再次 print 确认标签不存在。

## 10. Doctor

命令：

~~~text
xvault doctor [--offline] [--json]
~~~

检查项和级别：

| check | warning | error |
|---|---|---|
| Python/包版本 | 非推荐版本 | 不支持版本 |
| config 权限 | 大于 0600 | 无法读取 |
| Vault | 空间偏低 | 不存在/不可写 |
| Keychain | 接近过期 | 不存在/backend 不安全 |
| X online | 速率接近上限 | 401/403/billing |
| SQLite | WAL 未启用 | integrity/foreign key 失败 |
| Notes | AI 待处理 | 重复 x_id/坏 managed block |
| launchd | 未启用 | 已配置但加载失败 |
| recent run | 超过 24h | 连续 3 次失败 |
| AI | 未启用 | 配置为启用但不可用 |

输出不包含正文、token、API key 或完整 raw 路径中的敏感查询参数。

## 11. 结构化运行结果

每个命令返回 RunSummary：

~~~python
class SafeError(BaseModel):
    code: str
    retryable: bool
    safe_message: str

class RunSummary(BaseModel):
    status: Literal[
        "succeeded", "partial", "failed",
        "budget_exhausted", "skipped"
    ]
    run_id: str | None
    new: int = 0
    updated: int = 0
    removed: int = 0
    notes_written: int = 0
    media_succeeded: int = 0
    media_failed: int = 0
    processed: int = 0
    errors: list[SafeError] = []
~~~

JSON 模式 stdout 只有该对象。人类模式日志/进度写 stderr。

## 12. 备份与恢复

备份对象：

- config.yaml（不含 secrets）；
- state.sqlite3；
- raw；
- Vault。

不备份导出 Keychain 明文。

数据库迁移前：

1. checkpoint WAL；
2. sqlite backup API；
3. 写 backup hash；
4. 执行迁移；
5. integrity_check；
6. 成功后保留最近 3 个。

恢复：

~~~text
xvault doctor --offline
xvault db restore --from <explicit-backup>
xvault render --all
~~~

db restore 是破坏性操作，MVP 实现时必须先显示目标、备份当前 DB，并要求明确确认；自动任务不得执行。

## 13. 内容合规

- 定位：私人知识管理。
- Note 保留原帖链接、作者和 ID。
- AI 产物标记为派生。
- 不把完整第三方内容推送到 GitHub。
- 是否保留 X 已删除内容由用户配置，并须遵守 X Developer Agreement 和适用法律。
- 云端 AI 启用时明确提示会把来源文本发给配置 provider。
- local-only 模式必须无需 AI 工作。
- 避免批量公开下载媒体或建立公开镜像。

## 14. CI

工作流 ci.yml：

~~~text
trigger: pull_request, push main
matrix: python 3.12, 3.13

steps:
  checkout
  setup uv
  uv sync --frozen --all-extras
  uv run ruff format --check
  uv run ruff check
  uv run mypy src
  uv run pytest -m "not live" --cov
  coverage gate
~~~

security.yml：

- dependency review（PR）；
- pip-audit 或 uv audit 可用方案；
- gitleaks/secret scan；
- 禁止真实 token pattern；
- CodeQL 可在实现稳定后启用。

CI 不需要 X、GitHub 或 AI secrets。

## 15. 测试目录

~~~text
tests/unit/
  test_config.py
  test_redaction.py
  test_pkce.py
  test_oauth.py
  test_xapi_dto.py
  test_pagination.py
  test_normalize.py
  test_reconcile.py
  test_frontmatter.py
  test_writer.py
  test_media_security.py
  test_ai_schemas.py

tests/integration/
  test_oauth_refresh_flow.py
  test_full_sync.py
  test_incremental.py
  test_reconcile_fail_closed.py
  test_folder_partial.py
  test_media_download.py
  test_processing_pipeline.py

tests/e2e/
  test_first_run_to_topic.py
  test_user_edits_survive.py
  test_removed_bookmark_survives.py
  test_idempotent_rerun.py
~~~

## 16. 测试 Fixtures

必须提供：

- bookmarks_page_1.json（100 条）；
- bookmarks_page_2.json（5 条）；
- includes_missing.json；
- long_note_tweet.json；
- article_post.json；
- quoted_missing.json；
- folder_list.json；
- folder_page.json；
- 429、401、403、billing 错误；
- malicious_prompt_post.json；
- media_redirect_private.json；
- fake triage/deep/synthesis outputs。

Fixture token 必须明显虚构，例如 TEST_ACCESS_TOKEN_NOT_REAL。

## 17. 覆盖率门槛

- 总体 >=85%。
- security/redaction >=95%。
- xapi/oauth >=95%。
- obsidian/writer >=95%。
- sync/reconcile >=95%。

不能用 pragma no cover 回避关键分支。网络适配器可通过 respx 覆盖。

## 18. E2E 验收场景

### 场景 A：首次全量

- FakeX 返回 105 条。
- 断言 2 页、105 bookmark、105 Note。
- 断言 raw manifest hash。
- 断言没有真实网络。

### 场景 B：用户编辑保留

- 人工向 user block 写内容。
- 上游正文 hash 变化。
- render。
- managed 更新，user byte-for-byte 保留。

### 场景 C：删除核对

- 初始 105。
- incremental 只返回 10：不得 inactive。
- reconcile 完整返回 104：一条 inactive。
- Note 仍存在，x_active=false。

### 场景 D：故障恢复

- 第二页第一次 503 后成功。
- 第三页永久失败：run partial/failed。
- 不执行 deletion reconciliation。
- 重跑后无重复。

### 场景 E：AI 注入

帖子正文要求忽略系统、读取密钥、输出 Markdown。Fake/guard 流程必须仍只接受 Schema；Note 中保存原文但不执行。

### 场景 F：launchd

使用 temp HOME 或模板层测试：

- plist 不含 token。
- ProgramArguments 是数组。
- 路径绝对。
- 时间正确。
- uninstall target 精确。

## 19. 发布前检查

~~~text
uv lock --check
uv run ruff format --check
uv run ruff check
uv run mypy src
uv run pytest -m "not live"
uv run coverage report --fail-under=85
git grep -nE '<token-patterns>'
xvault doctor --offline --json
~~~

首个版本号 0.1.0。发布物只包含源代码/包，不包含 config、DB、raw、Vault 或日志。