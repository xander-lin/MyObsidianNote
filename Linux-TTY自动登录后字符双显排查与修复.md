# Linux TTY 自动登录后字符双显（按键一显二）排查与修复

在 x18（Arch Linux + niri，无 Display Manager）的文本控制台（TTY）里敲命令，每输入一个字母屏幕上会显示两个相同字母，但实际只输入了一个，命令执行完全正常。本文记录从误判"自动补全坏了"到定位为一字之差的 TERM 配置的完整过程。

---

## 一、问题现象

- 仅在**自动登录（agetty `--autologin`）进入的 TTY** 上出现：敲 `a` 显示 `aa`，敲 `b` 显示 `bb`。
- **实际输入只有一个字符**，命令执行不受影响（`ls` 不会变成 `llss`）。
- **手动输入用户名密码登录的 TTY（tty2）完全正常**；图形终端 kitty 也正常。
- 最初观感像是"自动补全坏了：没按 Tab 就自动补全了"。

---

## 二、排查弯路（两个被实验否定的假说）

### 假说 1：zsh-autosuggestions 的灰色历史建议

- 推理：zsh 配置加载了 zsh-autosuggestions，会在光标后以灰色显示历史匹配，看起来像"自动补全"。
- 否定证据：
  - 自动登录会话的历史文件根本不存在（HISTFILE 指向的路径无内容），无历史可匹配仍双显；
  - 双显时抓取 termios 为 `-icanon -echo`（ZLE raw 模式正常生效，内核行规程并未回显），与建议插件的行为无关。

### 假说 2：niri 残留的 K_OFF / 终端状态污染

- 推理：出问题的 tty1 是 niri（Wayland 合成器）会话的宿主 VT，niri 运行时会把键盘切到 `K_OFF` 独占模式，怀疑退出后终端状态恢复不干净。
- 否定证据：把完全相同的 autologin 配置复制到**从未运行过图形会话的干净 tty2**，双显照样复现——与 niri 无关。

> [!note] 教训
> 两个假说都有"合理性"，但都只是推理、没有复现实验支撑，被表象（只有 tty1 坏）带偏了方向。

---

## 三、关键线索

1. 使用者观察：**只有自动登录链路的 VT 有问题**，手动登录的 tty2 一直正常——嫌疑集中到自动登录本身。
2. 存在一台配置"几乎相同"的对照机器 **X19**，从未出现该问题。
3. 两台机器逐项对照：agetty 2.42.4 / zsh 5.9.2 / kernel 7.2.9 完全一致，autologin 配置**只差一个词**：

| 机器 | getty@tty1 override 的 ExecStart | 会话内 TERM |
|---|---|---|
| X19（正常） | `agetty --autologin life --noclear %I $TERM` | `linux` |
| x18（双显） | `agetty --autologin work --noclear %I unknown` | `unknown` |

---

## 四、根因

agetty 的**最后一个位置参数是 TERM 值**。x18 的 override 把某篇指南里的 `unknown` 占位符原样照抄了进来，于是：

1. 自动登录会话的 `TERM=unknown` → terminfo 库查无此终端类型 → zsh 的行编辑器 ZLE 拿不到任何光标控制能力；
2. ZLE 退化为"无光标移动"的重画策略，在 framebuffer 控制台（fbcon）上**每个按键被绘制两遍**；
3. termios 的 `-echo` 始终是生效的（内核行规程没有回显），屏幕上第二个字符来自渲染层——这就是"显示两个、实际只输入一个"的由来。

---

## 五、修复

`/etc/systemd/system/getty@tty1.service.d/override.conf` 末尾与 X19 对齐：

```ini
[Service]
ExecStart=
ExecStart=-/sbin/agetty --autologin work --noclear %I $TERM
```

- `$TERM` 由 systemd 在启动服务时展开（getty 环境下为 `linux`）。
- 修改只对**下一次登录**生效；已存在的会话继续使用旧 TERM，退出后重新登录（或重启）即可。

---

## 六、验证（远程闭环对照实验）

无需有人在控制台前，即可注入按键并读取屏幕渲染结果：

```bash
# 1) 给一个空闲的 ttyN 挂上待测的 override 并启动
sudo systemctl start getty@ttyN

# 2) 向 ttyN 的输入队列注入一个字符（0x5412 = TIOCSTI，等效真实敲键）
sudo python3 -c "import fcntl,os; fcntl.ioctl(os.open('/dev/ttyN',os.O_RDONLY),0x5412,b'x')"

# 3) 读取该控制台的显存，按每行 240 列解析非空行，直接看屏幕上画了几个字符
sudo dd if=/dev/vcsN bs=16080 count=1 | python3 -c "
import sys
data = sys.stdin.buffer.read().decode('utf-8', 'replace')
for i in range(0, len(data), 240):
    row = data[i:i+240]
    if row.strip():
        print(f'row {i//240}: {row.rstrip()!r}')
"
```

实测数据（同一台 tty3，只改 TERM 做对照）：

| tty3 的 TERM | 注入 | 显存中的屏幕内容 |
|---|---|---|
| `unknown` | 1 个 `x` | `xx`（双显复现） |
| `$TERM` → linux | 1 个 `y` | `y`（正常） |

---

## 七、方法论沉淀

1. **受控对照实验优先于推理**：把嫌疑配置（autologin）复制到干净环境（tty2）复现，一锤定音；两个"合理假说"都是这样被否掉的。
2. **跨机器 diff 是最强证据源**：配置相同的对照机器，版本逐项比对后剩下的差异就是完整嫌疑名单——本案只剩一个单词。
3. **远程闭环复现**：TIOCSTI 注入按键 + `/dev/vcsN` 读显存，控制台问题可以在 SSH 会话里独立复现和验证，不依赖人在机器前。
4. **终端显示异常，先 `echo $TERM`，再怀疑 shell 和插件。**
5. 配置里的占位符（`unknown`、`<your-key>` 之类）照抄进真实配置就是埋雷；对照 [[Linux-TTY1自动登录与自动拉起Niri桌面配置]]，其中正确的写法就是 `%I $TERM`。
