# GPT-5.6-Luna-xhigh 开发交接

## 使用方式

在 Codex 中打开本仓库，选择模型 GPT-5.6-Luna，reasoning effort=xhigh。首次只让它实施 M0，不要一次要求完成 M0–M8。

## 首次可直接复制的提示词

~~~text
你现在负责实现私有仓库 hilper/x-obsidian-knowledge-pipeline。

先完整阅读根目录 AGENTS.md，然后依次完整阅读：
1. docs/TECHNICAL_DESIGN.md
2. docs/DATA_SYNC_SPEC.md
3. docs/AI_PROCESSING_SPEC.md
4. docs/SECURITY_OPERATIONS_TESTING.md
5. docs/IMPLEMENTATION_PLAN.md

本次只实施 IMPLEMENTATION_PLAN.md 的 M0，不要开始 M1，也不要改写已确认架构。

执行要求：
- 检查仓库当前状态和已有改动。
- 创建并使用分支 codex/m0-scaffold。
- 按 M0 文件清单实现 Python 3.12、uv、hatchling、Typer、质量与 CI 骨架。
- 不创建真实配置、凭据、Vault、数据库、raw 或媒体。
- Fixtures 只能是假数据。
- 运行 M0 全部门禁命令并修复到通过。
- 对安全与 .gitignore 做自检。
- 完成后报告：改动文件、架构实现、测试命令及真实结果、未实现内容、下一步。
- 不创建或合并 PR，除非我在后续明确要求。
- 如果需要真实账户、API、付费网络调用、删除/覆盖或扩大权限，停止并向我说明。
~~~

## 后续里程碑提示词模板

~~~text
继续实现 hilper/x-obsidian-knowledge-pipeline 的 M<编号>。

先重新阅读 AGENTS.md 和 docs/IMPLEMENTATION_PLAN.md 中 M<编号>，并阅读该里程碑引用的全部规范文档。检查上一里程碑门禁和仓库当前状态。

本次只做 M<编号>：
- 使用指定 codex/m<编号>-<slug> 分支。
- 实现文件、接口、错误语义和测试矩阵。
- 不扩大 OAuth scope。
- 不接触真实密码、OTP、setup key、recovery code、token 或 API key。
- 不使用浏览器抓取 X。
- 不覆盖用户 Obsidian 内容。
- CI/live test 不访问真实付费 API。
- 运行本里程碑和全局门禁，修复到通过。
- 最后报告真实证据和仍需用户完成的外部依赖。
- 不创建或合并 PR，除非我明确要求。
~~~

## 评审提示词

~~~text
对当前 M<编号> 实现做一次独立代码评审。以 AGENTS.md 和 docs 下所有规范为验收依据。

重点检查：
1. 是否越过当前里程碑；
2. OAuth 和 Keychain 是否泄漏凭据；
3. 同步删除语义是否 fail-closed；
4. 幂等、事务和原子写入；
5. Obsidian user block 是否保持；
6. prompt injection、Schema 和 evidence 校验；
7. 媒体 SSRF/大小/重定向；
8. launchd 是否含密钥或错误权限；
9. 测试是否覆盖失败分支；
10. 是否存在声称通过但未实际运行的验证。

先列按严重度排序的问题，再列验证命令。没有问题时明确说明未发现阻断项，但不要省略残余风险。
~~~

## 实施节奏

| 次序 | 里程碑 | 预期产物 |
|---:|---|---|
| 1 | M0 | 工程/CI 骨架 |
| 2 | M1 | 配置、路径、日志、Keychain |
| 3 | M2 | SQLite/迁移/Repository |
| 4 | M3 | X OAuth/API |
| 5 | M4 | 全量/增量/reconcile |
| 6 | M5 | Obsidian/媒体 |
| 7 | M6 | AI/Topic |
| 8 | M7 | doctor/launchd/E2E |
| 9 | M8 | 用户文档/发布准备 |

## 用户必须亲自完成的步骤

Luna 不得代替用户完成或索取：

- X Developer 计费与 App 创建确认；
- X OAuth 浏览器授权；
- X/AI 密钥输入；
- OTP、TOTP setup key、恢复码；
- 指定真实 Obsidian Vault；
- 任何会产生费用的 live test 确认；
- 删除/覆盖真实数据确认。

## 完成判断

模型说“完成”不构成证据。每个里程碑必须提供：

- 实际 git diff 范围；
- 实际运行的命令；
- exit code；
- 测试通过数；
- 覆盖率；
- CI 状态（若已推送）；
- 未验证的外部依赖。

如果上下文不足，重新读取仓库文档，不得凭记忆补架构。