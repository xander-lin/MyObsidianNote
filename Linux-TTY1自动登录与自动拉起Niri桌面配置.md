# Linux TTY1 自动登录与自动拉起 Niri 桌面配置

在没有安装 Display Manager（如 GDM、SDDM、LightDM 等）的轻量化 Linux 环境中，可以通过 `systemd agetty` 实现 TTY1 免密自动登录，并结合 Shell 配置在登录后自动拉起 Wayland Compositor（如 Niri）。

---

## 一、TTY1 自动登录配置

利用 systemd 的 drop-in 机制覆盖默认的 `getty@tty1` 服务配置。

- **配置文件**：`/etc/systemd/system/getty@tty1.service.d/override.conf`
- **配置内容**：

```ini
[Service]
ExecStart=
ExecStart=-/sbin/agetty --autologin life --noclear %I $TERM
```

- **作用**：系统启动时，getty 服务直接以指定用户（如 `life`）免密自动登录到 tty1，无需手动输入用户名与密码。

---

## 二、Zsh 配置目录重定向（可选）

为了遵循 XDG 规范保持用户家目录整洁，可将 Zsh 配置集中存放。

- **配置文件**：`~/.zshenv`
- **配置内容**：

```bash
export ZDOTDIR="$HOME/.config/zsh"
```

- **作用**：将 Zsh 的所有后续配置文件读取路径重定向至 `~/.config/zsh/`。

---

## 三、TTY1 自动拉起 niri-session

在登录 Shell 初始化脚本中判断当前终端环境，符合条件时启动桌面。

- **配置文件**：`~/.config/zsh/.zprofile`（若未重定向 ZDOTDIR 则为 `~/.zprofile`）
- **配置内容**：

```bash
# Force disable client-side decorations for GTK & Qt apps
export GTK_CSD=0
export QT_WAYLAND_DISABLE_WINDOW_DECORATION=1

# Start Niri automatically on the first TTY login.
if [[ "$(tty)" == "/dev/tty1" ]] && [[ -z "$ZSH_EXECUTION_STRING" ]] && ! pgrep -x niri > /dev/null; then
    niri-session
fi
```

- **判断逻辑说明**：
  1. `[[ "$(tty)" == "/dev/tty1" ]]`：判断当前终端是否为主 TTY（`/dev/tty1`），避免在其他虚拟终端（tty2~tty6）登录或 SSH 登录时触发。
  2. `[[ -z "$ZSH_EXECUTION_STRING" ]]`：确保不是在执行单条非交互式命令（如 `zsh -c "..."`）。
  3. `! pgrep -x niri > /dev/null`：检查当前系统中尚无正在运行的 `niri` 进程，防止重复启动。
  4. 满足以上所有条件时，自动执行 `niri-session` 启动 Wayland 桌面会话。

---

## 四、总结整体流程

```text
系统开机 
  ↓
systemd agetty 自动免密登录用户到 tty1 
  ↓
启动登录 Shell (Zsh) 
  ↓
读取 ~/.config/zsh/.zprofile 
  ↓
环境判断（位于 tty1 且无 niri 进程运行）
  ↓
自动拉起 niri-session 进入 Wayland 桌面
```
