# 数据、同步与 Obsidian 写入规范

本文是 TECHNICAL_DESIGN.md 的强制细化。实现中若代码与本文冲突，以本文和根目录 AGENTS.md 为准。

## 1. X API 请求

### 1.1 当前用户

~~~http
GET https://api.x.com/2/users/me
Authorization: Bearer <access-token>
~~~

用途：OAuth 成功后的身份确认，以及 auth status 的可选在线检查。

### 1.2 收藏分页

~~~http
GET https://api.x.com/2/users/<user-id>/bookmarks
~~~

参数：

~~~text
max_results=100
pagination_token=<only-when-present>
tweet.fields=id,text,author_id,created_at,conversation_id,attachments,referenced_tweets,entities,lang,possibly_sensitive,note_tweet,article,edit_history_tweet_ids
expansions=author_id,attachments.media_keys,referenced_tweets.id,referenced_tweets.id.author_id
user.fields=id,name,username,profile_image_url,verified,created_at
media.fields=media_key,type,url,preview_image_url,alt_text,duration_ms,variants,width,height
~~~

兼容策略：

1. 首次请求使用完整字段集。
2. 若 400 明确指出字段不支持，记录 event=xapi.fields.downgrade。
3. 移除不支持字段后只重试一次。
4. 兼容字段集集中定义在 xapi/fields.py；不得散落在业务代码。
5. 任意响应先保存 raw，再解析。
6. includes 缺失是合法输入，不能 KeyError。
7. ID 永远按字符串处理。

### 1.3 收藏文件夹

~~~http
GET /2/users/<user-id>/bookmarks/folders
GET /2/users/<user-id>/bookmarks/folders/<folder-id>
~~~

文件夹失败时：

- 总收藏正文继续；
- sync_run.status=partial；
- error code=FOLDER_SYNC_FAILED；
- 不把已有 membership 设 inactive；
- 下次 folder sync 可独立重试。

## 2. 分页协议

PageResult DTO：

~~~python
class PageResult(BaseModel):
    data: list[PostDTO] = []
    includes: IncludesDTO = IncludesDTO()
    result_count: int
    next_token: str | None
    raw: dict[str, Any]
~~~

循环约束：

~~~text
token = None
seen_token_hashes = set()
page_number = 0

while True:
  response = fetch(token)
  page_number += 1
  raw_store.write(run_id, page_number, response)
  parse_and_commit(response)

  if response.next_token is None:
    break

  token_hash = sha256(response.next_token)
  if token_hash in seen_token_hashes:
    raise PaginationLoop
  seen_token_hashes.add(token_hash)
  token = response.next_token
~~~

- next_token 不跨运行复用。
- 只保存 token hash 到日志/DB，raw 中的上游 token 文件权限必须为 0600。
- 连续两页 data 为空但仍有 next_token，终止并返回 UPSTREAM_PROTOCOL_ERROR。
- page_size 固定 100，除测试和官方限制变化外不自动缩小。
- 每页提交一个 DB 事务；单页失败不伪装整次成功。

## 3. HTTP 重试矩阵

| 情况 | 行为 |
|---|---|
| connect timeout、read timeout | 指数退避 + jitter，最多 5 次 |
| 502/503/504 | 同上 |
| 429 | 读取 reset header；等待不超过配置上限，否则延后 |
| 401 | refresh token，然后重放一次 |
| 第二次 401 | needs_reauth，退出码 3 |
| 403 | 不重试；报告 scope/账户/地区限制 |
| 402/credits | 不重试；billing_blocked，退出码 9 |
| 单条引用 404 | target unavailable；主流程继续 |
| 主 bookmarks 404 | 终止，退出码 4 |
| 400 | 字段兼容降级一次，否则开发错误 |
| JSON 无法解析 | 最多重试一次，raw 进入 quarantine |

每次请求设置 User-Agent，格式：

~~~text
XVault/<version> (+https://github.com/hilper/x-obsidian-knowledge-pipeline)
~~~

日志不得包含 Authorization header、完整 OAuth code 或 refresh token。

## 4. 原始存储

路径：

~~~text
~/Library/Application Support/XVault/raw/<run-id>/
  manifest.json
  bookmarks-page-0001.json
  bookmarks-page-0002.json
  folders.json
  folder-<id>-page-0001.json
~~~

写入：

1. 序列化为 UTF-8 JSON，键排序、末尾换行。
2. 先写同目录临时文件。
3. flush + fsync。
4. chmod 0600。
5. os.replace 原子替换。
6. manifest 记录 sha256、bytes、kind、page、created_at。
7. raw 不参与 Markdown 渲染，只作为回溯证据。

RawStore 协议：

~~~python
class RawStore(Protocol):
    async def put_json(
        self,
        run_id: str,
        logical_name: str,
        payload: Mapping[str, Any],
    ) -> RawObjectRef: ...
~~~

## 5. SQLite 设置

连接初始化：

~~~sql
PRAGMA journal_mode=WAL;
PRAGMA foreign_keys=ON;
PRAGMA synchronous=NORMAL;
PRAGMA busy_timeout=5000;
~~~

测试和 doctor 可运行：

~~~sql
PRAGMA integrity_check;
PRAGMA foreign_key_check;
~~~

所有迁移前自动备份 DB 到 state.sqlite3.backup-<timestamp>；迁移成功后保留最近 3 个，失败不得替换原库。

## 6. 核心表

### 6.1 sync_runs

~~~sql
CREATE TABLE sync_runs (
  id TEXT PRIMARY KEY,
  kind TEXT NOT NULL CHECK(kind IN ('full','incremental','reconcile','folder')),
  started_at_ms INTEGER NOT NULL,
  finished_at_ms INTEGER,
  status TEXT NOT NULL CHECK(status IN (
    'running','succeeded','partial','failed',
    'budget_exhausted','cancelled','abandoned'
  )),
  pages_fetched INTEGER NOT NULL DEFAULT 0,
  resources_read INTEGER NOT NULL DEFAULT 0,
  new_bookmarks INTEGER NOT NULL DEFAULT 0,
  updated_bookmarks INTEGER NOT NULL DEFAULT 0,
  missing_bookmarks INTEGER NOT NULL DEFAULT 0,
  error_code TEXT,
  error_summary TEXT
);
~~~

启动时把超过 6 小时仍为 running 的记录设 abandoned。

### 6.2 users

~~~sql
CREATE TABLE users (
  id TEXT PRIMARY KEY,
  username TEXT NOT NULL,
  display_name TEXT,
  profile_image_url TEXT,
  verified INTEGER,
  raw_json TEXT NOT NULL,
  content_hash TEXT NOT NULL,
  first_seen_at_ms INTEGER NOT NULL,
  last_seen_at_ms INTEGER NOT NULL
);
~~~

### 6.3 posts

~~~sql
CREATE TABLE posts (
  id TEXT PRIMARY KEY,
  author_id TEXT,
  text TEXT NOT NULL,
  lang TEXT,
  created_at_ms INTEGER,
  conversation_id TEXT,
  possibly_sensitive INTEGER,
  canonical_url TEXT NOT NULL,
  raw_json TEXT NOT NULL,
  content_hash TEXT NOT NULL,
  capture_status TEXT NOT NULL CHECK(capture_status IN (
    'complete','partial','unavailable'
  )),
  thread_status TEXT NOT NULL CHECK(thread_status IN (
    'single','partial','complete','unknown'
  )),
  first_seen_at_ms INTEGER NOT NULL,
  last_seen_at_ms INTEGER NOT NULL,
  FOREIGN KEY(author_id) REFERENCES users(id)
);
CREATE INDEX posts_conversation_idx ON posts(conversation_id);
CREATE INDEX posts_author_idx ON posts(author_id);
~~~

canonical_url 使用 includes 中的 username；缺失时使用 https://x.com/i/web/status/<id>。

content_hash 输入为规范化对象：

~~~json
{
  "id": "...",
  "text": "...",
  "author_id": "...",
  "created_at": "...",
  "conversation_id": "...",
  "attachments": {},
  "referenced_tweets": [],
  "lang": "...",
  "note_tweet": {},
  "article": {}
}
~~~

禁止把 public_metrics 放入 content_hash，避免点赞数变化导致反复重写 Note。

### 6.4 bookmarks

~~~sql
CREATE TABLE bookmarks (
  post_id TEXT PRIMARY KEY,
  active INTEGER NOT NULL DEFAULT 1,
  first_seen_at_ms INTEGER NOT NULL,
  last_seen_at_ms INTEGER NOT NULL,
  removed_at_ms INTEGER,
  source_note_path TEXT,
  render_hash TEXT,
  triage_status TEXT NOT NULL DEFAULT 'pending',
  deep_status TEXT NOT NULL DEFAULT 'not_queued',
  user_status TEXT NOT NULL DEFAULT 'inbox',
  FOREIGN KEY(post_id) REFERENCES posts(id)
);
~~~

### 6.5 media

~~~sql
CREATE TABLE media (
  media_key TEXT PRIMARY KEY,
  post_id TEXT NOT NULL,
  media_type TEXT NOT NULL,
  remote_url TEXT,
  preview_url TEXT,
  alt_text TEXT,
  width INTEGER,
  height INTEGER,
  duration_ms INTEGER,
  variants_json TEXT,
  local_path TEXT,
  sha256 TEXT,
  byte_size INTEGER,
  download_status TEXT NOT NULL DEFAULT 'pending' CHECK(download_status IN (
    'pending','downloaded','linked','failed','quarantined','skipped'
  )),
  last_error TEXT,
  raw_json TEXT NOT NULL,
  FOREIGN KEY(post_id) REFERENCES posts(id)
);
CREATE INDEX media_post_idx ON media(post_id);
~~~

### 6.6 references

~~~sql
CREATE TABLE post_references (
  source_post_id TEXT NOT NULL,
  target_post_id TEXT NOT NULL,
  relation_type TEXT NOT NULL CHECK(relation_type IN (
    'quoted','replied_to','retweeted'
  )),
  PRIMARY KEY(source_post_id,target_post_id,relation_type)
);
~~~

target 尚未获取时允许先只记录 ID；不要创建违反外键的假 post。

### 6.7 folders

~~~sql
CREATE TABLE bookmark_folders (
  id TEXT PRIMARY KEY,
  name TEXT NOT NULL,
  raw_json TEXT NOT NULL,
  last_seen_at_ms INTEGER NOT NULL
);

CREATE TABLE bookmark_folder_memberships (
  folder_id TEXT NOT NULL,
  post_id TEXT NOT NULL,
  active INTEGER NOT NULL DEFAULT 1,
  last_seen_at_ms INTEGER NOT NULL,
  PRIMARY KEY(folder_id,post_id)
);
~~~

### 6.8 processing 和 topic

~~~sql
CREATE TABLE processing_jobs (
  id TEXT PRIMARY KEY,
  post_id TEXT,
  job_type TEXT NOT NULL CHECK(job_type IN ('triage','deep','synthesis')),
  input_hash TEXT NOT NULL,
  provider TEXT NOT NULL,
  model TEXT NOT NULL,
  prompt_version TEXT NOT NULL,
  status TEXT NOT NULL CHECK(status IN (
    'queued','running','succeeded','failed','quarantined','skipped'
  )),
  attempts INTEGER NOT NULL DEFAULT 0,
  output_json TEXT,
  error_summary TEXT,
  created_at_ms INTEGER NOT NULL,
  updated_at_ms INTEGER NOT NULL
);
CREATE UNIQUE INDEX processing_job_dedupe
ON processing_jobs(post_id,job_type,input_hash,prompt_version);

CREATE TABLE topics (
  slug TEXT PRIMARY KEY,
  title_zh TEXT NOT NULL,
  description_zh TEXT,
  managed INTEGER NOT NULL DEFAULT 1,
  note_path TEXT,
  updated_at_ms INTEGER NOT NULL
);

CREATE TABLE topic_memberships (
  topic_slug TEXT NOT NULL,
  post_id TEXT NOT NULL,
  confidence REAL NOT NULL,
  source TEXT NOT NULL CHECK(source IN ('rule','ai','user')),
  PRIMARY KEY(topic_slug,post_id,source)
);
~~~

## 7. 规范化顺序

每页事务固定顺序：

1. 解析 DTO，收集 validation warnings。
2. upsert users。
3. upsert includes 中的 referenced posts。
4. upsert主 posts。
5. upsert references。
6. upsert media。
7. upsert bookmarks，active=1，last_seen_at=now。
8. 更新 sync_run 计数。
9. commit。

任何不可解析单条：

- 保存 raw；
- 写 quarantine record；
- sync_run=partial；
- 继续同页其他对象；
- 不允许静默丢弃。

## 8. 同步算法

### 8.1 Full

Full 用于首次导入和显式重新抓取：

~~~text
lock
create full run
paginate all bookmarks
sync folders
render new/changed notes
schedule media
finish succeeded/partial
unlock
~~~

Full 不判断删除，因为文件夹或上游短暂不完整不应导致状态变化。

### 8.2 Fast incremental

~~~text
known_streak = 0
for page in bookmarks:
  for post in page:
    if known id and unchanged content_hash:
      known_streak += 1
    else:
      known_streak = 0
      upsert
  if known_streak >= configured threshold:
    stop reason=known_streak
  if pages >= max_pages:
    stop reason=page_budget
~~~

语义：

- 只保证读取过页面中出现的新/变更项。
- 绝不根据未见项标记 inactive。
- 停止原因进入 run summary。
- 每周 reconcile 修正遗漏和删除状态。

### 8.3 Reconcile

~~~text
current_seen = temporary table keyed by post_id
paginate all bookmarks successfully
insert every id into current_seen
if and only if terminal page reached:
  set active=0, removed_at=now
  where bookmarks.active=1
    and post_id not in current_seen
restore active=1 for reappeared ids
commit reconciliation transaction
~~~

如果任何页面失败：

- 不执行 inactive 更新；
- run=failed/partial；
- current_seen 临时数据丢弃；
- 已获取的新内容仍可保留；
- 用户 Note 不删除。

### 8.4 幂等

- X ID 是自然键。
- 规范化内容 hash 控制变更。
- render_hash 控制 Note 重写。
- media_key + sha256 控制媒体重复。
- processing job 由 post_id + type + input_hash + prompt_version 去重。
- 同步重跑不得改变无变化文件 mtime。
- 并发命令通过 fcntl.flock 返回退出码 7，不排队无限等待。

## 9. 媒体安全

下载器：

1. 只接收 API 返回的 URL。
2. scheme 必须 https。
3. DNS 解析后拒绝 loopback、link-local、private、reserved 地址。
4. 每次重定向重新校验。
5. 最多 5 次重定向。
6. 流式下载，Content-Length 只作为预检查。
7. 超过 media_max_bytes 立即终止并删除 temp。
8. 根据魔数和 Content-Type 判定扩展名。
9. MIME 冲突进入 quarantine。
10. 原子移动到最终路径。

路径：

~~~text
_assets/x/<post-id>/<media-key>-<sha256前12位>.<ext>
~~~

视频默认只保存 preview 和远程链接；download_original_video=false。

媒体失败只影响 media.download_status，Note 正文仍生成，run 最多 partial。

## 10. Obsidian Note

### 10.1 路径和定位

默认：

~~~text
10 Sources/X/x-<post-id>.md
~~~

查找优先级：

1. bookmarks.source_note_path 且 x_id 匹配。
2. 默认路径存在且 x_id 匹配。
3. 在受管 source_dir 中按 frontmatter x_id 精确查找。
4. 0 个：创建。
5. 多于 1 个：冲突，退出码 8，不覆盖。

禁止扫描 Vault 之外路径。

### 10.2 Frontmatter 合并

程序管理：

~~~yaml
xvault_schema: 1
x_id: "..."
x_url: "..."
x_author: "@..."
x_author_id: "..."
x_created_at: "..."
x_first_seen_at: "..."
x_active: true
x_capture_status: complete
x_thread_status: partial
x_folders: []
x_language: en
x_content_hash: "sha256:..."
x_triage_status: succeeded
x_deep_status: not_queued
~~~

用户管理：

~~~yaml
status: inbox
topics: []
verification: unverified
aliases: []
tags: []
~~~

规则：

- x_ 前缀字段可更新。
- 用户字段存在时永不覆盖。
- 新文件可写默认用户字段。
- AI 只能写 ai_suggested_status、ai_suggested_topics。
- 未识别字段原样保留。
- YAML 错误进入冲突，不自动修复。

### 10.3 Managed/User block

~~~markdown
<!-- xvault:managed:start -->
# @author：标题

## 原始内容

...

## 上下文

...

## AI 分诊

...

## 来源

- [X 原帖](...)
<!-- xvault:managed:end -->

<!-- xvault:user:start -->
## 我的判断

## 验证记录

## 可复现步骤

## 内容选题

<!-- xvault:user:end -->
~~~

更新：

1. 解析 frontmatter。
2. 校验 x_id。
3. 精确定位一个 managed block。
4. 保留 user block 和其余未知正文。
5. 生成新 managed block。
6. 计算 render_hash。
7. 无变化不写。
8. 有变化写 temp、fsync、os.replace。
9. 更新 DB source_note_path/render_hash。

managed 标记为 0 或大于 1 时不得覆盖。

### 10.4 引用和完整度

Note 必须显式显示：

- capture_status；
- thread_status；
- 引用帖/回复是否已获取；
- 媒体成功、链接或失败状态；
- AI prompt_version/model/processed_at；
- 原帖 URL 与 ID。

不得把引用帖内容拼接成原作者正文。

## 11. Topic Note

路径：

~~~text
20 Topics/<topic-slug>.md
~~~

同样使用 managed/user block。Managed 内容按数据库查询生成，用户可在 user block 写自己的结论。

每个关键句后列来源：

~~~markdown
- 方法 A 在多个案例中被重复提及。[[x-123]] [[x-456]]
~~~

来源少于 3 条时标题用“当前观察”，不得用“核心共识”。

## 12. 成本与运行预算

每个 run 记录：

- request_count；
- resource_count；
- owned_reads；
- thread_reads；
- media_bytes；
- AI input/output usage；
- estimated_cost。

支持：

~~~text
xvault sync full --max-owned-reads 10000
xvault sync incremental --max-pages 10
xvault process triage --limit 100
~~~

达到预算：

- 停在一致性边界；
- 已提交页保留；
- run=budget_exhausted；
- 返回退出码 9；
- 不把未完成 reconcile 当成功。

## 13. 必测场景

1. 105 条跨两页，无遗漏。
2. next_token 重复，检测循环。
3. includes 缺失，不崩溃。
4. 一条坏 DTO 进入 quarantine，其余入库。
5. 401 刷新成功。
6. 第二次 401 needs_reauth。
7. 429 超出等待上限。
8. 文件夹失败但正文成功。
9. Fast incremental 达阈值停止且不 inactive。
10. Reconcile 中间页失败且不 inactive。
11. 成功 reconcile 标记 removed，文件保留。
12. Removed 后重现恢复 active。
13. 人工修改 user block 后上游更新，用户内容不变。
14. managed block 缺失，冲突不覆盖。
15. 同 x_id 多文件，doctor 报错。
16. 媒体重定向到私网，被拒绝。
17. 大文件超限，temp 清理。
18. 无变化重跑 mtime 不变。
19. DB 崩溃后重跑无重复。
20. 预算停止返回结构化 summary。