# 实施计划（M0–M8）

目标：让 GPT-5.6-Luna xhigh 或人工开发者按顺序实现，不需要重新做架构设计。

通用门禁：每个里程碑结束必须运行其列出的命令；失败不得进入下一里程碑。每个 PR 只包含一个里程碑。禁止在 M0–M8 期间改用浏览器抓取 X。

## M0：工程骨架与质量门禁

分支：codex/m0-scaffold

### 文件

~~~text
pyproject.toml
uv.lock
src/xvault/__init__.py
src/xvault/__main__.py
src/xvault/cli.py
src/xvault/constants.py
src/xvault/errors.py
src/xvault/clock.py
tests/unit/test_cli.py
.github/workflows/ci.yml
.github/workflows/security.yml
.gitignore
~~~

### 任务

1. Python requires-python >=3.12。
2. hatchling 构建，src layout。
3. 配置 uv dependency groups：dev、test。
4. 引入 typer、pydantic、pydantic-settings、httpx、sqlalchemy、aiosqlite、jinja2、keyring、tenacity、structlog、PyYAML。
5. 开发依赖：pytest、pytest-asyncio、respx、freezegun、ruff、mypy、coverage。
6. CLI 根命令输出版本和 help。
7. 定义 AppError 基类、稳定 error_code 和 exit_code。
8. 定义 Clock protocol 与 SystemClock/FrozenClock。
9. .gitignore 覆盖：
   - config.yaml；
   - state.sqlite3*；
   - raw、logs、quarantine；
   - Obsidian Vault 常见目录；
   - .env；
   - token/credential fixtures。
10. CI 执行 format check、lint、mypy、pytest。
11. security workflow 加 secret scan/dependency audit；若某 action 当前不可用，记录 TODO 而非静默移除。

### 接口

~~~python
class AppError(Exception):
    code: str
    exit_code: int
    retryable: bool
    safe_message: str
~~~

### 测试

- xvault --help。
- xvault --version。
- AppError 映射退出码。
- stdout/stderr 分离。
- import 所有模块无副作用。

### 门禁

~~~text
uv sync --all-extras
uv run ruff format --check
uv run ruff check
uv run mypy src
uv run pytest
~~~

完成标准：CI 绿色，仓库无真实凭据。

## M1：配置、路径、日志与 Keychain

分支：codex/m1-config-security

### 文件

~~~text
src/xvault/config.py
src/xvault/logging.py
src/xvault/security/keychain.py
src/xvault/security/redaction.py
src/xvault/obsidian/paths.py
tests/unit/test_config.py
tests/unit/test_paths.py
tests/unit/test_keychain.py
tests/unit/test_redaction.py
~~~

### 任务

1. 实现完整 Config Pydantic 模型。
2. 加载顺序：defaults -> YAML -> CLI overrides。
3. 拒绝相对 Vault、主目录、仓库根和越界并发。
4. 实现 safe_join，阻断 .. 与 symlink escape。
5. 实现 CredentialStore protocol。
6. MacKeychainStore 使用 keyring；拒绝 plaintext backend。
7. InMemoryCredentialStore 仅测试。
8. SecretStr 不进入 repr。
9. structlog JSON 和 human renderer。
10. RedactionProcessor 覆盖规范中的键名和模式。
11. xvault init：
    - 询问/参数获取 Vault 绝对路径；
    - 创建应用支持目录和默认 config；
    - 不创建 Keychain secret；
    - ai.enabled=false。
12. xvault config show 隐藏敏感项，只输出来源和有效值。
13. 文件权限：应用目录 0700，config 0600。

### CLI

~~~text
xvault init --vault <absolute-path>
xvault config show [--json]
~~~

### 测试

- 正常/未知 config version。
- 相对路径拒绝。
- HOME、/、仓库根拒绝。
- safe_join symlink escape。
- plaintext keyring 拒绝。
- token/header/setup key/OTP 脱敏。
- config show 无 SecretStr。

### 门禁

M0 命令 + security 模块覆盖率 >=95%。

## M2：SQLite、迁移与仓储层

分支：codex/m2-storage

### 文件

~~~text
src/xvault/storage/db.py
src/xvault/storage/models.py
src/xvault/storage/repositories.py
src/xvault/storage/migrations/0001_initial.sql
src/xvault/storage/migrations/runner.py
tests/unit/test_storage_models.py
tests/integration/test_migrations.py
tests/integration/test_repositories.py
~~~

### 任务

1. 按 DATA_SYNC_SPEC 创建所有表与索引。
2. 初始化 PRAGMA WAL/foreign_keys/synchronous/busy_timeout。
3. 实现 schema_version。
4. migration 前 SQLite backup API。
5. Repository protocols：
   - SyncRunRepository；
   - UserRepository；
   - PostRepository；
   - BookmarkRepository；
   - MediaRepository；
   - FolderRepository；
   - ProcessingRepository；
   - TopicRepository。
6. 每页事务 UnitOfWork。
7. upsert 返回 created/updated/unchanged。
8. content_hash 不包含 public_metrics。
9. 启动时把超时 running run 标 abandoned。
10. doctor DB integrity helper。

### 测试

- 空库迁移。
- 重复迁移幂等。
- 备份存在。
- foreign key 开启。
- upsert 状态。
- metrics 变化不改 content_hash。
- 事务回滚。
- abandoned run。
- integrity_check。

### 门禁

数据库集成测试全部通过；migration 可从空库运行两次。

## M3：X OAuth 与 API Client

分支：codex/m3-xapi

### 文件

~~~text
src/xvault/xapi/oauth.py
src/xvault/xapi/client.py
src/xvault/xapi/dto.py
src/xvault/xapi/fields.py
src/xvault/xapi/errors.py
tests/unit/test_pkce.py
tests/unit/test_xapi_dto.py
tests/integration/test_oauth_flow.py
tests/integration/test_xapi_retry.py
tests/fixtures/xapi/
~~~

### 任务

1. 实现 PKCE S256、state、固定 localhost callback。
2. 浏览器打开接口通过 BrowserOpener protocol，测试不真开。
3. 回调 server 只绑定 127.0.0.1:8731。
4. 校验 path、method、state、timeout。
5. token exchange 后 Keychain 保存。
6. /2/users/me 验证身份和 scopes。
7. 主动刷新与 401 单次刷新。
8. XApiClient 使用单一 httpx AsyncClient。
9. 实现 bookmarks/folders/threads 所需 DTO。
10. 实现完整 fields 与一次兼容降级。
11. 实现 429/reset、5xx backoff、credits、403、404 语义。
12. 不打印 headers/body 中敏感项。
13. auth status 在线/离线。
14. auth logout 精确删除 XVault OAuth key。

### CLI

~~~text
xvault auth login
xvault auth status [--offline]
xvault auth logout
~~~

### 测试

- verifier 长度和 S256。
- state mismatch。
- callback timeout。
- scope 缺失。
- token 保存失败不更新 DB。
- 到期前刷新。
- 401 刷新成功/再次 401。
- 429。
- 403/billing。
- 字段降级仅一次。
- includes 缺失。
- 所有错误日志脱敏。

### 门禁

oauth/security 覆盖率 >=95%；无 live token。

## M4：Raw、全量/增量/核对与文件夹

分支：codex/m4-sync

### 文件

~~~text
src/xvault/sync/coordinator.py
src/xvault/sync/full.py
src/xvault/sync/incremental.py
src/xvault/sync/reconcile.py
src/xvault/sync/normalize.py
src/xvault/storage/raw.py
src/xvault/locking.py
tests/unit/test_pagination.py
tests/unit/test_normalize.py
tests/integration/test_full_sync.py
tests/integration/test_incremental.py
tests/integration/test_reconcile_fail_closed.py
tests/integration/test_folder_partial.py
~~~

### 任务

1. 实现 RawStore 原子写、0600、manifest hash。
2. 实现全局 fcntl 非阻塞锁。
3. 实现 next_token 循环检测。
4. 每页 raw 后 DB 事务。
5. 规范化 users/includes/references/media/bookmarks。
6. 单条坏 DTO quarantine + run partial。
7. Full 完整分页，但不判断删除。
8. Incremental known_streak/page_budget，绝不 inactive。
9. Reconcile 使用 temp seen 表，终止页成功后才 inactive。
10. reappeared 恢复 active。
11. Folder 同步独立错误语义。
12. RunSummary/结构化错误。
13. API 预算中止。
14. 命令取消时 run=cancelled。
15. running 超时恢复。

### CLI

~~~text
xvault sync full
xvault sync incremental
xvault sync reconcile
xvault sync folders
~~~

### Fixture

精确提供 100 + 5 条两页数据、空 includes、坏 DTO、重复 next_token、folder failure。

### 测试

以 DATA_SYNC_SPEC 的 20 个场景为准，尤其：

- 105 条无遗漏；
- 重跑无重复；
- incremental 不删除；
- failed reconcile 不删除；
- successful reconcile inactive；
- removed 重现；
- budget stop 一致。

### 门禁

所有同步集成测试通过；sync/reconcile >=95%。

## M5：Obsidian Renderer 与媒体

分支：codex/m5-obsidian

### 文件

~~~text
src/xvault/obsidian/frontmatter.py
src/xvault/obsidian/renderer.py
src/xvault/obsidian/writer.py
src/xvault/obsidian/templates/source.md.j2
src/xvault/obsidian/templates/topic.md.j2
src/xvault/obsidian/templates/weekly.md.j2
src/xvault/sync/media.py
tests/unit/test_frontmatter.py
tests/unit/test_renderer.py
tests/unit/test_writer.py
tests/unit/test_media_security.py
tests/integration/test_render_flow.py
tests/integration/test_media_download.py
tests/snapshots/
~~~

### 任务

1. 实现 Note 路径定位和重复 x_id 冲突。
2. frontmatter 分离 managed x_ 与 user fields。
3. managed/user block 精确解析。
4. 模板 StrictUndefined。
5. user block byte-for-byte 保留。
6. render_hash；无变化不写、mtime 不变。
7. temp + fsync + os.replace。
8. source note 显示完整度、来源、AI 元数据占位。
9. 媒体 URL 校验、DNS/allowlist、重定向、大小、MIME、hash。
10. 视频默认不下载原文件。
11. media failure 不阻断 Note。
12. removed 只更新 x_active=false。
13. render --all 可从 DB 重建全部受管 Note。

### CLI

~~~text
xvault render [--all] [--post-id ID]
xvault media retry [--post-id ID]
~~~

### 测试

- frontmatter 未知字段保留。
- user block 保留。
- managed block 0/2 个冲突。
- 重复 x_id。
- symlink escape。
- mtime。
- SSRF 私网。
- redirect 私网。
- 超大文件。
- MIME 冲突 quarantine。
- media partial。

### 门禁

writer >=95%；snapshot review 通过。

## M6：AI 分诊、深度分析和 Topic

分支：codex/m6-ai

### 文件

~~~text
src/xvault/processing/provider.py
src/xvault/processing/schemas.py
src/xvault/processing/prompts.py
src/xvault/processing/triage.py
src/xvault/processing/deep.py
src/xvault/processing/clustering.py
src/xvault/processing/synthesis.py
tests/unit/test_ai_schemas.py
tests/unit/test_prompt_guard.py
tests/integration/test_processing_pipeline.py
tests/integration/test_topic_synthesis.py
tests/fixtures/ai/
~~~

### 任务

1. 实现三个 Provider。
2. API key 从 Keychain。
3. 固定 triage.v1/deep.v1/synthesis.v1。
4. 实现 AI_PROCESSING_SPEC 全部 Pydantic Schema。
5. 输入长度/引用数限制。
6. strict structured output。
7. source_id 白名单。
8. evidence_span 规范化匹配。
9. 非法 JSON 一次修复，二次 quarantine。
10. processing_job 去重。
11. 分数代码计算。
12. 深度触发/优先队列/每日预算。
13. 规则 + AI Topic，user > rule > ai。
14. 用户 exclusion。
15. Topic 少于 3 来源无 consensus。
16. 将结果渲染进 managed block。
17. 云端 AI 首次启用提示并写 consent_at（非敏感）。
18. ai.disabled 下同步完全可用。

### CLI

~~~text
xvault process triage [--limit N] [--post-id ID]
xvault process deep [--limit N] [--post-id ID]
xvault process requeue --prompt-version <v>
xvault synthesize topics
xvault synthesize weekly
~~~

### 测试

以 AI_PROCESSING_SPEC 的矩阵为准；CI 全部使用 FakeProvider。

### 门禁

恶意帖子 fixture 不改变指令；非法 evidence 不入 Note；Topic 可追溯。

## M7：Doctor、launchd 与端到端

分支：codex/m7-operations

### 文件

~~~text
src/xvault/doctor/checks.py
src/xvault/daemon/launchd.py
src/xvault/daemon/templates/*.plist.j2
src/xvault/scheduled.py
tests/unit/test_doctor.py
tests/unit/test_launchd.py
tests/e2e/test_first_run_to_topic.py
tests/e2e/test_user_edits_survive.py
tests/e2e/test_removed_bookmark_survives.py
tests/e2e/test_idempotent_rerun.py
~~~

### 任务

1. 实现 SECURITY_OPERATIONS_TESTING 的 doctor 检查。
2. JSON 与人类输出。
3. 生成两个 user LaunchAgent。
4. plutil lint。
5. bootstrap/print/bootout 封装成 SystemService protocol，测试 fake。
6. plist 无 secrets、ProgramArguments 数组、路径绝对。
7. scheduled weekly 顺序与部分失败语义。
8. E2E FakeX + FakeAI + temp Vault。
9. 性能基准 1000 条。
10. 日志保留 30 天。
11. 失败连续计数和健康摘要。

### CLI

~~~text
xvault doctor [--offline]
xvault daemon install
xvault daemon status
xvault daemon uninstall
xvault scheduled weekly
~~~

### 门禁

全部 E2E 通过；1000 条达到技术设计性能目标。

## M8：文档、Live smoke 与 0.1.0

分支：codex/m8-release

### 文件

~~~text
README.md
docs/USER_GUIDE.md
docs/TROUBLESHOOTING.md
docs/PRIVACY.md
docs/X_DEVELOPER_SETUP.md
CHANGELOG.md
LICENSE
~~~

### 任务

1. 首次安装、X App、OAuth、Vault、AI、launchd 完整教程。
2. 说明 X API 当前计费需在 Console 核对。
3. 说明全量边界和收藏时间语义。
4. 说明云端 AI 数据发送。
5. 说明备份、恢复、卸载。
6. Live smoke 只读最多 10 条，人工显式。
7. 完整 secret scan。
8. 版本 0.1.0。
9. GitHub Release 仅在用户授权时创建；默认只准备。
10. 生成最终验收报告，列命令与结果。

### 发布门禁

~~~text
uv lock --check
uv run ruff format --check
uv run ruff check
uv run mypy src
uv run pytest -m "not live"
uv run coverage report --fail-under=85
xvault doctor --offline --json
~~~

## PR 规则

每个 PR：

1. 标题：[M#] 简述。
2. 描述包含目标、实现、非范围、测试证据、安全影响、后续。
3. 不混入下一里程碑。
4. 至少一次 self-review。
5. CI 绿色。
6. 不提交 generated Vault、DB、raw、media、log。
7. 修改架构必须先改 docs 并说明原因。

## Luna 停止条件

遇到以下情况必须停止并请求用户，不得自行扩大权限：

- 需要真实 X Client ID、API credits 或 OAuth 授权；
- 需要真实 AI API key；
- Vault 路径未知；
- X 官方字段/计费与文档冲突；
- 必须申请 write scope 才能继续；
- 需要删除/覆盖非受管文件；
- 发现仓库已有未预期用户改动；
- Live test 会产生费用而用户未确认。

## 最终 Definition of Done

- M0–M8 全部合并。
- CI 全绿。
- 105 条与 1000 条场景通过。
- 凭据只在 Keychain。
- 首次全量、增量、核对、Note 更新、AI、Topic、launchd 全流程验证。
- 无用户内容覆盖。
- 无真实数据进入 GitHub。
- doctor 给出健康状态。
- 文档允许新用户独立安装。