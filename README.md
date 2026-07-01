# Codex Windows Sandbox Proxy Fix

A community-tested workaround for Codex Desktop on Windows when sandboxed tools fail because the sandbox helper repeatedly inherits a local proxy port.

[中文说明](README.zh-CN.md)

> [!IMPORTANT]
> This is a community workaround, not an official OpenAI fix. It was reproduced on Codex Desktop for Windows after version `26.616.6631.0`, with `sandbox = "elevated"` and proxy variables provided through `%USERPROFILE%\.codex\.env`.

## Symptoms

- `apply_patch`, `Get-Content`, or other sandboxed tools hang or fail.
- Windows shows a dialog from `codex-windows-sandbox-setup.exe`: `The specified module could not be found` / `找不到指定的模块`.
- Codex reports `CreateProcessAsUserW failed: 5` or `ShellExecuteExW failed to launch setup helper: 1223`.
- A patch may be written successfully while Codex still hangs at the final step.
- Elevated execution works while normal sandboxed execution fails.
- Reinstalling, repairing, resetting the app, or rebuilding `.sandbox` does not reliably fix it.

## What appears to happen

Codex itself needs the proxy, but the Windows sandbox setup helper also inherits proxy environment variables. The proxy port is written to:

```text
%USERPROFILE%\.codex\.sandbox\setup_marker.json
```

For example:

```json
"proxy_ports": [7890]
```

A later sandboxed operation can trigger `codex-windows-sandbox-setup.exe` to refresh sandbox or firewall state, causing the dialog and tool failure.

Clearing `proxy_ports` while Codex is still running is not enough. An old Codex process can write the port back.

## The missing edge case: WSS proxy variables

The original community workaround excluded:

```text
HTTP_PROXY HTTPS_PROXY ALL_PROXY
http_proxy https_proxy all_proxy
```

That fix can still fail when `.codex\.env` also defines `WSS_PROXY` or `wss_proxy`. In our reproduction, `proxy_ports` was repopulated until both WSS variants were excluded too.

## Fix

### 1. Fully exit Codex

Exit Codex, including the system tray process. Open Task Manager and confirm that no Codex-related process remains.

### 2. Update `config.toml`

Open `%USERPROFILE%\.codex\config.toml` and add or update the following settings. Do not create a second `[features]` section.

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

Other existing settings can remain. Keep `network_proxy = true` so Codex and Remote can still use the proxy.

### 3. Clear the stored proxy port

Open `%USERPROFILE%\.codex\.sandbox\setup_marker.json` and change only:

```json
"proxy_ports": []
```

Keep the rest of the JSON unchanged.

### 4. Start Codex normally

Start Codex without running it as Administrator. Your `.codex\.env` may remain in place. In our case, Remote continued to work after the exclusion list was corrected.

## Verification

Ask Codex to perform three consecutive operations:

1. Create a test file with `apply_patch`.
2. Modify the same file with `apply_patch`.
3. Delete it with `apply_patch`.

Confirm that no helper dialog appears, all operations return normally, Remote still connects, and `proxy_ports` stays empty.

## If it still fails

- Check Task Manager again for background Codex or launcher processes.
- If `proxy_ports` is populated again, look for additional proxy variables in `.codex\.env` or the Windows environment.
- Check uppercase and lowercase variants, especially `WSS_PROXY` and `wss_proxy`.
- Verify that there is only one `[features]` section.
- Restart Codex after changing `config.toml`.

Inspect proxy variable names without printing their values:

```powershell
Get-ChildItem Env: |
  Where-Object Name -Match '^(HTTP|HTTPS|ALL|WSS)_PROXY$' |
  Select-Object Name
```

## Temporary fallback

```toml
[windows]
sandbox = "unelevated"
```

This reduces sandbox isolation. Use it only temporarily, then restore `sandbox = "elevated"`.

## Security notes

- Do not publish your `.env`; it may contain proxy credentials or tokens.
- Do not modify permissions or encryption attributes inside `C:\Program Files\WindowsApps`.
- Do not permanently disable sandboxing only to hide this error.

## Related discussions

- [openai/codex issue #29200](https://github.com/openai/codex/issues/29200)
- [Reddit report and original workaround](https://www.reddit.com/r/codex/comments/1uccr76/comment/ot2wdz0/)
- [Chinese technical investigation](https://jishuzhan.net/article/2069609038043770881)

## Status

Validated on July 1, 2026. Future Codex releases may change this behavior or make the workaround unnecessary.

## License

MIT
