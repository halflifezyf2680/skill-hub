# Skill Hub Manual

## 1. 路由协议

Skill Hub 的核心是渐进式披露：不把所有 skill 正文加载到 context，而是通过工具逐层定位后按需读取。

```
search_skills(query)
  │
  ├─ 命中了 → 检查 directMatch
  │             ├─ directMatch 存在 → 读 description 判断是否对路
  │             │    ├─ 对路 → read_skill(skill)
  │             │    └─ 不对路 → 阅读返回的 groupDescription，重新选
  │             └─ 无 directMatch → 阅读 groupDescription，选 group
  │
  └─ 没命中 → list_skill_groups() → 阅读所有组描述 → 选 group
                  │
                  ▼
             list_group_skills(group) → 看 skill 名称 → 选目标
                  │
                  ▼
             read_skill(skill) → 加载完整正文
```

**规则：**
- search_skills 是辅助入口，不是必须。搜不到就走 list_skill_groups 兜底。
- 一次只读 1 个 skill，读完后评估，不够再补。
- 语义判断由 LLM 完成，搜索引擎只做 token 过滤。

## 2. 工具详解

### search_skills

搜索匹配的组。query 会拆分为 token，与每个组的 ID、描述、关键词、别名以及组内所有 skill 名称做 includes 匹配。

**参数：**

| 参数 | 类型 | 必填 | 说明 |
|------|------|------|------|
| `query` | string | 是 | 搜索关键词，支持中英文混合 |
| `limit` | number | 否 | 最多返回几个组，默认 8，最大 12 |

**返回结构：**

```json
{
  "query": "前端开发",
  "returned": 2,
  "groups": [
    {
      "group": "engineering",
      "groupDescription": "软件工程、系统架构、前后端实现...",
      "skillNames": ["Frontend Developer", "Software Architect", ...],
      "directMatch": null
    },
    {
      "group": "design",
      "groupDescription": "UI/UX、品牌、视觉系统...",
      "skillNames": ["UI Designer", "UX Architect", ...],
      "directMatch": {
        "skillId": "engineering-engineering-frontend-developer",
        "skillName": "Frontend Developer",
        "description": "前端开发专家..."
      }
    }
  ]
}
```

`directMatch`：当 query 精确或模糊命中某个 skill 名称时出现，附带该 skill 的 description，LLM 可以直接判断是否对路而不用再调一次 read_skill。

### list_skill_groups

列出所有组。当 search_skills 返回空结果时用这个兜底。

**参数：** 无

**返回结构：**

```json
{
  "groups": [
    { "group": "academic-research", "groupDescription": "学术研究、学习规划..." },
    { "group": "design", "groupDescription": "UI/UX、品牌、视觉系统..." },
    ...
  ],
  "loadedSkills": 232,
  "issueCount": 0
}
```

### list_group_skills

选定组后，查看组内所有 skill。

**参数：**

| 参数 | 类型 | 必填 | 说明 |
|------|------|------|------|
| `group` | string | 是 | 组名，如 "engineering" |

**返回结构：**

```json
{
  "group": "engineering",
  "groupDescription": "软件工程、系统架构...",
  "skills": [
    {
      "skillId": "engineering-engineering-code-reviewer",
      "skillName": "Code Reviewer",
      "keywords": ["code", "review", "质量"],
      "skillPath": "/path/to/SKILL.md"
    },
    ...
  ]
}
```

### read_skill

加载 skill 的完整正文。这是最终步骤，返回内容包括 SKILL.md 正文以及 references/ 和 assets/ 目录下的所有文件。

**参数：**

| 参数 | 类型 | 必填 | 说明 |
|------|------|------|------|
| `skill` | string | 是 | skill 名称（如 "Code Reviewer"）或 skill ID（如 "engineering-engineering-code-reviewer"） |

**返回结构：**

```json
{
  "skillId": "engineering-engineering-code-reviewer",
  "skillName": "Code Reviewer",
  "description": "专业的代码审查专家...",
  "group": "engineering",
  "keywords": ["code", "review", "质量"],
  "markdown": "# Code Reviewer\n\n完整的 SKILL.md 正文内容...",
  "references": [
    { "filename": "review-checklist.md", "content": "..." }
  ],
  "assets": [
    { "filename": "template.md", "content": "..." }
  ],
  "related": [
    { "skillId": "...", "skillName": "Software Architect" }
  ]
}
```

### install_skills

从本地目录批量安装 skill 包。支持单 skill 目录和包含多个 skill 的父目录。安装后自动重建索引。

**参数：**

| 参数 | 类型 | 必填 | 说明 |
|------|------|------|------|
| `sourcePath` | string | 是 | skill 包目录或包含多个包的父目录 |

**返回结构：**

```json
{
  "action": "installed",
  "sourcePath": "/path/to/source",
  "installed": [
    { "skillId": "my-skill", "skillName": "My Skill", "group": "engineering" }
  ],
  "skipped": [],
  "errors": []
}
```

### create_skill

在 hub 中创建新的 skill 包。自动完成分组、索引、元数据生成。

**参数：**

| 参数 | 类型 | 必填 | 说明 |
|------|------|------|------|
| `name` | string | 是 | skill 显示名称 |
| `description` | string | 是 | skill 描述，用于搜索索引和自动分组 |
| `skillMarkdown` | string | 是 | 完整的 SKILL.md 内容，frontmatter 可选（会被 name/description 覆盖） |

**返回结构：**

```json
{
  "action": "created",
  "skillId": "my-skill",
  "skillName": "My Skill",
  "group": "engineering"
}
```

### create_group

创建新的自定义组。仅在现有组无法覆盖的领域才创建。

**参数：**

| 参数 | 类型 | 必填 | 说明 |
|------|------|------|------|
| `group` | string | 是 | 组 ID，kebab-case |
| `groupDescription` | string | 是 | 组描述，面向 LLM |
| `keywords` | string[] | 否 | 路由关键词 |
| `aliases` | string[] | 否 | 别名 |

### update_group

更新组的定义。可以改组 ID、描述、关键词、别名。更新后自动重建索引，所有 skill 重新分组。

**参数：**

| 参数 | 类型 | 必填 | 说明 |
|------|------|------|------|
| `group` | string | 是 | 当前组 ID |
| `newGroup` | string | 否 | 新组 ID |
| `groupDescription` | string | 否 | 新描述 |
| `keywords` | string[] | 否 | 新关键词（替换，非追加） |
| `aliases` | string[] | 否 | 新别名（替换，非追加） |

### delete_group

删除空的自定义组。内置组不可删除，非空组不可删除。

**参数：**

| 参数 | 类型 | 必填 | 说明 |
|------|------|------|------|
| `group` | string | 是 | 要删除的组 ID |

### validate_skills

校验 hub 中 skill 的完整性。可以校验单个或全部。

**参数：**

| 参数 | 类型 | 必填 | 说明 |
|------|------|------|------|
| `skill` | string | 否 | skill ID 或名称。省略则校验全部 |

**返回结构：**

```json
{
  "total": 232,
  "issues": [
    {
      "skillId": "my-skill",
      "skillName": "My Skill",
      "code": "duplicate_skill_name",
      "severity": "review_required",
      "message": "duplicate name found in: other-skill"
    }
  ]
}
```

**问题类型：**

| code | severity | 说明 |
|------|----------|------|
| `missing_skill_file` | blocked | 目录存在但 SKILL.md 缺失 |
| `invalid_frontmatter` | blocked | 缺少 name 或 description |
| `duplicate_skill_name` | review_required | 多个 skill 同名 |
| `generic_group` | review_required | 被分到 specialized-domain 兜底组 |

### get_hub_status

查看 hub 整体运行状态。

**参数：** 无

**返回结构：**

```json
{
  "groups": 16,
  "loadedSkills": 232,
  "issues": 0,
  "indexUpdatedAtMs": 1713000000000,
  "watcher": { "running": true, "lastEventAtMs": 1713000010000, "lastError": null },
  "imports": { "raw": 0, "repaired": 0 }
}
```

### list_import_candidates

列出 staging/imports/ 中的待审查候选。

**参数：**

| 参数 | 类型 | 必填 | 说明 |
|------|------|------|------|
| `status` | string | 否 | 过滤状态：`ready`、`review_required`、`blocked`、`repaired` |
| `limit` | number | 否 | 最多返回几个，默认 30，最大 100 |

**返回结构：**

```json
{
  "status": null,
  "returned": 5,
  "candidates": [
    {
      "id": "package:my-skill/SKILL.md",
      "sourcePath": "/path/to/SKILL.md",
      "relativePath": "my-skill/SKILL.md",
      "inferredTargetId": "my-skill",
      "status": "ready",
      "issues": [],
      "kind": "package",
      "name": "My Skill",
      "description": "..."
    }
  ]
}
```

### read_import_candidate

读取一个导入候选的完整内容，用于审查。

**参数：**

| 参数 | 类型 | 必填 | 说明 |
|------|------|------|------|
| `id` | string | 是 | 候选 ID，来自 list_import_candidates |

**返回内容：** 原始 markdown、frontmatter 解析结果、问题列表、附近组信息（辅助判断该归入哪个组）。

### write_repaired_import

将修复后的 skill 写入 staging/repaired/，不修改原始文件。

**参数：**

| 参数 | 类型 | 必填 | 说明 |
|------|------|------|------|
| `id` | string | 是 | 候选 ID |
| `skillMarkdown` | string | 是 | 修复后的完整 SKILL.md |
| `targetId` | string | 否 | 最终 skill ID，默认从候选推断 |
| `notes` | string | 否 | 修复说明 |

## 3. Skill 包格式

每个 skill 是一个目录，必须包含 `SKILL.md`：

```markdown
---
name: my-skill
description: 这个 skill 做什么
---

# My Skill

Skill 正文内容...
```

**必填 frontmatter：** `name`（显示名称）、`description`（用于搜索和自动分组）

**可选目录：** `references/`（参考文件）、`assets/`（资源文件），read_skill 时一并返回。

**自动生成：** `meta.json`（skillId、skillName、description、group、keywords、updatedAtMs、status）

## 4. 组体系

内置 16 个组：

| 组 ID | 描述 |
|-------|------|
| `engineering` | 软件工程、系统架构、前后端实现、数据与 AI 工程 |
| `design` | UI/UX、品牌、视觉系统、设计语言 |
| `product` | 产品规划、需求分析、用户反馈、路线图 |
| `project-management` | 项目推进、任务协调、里程碑、交付 |
| `marketing` | 品牌营销、内容营销、增长叙事 |
| `paid-media` | 广告投放、媒介优化、买量 |
| `sales` | 销售、售前、线索管理、成交推进 |
| `finance` | 财务、预算、核算、定价 |
| `legal-compliance` | 法务、合规、审计、政策、风险 |
| `hr-talent` | 招聘、人才、培训、组织发展 |
| `support-operations` | 客服、运营支持、行政协助 |
| `supply-chain` | 供应链、采购、物流、库存 |
| `academic-research` | 学术研究、学习规划、人文社科 |
| `testing-qa` | 测试、质量保障、评估、验证 |
| `spatial-gaming` | 游戏开发、空间计算、XR、3D |
| `specialized-domain` | 兜底组，无法归入其他组的 skill |

每个组的 keywords 为中英双语。安装 skill 时自动按加权匹配分组（keywords 权重 4 > aliases 权重 3 > group ID 片段权重 2 > groupDescription 片段权重 1）。

内置组不可删除。自定义组通过 create_group 创建，空组可通过 delete_group 删除。

## 5. 环境变量

| 变量 | 默认值 | 说明 |
|------|--------|------|
| `SKILL_HUB_ROOT` | `<package>/data/hub` | 数据根目录 |
| `SKILL_ROUTER_SEARCH_LIMIT` | `8` | search_skills 默认返回上限 |
| `SKILL_ROUTER_MAX_KEYWORDS` | `12` | 每个 skill 自动提取的最大关键词数 |
| `SKILL_ROUTER_MAX_RELATED_SKILLS` | `5` | read_skill 返回的最大关联 skill 数 |
| `SKILL_ROUTER_WATCH` | `1` | 是否启用文件监听 |
| `SKILL_ROUTER_WATCH_USE_POLLING` | `0` | 是否使用轮询 |
| `SKILL_ROUTER_WATCH_INTERVAL_MS` | `100` | 轮询间隔 |
| `SKILL_ROUTER_WATCH_STABILITY_MS` | `300` | 写入稳定等待时间 |
| `SKILL_ROUTER_WATCH_SYNC_DELETE` | `1` | 删除 SKILL.md 时是否同步移除索引 |
