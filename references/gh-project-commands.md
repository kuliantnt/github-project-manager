# gh project / GraphQL 命令清单

github-project-manager 使用的 `gh project` 与 `gh api graphql` 命令。所有写操作前先读对应真实 ID，写后重新读取验证。

**这是通用清单，不假定 owner / repo / Project 编号。** 运行时用变量替换：

```bash
# 1. 推断 owner/repo（优先当前 git 仓）
REPO=$(gh repo view --json owner,name -q '.owner.login + "/" + .name' 2>/dev/null)
OWNER=$(gh repo view --json owner -q '.owner.login' 2>/dev/null)
# 兜底：git config --get remote.origin.url 解析

# 2. 选 Project（列出后让用户指定编号；变量 PROJ = project number）
gh project list --owner "$OWNER" --limit 50
```

读取权限：

```bash
gh auth status     # 确认含 repo / project（组织 Project 可能需 read:org）
# 缺 project scope 时：
gh auth refresh -h github.com -s project,read:project
```

## 1. 读取 Project 字段与选项 ID（写任何字段前必做）

```bash
gh project field-list "$PROJ" --owner "$OWNER" --limit 100
# 输出每行：<显示名> <类型> <field ID>
```

单选字段需再取 option ID（设置单选值必须用 option ID，不是显示名）：

```bash
gh api graphql -f query='
query{
  node(id:"PVTSSF_xxxxx"){
    ... on ProjectV2SingleSelectField{ name options{ id name color } }
  }
}'
```

字段名 / 选项语言不固定（英文或中文均可能），以实际读取为准。

## 2. 读取 Project 当前 items（加 item 前用它验重）

```bash
gh project item-list "$PROJ" --owner "$OWNER" --limit 200
# 输出含 Title、item ID（PVTI_...）、来源 Issue/PR 编号。比对编号，已存在则跳过。
```

## 3. 取 Issue / PR 的 node ID

```bash
gh issue view <num> -R "$REPO" --json id -q .id     # -> "I_kw..."
gh pr   view <num> -R "$REPO" --json id -q .id     # -> "PR_kw..."
```

## 4. 将 Issue / PR 加入 Project

加前 `item-list` 比对确认未加入，再：

```bash
gh project item-add "$PROJ" --owner "$OWNER" --url "https://github.com/$REPO/issues/<num>"
# PR：
gh project item-add "$PROJ" --owner "$OWNER" --url "https://github.com/$REPO/pull/<num>"
```

输出含新 item ID。若失败报权限再走 GraphQL（第 7 段）。

## 5. 设置单选字段

需要：itemID、fieldID、optionID（第 1 步取到）。

```bash
gh project item-edit --id <itemID> --project-id <PVT_...> --field-id <PVTSSF_...> \
  --single-select-option-id <optionID>
# 参数名随 CLI 版本略有差异，先 `gh project item-edit --help` 核对再执行。
```

设置后**必须重新读取验证**：

```bash
gh project item-list "$PROJ" --owner "$OWNER" --limit 200 | grep -E "<关键字>"
```

## 6. 创建单选字段（缺选项时，先查无同义再建）

```bash
# 选项用英文逗号分隔；选项字符串内部不得含英文逗号
gh project field-create "$PROJ" --owner "$OWNER" --name "优先级" \
  --data-type SINGLE_SELECT \
  --single-select-options "🔴 P0 紧急,🟠 P1 高,🟡 P2 中,🟢 P3 低"

# 文本 / 日期 / 数字
gh project field-create "$PROJ" --owner "$OWNER" --name "下步动作" --data-type TEXT
gh project field-create "$PROJ" --owner "$OWNER" --name "目标日期" --data-type DATE
```

## 7. 备用：GraphQL 写 item / 字段

`gh project` 不支持时用 GraphQL。常用 mutation：

- `addProjectV2ItemById`（加 item，input：`projectId` + `contentId`=issue/PR node ID）
- `updateProjectV2ItemFieldValue`（改字段，input：`projectId` + `itemId` + `fieldId` + `value.singleSelectOptionId`）

```bash
gh api graphql -f query='
mutation{ addProjectV2ItemById(input:{projectId:"PVT_kwXxx",contentId:"I_kwXxx"}){
  item{ id } } }'

gh api graphql -f query='
mutation{ updateProjectV2ItemFieldValue(input:{
  projectId:"PVT_kwXxx",
  itemId:"PVTI_kwXxx",
  fieldId:"PVTSSF_kwXxx",
  value:{singleSelectOptionId:"optId"}
}){ projectV2Item{ id } } }'
```

## 8. Issue / PR 常用读命令

```bash
gh issue list -R "$REPO" --state open  --limit 100 --json number,title,labels,assignees,milestone
gh issue list -R "$REPO" --state closed --limit 50  --json number,title,closedAt
gh issue view <num> -R "$REPO" --json title,body,labels,state,closedByPullRequestsReferences,comments
gh pr    view <num> -R "$REPO" --json title,state,merged,closingIssuesReferences
```

`closingIssuesReferences` 用于判断「PR 已合并、对应 Issue 可关闭」。

## 9. 父子任务关系（官方关联，优于正文清单）

```bash
gh issue edit <child>  -R "$REPO" --add-parent <parent>
gh issue edit <parent> -R "$REPO" --add-sub-issue <child>
gh issue view <parent> -R "$REPO" --json subIssuesIssues
```

## 10. 关闭 Issue（需留说明评论）

```bash
gh issue comment <num> -R "$REPO" --body "迁移说明 / 完成证据，主 Issue #xx"
gh issue close   <num> -R "$REPO" --reason completed   # 或 "not planned" / duplicate
```

`--reason` 取值：`completed`、`not planned`、`duplicate`（部分版本）。duplicate 常配合打 `duplicate` 标签。

## 11. 克隆 / 新建看板（需授权）

### 克隆（带字段+选项+视图，不带实 item 字段值）

```bash
gh project copy <src-num> --source-owner <src-owner> --target-owner <dst-owner> \
  --title "<新看板标题>" [--drafts]
# --drafts 才带 draft issue item，一般不加
# 返回新 project number 与 node id
```

验证字段与选项到位：

```bash
gh project field-list <new-num> --owner <dst-owner> --limit 100
gh api graphql -f query='query{node(id:"PVTSSF_xxx"){... on ProjectV2SingleSelectField{ options{ id name } }}}'
```

### 从零建

```bash
gh project create --owner <owner> --title "<新看板标题>"   # 返回新 number + url + node id
```

### 逐字段创建

```bash
# 单选：选项用英文逗号分隔，选项名不得含英文逗号
gh project field-create <num> --owner <owner> --name "优先级" \
  --data-type SINGLE_SELECT \
  --single-select-options "P0 紧急,P1 高,P2 中,P3 低"
gh project field-create <num> --owner <owner> --name "下步动作" --data-type TEXT
gh project field-create <num> --owner <owner> --name "目标日期" --data-type DATE
```

### 常见字段建议集（仅建议，原样复用源 Project 选项，勿自造）

字段建议见 SKILL.md §C「常见字段建议」表。**模块值必须依据目标仓库业务确定**，本 Skill 不提供固定取值。选项命名 / 是否带 emoji / 中英文，以源 Project 或用户偏好为准——不要强加一套写法。

字段建完后，加 Issue/PR 并设值走第 4、5 段。