# Neo-Ralph Project

## 项目定位

**Neo-Ralph = frankbria/ralph-claude-code 的躯干 + snarktank/ralph 的灵魂**

在 frankbria/ralph-claude-code 基础上融合 snarktank/ralph 的优点，打造新一代自主 AI 开发循环工具。

## 核心目标

| 特性 | 来源 | 状态 |
|------|------|------|
| 单任务/迭代约束 | snarktank | 🔄 待实现 |
| 强制质量检查 | snarktank | 🔄 待实现 |
| 交互式 PRD 生成 | snarktank | 🔄 待实现 |
| Git 核心记忆 | snarktank | 🔄 待实现 |
| 速率限制 | frankbria | ✅ 已有 |
| 断路器 | frankbria | ✅ 已有 |
| 会话管理 | frankbria | ✅ 已有 |
| 实时监控 | frankbria | ✅ 已有 |

## 改造计划

### Phase 1: 核心约束 (P0)
- [ ] 修改 `ralph_loop.sh` 强制每次只处理 1 个任务
- [ ] 在 `.ralphrc` 中添加 `QUALITY_CHECKS` 配置
- [ ] 在循环中添加质量检查强制执行

### Phase 2: PRD 生成 (P1)
- [ ] 创建 `ralph_prd.sh` 命令
- [ ] 集成 snarktank 的 PRD 技能
- [ ] 支持交互式需求澄清

### Phase 3: Git 记忆增强 (P1)
- [ ] 创建 `git_memory.sh` 模块
- [ ] 记录每次迭代的 git 状态
- [ ] 增强 `progress.txt` 更新逻辑

### Phase 4: 模式切换 (P2)
- [ ] 添加 `--mode` CLI 参数
- [ ] 实现 short/long/auto 三种模式
- [ ] 根据任务复杂度自动判断

## 使用方式

```bash
# 生成交互式 PRD
ralph-prd

# 短任务模式（严格）
ralph --mode short --monitor

# 长任务模式（宽松）
ralph --mode long --monitor

# 自动模式（推荐）
ralph --mode auto --monitor
```

## 项目链接

- **GitHub**: https://github.com/sickone075-web/neo-ralph
- **上游**: https://github.com/frankbria/ralph-claude-code
- **灵感来源**: https://github.com/snarktank/ralph
