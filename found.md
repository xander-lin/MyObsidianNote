# Lessons Learned

## 1. 笔记里的配置被落地时改坏：`$TERM` 被写成字面量 `unknown`（2026-10-12）

- **Context**: 《Linux-TTY1自动登录与自动拉起Niri桌面配置》笔记中
  agetty 的 ExecStart 写的是正确的 `%I $TERM`，但 x18 落地时末位被换成
  字面量 `unknown`（来源不详，疑为照抄了其他指南的占位符）。结果所有
  autologin 会话 `TERM=unknown`，zsh ZLE 无 terminfo，控制台每个按键
  绘制两次（"按键双显"），排查耗时且先后误判为 zsh-autosuggestions 和
  niri 状态残留。
- **Root cause**: 配置示例中环境相关的值没有醒目警告，落地时无法分辨
  哪些位置允许替换、哪些必须原样保留；机器故障排查时也没有先和库内
  母本笔记 diff。
- **Correction**: x18 的 getty@tty1 override 已改回 `%I $TERM`（与 X19
  一致）；母本笔记已补 `> [!warning]` callout，并新增
  《Linux-TTY自动登录后字符双显排查与修复》完整记录。
- **Prevention**:
  - 配置类笔记中所有可变/环境相关的值必须就地标注或加警告 callout；
  - 排查机器问题时，先 `diff` 机器上的实际配置与库内对应笔记；
  - 终端显示异常的第一步是 `echo $TERM`，而不是怀疑 shell 插件。

（其余待记：本项目暂无其他已核实的失误记录。）
