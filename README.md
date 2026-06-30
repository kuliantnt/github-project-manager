# github-project-manager

> 一个可复用的 AI Coding Agent Skill，用于管理 GitHub Issue、Pull Request 与 GitHub Projects v2 看板。面向任意 GitHub 仓库，不修改业务代码。

<!-- TODO: 一句话卖点 /.banner / 截图 -->

## 它解决什么问题

<!-- TODO: 补场景描述 -->

日常 GitHub 项目维护常靠临时指令，每次都要重新解释规则，容易造成字段不统一、重复卡片、误关闭 Issue。本 Skill 把这套规则固化成一个可复用、可被 Agent 自动加载的 Skill。

主要能力：

- **Issue 盘点**：分类 Epic / Feature / Bug / Investigation / Documentation，识别「已完成未关闭」「重复」「范围重叠」「暂不处理」。
- **Issue 整理**：改标题/正文/标签、加迁移说明、关闭（completed / duplicate / not planned，关闭前必留说明评论，不静默关闭）。
- **Projects v2 字段管理**：读取并设置 Status / 优先级 / 类型 / 模块等字段，**先读真实 ID 全复用，不造同义字段，加 item 前先 dedup，写后重新读取验证**。
- **状态同步**：按事实梯度推断 Status，不因存在 PR 就标 Done。
- **路线图维护**：Epic 正文只保留路线图，子任务用官方父子关联。
- **看板克隆 / 初始化**：`gh project copy` 优先，从零建用 `gh project create` + `field-create`。

## 安装

将本目录放入 Agent 的 skills 目录即可自动发现（SKILL.md 的 frontmatter 是触发依据）：

```bash
# Codex 默认路径
cp -r github-project-manager ~/.codex/skills/

# 或自定义 CODEX_HOME
cp -r github-project-manager "$CODEX_HOME/skills/"
```

<!-- TODO: 其他 Agent 加载方式 / 备注 -->

## 使用

安装后，用自然语言触发即可：

```text
用 github-project-manager 整理当前仓库未关闭 Issue
把 issue #101 加入 Project，设 类型=Feature、优先级=P1
检查哪些 Issue 已由 PR 完成，可以关闭或合并
更新 Agent 路线图 epic
建一个类似的看板（克隆现有 Project）
```

<!-- TODO: 更多示例 / 输出示例 -->

## 权限要求

`gh` CLI token 至少含：

- `repo`
- `project`
- 组织 Project 可能还需 `read:org` / `read:project`

缺 scope 时 Skill 会输出 `gh auth refresh -h github.com -s project,read:project`，并保留已完成只读分析，不谎称完成。

## 目录结构

```
github-project-manager/
├── SKILL.md                        # 入口（Agent 加载）
├── agents/openai.yaml              # UI 元数据
└── references/
    └── gh-project-commands.md      # gh project / GraphQL 命令清单
```

## 设计原则

- **通用**：运行时推断 owner/repo/Project，不硬编码任何私有信息。
- **先读后写**：所有 node ID / 选项 ID 现场读取，不按显示名猜测。
- **不造同义字段**：已有等价字段一律复用。
- **最小授权**：高影响操作（关闭 Issue、删字段、克隆/新建看板）需逐项确认。
- **不碰业务代码**。

## License

MIT — 见 [LICENSE](./LICENSE)。

<!-- TODO: 贡献指南 / 行为准则 / 版本说明 -->