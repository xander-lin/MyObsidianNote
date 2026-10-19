# Linux fcitx5 IME 相关包清理（仅保留 rime）与不可删项

x18 的输入法只用雾凇拼音（fcitx5 + fcitx5-rime），2026-10-19 对系统里 IME 相关的包做了一次盘点和清理。本文记录哪些删了、哪些**删不得**（依赖链条）以及清理后的验证。

---

## 一、盘点结果

`pacman -Q | grep -iE "ibus|fcitx|rime|..."` 命中 20 个真 IME 相关包（`libusb`、`dotnet-runtime`、`perl-timedate`、`shared-mime-info` 等是 "ime"/"mime" 的字面误命中）。

---

## 二、已删除（实测 2026-10-19）

```bash
sudo pacman -Rns fcitx5-chinese-addons
```

实际移除 5 个包、共 343.45 MiB：

| 包 | 删除原因 |
|---|---|
| `fcitx5-chinese-addons` | fcitx5 原生拼音/双拼/表格/云拼音引擎，profile 里只配了 rime，完全未用 |
| `libime` | 只被 chinese-addons 依赖，随之成为孤包 |
| `qt6-webengine` / `qt6-webchannel` / `qt6-positioning` | 只是 chinese-addons 拼音词典管理器 GUI 的依赖，共 ~300 MiB |

清理后 emoji 模块不受影响（`libemoji.so` 在 fcitx5 核心包里，不在 chinese-addons）。

---

## 三、不可删项（依赖链条核实）

**rime 方案数据包一个都删不得**：

```
librime →（硬依赖）→ librime-data →（硬依赖）→ rime-prelude / rime-essay /
    rime-luna-pinyin / rime-bopomofo / rime-cangjie / rime-stroke / rime-terra-pinyin
```

Arch 把经典 rime 方案（朙月、注音、仓颉、五笔画、地球拼音）全部捆在 `librime-data` 里，而 `librime` 又硬依赖它。删任何一级都会级联到 `librime`，进而杀掉 `fcitx5-rime`。`pactree -r` 逐包核实过。

其余保留项及用途：`fcitx5`（核心）、`fcitx5-gtk`/`fcitx5-qt`（GTK/Qt 应用的输入法模块）、`fcitx5-rime`（引擎桥接）、`fcitx5-configtool`（GUI 设置）、`rime-ice-git`（雾凇拼音本体，含的双拼 schema 文件同属此包、不单可删）、`fcitx5-ainput-git`（语音输入）、`noto-fonts-cjk`（中文字体）、`fcitx5-theme-wechat`（候选窗皮肤，可选）。

---

## 四、清理后验证

```bash
pgrep -x fcitx5                                        # 进程仍在（删包时模块并未加载）
busctl --user call org.fcitx.Fcitx5 /rime \
    org.fcitx.Fcitx.Rime1 GetCurrentSchema             # 返回 "rime_ice"
```

运行中的 fcitx5 无需重启：被删模块（pinyin/table 等）本就没有被按需加载。

---

## 五、遗留（无害，未处理）

`~/.config/fcitx5/conf/` 下的孤儿用户配置：`pinyin.conf`、`punctuation.conf`、`chttrans.conf`（属被删的 chinese-addons）、`vinput.conf`、`vinput.conf_grylSw`、`notifications.conf_AR0dw7`（旧 vinput 插件与写配置时的临时文件残留）。均为惰性文件，留着不影响运行。

相关：[[Linux-Rime大写锁定时Delete误删已上屏文本排查与修复]]
