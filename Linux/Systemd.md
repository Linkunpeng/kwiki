`systemd`是现代Linux发行版中广泛使用的系统和服务管理器，它旨在提供一个更快速、更灵活、更强大的系统初始化和管理框架，取代了传统的System V init系统。

---

## systemd 的工作原理

`systemd`以进程ID 1 (PID 1) 运行，是系统中第一个启动的用户空间进程，也是最后一个终止的进程。它负责启动和管理系统中的所有其他进程和服务。其核心工作原理和特性包括：

* **单元（Units）**：`systemd`将所有系统资源（如服务、挂载点、设备、套接字等）抽象为“单元”。每个单元都有一个对应的配置文件，通常以 `.unit` 后缀命名（例如，`.service`、`.mount`、`.socket`、`.target`等）。
* **并行化启动**：与传统的串行启动方式不同，`systemd`能够并行启动服务，显著缩短了系统启动时间。它通过分析单元之间的依赖关系，尽可能同时启动没有相互依赖的服务。
* **基于依赖关系的服务控制**：`systemd`通过清晰的依赖关系（例如，A服务需要B服务先启动）来管理服务的启动和停止顺序。这种依赖关系是事务性的，这意味着如果某个依赖的服务启动失败，相关的服务也会受到影响或停止。
* **按需启动（Socket/D-Bus Activation）**：`systemd`支持套接字和D-Bus激活。这意味着服务可以在首次被访问时才启动，而不是在系统启动时就立即启动，从而节省了系统资源并加快了启动速度。
* **cgroups 进程跟踪**：`systemd`利用Linux内核的cgroups（控制组）功能来精确跟踪和管理服务进程，即使进程fork出子进程或脱离父进程，`systemd`也能保持对其的控制。
* **Target（目标）**：Target 单元类似于System V init中的运行级别（runlevel），它们是一组相关的单元的集合，定义了系统启动到某种状态所需的服务。例如，`multi-user.target` 对应多用户命令行模式，`graphical.target` 对应图形界面模式。
* **统一日志管理（journald）**：`systemd`集成了`journald`日志系统，将所有系统和服务的日志集中管理，并以结构化二进制格式存储，方便查询和分析。
* **其他组件**：`systemd`不仅仅是一个init系统，它还包含许多其他重要的组件，如`logind`（用户会话管理）、`networkd`（网络配置）、`timesyncd`（时间同步）、`resolved`（DNS解析）等。

---

## 关联的目录与文件

`systemd`的配置文件（单元文件）通常分散在几个标准目录中，它们之间存在优先级关系，允许管理员覆盖或扩展系统默认配置：

* **/usr/lib/systemd/system/**：这个目录存放由软件包安装的默认`systemd`单元文件。通常不建议直接修改这里的文件，因为软件包更新可能会覆盖你的修改。
* **/etc/systemd/system/**：这个目录用于存放系统管理员自定义或修改的单元文件。通过`systemctl enable`命令启用服务时，通常会在这个目录下创建指向`/usr/lib/systemd/system/`中文件的符号链接。在这里创建或修改的文件会覆盖`/usr/lib/systemd/system/`中的同名文件。
* **/run/systemd/system/**：这个目录存放运行时动态生成的单元文件，优先级最高。通常由`systemd`自身或其他程序在运行时创建。
* **`/etc/systemd/user/` 或 `$HOME/.config/systemd/user/`**：这些目录用于存放**用户级别**的`systemd`单元文件。用户可以定义和管理自己的服务，这些服务只在用户登录时运行。
* **/var/log/journal/**：`journald`存储日志的默认位置。日志以二进制格式存储，需要通过`journalctl`命令查看。
* **/etc/systemd/system.conf**：`systemd`的全局配置文件，可以用来修改`systemd`守护进程的整体行为。

---

## 常用的配置与功能

### 1. `systemctl` 命令

`systemctl`是`systemd`的主要命令行工具，用于管理和控制`systemd`单元。

**常用操作：**

* **启动服务**：`sudo systemctl start <service_name>.service`
* **停止服务**：`sudo systemctl stop <service_name>.service`
* **重启服务**：`sudo systemctl restart <service_name>.service`
* **重新加载服务配置**：`sudo systemctl reload <service_name>.service`
* **查看服务状态**：`systemctl status <service_name>.service`
* **启用服务（开机自启动）**：`sudo systemctl enable <service_name>.service`
* **禁用服务（取消开机自启动）**：`sudo systemctl disable <service_name>.service`
* **查看所有已加载的单元**：`systemctl list-units`
* **查看所有服务单元**：`systemctl list-units --type=service`
* **查看单元文件内容**：`systemctl cat <unit_name>`
* **重新加载所有`systemd`配置**：`sudo systemctl daemon-reload` (当你手动修改了单元文件后，需要执行此命令使更改生效)

### 2. 单元文件 (`.service`) 的常用配置项

服务单元文件通常包含多个节（Section），最常见的有`[Unit]`、`[Service]`和`[Install]`。

**`[Unit]` 节**：定义单元的通用信息和依赖关系。

* **`Description`**：对单元的简短描述。
* **`Documentation`**：指向相关文档的URL。
* **`After=<unit>`**：指定此单元应在 `<unit>` 之后启动。
* **`Before=<unit>`**：指定此单元应在 `<unit>` 之前启动。
* **`Requires=<unit>`**：强依赖关系，如果 `<unit>` 未启动或失败，此单元也无法启动。
* **`Wants=<unit>`**：弱依赖关系，建议启动 `<unit>`，但即使 `<unit>` 启动失败也不会影响此单元。
* **`Conflicts=<unit>`**：冲突关系，如果此单元启动，则 `<unit>` 必须停止。

**`[Service]` 节**：定义服务的具体行为。

* **`Type=`**：定义服务进程的启动类型，常见值：
    * `simple` (默认)：`ExecStart` 指定的命令是主进程。
    * `forking`：`ExecStart` 命令会 fork 出子进程作为主进程，父进程退出。通常用于传统守护进程。
    * `oneshot`：一次性进程，`systemd` 会等待命令执行完毕。常用于执行一次性任务或初始化脚本。
    * `notify`：服务启动完成后会通过 `sd_notify()` 通知 `systemd`。
    * `dbus`：服务通过D-Bus激活。
* **`ExecStart=`**：指定启动服务时执行的命令或脚本。
* **`ExecStartPre=`** / **`ExecStartPost=`**：在 `ExecStart` 之前/之后执行的命令。
* **`ExecReload=`**：指定重新加载服务时执行的命令。
* **`ExecStop=`**：指定停止服务时执行的命令。
* **`ExecStopPost=`**：在 `ExecStop` 之后执行的命令。
* **`Restart=`**：定义服务在何种情况下自动重启，常见值：
    * `no` (默认)：不重启。
    * `on-success`：只有成功退出时才重启。
    * `on-failure`：只有失败退出时才重启。
    * `always`：总是重启。
* **`RestartSec=`**：服务自动重启前的等待秒数。
* **`TimeoutSec=`**：`systemd`停止服务时等待的秒数。
* **`Environment=`**：为服务设置环境变量。
* **`WorkingDirectory=`**：设置服务的工作目录。
* **`User=`** / **`Group=`**：指定服务运行的用户和组。
* **安全特性 (例如：`ProtectHome=true`, `PrivateTmp=true`, `ReadOnlyPaths=`)**：`systemd`提供了丰富的安全特性，可以限制服务对文件系统、网络、进程的访问，从而增强系统的安全性。

**`[Install]` 节**：定义当单元被启用或禁用时的行为。

* **`WantedBy=<target>`**：当此单元被启用时，会在 `<target>.wants/` 目录下创建指向此单元的符号链接。例如，`WantedBy=multi-user.target`表示在多用户模式下启动。
* **`RequiredBy=<target>`**：类似于`WantedBy`，但表示强依赖。
* **`Alias=`**：为单元提供额外的别名。

### 3. `journalctl` 命令

`journalctl`用于查询和显示`systemd journal`日志。

**常用操作：**

* **查看所有日志**：`journalctl`
* **查看特定服务的日志**：`journalctl -u <service_name>.service`
* **实时跟踪日志**：`journalctl -f`
* **查看特定时间范围的日志**：`journalctl --since "2024-01-01" --until "2024-01-02 03:00:00"`
* **查看内核日志**：`journalctl -k`
* **查看指定数量的最新日志**：`journalctl -n 20`
* **以不同格式输出日志**：`journalctl -o json`, `journalctl -o short-iso`

`systemd`的强大之处在于其统一的管理方式和丰富的配置选项，使得Linux系统的启动、服务管理和故障排查更加高效和便捷。
