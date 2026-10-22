# Linux Rime 开启大写锁定时无法编辑预编辑框排查与修复

x18（Arch Linux + niri + fcitx5-rime，方案雾凇拼音）上，**Caps Lock 开启**时输入拼音，预编辑框能正常显示，但预编辑框里的内容改不动——按 BackSpace 删不掉刚输入的字母、Esc 取消不掉、Delete 无反应、方向键移不进去。这些按键并没有消失，而是被 Rime **原样透传给当前应用**，于是实际被编辑的是应用里**已经上屏**的文本。Caps Lock 关闭时一切正常。本文记录从复现到 librime 源码级根因、再到"配置层补绑"修复的全过程。

---

## 一、问题现象

- Caps Lock 灯亮 → 输入拼音（预编辑可见）→ 按 BackSpace / Delete / Esc / 方向键：**预编辑框毫无反应**，按键落到当前应用上；
  - BackSpace 删掉的是光标**前**已上屏的字，Delete 删掉的是光标**后**已上屏的字，方向键移动的是应用里的光标——想改拼音，改的却是正文。
- Caps Lock 关闭 → 同样操作：按键被输入法消费，预编辑框正常可编辑（BackSpace 删拼音字母、Esc 清空预编辑），应用文本不受影响；
- 同一场景下 回车 / 空格（选候选）同样受影响。
- 判断要点：**只在 Caps Lock 亮着时复现**；症状是"按键透传给应用"，不是"输入法无响应"。

---

## 二、根因（librime 1.17.0 + fcitx5-rime 5.1.16 源码级确认）

1. **fcitx5-rime 原样传递修饰位**（`src/rimestate.cpp`）：按键状态掩码显式包含 `KeyState::CapsLock`，Caps Lock 锁定时每个键事件都带 `Lock` 位（kLockMask = 1<<1）传给 librime 的 `process_key`。
2. **librime 键位是"键 + 修饰位"精确匹配**（`src/rime/gear/key_binding_processor_impl.h`）：
   - editor 内建键位形如 `{XK_Delete, 0}` → DeleteChar（预编辑内前向删除）；
   - `KeyEvent::operator==` **含 modifier_ 逐位比较**；
   - 回退机制 `FallbackOptions::All` 只处理 **Shift**（ShiftAsControl / IgnoreShift），**没有 Lock 回退**。
3. 于是 Caps Lock 开启时 Delete/BackSpace/Esc/Return/空格 全部失配 → editor 返回 kNoop → fcitx5 把按键**原样转发给应用**：预编辑框因此完全不可编辑，应用则按自己的语义处理这次按键（删除、取消、移动光标）。
4. 同理失效的还有：navigator 的方向键、key_binder 里修饰位为 0 的配置绑定（Tab、minus/equal 翻页、小键盘键）。
5. 2026-10 查证 librime master 的 `key_binding_processor.*` 与 1.17.0 完全一致，**上游未修复**。

---

## 三、修复思路与配置

不在 fcitx5-rime 层面剥 Lock 位（会破坏 `good_old_caps_lock` 的大写直通），而是**在 key_binder 里显式补 Lock 变体绑定**。`Lock` 是 librime 认可的修饰位名（`key_table.cc` 的 `modifier_name[]`，`KeyEvent::Parse` 可解析），`send:` 会把去掉 Lock 位的同名键**重放**回完整处理链，从而命中内建键位：

**配置文件**：`~/.local/share/fcitx5/rime/default.custom.yaml`（对全部方案生效，经各方案的 `import_preset: default` 并入）

**配置内容**（在原有 `patch:` 下追加）：

```yaml
patch:
  __include: rime_ice_suggestion:/
  "ascii_composer/switch_key/Caps_Lock": noop
  "key_binder/bindings/+":
    - {accept: "Lock+Delete", send: Delete, when: composing}
    - {accept: "Lock+BackSpace", send: BackSpace, when: composing}
    - {accept: "Lock+Escape", send: Escape, when: composing}
    - {accept: "Lock+Return", send: Return, when: composing}
    - {accept: "Lock+space", send: space, when: has_menu}
```

> [!note] 键名说明
> 主回车键的 keysym 名是 **`Return`**（X11 命名），没有叫 "Enter" 的键；小键盘回车是 `KP_Enter`。

**部署**：

```bash
rime_deployer --build ~/.local/share/fcitx5/rime /usr/share/rime-data ~/.local/share/fcitx5/rime/build
# 之后重启 fcitx5（Wayland 会话内）：
WAYLAND_DISPLAY=wayland-1 DISPLAY=:0 fcitx5 -d   # 先 pkill -x fcitx5，且必须 detached
```

---

## 四、验证（无头 harness，不碰运行中的输入法）

写一个极小的 C 程序直接调 librime C API，对 user 目录的**拷贝**运行（避免与 fcitx5 并发打开 userdb）：

```c
// 核心：模拟 fcitx5-rime 的按键投递。Caps Lock 锁定 => 修饰位带 1<<1
static Bool press(RimeSessionId id, int keycode, Bool caps_on) {
    return api->process_key(id, keycode, caps_on ? (1 << 1) : 0);
}
// 标准初始化：api->setup/initialize/start_maintenance/join_maintenance_thread
// traits.user_data_dir 指向拷贝目录；select_schema(id, "rime_ice")
// 返回值 handled=0 表示 fcitx5 会把键转发给应用（即 bug），handled=1 表示 IME 已消费
```

实测对照（输入 `nihao` 后按键；`handled` 为判据）：

| 按键（Caps Lock 开，带 Lock 位） | 修复前 | 修复后 |
|---|---|---|
| Delete | handled=0（透传给应用，预编辑不可编辑） | **handled=1** |
| BackSpace | handled=0 | **handled=1** |
| Escape | handled=0 | **handled=1**（且正确清空预编辑） |
| 主回车 Return | handled=0 | **handled=1**（提交原文 "nihao"） |
| 小键盘 KP_Enter | — | handled=0（未覆盖，需要时补一行） |
| Left（navigator 方向键） | handled=0 | handled=0（同类残留，见下） |

---

## 五、残留与注意事项

- **未覆盖（同类残留，按需加行）**：navigator 方向键（Left/Right/Home/End）、Tab、minus/equal 翻页、小键盘键。每个键一行 `Lock+<键> → send <键>` 即可。
- **语义提示**：caret 在预编辑末尾时，rime 的 Delete 本来就是"消费但无可删"（前向删除需要先用方向键把 caret 移进去）；修复目标是与 Caps 关闭时行为完全一致，不是改变 rime 语义。
- **不要整体剥 Lock 位**：librime 的 `good_old_caps_lock: true`（大写锁定时字母直通英文）依赖事件里的 Lock 位，fcitx5-rime 层剥位会把字母输入也改坏。只能按键补绑。

---

## 六、方法论沉淀

1. **"大写锁定 + 输入法行为异常"先想修饰位精确匹配**：librime 的键位绑定（内建和配置）都要求修饰位逐位相等，Lock/NumLock 位会让所有 mask-0 绑定集体失配，且失败是静默的——表现不是输入法报错或死掉，而是按键被透传给应用（预编辑框无响应，正文却跟着变）。
2. **IME 键行为可以用 librime C API 写无头 harness 断言**（`process_key` 返回值即"是否被输入法消费"），比 GUI 复现快、可批量、可对照；userdb 不能两进程同开，务必对**拷贝**测试。
3. 判断"键被谁处理"的最短路径：读 IME 引擎源码里键位匹配的**比较函数**，确认修饰位是否参与相等判断、有哪些回退。
4. 相关：[[Fcitx5-Rime-雾凇拼音默认英文标点配置]]（同库的 rime 方案层配置笔记）。
