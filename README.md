# SNNLU/.github

**这里曾经放组织级的复用工作流与模板，现已废弃。**

方案改为每个仓库自带工作流，当前分工如下：

| 目标 | 现在的做法 |
| --- | --- |
| 文章（纯 Markdown）审查 + 自动合并 | `SNNLU/snnlu-web` → `.github/workflows/md-fastlane.yml` |
| 代码与配置审查 | CodeRabbit（配置在仓库根的 `.coderabbit.yaml`） |
| 合并到 main 后构建与部署 | `SNNLU/snnlu-web` → `.github/workflows/deploy-on-merge.yml` |

本仓库已不再包含任何工作流，可以安全删除；也可以留作以后组织级配置的落脚点。
