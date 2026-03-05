# Neo-Ralph 架构设计文档

> **Neo-Ralph = snarktank/ralph 的简洁 + frankbria 的保护机制**

本文档描述 Neo-Ralph 的架构设计、核心模块和数据流。

**版本**: v1.0.0  
**最后更新**: 2026-03-05

---

## 1. 设计目标

### 1.1 核心理念

**采用 snarktank/ralph 的 Git 分支管理逻辑，保持简洁，只增加必要的保护机制。**

### 1.2 设计原则

| 原则 | 说明 |
|------|------|
| **一个需求 = 一个分支** | Git 分支天然隔离，简单有效 |
| **最少文件** | 只有 prd.json + progress.txt 两个核心文件 |
| **自动存档** | 分支切换时自动备份，防止数据丢失 |
| **人类审查** | PR/MR 需要人工 merge，最终控制 |
| **可选保护** | 速率限制、断路器可选择性启用 |

---

## 2. 系统架构

### 2.1 整体架构

```
┌─────────────────────────────────────────────────────────┐
│                    用户接口层                            │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐      │
│  │ ralph prd   │  │ ralph       │  │ ralph       │      │
│  │ (需求管理)  │  │ (执行循环)  │  │ prd --list  │      │
│  └─────────────┘  └─────────────┘  └─────────────┘      │
└─────────────────────────────────────────────────────────┘
                          │
                          ▼
┌─────────────────────────────────────────────────────────┐
│                    核心引擎层                            │
│  ┌─────────────────────────────────────────────────┐    │
│  │              ralph_loop.sh                       │    │
│  │  ┌──────────────┐  ┌──────────────┐            │    │
│  │  │ 分支管理器    │  │ 任务调度器    │            │    │
│  │  │ Branch Mgr  │  │ Task Scheduler│            │    │
│  │  └──────────────┘  └──────────────┘            │    │
│  │  ┌──────────────┐  ┌──────────────┐            │    │
│  │  │ 退出检测器    │  │ 存档管理器    │            │    │
│  │  │ Exit Detector│  │ Archive Mgr  │            │    │
│  │  └──────────────┘  └──────────────┘            │    │
│  └─────────────────────────────────────────────────┘    │
└─────────────────────────────────────────────────────────┘
                          │
                          ▼
┌─────────────────────────────────────────────────────────┐
│                    基础设施层                            │
│  ┌────────────┐ ┌────────────┐ ┌────────────┐          │
│  │ Git        │ │ Claude     │ │ File       │          │
│  │ Repository │ │ Code CLI   │ │ System     │          │
│  └────────────┘ └────────────┘ └────────────┘          │
└─────────────────────────────────────────────────────────┘
```

---

## 3. 需求管理（Git 分支模式）

### 3.1 核心设计

**一个需求 = 一个 Git 分支 + 一套 prd.json**

```
需求创建
    │
    ▼
创建 Git 分支 (feature/login-20260305)
    │
    ▼
生成 .ralph/prd.json
    │
    ▼
循环执行任务
    │
    ▼
需求完成 → Merge 到 main
    │
    ▼
自动存档到 .ralph/archive/
```

---

### 3.2 目录结构

```
my-project/
├── .ralph/
│   ├── prd.json              # 当前需求任务列表
│   ├── progress.txt          # 当前需求进度日志
│   ├── .last-branch          # 上次分支记录（用于存档）
│   │
│   └── archive/              # 自动存档
│       ├── 2026-03-05-login/
│       │   ├── prd.json
│       │   └── progress.txt
│       └── ...
│
├── .ralphrc                  # 项目配置（可选）
└── src/                      # 源代码
```

---

### 3.3 prd.json Schema

```json
{
  "$schema": "http://json-schema.org/draft-07/schema#",
  "type": "object",
  "required": ["project", "branchName", "userStories"],
  "properties": {
    "project": {
      "type": "string",
      "description": "项目名称"
    },
    "branchName": {
      "type": "string",
      "description": "Git 分支名称",
      "pattern": "^feature/.*$"
    },
    "description": {
      "type": "string",
      "description": "需求描述"
    },
    "createdAt": {
      "type": "string",
      "format": "date-time",
      "description": "创建时间"
    },
    "userStories": {
      "type": "array",
      "items": {
        "type": "object",
        "required": ["id", "title", "passes"],
        "properties": {
          "id": {
            "type": "string",
            "pattern": "^story-[0-9]{3}$"
          },
          "title": {
            "type": "string"
          },
          "description": {
            "type": "string"
          },
          "acceptanceCriteria": {
            "type": "array",
            "items": {"type": "string"}
          },
          "priority": {
            "type": "integer",
            "minimum": 1
          },
          "passes": {
            "type": "boolean"
          },
          "completedAt": {
            "type": ["string", "null"],
            "format": "date-time"
          }
        }
      }
    },
    "metadata": {
      "type": "object",
      "properties": {
        "totalStories": {"type": "integer"},
        "completedStories": {"type": "integer"},
        "lastUpdated": {"type": "string", "format": "date-time"}
      }
    }
  }
}
```

---

### 3.4 示例 prd.json

```json
{
  "project": "my-app",
  "branchName": "feature/login-20260305",
  "description": "用户登录功能",
  "createdAt": "2026-03-05T10:00:00Z",
  "userStories": [
    {
      "id": "story-001",
      "title": "添加登录表单组件",
      "description": "作为用户，我想要输入邮箱密码登录",
      "acceptanceCriteria": [
        "表单包含邮箱和密码输入框",
        "密码输入框显示为掩码"
      ],
      "priority": 1,
      "passes": false,
      "completedAt": null
    },
    {
      "id": "story-002",
      "title": "实现 JWT token 验证",
      "priority": 2,
      "passes": false,
      "completedAt": null
    }
  ],
  "metadata": {
    "totalStories": 2,
    "completedStories": 0,
    "lastUpdated": "2026-03-05T10:00:00Z"
  }
}
```

---

## 4. 核心命令

### 4.1 命令列表

```bash
# 需求管理
ralph prd [name]           # 创建/切换需求
ralph prd --list           # 查看所有需求
ralph prd --archive        # 手动存档

# 执行控制
ralph                      # 执行当前需求
ralph --stop               # 停止执行

# 状态查看
ralph status               # 查看当前状态
ralph logs                 # 查看日志
```

---

### 4.2 ralph prd [name] - 创建/切换需求

```bash
$ ralph prd login

┌─────────────────────────────────────────┐
│ 创建新需求：login                        │
├─────────────────────────────────────────┤
│                                         │
│ 正在创建分支：feature/login-20260305    │
│ 正在初始化 prd.json...                  │
│                                         │
│ 请用一句话描述需求：                    │
│ > 用户登录，支持邮箱和 Google 登录       │
│                                         │
│ 正在启动 Claude 生成 PRD...             │
│                                         │
└─────────────────────────────────────────┘

✅ 分支已创建：feature/login-20260305
✅ prd.json 已生成

是否开始执行？[y/n]
```

**实现逻辑**:

```bash
#!/bin/bash
# ralph_prd.sh

FEATURE_NAME=$1
DATE_SUFFIX=$(date +%Y%m%d)
BRANCH_NAME="feature/$FEATURE_NAME-$DATE_SUFFIX"

# 1. 创建分支
git checkout -b "$BRANCH_NAME"

# 2. 初始化 prd.json
cat > .ralph/prd.json << EOF
{
  "project": "$(basename $(pwd))",
  "branchName": "$BRANCH_NAME",
  "description": "",
  "createdAt": "$(date -Iseconds)",
  "userStories": []
}
EOF

# 3. 启动 Claude 生成 PRD
claude --dangerously-skip-permissions --prompt "
生成 PRD，功能描述：$(cat)
输出格式：prd.json
"

# 4. 提交
git add .ralph/prd.json
git commit -m "feat: 初始化 PRD - $FEATURE_NAME"

# 5. 询问执行
read -p "是否开始执行？[y/n] " answer
[ "$answer" = "y" ] && ./ralph_loop.sh
```

---

### 4.3 ralph prd --list - 查看需求列表

```bash
$ ralph prd --list

┌─────────────────────────────────────────┐
│ 需求列表                                 │
├─────────────────────────────────────────┤
│                                         │
│  ✅ feature/dashboard    5/5 完成       │
│  🔄 feature/login       3/5 进行中     │
│  ⏸️  feature/api        2/10 已暂停    │
│                                         │
│  共 3 个需求，1 个进行中                  │
│                                         │
└─────────────────────────────────────────┘
```

**实现逻辑**:

```bash
#!/bin/bash
# ralph_prd_list.sh

echo "需求列表："
echo ""

for branch in $(git branch | grep 'feature/' | sed 's/\*//' | tr -d ' '); do
    prd=$(git show "$branch:.ralph/prd.json" 2>/dev/null)
    
    if [ -n "$prd" ]; then
        total=$(echo "$prd" | jq '.userStories | length')
        completed=$(echo "$prd" | jq '[.userStories[] | select(.passes == true)] | length')
        
        # 状态判断
        if [ "$total" -eq "$completed" ]; then
            status="✅"
        elif git branch | grep -q "\* $branch"; then
            status="🔄"
        else
            status="⏸️"
        fi
        
        created=$(echo "$prd" | jq -r '.createdAt // "未知"' | cut -c1-10)
        echo "  $status $branch - $completed/$total 完成 ($created)"
    fi
done
```

---

### 4.4 ralph - 执行当前需求

```bash
$ ralph

Starting Ralph - Max iterations: 10
===============================================================
  Ralph Iteration 1 of 10
===============================================================

当前任务：story-001 - 添加登录表单组件
执行 Claude Code...
运行质量检查...
Git 提交：a1b2c3d
更新 prd.json...

迭代 1 完成。继续...
```

---

## 5. 存档机制

### 5.1 自动存档（分支切换时）

```bash
# 检测分支变化
CURRENT_BRANCH=$(jq -r '.branchName' .ralph/prd.json)
LAST_BRANCH=$(cat .ralph/.last-branch 2>/dev/null)

if [ "$CURRENT_BRANCH" != "$LAST_BRANCH" ] && [ -n "$LAST_BRANCH" ]; then
    # 存档
    DATE=$(date +%Y-%m-%d)
    FOLDER_NAME=$(echo "$LAST_BRANCH" | sed 's|^feature/||')
    ARCHIVE_FOLDER=".ralph/archive/$DATE-$FOLDER_NAME"
    
    mkdir -p "$ARCHIVE_FOLDER"
    cp .ralph/prd.json "$ARCHIVE_FOLDER/"
    cp .ralph/progress.txt "$ARCHIVE_FOLDER/"
    
    echo "已存档：$ARCHIVE_FOLDER"
    
    # 重置进度
    echo "# Ralph Progress Log" > .ralph/progress.txt
fi

# 记录当前分支
echo "$CURRENT_BRANCH" > .ralph/.last-branch
```

---

### 5.2 存档目录结构

```
.ralph/archive/
├── 2026-03-05-login/
│   ├── prd.json              # 最终任务列表
│   └── progress.txt          # 完整进度日志
├── 2026-03-04-dashboard/
│   └── ...
└── ...
```

---

### 5.3 清理策略

```bash
# 定期清理过期存档（心跳任务）
cleanup_archives() {
    local retention_days=${ARCHIVE_RETENTION_DAYS:-90}
    find .ralph/archive/ -type d -mtime +$retention_days -exec rm -rf {} \;
}
```

---

## 6. 循环执行流程

### 6.1 完整流程

```
┌─────────────────────────────────────────────────────────┐
│  循环开始                                                │
└─────────────────────────────────────────────────────────┘
    │
    ▼
┌─────────────────────────────────────────────────────────┐
│  1. 检查分支变化，自动存档                               │
└─────────────────────────────────────────────────────────┘
    │
    ▼
┌─────────────────────────────────────────────────────────┐
│  2. 切换到 prd.json 定义的分支                           │
└─────────────────────────────────────────────────────────┘
    │
    ▼
┌─────────────────────────────────────────────────────────┐
│  3. 读取 prd.json，选择 passes=false 的任务              │
└─────────────────────────────────────────────────────────┘
    │
    ▼
┌─────────────────────────────────────────────────────────┐
│  4. 读取 progress.txt，作为上下文                        │
└─────────────────────────────────────────────────────────┘
    │
    ▼
┌─────────────────────────────────────────────────────────┐
│  5. 调用 Claude Code 执行任务                            │
└─────────────────────────────────────────────────────────┘
    │
    ▼
┌─────────────────────────────────────────────────────────┐
│  6. 运行质量检查（typecheck/test）                      │
└─────────────────────────────────────────────────────────┘
    │
    ├─── 失败 ───► 记录失败，继续下次迭代
    │
    通过
    │
    ▼
┌─────────────────────────────────────────────────────────┐
│  7. Git 提交                                             │
└─────────────────────────────────────────────────────────┘
    │
    ▼
┌─────────────────────────────────────────────────────────┐
│  8. 更新 prd.json (passes: true)                        │
└─────────────────────────────────────────────────────────┘
    │
    ▼
┌─────────────────────────────────────────────────────────┐
│  9. 追加 progress.txt                                    │
└─────────────────────────────────────────────────────────┘
    │
    ▼
┌─────────────────────────────────────────────────────────┐
│  10. 退出检测（所有 passes=true？）                     │
└─────────────────────────────────────────────────────────┘
    │
    ├─── 是 ───► 输出 <promise>COMPLETE</promise>，退出
    │
    否
    │
    ▼
(回到步骤 1)
```

---

### 6.2 核心代码

```bash
#!/bin/bash
# ralph_loop.sh (简化版)

PRD_FILE=".ralph/prd.json"
PROGRESS_FILE=".ralph/progress.txt"
MAX_ITERATIONS=${1:-10}

for i in $(seq 1 $MAX_ITERATIONS); do
    echo "==============================================================="
    echo "  Ralph Iteration $i of $MAX_ITERATIONS"
    echo "==============================================================="
    
    # 1. 检查分支变化，自动存档
    check_branch_change
    
    # 2. 切换到 PRD 分支
    CURRENT_BRANCH=$(jq -r '.branchName' "$PRD_FILE")
    git checkout "$CURRENT_BRANCH" 2>/dev/null
    
    # 3. 获取下一个任务
    TASK=$(jq -r '.userStories[] | select(.passes == false) | @json' "$PRD_FILE" | head -1)
    
    if [ -z "$TASK" ]; then
        echo "✅ 所有任务完成！"
        echo "<promise>COMPLETE</promise>"
        exit 0
    fi
    
    # 4. 执行任务
    OUTPUT=$(claude --dangerously-skip-permissions --prompt "
完成任务：$(echo "$TASK" | jq -r '.title')
描述：$(echo "$TASK" | jq -r '.description')
验收标准：$(echo "$TASK" | jq -r '.acceptanceCriteria[]')
" 2>&1)
    
    # 5. 质量检查
    if ! npm run typecheck && npm run test; then
        echo "❌ 质量检查失败"
        continue
    fi
    
    # 6. Git 提交
    git add -A
    git commit -m "完成：$(echo "$TASK" | jq -r '.id') - $(echo "$TASK" | jq -r '.title')"
    
    # 7. 更新 prd.json
    TASK_ID=$(echo "$TASK" | jq -r '.id')
    jq "(.userStories[] | select(.id == \"$TASK_ID\")).passes = true" "$PRD_FILE" > tmp.json
    mv tmp.json "$PRD_FILE"
    
    # 8. 追加进度
    echo "## $(date) - $TASK_ID" >> "$PROGRESS_FILE"
    echo "- 完成：$(echo "$TASK" | jq -r '.title')" >> "$PROGRESS_FILE"
    echo "---" >> "$PROGRESS_FILE"
    
    echo "迭代 $i 完成。继续..."
done

echo "❌ 达到最大迭代次数"
exit 1
```

---

## 7. 权限机制

### 7.1 默认模式：全权限

```bash
# 同 snarktank/ralph
claude --dangerously-skip-permissions --prompt "..."
```

**理由**：
- ✅ 简单，无配置
- ✅ Git 分支隔离提供安全边界
- ✅ 质量检查防止烂代码
- ✅ 人类最终审查（PR/MR）

---

### 7.2 可选模式：限制模式

```bash
# .ralphrc 配置
PERMISSION_MODE="restricted"
ALLOWED_TOOLS="Write,Read,Edit,Bash(git *),Bash(npm *)"

# ralph_loop.sh 使用
claude --allowed-tools "$ALLOWED_TOOLS" --prompt "..."
```

**适用场景**：生产环境、核心项目

---

## 8. 配置文件

### 8.1 .ralphrc（可选）

```bash
# 项目配置
PROJECT_NAME="my-app"

# 权限模式
PERMISSION_MODE="full"  # full | restricted

# 限制模式配置
ALLOWED_TOOLS="Write,Read,Edit,Bash(git *),Bash(npm *)"

# 存档配置
ARCHIVE_RETENTION_DAYS=90

# 执行配置
MAX_ITERATIONS=10
QUALITY_CHECKS="npm run typecheck && npm run test"
```

---

## 9. 安全机制

### 9.1 多层保护

| 机制 | 说明 |
|------|------|
| **Git 分支隔离** | 每个需求在独立分支，不污染 main |
| **质量检查** | typecheck/test 必须通过才提交 |
| **进度追踪** | progress.txt 完整记录每次迭代 |
| **自动存档** | 分支切换时自动备份 |
| **人类审查** | PR/MR 需要人工 merge |

---

## 10. 对比分析

### 10.1 与 snarktank/ralph 对比

| 功能 | snarktank | Neo-Ralph |
|------|-----------|-----------|
| **分支管理** | ✅ | ✅ (同) |
| **需求列表** | ❌ | ✅ (新增) |
| **自动存档** | ✅ | ✅ (同) |
| **元数据** | ❌ | ✅ (createdAt 等) |
| **权限模式** | 全权限 | 可切换 |
| **配置** | ❌ | ✅ (可选) |

---

### 10.2 与 frankbria 对比

| 功能 | frankbria | Neo-Ralph |
|------|-----------|-----------|
| **需求隔离** | .ralph/ 子目录 | Git 分支 |
| **配置复杂度** | 高 | 低 |
| **文件数量** | 多 | 少 |
| **Git 耦合** | 低 | 高 |
| **学习成本** | 中 | 低 |

---

## 11. 最佳实践

### 11.1 分支命名

```bash
# 推荐格式
feature/<name>-<YYYYMMDD>

# 示例
feature/login-20260305
feature/dashboard-20260306
feature/api-refactor-20260307
```

### 11.2 任务拆分

```
✅ 好的任务（小）
- 添加数据库字段
- 添加 UI 组件
- 更新 API 端点

❌ 大的任务（拆分）
- 实现整个登录模块
- 重构所有 API
```

### 11.3 提交频率

```
每次迭代 = 1 次提交

保持 CI 绿色，不要累积多个任务再提交
```

---

## 12. 总结

Neo-Ralph 采用 **snarktank/ralph 的 Git 分支管理逻辑**，保持简洁：

1. **一个需求 = 一个分支** - Git 天然隔离
2. **最少文件** - prd.json + progress.txt
3. **自动存档** - 分支切换时备份
4. **需求列表** - 从 Git 分支读取
5. **人类审查** - PR/MR 人工 merge

**核心理念**：简单即美，Git 即真相。

---

## 附录

### A. 文件清单

| 文件 | 用途 |
|------|------|
| `ralph` | 主执行脚本 |
| `ralph_prd.sh` | 需求管理 |
| `ralph_loop.sh` | 循环执行 |
| `.ralph/prd.json` | 任务列表 |
| `.ralph/progress.txt` | 进度日志 |

### B. 快速开始

```bash
# 1. 创建需求
ralph prd login

# 2. 查看列表
ralph prd --list

# 3. 执行
ralph
```
