---
name: github-project-manager
description: 通用 GitHub Issue、Pull Request 与 GitHub Projects v2 看板管理 Skill，适用于任意 GitHub 仓库。能力包括：整理未关闭 Issue / 识别重复项与已合并未关闭项、维护 Epic 路线图父 Issue、把 Issue 或 PR 加入指定 Project、读取并设置 Status / Priority / Type / Module 等字段、根据 Issue 与 PR 真实状态同步看板、识别已存在的 Project item 避免重复卡片、克隆或初始化看板与字段、字段初始化建议。仅做项目管理，不修改业务代码。当用户说“整理 Issue”“维护路线图”“更新 Project 看板”“把 xx 加入 Project”“检查哪些 Issue 可以关闭”“同步 Project 字段”“建一个类似的看板”“初始化 Project 字段”等时触发。
---

# github-project-manager

通用 GitHub 项目管理 Skill。目标仓库与 Project 在运行时推断或由用户指定，**不假定任何固定的 owner、仓库、Project 编号或字段 ID**。

## 适用范围

管理 GitHub Issue、Pull Request、GitHub Projects v2。**不修改业务代码、不触碰与整理无关的仓库配置。**

## 首要：定位目标与权限

每次调用先确认运行时目标，不要凭空假设：

1. **仓库 owner/repo**：优先当前 git 仓远程；缺则问用户。
   ```bash
   gh repo view --json owner,name -q '.owner.login + "/" + .name' 2>/dev/null
   gh repo set-default --view 2>/dev/null   # 确认默认仓
   git config --get remote.origin.url       # 兜底解析
   ```
2. **目标 Project**：列出该 owner 的 Project，让用户指定编号或标题；若只有一个则默认它。
   ```bash
   gh project list --owner <owner> --limit 50
   ```
3. **权限**：`gh auth status`，确认含 `project` + `repo`（组织 Project 可能需 `read:org`/`read:project`）。缺 scope 则停止写入、保留只读分析并输出：
   ```bash
   gh auth refresh -h github.com -s project,read:project
   ```

确定前不进行任何写操作。

## 工作流程

1. 定位目标与权限（上方）。
2. **只读盘点**：Issue、PR、Project 字段与 items（命令见 `references/gh-project-commands.md`）。
3. **输出整理建议**，每项操作归入三类：
   - 安全只读分析（直接给结论）
   - 低风险可执行（用户说“直接整理/直接处理”时可执行）
   - 高影响操作（**始终需逐项授权**：关闭 Issue、删除 item/字段、创建新字段或选项、大批量关闭、克隆/新建看板）
4. **写操作**：先 dry-check，写后重新读取验证。
5. **输出变更报告**（见末尾）。

“直接处理”授权下可执行：更新 Issue 标题/正文/标签、加入已存在的 Project item、设置**已存在**的字段值、关闭内容已完整迁移或明确重复的 Issue（仍需留说明评论）。

**默认不得擅自执行**：删除 Project / 字段 / item、删除 Issue、大批量关闭仍存疑的 Issue、创建新字段或同义字段、克隆或新建看板、修改代码、修改无关仓库配置。

## A. Issue 盘点

读取并与下列维度分类：

- 状态：未关闭 / 最近关闭 / 已合并 PR 关联
- 类型：Epic 父任务（标题含 `epic:` 或正文标记路线图）、Feature、Bug、Investigation（`investigate:`）、Documentation
- 风险态：已完成但未关闭、重复项、范围重叠项、暂不处理项
- 来源：Issue 正文的父子任务清单、评论中的完成 / 迁移说明、PR 的 `closingIssuesReferences`

判定重复必须看正文与关联 PR，**不得仅因标题相似就判定重复**。

## B. Issue 整理

可执行：改标题、更新正文、增减标签、加迁移说明评论、关闭 Issue。

**关闭前置条件（缺一不可）**：
1. 工作确实完成，或内容已完整迁移到主 Issue；
2. 留下说明评论，指向主 Issue 或完成证据；
3. 关闭原因明确为 `completed` / `duplicate` / `not planned`；
4. 不得静默关闭。

链接父子用官方关联（避免在正文堆砌子任务清单）：

```bash
gh issue edit <child>  -R <repo> --parent <parent-num>    # 设父任务（注意：不存在 --add-parent）
gh issue edit <parent> -R <repo> --add-sub-issue <child>  # 反向追加子任务
```

验证父子关系用 `gh issue view <parent> --json subIssues`（字段名为 `subIssues`，不是 `subIssuesIssues`）。

## C. Projects v2 字段管理（核心：先读真实，再复用）

**优先 `gh project`，无法完成再 `gh api graphql`**。完整命令在 `references/gh-project-commands.md`，写前必读。

铁律（与具体仓库无关）：

- 一律使用真实 node ID（Project / item / field / option / issue / PR node），**不得按显示名称猜测**。`gh project field-list` 给出单选项 ID，设单选字段必须用 option ID。
- 加 item 前先 `item-list` 比对，**已存在不重复加**。
- 设置字段后**必须重新读取该 item 确认值生效**；不得伪造结果。
- 单选字段缺选项时，先 `field-list` 确认无同义选项，再 `field-create` 创建并回填 option ID。
- **不假定字段一定存在，不假定字段名一定是英文或中文**。字段名 / 选项名 / 语言以目标 Project 实际读取为准。

字段是「**读到的真实字段集**」优先；下方只是命名空间建议，缺失时才提，且需用户确认才创建。

### 常见字段建议（命名空间，按需采用）

| 字段语义 | 类型 | 常见选项梯度 |
|---|---|---|
| Status | 单选 | 待办 → 进行中 → 审查中 → 已完成 / 已取消（可含「阻塞」） |
| 优先级 / Priority | 单选 | P0 紧急 / P1 高 / P2 中 / P3 低 |
| 类型 / Type | 单选 | Feature / Bug / Refactor / Docs / Research / DevOps |
| 模块 / Module | 单选 | **必须依据目标仓库实际业务确定**，不在 Skill 内固定取值 |
| 工作量 | 单选 | XS / S / M / L / XL |
| 风险 | 单选 | 低 / 中 / 高 / 阻塞 |
| 验收状态 | 单选 | 待拆解 / 待验证 / 已覆盖 / 不适用 |
| 目标日期 | 日期 | — |
| 下步动作 | 文本 | — |

模块值没有通用答案，应从目标仓库的目录、labels 或历史 Issue 里归纳后再提议。**不要未经检查直接创建上述选项**；已有等价字段一律复用，不造同义词。

## D. 状态同步（Status 推断规则，与具体选项名解耦）

按事实梯度推断，映射到目标 Project **真实存在**的 Status 选项（用 §C 步骤读到的 option 名），**不得仅因存在 PR 就标 Done**：

- 新建、需求未细化 → 最低「待办」态
- 需求明确且可执行 → 「就绪」态（如无独立就绪项，并入待办）
- 已有开发分支或 open PR → 「进行中」
- PR 已合并、等待真机/人工验证，或 Investigation 有初步结论待复核 → 「审查/测试」态
- 已合并且验收完成、用户确认 → 「已完成」
- 依赖平台、权限或外部条件 → 「阻塞」（若无该选项，留在进行中并加评论说明）

必须能区分：`代码已完成` / `PR 已合并` / `测试已通过` / `真机已验证` / `用户已确认`。无结论的 Investigation 不得标 Done。

## E. 路线图（Epic）整理

对 Epic / 父 Issue 维护其正文为路线图结构，**只保留路线图信息**，不堆放子任务实现细节：

- 定位与目标
- 已完成任务（链到已关闭子 Issue）
- 当前主线（按优先级梯度）
- 关联但独立维护的任务
- 已迁移合并的重复 Issue
- 架构原则 / 验收状态 / 暂不包含范围

用 `gh issue edit` 更新正文，用官方父子关联维护结构，正文清单只做导航。

## F. 看板克隆与初始化（需明确授权）

用户要求“建一个类似的看板”/“初始化 Project”时，§为高影响操作，**需用户逐项授权**。

### 路径 1：克隆现有看板（首选）

```bash
gh project copy <src-num> --source-owner <src-owner> --target-owner <dst-owner> \
  --title "<新看板标题>" [--drafts]
```

克隆带字段、单选选项、视图布局，不带实 item 字段值。完成后回读新编号、`field-list` 确认选项齐备。

### 路径 2：从零建

```bash
gh project create --owner <owner> --title "<新看板标题>"
gh project field-create <num> --owner <owner> --name <字段名> --data-type SINGLE_SELECT \
  --single-select-options "选项A,选项B,选项C"   # 选项用英文逗号分隔，选项名不得含英文逗号
```

默认复刻上方「常见字段建议」的语义，选项名**原样照搬源 Project**（若有），不任由模型再发明。创字段前判断是否与源同义，避免同一看板建两套同字段。

### 流程

1. 确认目标所有者与新看板标题。
2. 有现成模板走 clone，纯增量走 create。
3. 创建后回读字段、选项 ID，在变更报告写明新 Project 编号与 URL。
4. 不加速任何 item，除非用户明确要求。

## 安全原则

- 不改业务代码；不伪造测试/合并/验证结果。
- 不仅凭标题相似判重；不静默关闭 Issue；不删历史讨论。
- 不创建重复字段；不覆盖用户已有 Project 配置。
- 不在设计任务外擅自克隆或新建 Project；建看板属高影响操作，需用户明确授权。
- 不把平台建设任务混入日常 Bug 看板，除非用户明确要求。
- Investigation 无结论不得标 Done。
- 一切结论以 GitHub 当前实际状态为准；缺权限如实说明并输出授权命令。

## 输出报告格式

每次整理结束统一输出：

```
## 整理结论
（本次发现的主要问题）

## Issue 变更
- 更新的 Issue
- 关闭的 Issue（含关闭原因 / 迁移到的主 Issue）

## Project 变更
- 加入的 Issue / PR
- 原本已存在的 item（未重复添加）
- 设置的字段值（字段名 -> 值）
- 新增的字段或选项（如有，注明已先查无同义项）

## 未执行事项
（因权限 / 信息不足 / 高风险而未执行，并说明原因）

## 验证结果
（重新查询后的最终状态）

## 后续建议
（只列真正需要后续处理的事项）
```

## 依赖

- `gh` CLI（含 `project` 系列子命令；`copy`/`create`/`field-create` 用于建看板）。
- token scope 至少：`repo`、`project`；组织 Project 可能需 `read:org`、`read:project`，以 GitHub 当时认证提示为准。
- 脆弱与建板命令清单见 `references/gh-project-commands.md`，执行前必读。