# Neo-Ralph 架构设计文档

> **Neo-Ralph = frankbria/ralph-claude-code 的躯干 + snarktank/ralph 的灵魂**

本文档描述 Neo-Ralph 的架构设计、核心模块和数据流。

---

## 1. 设计目标

### 1.1 核心问题

现有两个 Ralph 实现各有优劣：

| 项目 | 优势 | 劣势 |
|------|------|------|
| **snarktank/ralph** | 任务约束严格、Git 记忆清晰、PRD 生成友好 | 无系统保护、无速率限制、无会话管理 |
| **frankbria/ralph-claude-code** | 系统保护完善、速率限制、断路器、会话管理 | 任务约束弱、PRD 需手动编写、Git 记忆不强制 |

### 1.2 Neo-Ralph 目标

**融合两者优点，打造一个既安全又高效的自主 AI 开发循环工具：**

- ✅ 保留 frankbria 的系统保护（速率限制、断路器、会话管理）
- ✅ 引入 snarktank 的任务约束（单任务/迭代、强制质量检查）
- ✅ 新增交互式 PRD 生成能力
- ✅ 增强 Git 记忆机制
- ✅ 支持短任务/长任务双模式

---

## 2. 系统架构

### 2.1 整体架构

```
┌─────────────────────────────────────────────────────────────────┐
│                        用户接口层                                │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐              │
│  │ ralph       │  │ ralph-prd   │  │ ralph-monitor│             │
│  │ (主循环)    │  │ (PRD 生成)  │  │ (监控仪表盘) │              │
│  └─────────────┘  └─────────────┘  └─────────────┘              │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│                        核心引擎层                                │
│  ┌─────────────────────────────────────────────────────────┐    │
│  │                    ralph_loop.sh                         │    │
│  │  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐   │    │
│  │  │ 模式选择器    │  │ 任务调度器    │  │ 退出检测器    │   │    │
│  │  │ Mode Selector│  │ Task Scheduler│  │ Exit Detector │   │    │
│  │  └──────────────┘  └──────────────┘  └──────────────┘   │    │
│  │  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐   │    │
│  │  │ 速率限制器    │  │ 断路器       │  │ 会话管理器    │   │    │
│  │  │ Rate Limiter │  │ Circuit      │  │ Session      │   │    │
│  │  └──────────────┘  └──────────────┘  └──────────────┘   │    │
│  └─────────────────────────────────────────────────────────┘    │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│                        功能模块层                                │
│  ┌────────────┐ ┌────────────┐ ┌────────────┐ ┌────────────┐   │
│  │ Git Memory │ │Quality     │ │ PRD        │ │ Response   │   │
│  │ (Git 记忆)  │ │Check(质检) │ │Generator   │ │ Analyzer   │   │
│  │            │ │            │ │(PRD 生成)  │ │(响应分析)  │   │
│  └────────────┘ └────────────┘ └────────────┘ └────────────┘   │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│                        基础设施层                                │
│  ┌────────────┐ ┌────────────┐ ┌────────────┐ ┌────────────┐   │
│  │ Claude     │ │ Git        │ │ File       │ │ Logging    │   │
│  │ Code CLI   │ │ Repository │ │ System     │ │ System     │   │
│  └────────────┘ └────────────┘ └────────────┘ └────────────┘   │
└─────────────────────────────────────────────────────────────────┘
```

---

## 3. 核心模块设计

### 3.1 模式选择器 (Mode Selector)

**职责**: 根据任务复杂度自动选择运行模式

```
┌─────────────────────────────────────────────┐
│            模式选择逻辑                      │
├─────────────────────────────────────────────┤
│  输入：PRD 文件 / 任务列表                   │
│  处理：估算任务复杂度                        │
│  输出：short | long | auto                   │
└─────────────────────────────────────────────┘

复杂度估算因子:
- 任务数量
- 每个任务的字数
- 是否涉及多模块集成
- 是否有前端 UI 变更
- 是否有数据库变更

决策树:
IF 任务数 <= 3 AND 总字数 < 1000
  → SHORT_MODE (无状态，每次 fresh context)
ELSE IF 任务数 > 10 OR 有复杂集成
  → LONG_MODE (有状态，保持会话连续性)
ELSE
  → AUTO_MODE (根据运行时动态判断)
```

### 3.2 任务调度器 (Task Scheduler)

**职责**: 强制单任务/迭代约束

```
┌─────────────────────────────────────────────┐
│           任务调度流程                       │
├─────────────────────────────────────────────┤
│  1. 读取 .ralph/prd.json                    │
│  2. 筛选 passes=false 的任务                 │
│  3. 按 priority 升序排序                     │
│  4. 返回 TOP 1 任务（强制单任务）             │
│  5. 锁定任务（防止并发）                     │
└─────────────────────────────────────────────┘

关键约束:
- 每次迭代只处理 1 个任务
- 任务完成后才能选择下一个
- 支持任务依赖检查
- 唯一任务源：prd.json（结构化）
```

### 3.3 质量检查器 (Quality Checker)

**职责**: 强制质量检查，防止烂代码提交

```
┌─────────────────────────────────────────────┐
│           质量检查流程                       │
├─────────────────────────────────────────────┤
│  1. 任务执行完成                            │
│  2. 运行 .ralphrc 中定义的检查命令           │
│     - typecheck                             │
│     - lint                                  │
│     - test                                  │
│     - browser check (可选)                  │
│  3. 全部通过 → 允许提交                      │
│  4. 任一失败 → 记录失败，下次重试            │
└─────────────────────────────────────────────┘

配置示例 (.ralphrc):
QUALITY_CHECKS="npm run typecheck && npm run lint && npm run test"
ENFORCE_QUALITY_CHECKS=true
ENABLE_BROWSER_CHECK=false
```

### 3.4 Git 记忆模块 (Git Memory)

**职责**: 记录每次迭代的 git 状态，增强可追溯性

```
┌─────────────────────────────────────────────┐
│           Git 记忆记录流程                    │
├─────────────────────────────────────────────┤
│  1. 任务完成且质检通过                       │
│  2. 执行 git commit                         │
│  3. 记录到 .ralph/logs/git_history.log      │
│     格式：timestamp | task_id | commit_hash │
│  4. 追加到 progress.txt                     │
│  5. 更新 AGENTS.md (项目约定)               │
└─────────────────────────────────────────────┘

文件结构:
.ralph/
├── logs/
│   └── git_history.log    # Git 提交历史
├── progress.txt           # 迭代学习日志
└── AGENTS.md              # 项目约定和坑点
```

### 3.5 PRD 生成器 (PRD Generator)

**职责**: 交互式生成结构化 PRD 文档

```
┌─────────────────────────────────────────────┐
│           PRD 生成流程                       │
├─────────────────────────────────────────────┤
│  1. 用户输入功能描述                        │
│  2. Claude 询问澄清问题                     │
│     - 核心用户价值                          │
│     - 具体场景                              │
│     - 技术约束                              │
│     - 验收标准                              │
│  3. 生成结构化 PRD 文档                      │
│  4. 转换为 prd.json / fix_plan.md           │
└─────────────────────────────────────────────┘

输出文件:
.ralph/specs/prd-[feature].md  # 原始 PRD 文档（人类可读，仅供参考）
.ralph/prd.json                # 结构化任务列表（唯一任务源）
```

### 3.6 退出检测器 (Exit Detector)

**职责**: 智能判断何时停止循环

```
┌─────────────────────────────────────────────┐
│           双条件退出检测                     │
├─────────────────────────────────────────────┤
│  条件 1: completion_indicators >= 2         │
│    (从自然语言模式检测完成信号)              │
│                                             │
│  条件 2: EXIT_SIGNAL = true                 │
│    (Claude 明确输出退出信号)                 │
│                                             │
│  退出 = 条件 1 AND 条件 2                    │
└─────────────────────────────────────────────┘

其他退出条件:
- 所有任务标记完成
- 断路器打开超过阈值
- 达到最大迭代次数
- API 5 小时限制到达
```

---

## 4. 数据流

### 4.1 完整执行流程

```
用户启动
    │
    ▼
┌─────────────────┐
│ 读取 .ralphrc   │
│ 加载配置        │
└─────────────────┘
    │
    ▼
┌─────────────────┐
│ 速率限制检查    │
│ (是否超限？)    │
└─────────────────┘
    │
    ├─────── 是 ───────► 等待重置 / 退出
    │
    否
    │
    ▼
┌─────────────────┐
│ 断路器状态检查  │
│ (是否打开？)    │
└─────────────────┘
    │
    ├─────── 是 ───────► 等待冷却 / 手动干预
    │
    否
    │
    ▼
┌─────────────────┐
│ 模式选择        │
│ (short/long)    │
└─────────────────┘
    │
    ▼
┌─────────────────┐
│ 获取下一个任务  │
│ (单任务约束)    │
└─────────────────┘
    │
    ├─────── 无任务 ─────► 退出 (项目完成)
    │
    有任务
    │
    ▼
┌─────────────────┐
│ 执行 Claude Code│
│ (带上下文)      │
└─────────────────┘
    │
    ▼
┌─────────────────┐
│ 质量检查        │
│ (tests/lint)    │
└─────────────────┘
    │
    ├─────── 失败 ─────► 记录失败，下次重试
    │
    通过
    │
    ▼
┌─────────────────┐
│ Git 提交         │
│ 更新状态        │
└─────────────────┘
    │
    ▼
┌─────────────────┐
│ 退出检测        │
│ (是否完成？)    │
└─────────────────┘
    │
    ├─────── 是 ───────► 退出
    │
    否
    │
    ▼
(循环回速率限制检查)
```

### 4.2 状态文件

| 文件 | 用途 | 更新频率 |
|------|------|----------|
| `.ralphrc` | 项目配置 | 初始化时 |
| `.ralph/prd.json` | **唯一任务源**（结构化） | 每次任务完成 |
| `.ralph/specs/prd-*.md` | 原始 PRD 文档（人类可读） | PRD 生成时 |
| `.ralph/progress.txt` | 学习日志 | 每次迭代 |
| `.ralph/logs/git_history.log` | Git 历史 | 每次提交 |
| `.ralph/logs/ralph.log` | 执行日志 | 实时 |
| `.ralph/status.json` | 当前状态 | 每次循环 |

**注意**: `fix_plan.md` 已废弃，不再使用。所有任务统一存储为 `prd.json`。
| `.ralph/.ralph_session` | 会话 ID | 会话变化时 |
| `.ralph/AGENTS.md` | 项目约定 | 每次迭代 |

---

## 5. 关键算法

### 5.1 任务复杂度估算

```bash
estimate_complexity() {
    local prd_file=$1
    
    # 基础分数：任务数量
    local task_count=$(jq '.userStories | length' "$prd_file")
    local score=$task_count
    
    # 字数因子
    local word_count=$(wc -w < "$prd_file")
    score=$((score + word_count / 200))
    
    # 复杂度关键词
    local has_integration=$(grep -ci "集成\|对接\|API\|microservice" "$prd_file")
    local has_ui=$(grep -ci "界面\|UI\|前端\|component" "$prd_file")
    local has_db=$(grep -ci "数据库\|migration\|schema" "$prd_file")
    
    score=$((score + has_integration * 3 + has_ui * 2 + has_db * 2))
    
    echo $score
}

# 阈值判断
if [ $score -lt 5 ]; then
    MODE="short"
elif [ $score -gt 15 ]; then
    MODE="long"
else
    MODE="auto"
fi
```

### 5.2 速率限制算法

```bash
check_rate_limit() {
    local current_hour=$(date +%Y%m%d%H)
    local last_hour=$(cat .ralph/.last_hour 2>/dev/null || echo "0")
    local call_count=$(cat .ralph/.call_count 2>/dev/null || echo "0")
    
    # 小时重置
    if [ "$current_hour" != "$last_hour" ]; then
        call_count=0
        echo "$current_hour" > .ralph/.last_hour
    fi
    
    # 检查是否超限
    if [ "$call_count" -ge "$MAX_CALLS_PER_HOUR" ]; then
        echo "速率限制已到达 ($call_count/$MAX_CALLS_PER_HOUR)"
        return 1
    fi
    
    # 增加计数
    call_count=$((call_count + 1))
    echo "$call_count" > .ralph/.call_count
    
    return 0
}
```

### 5.3 断路器状态机

```
状态转换:

CLOSED (正常) 
    │
    ├─ 3 次无进度 ────────┐
    ├─ 5 次同错 ──────────┼──► OPEN (打开)
    │                    │
    │                    │ 等待 CB_COOLDOWN_MINUTES
    │                    │
    │                    ▼
    │              HALF_OPEN (半开)
    │                    │
    │                    ├─ 成功 ──► CLOSED
    │                    │
    │                    └─ 失败 ──► OPEN
    │
    └─ 手动重置 ────────────────► CLOSED
```

---

## 6. 配置设计

### 6.1 .ralphrc 配置项

```bash
# 项目基础配置
PROJECT_NAME="my-project"
PROJECT_TYPE="typescript"

# 模式配置
DEFAULT_MODE="auto"  # short | long | auto

# 速率限制
MAX_CALLS_PER_HOUR=100

# 断路器
CB_NO_PROGRESS_THRESHOLD=3
CB_SAME_ERROR_THRESHOLD=5
CB_COOLDOWN_MINUTES=30
CB_AUTO_RESET=false

# 质量检查（新增）
QUALITY_CHECKS="npm run typecheck && npm run test"
ENFORCE_QUALITY_CHECKS=true
ENABLE_BROWSER_CHECK=false
BROWSER_CHECK_COMMAND="npx playwright test"

# Git 配置（新增）
GIT_COMMIT_ON_SUCCESS=true
GIT_COMMIT_MESSAGE_PREFIX="完成："

# 会话管理
SESSION_CONTINUITY=true
SESSION_EXPIRY_HOURS=24

# Claude Code 配置
CLAUDE_CODE_CMD="claude"
CLAUDE_TIMEOUT_MINUTES=15
CLAUDE_OUTPUT_FORMAT="json"

# 工具权限
ALLOWED_TOOLS="Write,Read,Edit,Bash(git *),Bash(npm *),Bash(pytest)"
```

---

## 7. CLI 设计

### 7.1 主命令

```bash
# 启动 Ralph（默认 auto 模式）
ralph

# 指定模式
ralph --mode short    # 短任务模式
ralph --mode long     # 长任务模式
ralph --mode auto     # 自动模式

# 严格模式（强制质量检查）
ralph --strict

# 带监控
ralph --monitor

# 实时流输出
ralph --live

# 自定义超时
ralph --timeout 30

# 查看状态
ralph --status
```

### 7.2 新增命令

```bash
# 生成交互式 PRD
ralph-prd

# 导入 PRD 文档
ralph-import requirements.md my-project

# 启用项目（交互式向导）
ralph-enable

# Git 记忆查看
ralph-git-history

# 电路 breaker 状态
ralph --circuit-status
```

---

## 8. 文件结构

```
neo-ralph/
├── src/
│   ├── ralph_loop.sh          # 核心循环（改造重点）
│   ├── ralph_cli.sh           # CLI 参数解析
│   ├── ralph_monitor.sh       # 监控仪表盘
│   ├── ralph_prd.sh           # 【新增】PRD 生成器
│   ├── git_memory.sh          # 【新增】Git 记忆模块
│   ├── quality_checker.sh     # 【新增】质量检查器
│   ├── mode_selector.sh       # 【新增】模式选择器
│   ├── circuit_breaker.sh     # 断路器模块
│   └── session_manager.sh     # 会话管理模块
│
├── config/
│   └── .ralphrc.template      # 配置模板（增强版）
│
├── docs/
│   ├── neo-ralph-design.md    # 本文档
│   ├── modification-plan.md   # 修改计划
│   └── user-guide.md          # 用户指南
│
├── tests/
│   ├── unit/
│   │   ├── test_mode_selector.bats
│   │   ├── test_quality_checker.bats
│   │   └── test_git_memory.bats
│   └── integration/
│       └── test_full_loop.bats
│
└── ... (其他上游文件)
```

---

## 9. 安全与边界

### 9.1 安全保护

| 保护机制 | 描述 |
|----------|------|
| 速率限制 | 防止 API 费用失控 |
| 断路器 | 防止无限错误循环 |
| 工具白名单 | 防止危险命令执行 |
| 会话过期 | 防止长期上下文污染 |
| 质量检查 | 防止烂代码累积 |

### 9.2 边界约束

| 约束 | 实现方式 |
|------|----------|
| 单任务/迭代 | 任务调度器强制返回 1 个任务 |
| 强制质检 | 质检失败则跳过提交 |
| PRD 格式 | 严格 prd.json schema |
| 前端验证 | 可选 browser check |

---

## 10. 性能考虑

### 10.1 Token 优化

- **SHORT_MODE**: 每次 fresh context，Token 消耗低
- **LONG_MODE**: 保持会话，但限制上下文长度

### 10.2 执行优化

- 并行质量检查（typecheck + lint + test）
- 增量 Git 提交（只提交变更文件）
- 日志轮转（防止日志文件过大）

---

## 11. 扩展性

### 11.1 插件系统（未来）

```bash
# 预留插件接口
plugins/
├── prd-generator/     # PRD 生成插件
├── quality-checks/    # 质量检查插件
└── notifications/     # 通知插件
```

### 11.2 多 AI 工具支持（未来）

```bash
# 支持切换 AI 工具
CLAUDE_CODE_CMD="claude"  # 或
AMP_CMD="amp"
```

---

## 12. 附录：prd.json Schema

### 完整 Schema

```json
{
  "$schema": "http://json-schema.org/draft-07/schema#",
  "type": "object",
  "required": ["projectName", "userStories"],
  "properties": {
    "projectName": {
      "type": "string",
      "description": "项目名称"
    },
    "createdAt": {
      "type": "string",
      "format": "date-time",
      "description": "PRD 创建时间"
    },
    "userStories": {
      "type": "array",
      "items": {
        "type": "object",
        "required": ["id", "title", "passes"],
        "properties": {
          "id": {
            "type": "string",
            "description": "任务唯一 ID",
            "pattern": "^story-[0-9]{3}$"
          },
          "title": {
            "type": "string",
            "description": "任务标题"
          },
          "description": {
            "type": "string",
            "description": "任务描述（用户故事格式）"
          },
          "acceptanceCriteria": {
            "type": "array",
            "items": {"type": "string"},
            "description": "验收标准列表"
          },
          "technicalNotes": {
            "type": "array",
            "items": {"type": "string"},
            "description": "技术实现备注"
          },
          "priority": {
            "type": "integer",
            "minimum": 1,
            "description": "优先级（1=最高）"
          },
          "passes": {
            "type": "boolean",
            "description": "是否完成"
          },
          "completedAt": {
            "type": ["string", "null"],
            "format": "date-time",
            "description": "完成时间"
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

### 示例

```json
{
  "projectName": "user-login-feature",
  "createdAt": "2026-03-05T10:00:00Z",
  "userStories": [
    {
      "id": "story-001",
      "title": "添加登录表单组件",
      "description": "作为用户，我想要输入邮箱密码登录，以便访问个人账户",
      "acceptanceCriteria": [
        "表单包含邮箱和密码输入框",
        "密码输入框显示为掩码",
        "提交后验证用户凭证"
      ],
      "technicalNotes": [
        "使用 React Hook Form",
        "密码前端加密传输"
      ],
      "priority": 1,
      "passes": false,
      "completedAt": null
    },
    {
      "id": "story-002",
      "title": "实现 JWT token 验证",
      "description": "作为后端，我想要验证 JWT token，以便确保请求合法性",
      "acceptanceCriteria": [
        "验证 token 签名",
        "检查 token 过期时间",
        "无效 token 返回 401"
      ],
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

## 13. 存档机制

### 设计灵感

借鉴 [snarktank/ralph](https://github.com/snarktank/ralph) 的分支切换存档策略，并在此基础上增强。

### snarktank/ralph 的存档方式

```bash
# 存档触发条件：分支变更时
if [ "$CURRENT_BRANCH" != "$LAST_BRANCH" ]; then
    ARCHIVE_FOLDER="$ARCHIVE_DIR/$DATE-$FOLDER_NAME"
    cp "$PRD_FILE" "$ARCHIVE_FOLDER/"       # 存档 prd.json
    cp "$PROGRESS_FILE" "$ARCHIVE_FOLDER/"  # 存档 progress.txt
fi
```

**优点**：
- ✅ 自动触发，无需手动操作
- ✅ 按特征分组，清晰
- ✅ 保留上下文（prd.json + progress.txt）

**缺点**：
- ❌ 只在分支切换时存档
- ❌ 同一分支内多次运行不存档
- ❌ 无法回滚到某次迭代

---

### Neo-Ralph 存档策略

**三层存档机制**：

```
┌─────────────────────────────────────────────────────────┐
│                  Neo-Ralph 存档体系                      │
├─────────────────────────────────────────────────────────┤
│                                                         │
│  1. Git 自动提交（每次迭代）                            │
│     └─ prd.json 每次更新都 commit                        │
│                                                         │
│  2. 分支切换存档（自动）                                │
│     └─ 检测到分支变化时，备份到 archive/by-branch/      │
│                                                         │
│  3. 里程碑存档（关键节点）                              │
│     ├─ PRD 生成完成 → archive/prd-initial.json         │
│     ├─ 所有任务完成 → archive/prd-complete.json        │
│     └─ 手动触发 → archive/prd-manual-YYYYMMDD.json     │
│                                                         │
└─────────────────────────────────────────────────────────┘
```

---

### 存档目录结构

```
.ralph/
├── prd.json                    # 当前任务列表
├── progress.txt                # 当前进度日志
├── .last-run-id                # 上次运行 ID
├── .last-branch                # 上次分支名称
└── archive/
    ├── by-branch/              # 按分支存档
    │   ├── 2026-03-05-login-feature/
    │   │   ├── prd.json
    │   │   └── progress.txt
    │   └── 2026-03-04-dashboard/
    │       └── ...
    ├── by-milestone/           # 按里程碑存档
    │   ├── prd-initial.json    # 初始版本
    │   ├── prd-complete.json   # 完成版本
    │   └── prd-manual-20260305.json  # 手动存档
    └── by-iteration/           # 按迭代存档（可选）
        ├── iteration-001.json
        ├── iteration-002.json
        └── ...
```

---

### 存档触发逻辑

```bash
# 1. 分支切换检测（启动时）
check_branch_change() {
    local current_branch=$(jq -r '.branchName // empty' .ralph/prd.json)
    local last_branch=$(cat .ralph/.last-branch 2>/dev/null || echo "")
    
    if [ -n "$current_branch" ] && [ -n "$last_branch" ] && [ "$current_branch" != "$last_branch" ]; then
        archive_previous_run "$last_branch"
    fi
    
    echo "$current_branch" > .ralph/.last-branch
}

# 2. 里程碑存档（关键节点）
archive_milestone() {
    local milestone=$1  # initial|complete|manual
    local timestamp=$(date +%Y%m%d-%H%M%S)
    
    if [ "$milestone" = "initial" ]; then
        cp .ralph/prd.json .ralph/archive/by-milestone/prd-initial.json
    elif [ "$milestone" = "complete" ]; then
        cp .ralph/prd.json .ralph/archive/by-milestone/prd-complete.json
    elif [ "$milestone" = "manual" ]; then
        cp .ralph/prd.json .ralph/archive/by-milestone/prd-manual-$timestamp.json
    fi
}

# 3. Git 自动提交（每次迭代后）
git_auto_commit_prd() {
    if [ "$GIT_AUTO_COMMIT_PRD" = "true" ]; then
        git add .ralph/prd.json
        git commit -m "更新任务状态：$task_id"
    fi
}
```

---

### 配置项 (.ralphrc)

```bash
# 存档配置
ARCHIVE_ENABLED=true
ARCHIVE_ON_BRANCH_CHANGE=true      # 分支切换时存档（默认开启）
ARCHIVE_ON_MILESTONE=true          # 里程碑时存档（默认开启）
ARCHIVE_ON_ITERATION=false         # 每次迭代都存档（默认关闭，太占空间）
ARCHIVE_RETENTION_DAYS=90          # 备份保留天数（默认 90 天）
GIT_AUTO_COMMIT_PRD=true           # Git 自动提交 prd.json（默认开启）
```

---

### 存档文件清单

| 文件 | Git 提交 | 分支存档 | 里程碑存档 | 迭代存档 |
|------|----------|----------|------------|----------|
| `prd.json` | ✅ | ✅ | ✅ | 可选 |
| `progress.txt` | ✅ | ✅ | ❌ | ❌ |
| `AGENTS.md` | ✅ | ✅ | ❌ | ❌ |
| `status.json` | ❌ | ❌ | ❌ | ❌ |
| `logs/*` | 可选 | ❌ | ❌ | ❌ |

---

### 回滚操作

```bash
# 查看存档历史
ls -la .ralph/archive/by-branch/
ls -la .ralph/archive/by-milestone/

# 回滚到某个版本
cp .ralph/archive/by-branch/2026-03-05-login-feature/prd.json .ralph/prd.json

# Git 回滚（如果启用了 Git 自动提交）
git log --oneline .ralph/prd.json
git checkout <commit-hash> -- .ralph/prd.json
```

---

### 清理策略

```bash
# 定期清理过期存档（心跳任务）
cleanup_old_archives() {
    local retention_days=${ARCHIVE_RETENTION_DAYS:-90}
    find .ralph/archive/by-branch/ -type d -mtime +$retention_days -exec rm -rf {} \;
    find .ralph/archive/by-milestone/ -type f -mtime +$retention_days -exec rm -f {} \;
}
```

---

## 14. 总结

Neo-Ralph 通过以下设计实现"既安全又高效"的目标：

1. **模式双轨制** - 短任务轻量，长任务连续
2. **强制单任务** - 避免上下文过载
3. **质量门禁** - 防止烂代码累积
4. **Git 记忆** - 完整可追溯
5. **系统保护** - 速率限制 + 断路器
6. **统一任务源** - 只用 prd.json（结构化）
7. **三层存档** - Git + 分支 + 里程碑

下一步：编写 `modification-plan.md` 详细修改清单。
