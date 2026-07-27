# Agent View 架构设计

本文描述 Agent View 的架构设计。Agent View 是一个终端内的多 session 管理系统，允许用户从统一界面派发、监控和交互多个独立 Qwen Code 会话，支持后台运行、休眠恢复和独立 cwd 隔离。

## 1. 目标

- 从一个 Roster 界面派发多个独立 Qwen Code session。
- 每个 session 有独立的 PTY 和完整的原生 TUI 体验。
- 支持 attach/detach：断开后会话继续运行，随时重新接入。
- 支持后台运行：关闭终端后会话不中断。
- 支持独立 cwd 隔离：每个 session 有独立的工作目录。worktree 隔离作为后续推进方向。
- 支持 idle 休眠和自动唤醒。
- Supervisor 重启后能恢复已有会话。

## 2. 整体架构

采用 Supervisor-Worker 三层架构。

| 层 | 职责 | 特点 |
| --- | --- | --- |
| Supervisor | 管理所有 session 生命周期，调度 dispatch、attach、stop | 用户级守护进程，crash 后可重启 |
| PTY Host | 持有 PTY master fd，桥接终端字节流 | 轻量进程，Supervisor 重启时 Worker 不中断 |
| Worker | 完整的原生 Qwen Code 交互式进程 | 通过 PTY slave 和 sideband channel 双通道通信 |

三层职责分离后，Supervisor 可以专注于生命周期和调度，PTY Host 负责保持终端会话连续性，Worker 继续运行原生 Qwen Code TUI。

## 3. Supervisor

### 3.1 启动与单例保证

Supervisor 通过 double-check 锁定确保全局单例。

第一次检查避免不必要的锁竞争；第二次检查处理并发启动的竞争窗口。Supervisor 启动后定期执行 idle 休眠检查和资源清理。

### 3.2 会话生命周期

Supervisor 为每个 session 提供以下操作：

- dispatch：创建新的后台 session。
- adopt：接管当前前台 session，使其进入 Agent View 管理。
- attach：连接到已有 session 的终端流。
- detach：断开当前 attach，Worker 继续运行。
- stop：请求 session 停止当前工作。
- kill：强制终止 session 进程。
- hibernate：在 idle 后释放 Worker 进程。
- respawn：从持久化状态恢复 Worker。
- remove：删除 session 记录和状态文件。

### 3.3 数据模型

每个 session 在文件系统上有多个 JSON 状态文件，支持跨进程读取和 Supervisor 重启恢复：

| 文件 | 说明 |
| --- | --- |
| `state.json` | session 业务状态、进程状态、attach 状态、cwd 信息 |
| `launch.json` | 启动参数、环境、cwd、模型和终端尺寸 |
| `activity.json` | summary、waitingFor、lastResult、capabilities 等展示信息 |
| `worker.json` | Worker/PTY Host 进程信息、endpoint、heartbeat |
| `roster.json` | Roster 展示所需的排序、pin、rename 元数据 |

会话状态机描述业务逻辑：

```text
starting -> working -> needs_input -> working
working  -> idle -> completed
working  -> stopped
working  -> failed
```

进程状态机独立描述物理进程：

```text
starting -> alive -> hibernating -> hibernated
alive    -> exited
exited   -> restarting -> alive
```

两个状态机独立演化：一个 hibernated 的 session 可以是 working，表示业务上仍有可恢复的上下文，只是物理 Worker 进程被暂停了。

### 3.4 状态聚合与排序

Roster UI 需要将多个数据源聚合为可展示的行：

```text
state.json + launch.json + activity.json + worker.json + roster.json
  -> AgentViewSessionSnapshot
  -> presentation row
```

排序优先级：

1. pinned。
2. needs_input。
3. working。
4. ready/completed/stopped/failed。
5. 最近更新时间。

`needs_input` 排在前面，确保需要用户关注的 session 始终可见。

## 4. PTY Host 与终端桥接

### 4.1 为什么分离 PTY Host

将 PTY master fd 持有在独立进程中，而非 Supervisor 自身。

这样 Supervisor 重启后可以重新连接 PTY Host，客户端 reattach 时仍然能获得完整输出。如果 PTY master 在 Supervisor 中，Supervisor crash 会直接杀死 Worker。

### 4.2 Attach/Detach 流程

Attach 采用 lease 机制防止并发冲突：

```text
client attach
  -> supervisor acquire lease
  -> supervisor bridge socket to PTY Host
  -> terminal bytes stream both ways
  -> detach/close releases lease
```

Lease 的 TTL 保证客户端异常退出时不会永远占着 session。

## 5. Worker Sideband

Worker（原生 Qwen TUI）除了 PTY 之外，还通过独立的 sideband channel 与 Supervisor 通信。

Sideband 的价值在于提供结构化状态（summary、waitingFor 等），而不是让 Supervisor 去解析终端 ANSI 输出。Worker 内部的 TUI 框架集成了 sideband 逻辑：

- 检测是否在 Agent View 环境下运行。
- 根据当前 streaming state 和 pending tool calls 计算并上报 sessionState。
- 监听控制事件，处理来自 Roster 的远程 prompt 注入和工具确认回答。

## 6. Roster UI

### 6.1 架构

Roster UI 是一个独立的 Ink 应用，通过 Supervisor IPC 获取数据。

App 层管理所有 UI 状态，包括选中、prompt、peek 面板和分组模式。Roster 组件只负责渲染和转发键盘事件。

### 6.2 交互模型

| 快捷键 | 操作 |
| --- | --- |
| ↑/↓ | 移动选中 |
| Enter | attach 到选中 session |
| Space | 打开 peek 面板 |
| Shift+Enter | dispatch 新 session 并 attach |
| Ctrl+T | pin/unpin |
| Ctrl+R | rename，使用 prompt 文本 |
| Ctrl+X | stop；2 秒内再按 remove |
| Ctrl+S | 切换分组（state / directory） |
| Ctrl+C | 清空；再按退出 |
| ? | 显示帮助 |

### 6.3 Peek 面板

Peek 允许在不 attach 的情况下查看 session 状态和远程交互。

Peek 的 answer 能力是 Agent View 的核心价值之一：用户不需要 attach 就能解决 Agent 的阻塞问题。

### 6.4 Dispatch 流程

```text
user submits prompt
  -> roster calls supervisor dispatch
  -> supervisor creates session state
  -> supervisor launches PTY Host and Worker
  -> worker reports ready through sideband
  -> roster refreshes row
```

### 6.5 二次确认（Stop/Remove）

`Ctrl+X` 第一次执行 stop，并显示“再按一次 remove”的短暂提示。提示窗口内再次按下 `Ctrl+X` 才会 remove。提示过期后再次按下 `Ctrl+X` 重新视为第一次 stop。

## 7. Cwd 隔离（Worktree 后续推进）

当前实现中，每个 session 使用独立的 cwd（activeCwd），但不创建 git worktree。所有 session 共享同一个项目目录。

状态模型中保留了 worktree 字段（`mode: 'none' | 'worktree' | 'shared-unisolated'`），用于向前兼容。

后续推进方向：为每个 session 创建独立的 git worktree（在 `.qwen/worktrees/` 下），实现真正的多 session 写文件隔离。涉及：

- git worktree 创建与分支策略。
- worktree 生命周期管理，session remove 时清理。
- 安全性检查，确保只删除 Agent View 拥有的 worktree。
- 与 `--worktree` CLI flag 的集成。

## 8. 会话恢复

Worker 退出后 session 状态保留在文件系统中，支持后续恢复：

```text
supervisor starts
  -> reads persisted sessions
  -> probes worker/pty host state
  -> marks recoverable sessions
  -> respawn on attach/reply/request
```

恢复能力依赖文件系统状态、PTY Host endpoint 和 sideband 状态纠偏。

## 9. CLI 命令体系

| 命令 | 说明 |
| --- | --- |
| `qwen agents` | 启动 Roster 交互界面 |
| `qwen agents --json` | JSON 格式列出所有 session |
| `qwen daemon status` | 查看 Supervisor 状态 |
| `qwen daemon stop --any` | 停止 Supervisor |
| `qwen session attach <id>` | 接入指定 session |
| `qwen session logs <id>` | 查看 session 日志 |
| `qwen session stop <id>` | 停止 session |
| `qwen session kill <id>` | 强制终止 session |
| `qwen session respawn [id]` | 重启 hibernated session |
| `qwen session remove <id>` | 删除 session 及状态文件 |

Worker 内部支持 `/background`（`/bg`）命令，触发当前 session detach。

## 10. 关键设计决策

**PTY Host 分离**：将 PTY master fd 持有在独立进程中。Supervisor 可以安全重启（crash recovery、版本升级）而不中断正在运行的 Worker。这是整个架构最重要的决策，它使得“关闭终端后 session 继续运行”成为可能。

**文件系统状态持久化**：会话状态存为 JSON 文件而非内存或数据库。好处是：(1) 任何客户端进程都能直接读取，不需要先连接 Supervisor；(2) Supervisor 重启后自动恢复状态；(3) `qwen agents --json` 甚至在 Supervisor 未运行时也能工作。

**Sideband Channel**：Worker 通过独立于 PTY 的通道上报结构化状态。这使 Supervisor 能准确知道 Agent 在做什么（summary、waitingFor），而不需要解析终端 ANSI 输出流。

**Lease-based Attach**：防止两个客户端同时 attach 到同一 session 导致的终端输出混乱。Lease 有 TTL，客户端异常退出时自动释放，避免死锁。

**Double-tap Stop/Remove**：`Ctrl+X` 第一次 stop，2 秒内再按才 remove。避免误删的同时不需要弹出确认对话框，保持终端 UI 简洁。

**状态机分离**：会话状态（业务逻辑：working/idle/needs_input）和进程状态（物理层面：alive/hibernated/exited）独立演化。一个 hibernated 的 session 业务上仍然可以是 working，respawn 后立即继续。

## 11. 演示视频

待补充。
