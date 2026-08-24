# XVault：X 收藏到 Obsidian 的本地知识管线

> 当前状态：技术方案完成，代码尚未实施。仓库为私有仓库。

XVault 计划通过 X 官方 OAuth 2.0 和 Bookmarks API，全量/增量读取用户自己的收藏，将原始数据、媒体和结构化 Markdown 写入指定 Obsidian Vault，再执行可回溯的中文分诊、深度分析、主题聚类和主题综述。

## 设计原则

- X 密码、Cookie、OTP、TOTP setup key、恢复码永不进入程序。
- OAuth token 和 AI API key 仅存 macOS Keychain。
- 只申请 bookmark.read、tweet.read、users.read、offline.access。
- 不使用浏览器抓取 X。
- 不删除本地知识；X 中移除只标记 inactive。
- AI 输出必须区分事实、作者主张、推断和用户验证。
- Obsidian 更新只改 managed block，不覆盖用户编辑。
- GitHub 不保存真实 Vault、收藏、raw、数据库、媒体或日志。

## 文档

按顺序阅读：

1. [技术设计总纲](docs/TECHNICAL_DESIGN.md)
2. [数据、同步与 Obsidian 写入规范](docs/DATA_SYNC_SPEC.md)
3. [AI 处理与知识综合规范](docs/AI_PROCESSING_SPEC.md)
4. [安全、运行维护与测试规范](docs/SECURITY_OPERATIONS_TESTING.md)
5. [M0–M8 实施计划](docs/IMPLEMENTATION_PLAN.md)
6. [GPT-5.6-Luna-xhigh 开发交接](docs/LUNA_HANDOFF.md)
7. [Agent 强制规则](AGENTS.md)

## 推荐实施方式

1. 在 Codex 中打开本仓库。
2. 选择 GPT-5.6-Luna、reasoning effort=xhigh。
3. 复制 LUNA_HANDOFF.md 的首次提示词。
4. 只实施 M0。
5. M0 门禁通过后再逐步执行 M1–M8。
6. 每个里程碑单独分支、单独评审，不一次性实现全部。

## 外部前提

真正接入时需要用户亲自完成：

- X Developer App 与当前 API credits；
- X OAuth 浏览器授权；
- 明确的 Obsidian Vault 绝对路径；
- 是否启用云端 AI；
- AI provider 与 Keychain 凭据；
- 任何可能产生费用的 live test。

## 当前不应做的事

- 不要向仓库提交真实配置或收藏数据。
- 不要创建 write scope。
- 不要用 Cookie/Playwright 替代 X API。
- 不要在尚未完成 M0 时开始真实 OAuth。
- 不要把 AI 摘要作为独立事实核验。

## 官方依赖

- [X OAuth 2.0 PKCE](https://docs.x.com/fundamentals/authentication/oauth-2-0/user-access-token)
- [X Bookmarks API](https://docs.x.com/x-api/posts/bookmarks/introduction)
- [Get bookmarks](https://docs.x.com/x-api/users/get-bookmarks)
- [X API billing](https://docs.x.com/x-api/fundamentals/post-cap)

实际开发前应重新核对官方 API 字段、计费和访问策略。