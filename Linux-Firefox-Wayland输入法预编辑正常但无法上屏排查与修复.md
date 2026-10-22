# Linux Firefox 输入法预编辑框正常但无法上屏的排查与修复

在 Wayland（Niri）+ fcitx5 环境下，Firefox 中按拼音输入时预编辑框能正常显示，但最终文字无法上屏（commit 失败），且语音输入也无效果。本文记录根因定位过程与解决方案。

---

## 一、问题现象

- 仅在 **Firefox** 中出现，其他程序（Kitty、Zenity、ZCode 等）输入法正常。
- 预编辑框能弹出、能打字，但选词后文字**不上屏**。
- fcitx5 侧日志显示 commit 已发生（`Vinput [commit] ... program=firefox`），说明文字是被 Firefox 侧丢弃的。
- 关闭 Firefox 重开无效；重启 fcitx5 也无效。

---

## 二、根因

Firefox **profile 的 `prefs.js` 中被写入了 `focusmanager.testmode=true`**。

```js
user_pref("focusmanager.testmode", true);
```

该 pref 原本是 Firefox 自动化测试用项（让窗口在后台时也保持「焦点」），在 Wayland 下它会破坏输入法的 commit 路径，导致预编辑正常但文字不上屏。

---

## 三、`focusmanager.testmode` 是怎么被写进去的

**触发点是启动参数 `--remote-debugging-port`**（Firefox 远程调试 / WebDriver BiDi 所需）：

- Firefox 在收到 `--remote-debugging-port` 启动时，会**自己**向该实例所用的 profile 写入一整套自动化测试 pref：
  - `focusmanager.testmode` = `true`
  - `app.update.disabledForTesting` = `true`
  - `browser.dom.window.dump.enabled` = `true`
  - `dom.input_events.security.minTimeElapsedInMS` = `0`
  - `dom.permissions.testing.enabled` = `true`
  - `geo.provider.testing` = `true`
  - `services.settings.server` = `"data:,#remote-settings-dummy/v1"`
  - `general.useragent.updates.enabled` = `false`
  - `dom.input_events.security.minNumTicks` = `0`

**关键特性（隔离实验验证）：**

| 启动方式 | `prefs.js` 中 `focusmanager.testmode` |
|---|---|
| `firefox -profile P`（不带调试端口） | 无 |
| `firefox --remote-debugging-port -profile P` | `true`（被写入） |
| 之后用**普通**方式启动同一 profile | 仍是 `true`（**粘性，不会自动恢复**） |

即：**只要用调试端口启动过一次，该 profile 就会永久带上 `true`**，此后普通启动也无法恢复——这正是「关掉重开仍然不行」的原因。

**污染来源：** 面向 Agent 的 Firefox 调试桥（GetBrowser）在 `browser_debug {on:true}` 且**未指定 profile**、同时 Firefox 未运行时，会用**默认 profile** 启动 `--remote-debugging-port`（见 `bridge/browser-debug.mjs`），从而污染日常使用的默认 profile。

---

## 四、排查方法：对照实验隔离 pref

为精确锁定是哪一个 pref，构建了如下实验装置：

1. 编写测试页，用 `dump()` 把 `<textarea>` 的实时内容写到标准输出（`browser.dom.window.dump.enabled=true` 时可用），这样无需人工描述即可读取输入结果。
2. 每个候选 pref 单独放入独立 profile 的 `user.js`，各开一个 Firefox 实例，逐个测试。

```html
<textarea id="t"></textarea>
<script>
const t = document.getElementById('t');
t.focus();
t.addEventListener('input', () => dump("BTINPUT [" + t.value + "]\n"));
</script>
```

实验结果：

| 测试窗口 | 注入的 pref | 能否上屏 |
|---|---|---|
| control | 无 | ✅ 正常 |
| focus | `focusmanager.testmode=true` | ❌ **不上屏** |
| iesec | `dom.input_events.security.*` | ✅ 正常 |
| other | geo / permissions / testing 组 | ✅ 正常 |

→ **唯一罪魁是 `focusmanager.testmode=true`。**

---

## 五、解决方案

在每个受影响的 profile 目录下新建 `user.js`：

```js
// Guard against automation pollution: launching Firefox with
// --remote-debugging-port injects focusmanager.testmode=true into this
// profile, which breaks IME (fcitx) commit under Wayland. user.js is
// re-applied every startup and overrides prefs.js, so the input method
// keeps working even in this automation profile.
user_pref("focusmanager.testmode", false);
```

**路径示例：**

- `~/.config/mozilla/firefox/<日常profile>/user.js`
- `~/.config/mozilla/firefox/<调试profile>/user.js`

**为什么用 `user.js` 而不是直接改 `prefs.js`：**

- `user.js` 在**每次启动时覆盖 `prefs.js`**，且优先级更高。
- 即使之后再次用 `--remote-debugging-port` 启动，Firefox 注入的 `true` 也会被 `user.js` 的 `false` 压住。

**验证（反复注入下 guard 稳定）：**

| 场景 | 结果 |
|---|---|
| 带调试端口启动 + 有 `user.js` | `prefs.js` 中无 `focusmanager` ✅ |
| 连续 3 次带调试端口启动 + `user.js` | 始终被压住 ✅ |

---

## 六、对自动化调试是否有影响

**没有影响。** 用 GetBrowser 自身的可信输入 E2E（`node test/bidi.mjs`，WebDriver BiDi）在两种状态下分别验证：

| 配置 | `focusmanager.testmode` | BiDi 可信输入 E2E |
|---|---|---|
| 无 guard（Firefox 注入） | `true` | 5/5 通过 |
| 有 guard（强制 `false`） | `false` | 5/5 通过 |

原因：

- `focusmanager.testmode` 只关掉「窗口在后台时也当作有焦点」，而 BiDi 输入直接派发到 browsing context，不依赖 OS 焦点。
- 其余被注入的自动化 pref 本就服务于调试（禁用更新、假 remote-settings、允许 `dump` 等），对调试有益无害，且都不影响 IME。

---

## 七、预防建议

1. **调试浏览器始终使用独立 profile**：如 `firefox -P Agent --remote-debugging-port=9222`，避免把默认 profile 暴露给远程调试。
2. **让调试启动器在未显式指定 profile 时拒绝启动**，而不是回退到默认 profile。
3. **保留 `user.js` guard**：即使将来被调试端口再次启动，也不会破坏输入法。

---

## 八、备查命令

```bash
# 查看某 profile 是否被注入
grep focusmanager ~/.config/mozilla/firefox/<profile>/prefs.js

# 确认 Firefox 是否以远程调试启动
pgrep -af "firefox --remote-debugging-port"

# 查看当前会话输入法环境
env | grep -iE "IM_MODULE|XMODIFIERS"
```
