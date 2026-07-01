# Codex Windows 沙箱代理冲突修复

这是一个经过实际复现和验证的社区解决方案，用于处理 Codex Desktop Windows 版在使用本地代理时出现的沙箱写入失败。

[English](README.md)

> [!IMPORTANT]
> 这不是 OpenAI 官方修复。问题在 Codex Desktop for Windows `26.616.6631.0` 更新后出现，并在 `sandbox = "elevated"`、通过 `%USERPROFILE%\.codex\.env` 配置代理的环境中得到复现。

## 症状

- `apply_patch`、`Get-Content` 等沙箱工具卡住或失败。
- Windows 弹出来自 `codex-windows-sandbox-setup.exe` 的窗口，提示“找不到指定的模块”。
- Codex 返回 `CreateProcessAsUserW failed: 5` 或 `ShellExecuteExW failed to launch setup helper: 1223`。
- 文件有时已经成功写入，但 Codex 仍卡在最后一步。
- 管理员或外部执行可以工作，普通沙箱执行失败。
- 修复、重置、重装 Codex，甚至重建 `.sandbox` 后仍会复发。

## 前因后果

Codex 本身需要代理才能联网，但 Windows 沙箱辅助程序也继承了代理环境变量。代理端口随后会被写入：

```text
%USERPROFILE%\.codex\.sandbox\setup_marker.json
```

例如：

```json
"proxy_ports": [7890]
```

之后执行 `apply_patch` 等沙箱操作时，`codex-windows-sandbox-setup.exe` 可能再次尝试刷新沙箱或防火墙配置，从而触发弹窗和写入失败。

仅手动清空 `proxy_ports` 并不可靠。如果旧的 Codex 进程仍在运行，它会再次把端口写回去。

## 现有方案的遗漏：WSS 代理

Reddit 上的原始方案排除了以下变量：

```text
HTTP_PROXY HTTPS_PROXY ALL_PROXY
http_proxy https_proxy all_proxy
```

但如果 `.codex\.env` 中还定义了：

```text
WSS_PROXY
wss_proxy
```

端口仍可能重新进入 `setup_marker.json`。我们的第一次复测正是因此失败：`proxy_ports` 被重新写入；补充排除大小写两种 WSS 变量并彻底重启后，问题才真正消失。

## 修复步骤

### 1. 彻底退出 Codex

完全退出 Codex，包括系统托盘中的 Codex。打开任务管理器，确认所有 Codex 及相关启动器进程都已结束。

顺序很重要。旧进程不退出，清空的端口仍可能被写回。

### 2. 修改 `config.toml`

打开：

```text
%USERPROFILE%\.codex\config.toml
```

加入或修改以下配置。已有 `[features]` 时不要重复创建第二个。

```toml
[features]
network_proxy = true

[shell_environment_policy]
inherit = "all"
exclude = [
  "HTTP_PROXY",
  "HTTPS_PROXY",
  "ALL_PROXY",
  "WSS_PROXY",
  "http_proxy",
  "https_proxy",
  "all_proxy",
  "wss_proxy",
]

[windows]
sandbox = "elevated"
```

其他原有配置可以保留：

- `network_proxy = true` 让 Codex 和 Remote 继续正常使用代理。
- `exclude` 阻止工具和沙箱 shell 继承代理变量。
- 保留 `sandbox = "elevated"`，无需长期降低沙箱隔离强度。

### 3. 清空已记录的代理端口

打开：

```text
%USERPROFILE%\.codex\.sandbox\setup_marker.json
```

只修改 `proxy_ports`：

```json
"proxy_ports": []
```

保留其他 JSON 内容不变。

### 4. 正常启动 Codex

以普通方式启动 Codex，不需要管理员权限。

`.codex\.env` 可以保留。我们的实际验证中，补全排除项后，Remote 可以正常连接，同时沙箱文件写入也恢复正常。

## 验证方法

让 Codex 连续执行三次操作：

1. 用 `apply_patch` 创建一个测试文件。
2. 用 `apply_patch` 修改同一个文件。
3. 用 `apply_patch` 删除该文件。

确认：

- 没有再次弹出 `codex-windows-sandbox-setup.exe` 报错。
- 三次调用均正常返回，不再卡在最后一步。
- Remote 可以正常连接。
- `proxy_ports` 保持为空。

我们完成了创建、修改、删除的连续测试，均无弹窗和卡顿。

## 如果仍然失败

- 再次检查任务管理器，确认没有后台 Codex 或启动器进程。
- 重新打开 `setup_marker.json`。如果端口再次出现，说明仍有代理变量被继承。
- 检查 `.codex\.env` 和 Windows 环境变量中的大小写版本。
- 尤其检查容易遗漏的 `WSS_PROXY` 和 `wss_proxy`。
- 确认 `config.toml` 中只有一个 `[features]`。
- 修改 `config.toml` 后必须重启 Codex，旧进程不会自动重新加载配置。

只查看代理变量名、不显示具体值：

```powershell
Get-ChildItem Env: |
  Where-Object Name -Match '^(HTTP|HTTPS|ALL|WSS)_PROXY$' |
  Select-Object Name
```

## 临时绕过方案

如果仍无法修复，可以暂时改为：

```toml
[windows]
sandbox = "unelevated"
```

这个方案会降低沙箱隔离强度，只建议临时使用。修复代理继承问题后，应恢复为 `sandbox = "elevated"`。

## 安全提醒

- 不要公开自己的 `.env`，其中可能包含代理凭据或令牌。
- 不建议修改 `C:\Program Files\WindowsApps` 中的权限或加密属性，这可能破坏应用包完整性和后续更新。
- 不建议为了隐藏报错而长期关闭或降低沙箱保护。

## 相关讨论

- [openai/codex issue #29200](https://github.com/openai/codex/issues/29200)
- [Reddit 原始报告与代理排除方案](https://www.reddit.com/r/codex/comments/1uccr76/comment/ot2wdz0/)
- [中文技术排查文章](https://jishuzhan.net/article/2069609038043770881)

## 状态

本方案于 2026 年 7 月 1 日完成验证。后续 Codex 更新可能改变相关行为，或使该解决方案不再需要。

## 许可证

MIT
