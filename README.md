# dev-swarm monorepo

多 agent 研发流程的**单仓唯一真相源**：工具（引擎 + 看板）与需求数据同仓，一套 git 历史做迭代版本管理，整仓 tag 即版本。

## 目录结构

```text
├── dev-swarm.yaml          仓级主配置（三级分层第 2 级）
├── SPEC-目录规范.md         需求目录结构规约（全体 agent 强制）
├── SPEC-提交规约.md         分路径提交与 push 边界（含 §8.9 Approved-by）
├── SPEC-脱敏红线.md         敏感信息分口径
├── hooks/                  pre-push（工具路径强制 Approved-by: human）+ setup.sh
├── skill/                  dev-swarm 引擎：adapters/ protocol/ pipeline/ config/ scripts/ tests/
├── console/                只读看板：client（Vite+React）+ server（Express，独立 clone 轮询）
└── workspace/{需求名}/      需求协作数据（services.yaml + 01~12 产物 + _state/ + bug/ + reverify.md）
```

## 快速开始

```bash
git clone https://github.com/jerrey17/jerrey17.git
bash hooks/setup.sh        # 安装 pre-push 拦截（core.hooksPath=hooks）
```

- 全部 agent 开工前读物：`skill/protocol/agent-contract.md`（权限矩阵/台账三文件/开工信号）
- 回归：`bash skill/tests/run_all.sh`（四场景全绿为工具迭代前置）
- 看板：`cd console && npm install && npm run dev`（数据源配 `.local-state/console.config.yaml`）

## 规范索引

| 规范 | 内容 |
|------|------|
| 目录规范 | workspace/ 一级=需求名；固定子结构；`_state/` 命名承载隔离约束不可改 |
| 提交规约 | workspace/ 数据由负责 agent 自主提交；skill/ console/ 仓根为工具迭代，push 须 `Approved-by: human` |
| 脱敏红线 | 密码/token/连接串/webhook 绝不入库；内网实例地址不进公开仓库 |

## 历史

本仓原为 merge 测试仓（test-01~04.md 等测试文件已于 2026-10-03 经 human 批准清理）。
