# dev-swarm

把一次需求从分析做到复盘，并且规定好谁能动、谁不能动。

dev-swarm is a git-native workflow for multi-agent software delivery. Claude Code writes, Cursor reviews and tests, and a human approves the gates that should not be automatic. The source of truth is a requirement folder in git, not the chat transcript.

## 它解决什么问题

多个编码代理同时改一个仓库时，常见的失败是：写的人自己宣布通过、聊天记录里找不到上一手停在哪、两个人的未提交改动缠在同一个工作区里。

dev-swarm 把这些收成仓库里的文件：

- 每个需求一个目录，阶段产物、缺陷台账、下一手都在里面。
- 开工前先核对 `next.agent` 是不是自己，并且 `blockers` 为空。不是自己的活，直接拒绝。
- 写代码的代理不能把自己写的缺陷标成「已验证」。验证交给另一个代理。
- 发布、部署、工具本身的推送，停下来等人拍板。
- 证据要写清工程、目录和命令。读过代码不算环境已经通过。

## 三个角色

| 角色 | 运行时 | 负责 |
|------|--------|------|
| `cc-dev` | Claude Code | 需求分析、方案、编码、单测、缺陷修复、脚本 |
| `cursor-qa` | Cursor | 方案审查、代码审查、自动化测试、发布后的验证 |
| `human` | 人 | 口径拍板、部署、实例信息、工具变更的批准 |

审查方只列差异和问题。修正回到动手方。同一角色不审查自己刚写完的结论。

## 阶段库和三条链

阶段库有 14 个编号。一条需求在创建时冻结自己的链，链是阶段库的子集。库里有、链上没有的阶段，不产生文件。

| Profile | 适合 | 链 |
|---------|------|----|
| `toolchain` | 工具和流程本身。没有数据库、没有发布动作 | 9 段：分析 → 方案 → 审查 → 编码 → 代码审查 → 单测 → 自动化测试 → 复盘 → 知识库 |
| `java-business` | 有接口、数据、部署的业务需求 | 13 段：在 toolchain 上增加服务拆分、提测、发布清单、发布验证 |
| `data-script` | 只写刷数或取数脚本，不改业务代码 | 7 段：分析 → 方案 → 审查 → 脚本与验证脚本 → 自动化测试 → 复盘 → 知识库 |

`java-business` 在进开发之前要有「产研测评审」记录。`data-script` 在真正改数据之前停下来等人确认。默认 profile 是 `toolchain`，写在 `dev-swarm.yaml`。

## 一个需求长什么样

```text
workspace/{需求名}/
├── services.yaml          要动哪些代码，以及这条需求用哪条链
├── 01-需求分析.md          编号等于阶段库编号，链上没有的阶段不出文件
├── _state/
│   ├── current.md         总控：当前阶段、下一手、阻塞项
│   ├── decisions.md       拍过的口径
│   └── override.yaml      可选，只覆盖这一条需求的配置
├── bug/BUG-{NNN}/         status.md 只记当前状态，log.md 只追加
└── reverify.md            开工信号：代码版本、要验什么、在哪验
```

`current.md` 里的下一手是结构化字段，不是一段聊天摘要：

```yaml
next:
  agent: cursor-qa
  action: 复验 BUG-062
  blockers: []
```

缺陷状态只用六个词：待确认、待修复、已修复待验、已验证、已关闭、作废。打回时把复验行写成「已打回」，同一次提交把状态改回「待修复」。

## 仓库地图

```text
├── dev-swarm.yaml                 仓级配置：角色、阶段库、默认 profile、推送策略
├── skill/
│   ├── protocol/agent-contract.md 权限矩阵和台账协议
│   ├── pipeline/                  三条链的门禁，以及 01~14 操作卡
│   ├── adapters/cc/               Claude Code 的开工方式
│   ├── adapters/cursor/           Cursor 的开工方式
│   ├── scripts/                   工作区隔离、点名暂存、术语扫描
│   └── tests/                     协议回归
├── console/                       本地只读看板
├── bin/dev-swarm                  看板的启动、停止、状态、日志
├── hooks/                         pre-commit / pre-push
└── SPEC-*.md                      目录、提交、脱敏、变更
```

配置分三层，深合并，需求级覆盖仓级，仓级覆盖内置默认值：`skill/config/defaults.yaml` → `dev-swarm.yaml` → `workspace/{需求}/_state/override.yaml`。

## 开始使用

需要 Git、Python 3、Node.js。

```bash
git clone https://github.com/jerrey17/jerrey17.git
cd jerrey17
bash hooks/setup.sh
bash skill/tests/run_all.sh
```

`hooks/setup.sh` 把 `core.hooksPath` 指到仓库里的 `hooks/`。工具路径（`skill/`、`console/`、规范、仓级配置、本 README）的推送必须在提交说明里带 `Approved-by: human`。需求目录里的协作记录由负责的代理提交。默认 `push_policy` 是 `local-only`：阶段结束只提交，不推送。

让 Claude Code 加载适配：

```bash
mkdir -p .claude/skills/dev-swarm
cp skill/adapters/cc/SKILL.md .claude/skills/dev-swarm/SKILL.md
```

打开 `SKILL.md`，把 `MONOREPO` 设成这个克隆的绝对路径。触发词用 `/dev-swarm`、`swarm 需求`、`swarm 进度`。更宽的句子，例如「开始新需求」，不会启动这条流程。

让 Cursor 接手时，从该需求的 `_state/current.md` 抄出下一手，按 `skill/adapters/cursor/开工说明.md` 里的模板贴进会话。Cursor 先核对下一手是不是 `cursor-qa`，再读权限矩阵，只改允许改的文件。

## 本地看板

看板只监听本机 `127.0.0.1`，用来看需求走到哪、谁被挡住、哪些复验还没做。它不代替代理写代码。

```bash
cp console/.local-state/console.config.example.yaml console/.local-state/console.config.yaml
```

把 `dataClone` 设成这个仓库的绝对路径。这个目录里必须有 `dev-swarm.yaml`。然后：

```bash
./bin/dev-swarm start
```

页面在 http://127.0.0.1:5173 ，接口在 http://127.0.0.1:4600 。停止、状态和日志：

```bash
./bin/dev-swarm stop
./bin/dev-swarm status
./bin/dev-swarm logs
```

登录账号不进仓库。用 `node console/server/accounts.mjs add <账号>` 在本机生成 `console/.local-state/accounts.local.yaml`。

## 公开仓库的边界

`workspace/` 里的需求正文、缺陷记录和复验表默认不提交，只保留空目录。密码、token、连接串和 webhook 真值不入库；样例用占位符，真值放在已被 `.gitignore` 覆盖的本机文件里。

别人拿走这个框架之后，用自己的需求目录在本地跑。三条链里的业务名词可以按自己的发布方式改，协议不变的部分是：角色分开、下一手字段化、验证者不写下结论、关键门等人。

## 规范

| 文件 | 内容 |
|------|------|
| [SPEC-目录规范.md](SPEC-目录规范.md) | 需求目录里每个文件的位置 |
| [SPEC-提交规约.md](SPEC-提交规约.md) | 谁可以提交，什么情况下可以推送 |
| [SPEC-脱敏红线.md](SPEC-脱敏红线.md) | 哪些值永远不能进 git |
| [SPEC-变更流程.md](SPEC-变更流程.md) | 改协议和改配置的路径 |
| [skill/protocol/agent-contract.md](skill/protocol/agent-contract.md) | 权限矩阵、缺陷六态、开工信号 |

许可证文件还没有选定。在选定之前，请不要把这份代码并进别的产品。
