

# termux-mcp

一个运行在 Termux 里的 `Streamable HTTP` MCP 服务。安装完成后，直接在 Termux 里执行：

```sh
termux-mcp
termux-mcp-shutdown
```

## 一行安装

```sh
curl -fsSL https://raw.githubusercontent.com/yuxinjiang218-creator/termux-mcp/main/install.sh | sh
```

如果你的环境没有 `curl`，也可以用：

```sh
wget -qO- https://raw.githubusercontent.com/yuxinjiang218-creator/termux-mcp/main/install.sh | sh
```

安装脚本会自动完成这些事情：

- 安装缺失依赖：`git`、`nodejs`、`npm`、`patch`、`ripgrep`
- 克隆或更新仓库到 `~/termux-mcp`
- 执行 `npm install --omit=dev`
- 安装命令入口到 `~/bin`
- 自动把 `~/bin` 加进 `PATH`

安装后重新打开一次 Termux，或者执行：

```sh
source ~/.bashrc
```

## 使用方法

启动服务：

```sh
termux-mcp
```

停止服务：

```sh
termux-mcp-shutdown
```

默认地址：

```text
http://0.0.0.0:8765/mcp
```

## 已提供的 MCP Tools

- `exec_command`
- 执行短时、一次性 shell 命令
- `start_session`
- 启动一个可持续交互的终端会话，用于前台交互式任务
- `list_sessions`
- 查看当前活跃会话
- `write_session`
- 往一个运行中的 shell session 写入输入；需要原始 stdin 时可用 `raw=true`
- `read_session`
- 分块读取会话输出
- `kill_session`
- 结束会话
- `start_background_process`
- 启动可持续运行的后台服务或长任务；这是起服务的首选工具
- `list_background_processes`
- 列出由 termux-mcp 启动并追踪的后台进程
- `read_process_output`
- 分块读取后台进程 stdout/stderr 日志
- `stop_background_process`
- 温和停止后台进程，必要时自动强杀
- `read_file`
- 读取文本文件
- `list_files`
- 列出目录内容
- `search_text`
- 递归搜索文本
- `apply_patch`
- 以 patch 方式修改文件
- `view_image`
- 读取本地图片，并返回真正的图片内容给多模态客户端

## 工具选型规则

- 要跑一个会很快结束的普通命令，用 `exec_command`
- 要在同一个 shell 里反复执行前台命令、保留 cwd/环境/交互状态，用 `start_session` + `write_session` + `read_session`
- 要启动 HTTP 服务、dev server、watcher、bot、长循环任务，直接用 `start_background_process`
- 要看后台服务日志，用 `read_process_output`
- 要停后台服务，用 `stop_background_process`

## 接入手机客户端后能做什么

只要你的手机 AI 客户端支持 MCP，并且能连接这个服务地址，AI 就可以直接调用你在 Termux 里的终端和文件能力。

常见用途包括：

- 在 Termux 里直接执行命令
  - 例如 `ls`、`pwd`、`git status`、`npm install`、`node xxx.js`
- 读取和修改项目文件
  - 查看源码、搜索关键字、改配置、打补丁
- 进行多轮终端交互
  - 启动一个持续 session，连续读取输出并继续输入
- 让手机里的 AI 变成真正可执行的终端助手
  - 查日志
  - 修脚本
  - 跑项目命令
  - 管理仓库
  - 处理日常开发任务

一句话概括：

> 把支持 MCP 的手机 AI 客户端，直接变成你在 Termux 上的可执行终端助手。

## 可用环境变量

- `TERMUX_MCP_HOST`
- `TERMUX_MCP_PORT`
- `TERMUX_MCP_RUNTIME_DIR`
- `TERMUX_MCP_ALLOWED_ROOTS`
- `TERMUX_MCP_MAX_OUTPUT`
- `TERMUX_MCP_MAX_SESSION_READ_BYTES`
- `TERMUX_MCP_MAX_PROCESS_READ_BYTES`
- `TERMUX_MCP_MAX_IMAGE_BYTES`
- `TERMUX_MCP_SESSION_IDLE_MS`
- `TERMUX_MCP_RECENT_SESSION_TTL_MS`
- `TERMUX_MCP_BACKGROUND_STOP_GRACE_MS`
- `TERMUX_MCP_HOME`

示例：

```sh
TERMUX_MCP_PORT=9000 termux-mcp
```

### `read_session` 现在会做什么

为了避免终端会话把超长输出一次性灌给模型，`read_session` 改成了默认按块读取：

- 默认单次最多返回约 `12KB`
- 会告诉客户端还有多少 `remaining_bytes`
- 如果只是想偷看但不消费缓冲区，可以传 `peek=true`
- 如果输出在你读取前就已经太大，服务会丢弃更早的旧内容，并在结果里标出 `dropped_bytes`

### `write_session` 怎么用

- 默认模式下，`write_session` 会在输入末尾补一个换行，让 shell 命令正常执行
- 如果你是在和交互程序对话，需要原始 stdin，请传 `raw=true`
- 如果这个命令本来就应该持续跑着，比如服务、watch 模式、轮询脚本，不要用 `write_session`，直接改用 `start_background_process`

### 后台服务怎么跑

推荐把长时间运行的服务交给后台进程工具，而不是让 `exec_command` 或前台 session 一直挂着：

- `start_background_process` 启动服务并返回 `process_id`
- `list_background_processes` 查看它是否仍在运行
- `read_process_output` 读取日志
- `stop_background_process` 停掉服务

### `view_image` 返回的是什么

`view_image` 不再只返回文件元信息，而是会返回：

- 一段简短文字说明
- 一条原生 MCP `image` content

这样支持多模态的客户端就能把图片真正送进模型上下文，而不只是告诉模型“这里有一张图”。

## 卸载

仓库里自带卸载脚本：

```sh
~/termux-mcp/uninstall.sh
```
