# SNNLU 组织自动化：AI 审查 + 自动合并

这个仓库是组织级自动化的**单一真源**：审查逻辑只在这里改一处，所有仓库通过 `workflow_call` 复用。

```
SNNLU/.github/.github/workflows/ai-review-merge.yml   ← 审查逻辑（仅组织管理员可改，仓库无法本地削弱）
             ↑ workflow_call + secrets: inherit
<仓库>/.github/workflows/ai-review-merge.yml          ← 十几行调用者
<仓库>/.github/workflows/ci.yml                       ← 仓库自己的构建门禁（充当必需状态检查）
<仓库> ruleset protect-main                            ← 强制走 PR + 必需检查
```

## 已经生效并实测过的部分（SNNLU/snnlu-web）

| 项目 | 状态 | 验证方式 |
| --- | --- | --- |
| Allow auto-merge | 已开 | `gh api repos/SNNLU/snnlu-web --jq .allow_auto_merge` → `true` |
| ruleset `protect-main`（id 25221258） | 已生效 | 直推被拒：`GH013 … Changes must be made through a pull request. Required status check "build" is expected.` |
| CI job `build` | 实跑绿 | `npm ci → npm run build → 校验产物`，21s 通过 |
| 必需状态检查名 | `build` | 从 check-runs API 读回，非猜测 |

## 还需要你做的三件事

### 1. 配置 secret（不配的话 AI 审查会明确跳过，不会误批）

```bash
gh secret set LLM_API_KEY --repo SNNLU/snnlu-web    # DeepSeek / OpenAI 兼容 key
gh secret set BOT_TOKEN  --repo SNNLU/snnlu-web     # 另一个身份的 token，见下
```

可选（不设则默认 `https://api.deepseek.com/v1` + `deepseek-chat`）：

```bash
gh variable set LLM_BASE_URL --repo SNNLU/snnlu-web --body "https://api.deepseek.com/v1"
gh variable set LLM_MODEL    --repo SNNLU/snnlu-web --body "deepseek-chat"
```

### 2. `BOT_TOKEN` 必须是「另一个身份」——这是实测结论，不是建议

两个硬约束：

1. **GitHub 禁止任何身份批准自己的 PR**。如果 `BOT_TOKEN` 用的是开 PR 那个账号的 token，审批一定失败。
2. **SNNLU 当前的组织策略明确禁止 `GITHUB_TOKEN` 审批 PR**。实测：

   ```
   PUT /repos/SNNLU/snnlu-web/actions/permissions/workflow
   → 409 Conflict: The organization does not allow GitHub Actions to create or approve pull requests
   ```

   也就是说，网上大量教程里写的「用 `secrets.GITHUB_TOKEN` 去 `gh pr review --approve`」在 SNNLU **跑不通**。

做法二选一：

- **机器账号**（最简单）：注册一个专用账号（如 `snnlu-bot`），邀请进组织（免费组织席位不限），给目标仓库 write 权限，用它的 **fine-grained PAT** 设置 `BOT_TOKEN`（权限：Contents: read、Pull requests: write）。
- **GitHub App**（更规范、不占账号）：建 App → 安装到组织 → 用 `actions/create-github-app-token` 生成 token。注意 GitHub App 不能作为 CODEOWNERS 的 owner。

### 3. 相位二：加上「必须 1 个审批」

现在 ruleset 里 `required_approving_review_count = 0`，目的是**先不把你锁在门外**：因为 bot 还批不了，而你自己不能批自己的 PR，一旦要求 1 个审批，你自己的 PR 就只能靠 bypass 合并。

等 `BOT_TOKEN` 配好、并在某个真实 PR 上确认 bot 能成功 approve 之后，再执行：

```bash
gh api repos/SNNLU/snnlu-web/rulesets/25221258 > rs.json
python -c "import json;d=json.load(open('rs.json'));[r['parameters'].__setitem__('required_approving_review_count',1) for r in d['rules'] if r['type']=='pull_request'];json.dump(d,open('rs2.json','w'),ensure_ascii=False)"
gh api -X PUT repos/SNNLU/snnlu-web/rulesets/25221258 --input rs2.json
```

## 组织级设置（需要 `admin:org` scope）

当前登录的 token 只有 `gist, read:org, repo, workflow`，所以组织级操作还没做。先提权：

```bash
gh auth refresh -h github.com -s admin:org
```

然后可以配组织级 secret（多个仓库时省事）：

```bash
gh secret set LLM_API_KEY --org SNNLU --visibility all
gh secret set BOT_TOKEN   --org SNNLU --visibility all
```

关于「允许 GitHub Actions 创建或审批 PR」这个组织开关：**不建议打开**。打开之后，任何对仓库有写权限的人都能提交一个 PR、再让自己的 workflow 去 approve 它，直接绕过 required approvals —— 这正是 GitHub 当初加这个开关要防的事。用独立身份的 `BOT_TOKEN` 是更干净的方案。如果确实要开：

```bash
gh api -X PUT orgs/SNNLU/actions/permissions/workflow -F can_approve_pull_request_reviews=true
```

## 新增仓库时怎么接入

1. **Actions → New workflow → 搜 “AI Review & Auto-Merge”**（本仓库 `workflow-templates/` 提供的组织模板），提交即可。
2. 自己加一份 `ci.yml`，job 名建议就用 `build`。
3. 仓库 Settings → General → Pull requests：勾 Allow auto-merge、Automatically delete head branches。
4. Settings → Rules → 建 branch ruleset 保护默认分支：必须走 PR + 必需状态检查（填第 2 步的 job 名）。
5. secrets：用组织级共享，或逐仓库配 `LLM_API_KEY` / `BOT_TOKEN`。

## 安全边界（必须知晓）

- **AI 审批不等于人工审批。** diff 里可以藏指令诱导模型放行（prompt injection）。工作流已在提示词层面声明 diff 不可信，但这只是降低概率，不是防护墙。所以：凡是改动 `.github/**` 或 CODEOWNERS 的 PR，一律强制人工审批，不做自动审批与自动合并。
- **fork 来的 PR 拿不到 secrets**，因此不会被自动审查，走人工。这是刻意的取舍——用 `pull_request_target` 能让 fork PR 也拿到 secrets，但那等于把密钥暴露给不可信代码，不值得。
- 审查会把 diff 内容发给模型服务商。公开仓库的代码本来就是公开的，但仍需知晓。
- ruleset 给「仓库管理员」留了 `bypass_mode: pull_request` 的 bypass（应急可不等检查合 PR）。不需要的话，从规则里去掉 `bypass_actors`。

## 验证状态：哪些测过，哪些没测

**已实测：** 直推拦截、CI 门禁绿、必需检查名 `build`、ruleset 生效、无 checkout 取 diff（`gh pr diff -R`）。
**尚未实测（因缺 secret）：** 真实的 bot approve、auto-merge 排队合并、`--request-changes` 驳回路径。配好 `BOT_TOKEN` 后拿一个测试 PR 跑一遍即可确认。
