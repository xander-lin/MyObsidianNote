# Fcitx5-Rime 雾凇拼音默认英文标点配置指南

雾凇拼音（rime-ice）默认输出中文标点。如果希望在输入中文时默认使用英文/半角标点（常用于编写代码或 Markdown 文档），可以通过 Rime 的 custom patch 机制进行配置。

---

## 一、配置原理

1. **配置层级**：
   - fcitx5 的系统配置位于 `~/.config/fcitx5/`，而 Rime 方案层的配置位于 `~/.local/share/fcitx5/rime/`。标点输出行为由 Rime 方案层控制。
   - 不要直接修改 `build/` 目录中的生成文件，而应通过 `<schema>.custom.yaml` 进行 patch。
2. **`switches/@1`**：
   - 对应 `rime_ice.schema.yaml` 中的开关列表第二项 `ascii_punct`（开关列表顺序为：`ascii_mode`、`ascii_punct`、`traditionalization`、`emoji` 等）。
3. **`reset: 1`**：
   - `states` 中 `0` 为中文标点（`，。`），`1` 为英文标点（`,.`）。
   - 设置 `reset: 1` 使输入法每次启动/切换时默认重置为第二状态（英文标点）。
   - 临时切换中英文标点可使用快捷键 `Ctrl+Shift+3`（对应 `default.yaml` 中配置的 `key_binder`）。

---

## 二、配置步骤

### 1. 添加 patch 配置

在 `~/.local/share/fcitx5/rime/rime_ice.custom.yaml`（若文件不存在则新建）中添加：

```yaml
patch:
  "switches/@1/reset": 1
```

---

### 2. 重新部署 Rime

执行命令重新部署 Rime 配置：

```bash
fcitx5-remote -r
```

---

### 3. 验证生效

检查重新生成后的 schema 配置文件：

```bash
grep -A3 'ascii_punct' ~/.local/share/fcitx5/rime/build/rime_ice.schema.yaml
```

预期输出应包含 `reset: 1`：

```yaml
  - name: ascii_punct
    reset: 1
    states: ["，。", ",."]
```
