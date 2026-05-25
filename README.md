# Kimi Code VS Code Extension: China Endpoint (api.moonshot.cn) Setup Guide

> **Problem**: The official `moonshot-ai.kimi-code` VS Code extension defaults to the international endpoint `https://api.moonshot.ai/v1`. If you registered on `platform.moonshot.cn` (China mainland), your API key is **not valid** on the international endpoint. The two platforms use separate authentication systems.

---

## Recommended Solution: Config File

The official extension reads its configuration from `~/.kimi/config.toml`. This is the cleanest approach — it survives extension updates and does not require modifying installed files.

### Step 1: Create the config directory and file

```bash
mkdir -p ~/.kimi
```

### Step 2: Write `config.toml`

**Important**: TOML parses dots in table keys as nested paths. You **must** use a clean local alias (e.g., `k2_6`) for the TOML table name, while keeping the actual API model name (`kimi-k2.6`) in the `model` field.

```toml
default_model = "k2_6"
default_thinking = false

[providers.moonshot-cn]
type = "kimi"
base_url = "https://api.moonshot.cn/v1"
api_key = "YOUR_CN_API_KEY_HERE"

[models.k2_6]
provider = "moonshot-cn"
model = "kimi-k2.6"
max_context_size = 256000
capabilities = ["thinking", "always_thinking", "image_in"]
```

### Step 3: Reload VS Code

```bash
code --restart
```

Or use the Command Palette (`Ctrl+Shift+P`) → `Developer: Reload Window`.

---

## How It Works

### Why the auth error happens

The extension's bundled JavaScript contains a hardcoded constant:

```js
MOONSHOT_BASE_URL = "https://api.moonshot.ai/v1";
```

When you paste a `platform.moonshot.cn` API key into an extension configured for `api.moonshot.ai`, the server rejects it with **401 Invalid Authentication** because the key was issued for a different region/tenant.

### Config file resolution

The official extension resolves config via this logic (found in `dist/extension.js`):

```js
function mr(r) {
  let e = r || process.env.KIMI_SHARE_DIR || Vt.join(ho.homedir(), ".kimi");
  return {
    home: e,
    config: Vt.join(e, "config.toml"),
    // ...
  };
}
```

This means:
- Default path: `~/.kimi/config.toml`
- Override via env: `KIMI_SHARE_DIR=/custom/path`

The config is validated against a Zod/Pydantic schema that expects:
- `providers.<name>.base_url` — supports custom endpoints
- `providers.<name>.api_key` — your China-region key
- `models.<alias>.provider` — references the provider name
- `models.<alias>.model` — the actual model ID sent to the API

### TOML dot-parsing trap

TOML interprets `[models.kimi-k2.6]` as `[models.kimi-k2]` with sub-key `6`, causing:

```
models.kimi-k2.provider  Field required
models.kimi-k2.model     Field required
```

**Fix**: Use `[models.k2_6]` (no dots) and set `model = "kimi-k2.6"` inside the table.

### Temperature enforcement (K2.5 only)

The China endpoint for `kimi-k2.5` enforces `temperature: 1` and rejects any other value with a 400 error. `kimi-k2.6` does not have this restriction. If you use K2.5 on `.cn`, ensure the client does not send `temperature: 0.7`.

---

## Troubleshooting

| Symptom | Cause | Fix |
|---------|-------|-----|
| `401 Invalid Authentication` | Key from `.cn` sent to `.ai` endpoint | Use `base_url = "https://api.moonshot.cn/v1"` |
| `Field required` for `kimi-k2` | TOML parsed dot as nested key | Use `[models.k2_6]` instead of `[models.kimi-k2.6]` |
| `Default model ... not found` | `default_model` string doesn't match TOML table key | Ensure `default_model = "k2_6"` matches `[models.k2_6]` |
| `400` with temperature error | K2.5 on `.cn` rejects `temperature != 1` | Switch to K2.6 or patch client to send `temperature: 1` |

---

## File Paths Reference (Linux)

| Purpose | Path |
|---------|------|
| Extension install (official) | `~/.vscode/extensions/moonshot-ai.kimi-code-0.5.10-linux-x64/` |
| Config file | `~/.kimi/config.toml` |
| Extension logs | `~/.kimi/logs/kimi.log` |
| Global storage | `~/.config/Code/User/globalStorage/moonshot-ai.kimi-code/` |

---

## License

This guide is provided as-is for educational purposes. Use at your own risk. Patching extension files may violate the extension's terms of service.

