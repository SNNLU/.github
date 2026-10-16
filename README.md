# SNNLU 组织自动化：AI 审查 + 自动合并

本仓库是组织级自动化的**单一真源**：审查与合并策略只在这里改一处，所有仓库通过 `workflow_call` 复用。
各仓库**无法在本地削弱这道门禁** —— 想放宽必须改这个仓库。

```
SNNLU/.github/.github/workflows/ai-review-merge.yml   ← 策略与判定逻辑（仅组织管理员可改）
             ↑ workflow_call + secrets: inherit
<仓库>/.github/workflows/ai-review-merge.yml          ← 十几行调用者
<仓库>/.github/workflows/ci.yml                       ← 仓库自己的门禁（必需状态检查 build + content）
<仓库> ruleset protect-main                            ← 强制走 PR + 必需检查通过
```

## 策略：两条车道

| | 内容车道 | 代码车道 |
| --- | --- | --- |
| 触发条件 | 改动**全部**是 `content/` 下的 `.md` / `.mdx`，且文件数 ≤ 30 | 其余一切（站点代码、`site.json`、`layout/*.json`、`public/` 资源、`.github/`、README 等） |
| AI 审查 | 宽松：只拦"大病"（头部块损坏、缺 date、违规内容、密钥泄露、可疑外链） | 严格：bug、运行时异常、安全漏洞、构建风险 |
| 合并 | AI 通过 + 必需检查绿 → **自动合并** | **绝不自动合并**，由维护者亲自点 merge |
| AI 判"有问题" | 提交 request-changes 挡住合并 | 提交 request-changes 挡住合并 |

两条车道的判定是**白名单**：只有确定是纯文章的 PR 才走内容车道，新出现的文件类型默认落回代码车道（必须人工合并）。

## 已实测验证的部分

**内容结构校验**（`.github/scripts/validate-content.mjs`，作业名 `content`，本地构造用例全跑过）：

| 用例 | 结果 |
| --- | --- |
| 头部齐全的新闻 + 只有 title 的页面 | 退出码 0 |
| 缺 `date` | 报错："缺 date 会让列表接口在运行时崩掉" |
| `date: 2026/10/16` | 报错：格式应为 `YYYY-MM-DD` 或 `YYYY-MM-DD HH:mm` |
| 没有头部块（文件不以 `---` 开头） | 报错 |
| 文件名不含数字（文章 id 由文件名数字推导） | 报错 |
| `folder` 与所在目录不一致 | warning |
| 改动不涉及 content/ | 退出码 0，不误伤代码 PR |

**判定逻辑**（用桩 `gh` + 真实 `jq` 在本地把工作流的每一步整段执行，10 个用例全通过）：

| 场景 | 实际行为 |
| --- | --- |
| 内容车道 + AI 通过 | 发出 `pr merge --auto --squash` ✅ |
| 内容车道 + 门禁已全绿（GitHub 拒绝排队） | 自动回退为直接 `pr merge --squash` ✅ |
| 内容车道 + AI 驳回 | `request-changes`，**零**合并调用 ✅ |
| 代码车道 + AI 通过 | **零**合并调用，只留言提示由维护者合并 ✅ |
| 代码车道 + AI 驳回 | `request-changes` ✅ |
| 模型调用失败（HTTP 500） | 不合并、不封禁，留言说明"未给出可用结论" ✅ |
| 未配置 `LLM_API_KEY` | 明确跳过（`::notice`），不合并 ✅ |
| 上一轮 bot 驳回后作者已改好 | 撤销**自己**那条 request-changes 再合并 ✅ |
| 人工（非 bot）的 request-changes | **不碰** ✅ |

**仓库门禁**（SNNLU/snnlu-web）：

- 直推 main 被拒（实测）：`GH013 … Changes must be made through a pull request. Required status check "build" is expected.`
- 必需检查实测名：`build`（CI 实跑 21–30s 通过）、`content`（3s 通过）；ruleset id `25221258`，`required_approving_review_count = 0`。
- `Allow auto-merge` 已开；合并后自动删除源分支（实测生效）。

## 还需要你做的

### 1. 配置 `LLM_API_KEY`（不做的话两条车道都不会自动合并）

```bash
gh secret set LLM_API_KEY --repo SNNLU/snnlu-web     # DeepSeek / OpenAI 兼容 key
```

可选（默认 `https://api.deepseek.com/v1` + `deepseek-chat`）：

```bash
gh variable set LLM_BASE_URL --repo SNNLU/snnlu-web --body "https://api.deepseek.com/v1"
gh variable set LLM_MODEL    --repo SNNLU/snnlu-web --body "deepseek-chat"
```

`BOT_TOKEN` **现在不是必需的**：合并动作只需要 `GITHUB_TOKEN`（`contents: write` + `pull-requests: write`），
组织那条"禁止 Actions 创建或审批 PR"的策略只挡审批，不挡合并。若哪天出现
`Resource not accessible by integration`，再配 `BOT_TOKEN`（独立身份的 PAT 或 GitHub App token）即可 ——
工作流会自动改用它。**注意** `BOT_TOKEN` 不能是你自己的 PAT：GitHub 禁止自己批准自己的 PR。

### 2. 发布链路（**目前的真实缺口**）

`nuxt.config.ts` 里 `nitro.serverAssets` 把 `content/` 打包进服务端产物 —— 也就是说：

> **合并到 main 并不会让文章上线。** 必须有一次构建 + 部署。

仓库里目前没有任何部署配置（无 Pages、无 environments、无 Dockerfile/wrangler/vercel 文件），
所以"别人马上就能看到"这一步还没有着落。告诉我站点跑在哪（自建 VPS + nginx/PM2？Docker？Cloudflare/Vercel/Zeabur？），
我可以在内容车道自动合并之后接一个部署作业，做成"文章 PR 合并 → 自动构建 → 自动上线"。

### 3. 组织级设置（需要 `admin:org`，当前 token 没有）

```bash
gh auth refresh -h github.com -s admin:org
gh secret set LLM_API_KEY --org SNNLU --visibility all   # 多个仓库时共享，省去逐仓库配置
```

关于"允许 GitHub Actions 创建或审批 PR"这个组织开关：**仍然不建议打开**。打开后任何有写权限的人
都能用自己的 workflow 自审批自己的 PR，绕过 required approvals。当前设计不需要它。

### 4. 新增仓库怎么接入

1. **Actions → New workflow → 搜 "AI Review & Auto-Merge"**（本仓库 `workflow-templates/` 提供），提交。
2. 自己加 `ci.yml`：至少一个构建作业 + 一个内容校验作业（可从 snnlu-web 复制 `validate-content.mjs`）。
3. Settings → General → Pull requests：勾 **Allow auto-merge**、**Automatically delete head branches**。
4. Settings → Rules → 新建 branch ruleset 保护默认分支：要求走 PR + 必需状态检查填上面的作业名。

## 安全边界（必须知晓）

- **AI 审批 ≠ 人工审批。** diff 里可以藏指令诱导模型放行（prompt injection）。提示词里已声明 diff 不可信，但那是降低概率，不是防护墙。这也是为什么**代码车道永不自动合并**。
- 内容车道之所以可以自动合并：文章是数据不是代码；渲染器 `server/utils/markdown.ts` 对每块内容都做 HTML 转义（`<script>`、事件属性进不来）；而且只有同仓库分支的 PR 能拿到 secrets —— **fork 来的 PR 不会被自动审查、也不会被自动合并**，一律走人工。
- 想放宽内容车道（例如允许连同图片一起自动合并）：改 `ai-review-merge.yml` 里分流那一段的 `^content/.+\.mdx?$` 正则即可，改一处所有仓库生效。
- 密钥只放在 secrets 里；审查会把 diff 内容发给模型服务商（公开仓库的代码本来就是公开的，但仍需知晓）。
- ruleset 给"仓库管理员"留了 `bypass_mode: pull_request` 的 bypass（应急可不等检查合 PR），不需要可从 `bypass_actors` 去掉。

## 待办 / 已知未验证

- **真实 auto-merge 尚未跑过**：内容车道的合并需要 `LLM_API_KEY`，链路的合并/回退分支已用桩测覆盖，但第一次真实合并要等你配好 key 开一篇真文章。
- 部署不存在，合并 ≠ 上线（见上文第 2 条）。
- 组织级 secret / ruleset 需要 `admin:org` scope（见上文第 3 条）。
