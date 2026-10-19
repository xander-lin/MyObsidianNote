# Linux zsh 历史建议仅当前窗口生效（新窗口为空）排查与修复

在 x18（Arch Linux + niri + kitty + zsh）上，zsh-autosuggestions 的灰色历史建议只在"当前正在用的窗口"里出现，新开的终端窗口完全没有任何历史建议。本文记录定位为 HISTFILE 父目录缺失的过程。

---

## 一、问题现象

- 同一个终端窗口里：输入时有正常的灰色历史建议（zsh-autosuggestions 工作正常）。
- **新开的任何窗口**：建议为空，仿佛 zsh 从未有过历史。
- `history` / 上箭头在新窗口里也无旧命令。

---

## 二、根因

`~/.config/zsh/.zshrc`（本机 ZDOTDIR 指向 `~/.config/zsh/`，`~/.zshrc` 并不被读取）第 5 行：

```zsh
HISTFILE="$HOME/.local/state/zsh/history"
```

而 **父目录 `~/.local/state/zsh/` 从未创建过**。zsh **不会为 HISTFILE 自动创建父目录**，每次历史写入（`share_history` 是每条命令执行后即时追加）都静默失败，历史文件从未存在过。

新会话的历史是从 HISTFILE 加载的——文件不存在，自然是空；当前会话的建议正常，是因为 zsh-autosuggestions 读的是**会话内存历史**（本窗口刚敲过的命令），与文件无关。这正是"当前窗口有、新窗口无"的原因。

oh-my-zsh 的 `lib/history.zsh` 已经设好了 `share_history`、`HISTSIZE=50000`、`SAVEHIST=10000`，唯一缺的就是这个目录。

---

## 三、定位过程

1. `grep -rn hist ~/.config/zsh/` → 只有 HISTFILE 一行，无任何 `setopt` 相关配置；
2. `ls ~/.local/state/zsh/` → **目录不存在**（`~/.local/state/` 下有其他目录，唯独没有 zsh）;
3. 对照 oh-my-zsh 源码确认 share_history 默认开启 → 排除"没开共享"的假设，锁定目录缺失。

---

## 四、修复

```bash
mkdir -p ~/.local/state/zsh
```

并在 `~/.config/zsh/.zshrc` 的 HISTFILE 赋值后加一行**自愈**，新机器/目录被清后不再复发：

```zsh
HISTFILE="$HOME/.local/state/zsh/history"
mkdir -p "${HISTFILE:h}"   # zsh won't create the parent dir itself; without it history is never saved
```

`${HISTFILE:h}` 取 HISTFILE 的目录部分（zsh 修饰符 `:h`，等价于 dirname）。

---

## 五、验证

无需 GUI，管道喂交互 shell 即可闭环：

```bash
# 1) "窗口 1"：交互 zsh 里执行一条 marker 命令
echo 'echo marker-command-from-window-1' | zsh -i >/dev/null 2>&1

# 2) 历史文件应出现该行（extended_history 格式）
cat ~/.local/state/zsh/history
# : 1792378478:0;echo marker-command-from-window-1

# 3) "新窗口"：第二个交互 zsh 里 fc -ln 1 应能列出它
printf 'fc -ln 1\n' | zsh -i 2>/dev/null | grep -a marker
```

> [!warning] 验证方法的坑
> 不要 `grep` 会话的**可见输出**来判断历史是否加载——历史装入内存并不会打印出来。正确做法是在新会话**内部**执行 `fc -ln 1`（列出从第 1 条起的全部内存历史）再看输出。

---

## 六、方法论沉淀

1. **把历史/缓存/运行时文件指到 XDG 目录时，同一份配置里就地 `mkdir -p`**，不要假设目录已存在；zsh 对 HISTFILE 的写入失败是**静默**的，不会报任何错。
2. 诊断"新会话没历史"，第一步是 `ls -la $HISTFILE` 查文件和父目录存在性，而不是先怀疑 share_history / 插件。
3. 会话内存历史 ≠ 磁盘历史：autosuggestions 在当前窗口"看起来正常"恰恰掩盖了文件层的问题。
4. 相关：[[Linux-TTY自动登录后字符双显排查与修复]] 的假说 1 排查时就已观察到"历史文件根本不存在"，当时未深究；本文补上了它真正的根因。
