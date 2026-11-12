---
title: "Hermes Agent v0.16 Kanban Swarm 拆解：一个建图函数，和它不讲情面的默认值"
date: 2026-11-12T16:29:15+08:00
tags: ["Hermes Agent", "多智能体编排"]
author: "Facecode"
---

# Hermes Agent v0.16 Kanban Swarm 拆解：一个建图函数，和它不讲情面的默认值

设想一次 swarm 跑完：六张卡全绿，产物目录却是空的。再设想 verifier 在 summary 里写了「证据不足」，synthesizer 照常开工——门控根本没生效。还有一种更安静的失败：一张卡在 `ready` 里躺了两天，没人报错，也没人收到通知，因为 `assignee` 写的是一个不存在的 profile。

三个现象看起来无关，其实指向同一件事：Hermes 的 Kanban 比它的宣传口径朴素得多。它没有额外的消息总线，没有第二套调度器，也没有内容级门控。理解这一点，才不会把「多智能体平台」当成魔法。

先纠一个版本。`hermes kanban swarm` 是 v0.15.0（tag `v2026.5.28`，Velocity Release，PR #28443）引入的；v0.16.0（tag `v2026.6.5`，The Surface Release）在 Kanban 上的增量都在运维层，例如 `goal_mode` 卡片的 goal 循环、任务附件与 body 图片喂给 worker 的 vision、`default_assignee` 兜底、每 profile 并发上限、单次 run 的 terminate 端点。所以本文讲的是「v0.16 时间点上的 swarm 全貌」，机制部分以 tag `v2026.6.5` 的源码与文档为准。

## 一、swarm 是一个建图函数

打开 `hermes_cli/kanban_swarm.py`，模块 docstring 第一句就把叙事拆了：`This module intentionally does not introduce a second scheduler.` 它只做一件事：按固定形状写四类卡。

```text
planning root (completed immediately)
    ├─ parallel specialist workers (ready)
    └─ verifier (todo until all workers done)
         └─ synthesizer (todo until verifier done)
```

root 卡创建后立刻被 `complete_task()` 标成 done，summary 固定为「Swarm topology planned; root remains the shared blackboard.」，`metadata` 里写 `kind: kanban_swarm_v1`。worker 卡 `parents=[root]`，因为父卡已 done，它们立刻可派发；verifier 的父卡是全部 worker，synthesizer 的父卡是 verifier。整张图建完，这个模块的工作就结束了，之后完全交给原有的 dispatcher。

root 立刻 done，不只是为了解除依赖。它同时是审计锚点：事后回看一次 swarm，先找到 root 就能读到这次的 goal、worker 数量和图结构。它还是一张不需要被 claim 就能让所有 worker 往上追加内容的卡——黑板挂在上面，谁想说一句就追加一条注释，不必等锁。反过来想，如果把协调者做成一张 running 的卡，整窝 worker 会卡在第一张卡上——协调者自己也被卷进了任务状态机。

黑板同样不是新东西：它是 root 卡上的一条结构化 JSON 注释，前缀常量 `BLACKBOARD_PREFIX = "[swarm:blackboard] "`。源码注释解释了为什么做得这么土——状态全部留在现有的 `task_comments` / `task_events` 行里，dashboard、notifier、slash command、dispatcher 都不用改就能继续工作。这是明确的取舍：放弃独立的共享存储，换来零新增服务、零新增同步路径。多一个服务就多一处会挂的东西，而黑板真正需要的只是「同 key 后者覆盖前者」加 `_authors` 溯源。

```bash
hermes kanban init
hermes gateway start

hermes kanban swarm "设计一套多区域故障转移方案" \
  --worker researcher:调研现有故障转移方案 \
  --worker architect:产出架构草案 \
  --worker sre:评估运维成本 \
  --verifier reviewer \
  --synthesizer writer \
  --idempotency-key "failover-plan-2026-11"
```

`--worker` 可重复，`metavar` 是 `PROFILE:TITLE[:SKILL,SKILL]`，`--verifier` 与 `--synthesizer` 必填。官方 v0.16 文档这一段写的是 `--workers researcher,architect,sre`，而 v0.16 与 v0.23.1 的解析器里都没有 `--workers`——照抄文档会直接报 argparse 错误。第一个反直觉点：文档比代码旧，以解析器为准。

`--idempotency-key` 命中已存在的 root 时，`create_swarm()` 会先读 root 的 `topology` 黑板值，若里面已有 `worker_ids` / `verifier_id` / `synthesizer_id` 就直接返回既有拓扑，不重复建图。流水线重跑时，这一条比什么都重要。

## 二、`assignee` 就是一个 profile 名

任务定义里只有一个 `assignee`，值就是 profile 名。dispatcher 每 `dispatch_interval_seconds`（默认 60 秒）跑一轮，做五件事：回收过期 claim、回收已死 PID 的 worker、promote 满足依赖的任务、原子 claim、spawn 被指派的 profile。claim 锁的 TTL 是 `DEFAULT_CLAIM_TTL_SECONDS = 15 * 60`，lock 形如 `<host>:<pid>:<uuid>`。

为什么多 profile 共用一个 board，而不是各开一个？源码原话是「Profiles intentionally collapse onto a shared board: it IS the cross-profile coordination primitive.」派发时 dispatcher 会把 `HERMES_KANBAN_DB`、`HERMES_KANBAN_BOARD`、`HERMES_KANBAN_WORKSPACES_ROOT` 注入 worker 的环境，保证 worker 落在派发它的那个 DB 上。省掉这一步，跨 profile 协作就得再发明一套消息传递。

反直觉点二：`assignee` 写错不会报错。dispatcher 不会 spawn 不存在的 profile，也不计失败，卡就停在 `ready`，事件流里只留一条 `skipped_nonspawnable {assignee}`。`failure_limit`（默认 2）的熔断管的是「profile 存在但起不来」，管不到「名字拼错」，这是两条不同的路径。人看到的现象是卡安静地躺着，`hermes kanban stats` 里也只有一张 ready。所以建卡前先跑 `hermes kanban assignees`，对齐磁盘上真实存在的 profile。

## 三、门控靠依赖，不靠内容

依赖是 `task_links` 里的一行 parent→child，推进规则只有一条：所有 parent 都 `done` 时，`todo` 变 `ready`，并写一条 `promoted` 事件。没有附加条件。

于是 verifier body 里那段契约只有后半句是机制，前半句是约定：「complete only with metadata `{"gate": "pass"}` when evidence is sufficient; otherwise block with exact missing work.」`metadata.gate` 写成什么，dispatcher 都不看；verifier 一旦 `complete` 就是 done，synthesizer 立刻被 promote。想真正拦住，只能 `block`，或者让 synthesizer 的 body 自己去校验黑板上的 gate 值，把内容判断挪到 worker 侧。

这个取舍要说清楚：状态机不理解内容，也不该理解。让 promote 依赖「上一个 agent 说它通过了」，等于把整条流水线的正确性押在一句自然语言上。要么用 `block` 把人的注意力拉进来，要么承认门控是软的。

实践上更可靠的办法是把验收条件写成可执行的东西：verifier 通过时把结论写进黑板，synthesizer 的 body 第一句就是读黑板、比对 gate 值，不合规就自己 block。门控仍然不是机制，但至少不再依赖一句没有约束力的自然语言。

状态全集是 `triage`、`todo`、`scheduled`、`ready`、`running`、`blocked`、`review`、`done`、`archived`。`scheduled` 确实在全集里，不是文档笔误；`VALID_INITIAL_STATUSES` 只有 `{"running", "blocked"}`。

## 四、workspace 三形态：只有非 scratch 保得住产物

| 形态 | 真实路径 | 任务完成后 |
|---|---|---|
| `scratch`（默认） | `~/.hermes/kanban/workspaces/<id>/` | 删除 |
| `dir:<绝对路径>` | 你指定的目录 | 保留 |
| `worktree` | `.worktrees/<id>/` | 保留 |

`dir:` 必须是绝对路径，相对路径在 dispatch 阶段被拒。理由不是洁癖：相对路径取决于 dispatcher 当时的 CWD，官方把它定义为 confused-deputy 逃逸面——一张卡不该有能力写到别人目录里去。

worktree 形态解决的是另一种问题：多个 worker 要改同一个仓库，各自一个独立工作树加分支，互不覆盖，合并留给下游。它比 `dir:` 多一层隔离，代价是它只对 git 仓库有意义。

反直觉点三：`hermes kanban swarm` 没有 `--workspace` 参数（v0.16 与 v0.23.1 的 swarm 参数集一致），而 `create_swarm()` 的 `workspace_kind` 默认是 `scratch`。于是一键建出来的整窝卡跑完就删，产物和任务一起消失。v0.16 也没有 `kanban_attach`；`kanban_complete(artifacts=[...])` 确实存在，会把路径写进 `metadata.artifacts` 供 gateway notifier 上传，但「清理前把声明的产物复制到持久附件存储」这个保证要到 v0.23 文档才写明。

要留文件，就手搭同形状的图，给每张卡配上 `dir:<绝对路径>`（尖括号占位符换成对应命令返回的 id）：

```bash
hermes kanban create "Swarm: 多区域故障转移" --assignee orchestrator \
  --workspace dir:E:/work/failover
# root 建完立刻标 done（swarm 里由 create_swarm() 内部 complete_task() 做），
# 否则三张 worker 一直门控在它下面
hermes kanban complete <root_id>

hermes kanban create "调研现有方案" --assignee researcher --parent <root_id> \
  --workspace dir:E:/work/failover
hermes kanban create "架构草案" --assignee architect --parent <root_id> \
  --workspace dir:E:/work/failover
hermes kanban create "运维成本" --assignee sre --parent <root_id> \
  --workspace dir:E:/work/failover

# verifier 与 synthesizer 也要建；多父卡这里用 link 逐条挂（--parent 本身可重复，重复写也行），
# 参数顺序是「父卡在前、子卡在后」
hermes kanban create "验收" --assignee reviewer --workspace dir:E:/work/failover
hermes kanban create "成稿" --assignee writer --workspace dir:E:/work/failover

hermes kanban link <research_id> <verify_id>
hermes kanban link <architect_id> <verify_id>
hermes kanban link <sre_id> <verify_id>
hermes kanban link <verify_id> <synth_id>
```

运维参数按真实键名配一遍：

```yaml
kanban:
  dispatch_in_gateway: true
  dispatch_interval_seconds: 60
  failure_limit: 2
  dispatch_stale_timeout_seconds: 14400
  max_in_progress: 3
```

scratch 默认删除不是疏忽：一次性任务的目录不该长期堆着。

## 五、三个会咬人的坑

stale 规则值得单独记：任务跑超过 `dispatch_stale_timeout_seconds`（默认 4 小时）、且最近 1 小时没有 heartbeat 时，dispatcher 会给本地 worker 发 SIGTERM、把卡放回 `ready`，并且不计失败。所以同一张卡会安静地反复开始，`stats` 里没有任何失败痕迹。长流程就每几十分钟调一次 `kanban_heartbeat(note=...)`——它是纯副作用，不改任务状态。

不要同时跑 `hermes kanban daemon` 和 gateway 内嵌的 dispatcher：两者会抢 claim，官方明确不支持，`daemon` 已标 DEPRECATED。

`review` 状态在 v0.16 里存在，dispatcher 也有 `claim_review_task()`，但 worker 够不着。v0.16 的工具面只有 9 个：`kanban_show`、`list`、`complete`、`block`、`heartbeat`、`comment`、`create`、`unblock`、`link`；CLI 里也没有 `request-review` / `request-changes`。看到文档写 review 就以为能送审，会白等。这套车道在我本机的 v0.23.1 上已经完整（`kanban_request_review`、`kanban_request_changes`、`kanban_attach` 等），而且 review 不参与 block 循环计数；它具体在哪个版本补齐，我没核实到。

采用建议四条：

1. 建卡前用 `hermes kanban assignees` 对齐 profile，用 `hermes kanban dispatch --dry-run` 先看这一轮会派给谁；
2. 要留产物就别用 swarm 的默认值，手搭图加 `dir:<绝对路径>`，或在 worker 里显式声明 `artifacts`；
3. verifier 用 `block` 表达不通过，别只写在 summary 里；
4. 只跑一套 dispatcher，长任务勤 heartbeat。

反过来也要知道什么时候不该用 swarm：如果几件事之间没有真实依赖，手写几张卡加 `link` 更省事；swarm 会强制塞给你 verifier 和 synthesizer，你可能只要前者。

swarm 的价值不在于替你编排，而在于把这几类卡按依赖连好、把黑板塞进 root 卡，然后退场。真正干活的是那个 60 秒一轮、只认「父卡是否 done」的循环。接受这一点，调它的时候会快很多。

## 来源

1. v0.16.0 发布说明（tag `v2026.6.5`，The Surface Release）：https://github.com/NousResearch/hermes-agent/releases/tag/v2026.6.5
2. v0.15.0 发布说明（tag `v2026.5.28`，Velocity Release，PR #28443）：https://github.com/NousResearch/hermes-agent/releases/tag/v2026.5.28
3. `hermes_cli/kanban_swarm.py` @ `v2026.6.5`（拓扑、黑板前缀、verifier 契约、幂等）：https://raw.githubusercontent.com/NousResearch/hermes-agent/v2026.6.5/hermes_cli/kanban_swarm.py
4. `hermes_cli/kanban.py` @ `v2026.6.5`（swarm 子解析器、CLI 动词表）：https://raw.githubusercontent.com/NousResearch/hermes-agent/v2026.6.5/hermes_cli/kanban.py
5. `hermes_cli/kanban_db.py` @ `v2026.6.5`（四表 schema、`VALID_STATUSES`、`DEFAULT_CLAIM_TTL_SECONDS`、`skipped_nonspawnable`）：https://raw.githubusercontent.com/NousResearch/hermes-agent/v2026.6.5/hermes_cli/kanban_db.py
6. `tools/kanban_tools.py` @ `v2026.6.5`（9 个 worker 工具、`artifacts` 参数）：https://raw.githubusercontent.com/NousResearch/hermes-agent/v2026.6.5/tools/kanban_tools.py
7. v0.16 用户文档 `kanban.md`（workspace、dispatcher、事件表、swarm 示例误用 `--workers`）：https://raw.githubusercontent.com/NousResearch/hermes-agent/v2026.6.5/website/docs/user-guide/features/kanban.md
8. 未在 v0.16 实机运行：文中 v0.16 结论均出自 tag `v2026.6.5` 的源码与文档；本机安装版本为 v0.23.1，对比项（review 车道工具、附件工具）以本机文档为准。
