# Using Custom API Key in Kimi Code VS Code Extension

By default, the **Kimi Code** VS Code extension uses the built-in **Kimi subscription** (managed by Moonshot AI). However, you can configure it to use your own **API key** from [platform.moonshot.cn](https://platform.moonshot.cn) or [platform.moonshot.ai](https://platform.moonshot.ai).

This guide explains how to switch from the managed subscription to a custom API key, with support for both the **global** (`api.moonshot.ai`) and **China mainland** (`api.moonshot.cn`) endpoints.

---

## Why Use a Custom API Key?

- **China users**: Accounts registered on `platform.moonshot.cn` have API keys that only work with the `api.moonshot.cn` endpoint. These keys are **not valid** on the global `api.moonshot.ai` endpoint.
- **Global users**: If you registered on `platform.moonshot.ai`, your key works with `api.moonshot.ai`.
- **Flexibility**: Using your own API key lets you control billing, rate limits, and model access independently of the extension's built-in subscription.

---

## Endpoint Reference

| Region | Platform | API Endpoint |
|--------|----------|-------------|
| China mainland | [platform.moonshot.cn](https://platform.moonshot.cn) | `https://api.moonshot.cn/v1` |
| Global/International | [platform.moonshot.ai](https://platform.moonshot.ai) | `https://api.moonshot.ai/v1` |

> **Important**: API keys are **not interchangeable** between endpoints. A `.cn` key will fail with `401 Invalid Authentication` on the `.ai` endpoint, and vice versa.

---

## Configuration

The Kimi Code extension reads its configuration from `~/.kimi/config.toml`. This file lets you define custom providers and models.

### Step 1: Create the config directory

```bash
mkdir -p ~/.kimi
```

### Step 2: Write `config.toml`

Replace `YOUR_API_KEY_HERE` with your actual API key from the appropriate platform.

#### For China mainland users (`api.moonshot.cn`)

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

#### For global users (`api.moonshot.ai`)

```toml
default_model = "k2_6"
default_thinking = false

[providers.moonshot-global]
type = "kimi"
base_url = "https://api.moonshot.ai/v1"
api_key = "YOUR_GLOBAL_API_KEY_HERE"

[models.k2_6]
provider = "moonshot-global"
model = "kimi-k2.6"
max_context_size = 256000
capabilities = ["thinking", "always_thinking", "image_in"]
```

### Step 3: Reload VS Code

```bash
code --restart
```

Or use the Command Palette (`Ctrl+Shift+P`) → **Developer: Reload Window**.

---

## Key Configuration Details

### TOML Table Keys and Dots

TOML interprets dots in table names as nested paths. For example, `[models.kimi-k2.6]` is parsed as `[models.kimi-k2]` with a sub-key `6`, which causes validation errors.

**Solution**: Use a clean alias without dots for the TOML table name (e.g., `k2_6`), and put the actual model name in the `model` field:

```toml
[models.k2_6]          # <-- TOML table name (no dots)
provider = "moonshot-cn"
model = "kimi-k2.6"    # <-- actual API model ID (dots OK here)
```

The `default_model` field must match the TOML table name exactly:

```toml
default_model = "k2_6"  # <-- matches [models.k2_6]
```

### Provider Schema

The extension validates `config.toml` against this schema (extracted from `dist/extension.js`):

```
providers.<name>.type       = "kimi" | "openai_legacy" | "anthropic" | ...
providers.<name>.base_url   = custom endpoint URL
providers.<name>.api_key    = your API key
providers.<name>.env        = optional environment variables
providers.<name>.custom_headers = optional HTTP headers

models.<alias>.provider     = name of the provider above
models.<alias>.model        = actual model ID sent to the API
models.<alias>.max_context_size = context window size
models.<alias>.capabilities = ["thinking", "always_thinking", "image_in"]
```

### Config Resolution

The extension resolves the config file via this logic:

```js
function getConfigPath(customDir) {
  let base = customDir || process.env.KIMI_SHARE_DIR || path.join(os.homedir(), ".kimi");
  return path.join(base, "config.toml");
}
```

This means:
- **Default**: `~/.kimi/config.toml`
- **Override**: Set `KIMI_SHARE_DIR=/custom/path` environment variable

---

## Troubleshooting

| Symptom | Cause | Fix |
|---------|-------|-----|
| `401 Invalid Authentication` | API key from wrong endpoint (e.g., `.cn` key sent to `.ai`) | Ensure `base_url` matches the platform where your key was issued |
| `Field required` for `kimi-k2` | TOML parsed dot in model name as nested key | Use `[models.k2_6]` instead of `[models.kimi-k2.6]` |
| `Default model ... not found` | `default_model` string doesn't match TOML table key | Ensure `default_model = "k2_6"` matches `[models.k2_6]` |
| Config file not found | `~/.kimi/config.toml` missing or in wrong location | Verify path with `cat ~/.kimi/config.toml` |
| Extension still uses built-in subscription | `managed:kimi-code` provider takes priority | Remove or override the managed provider in config |

---

## File Paths Reference (Linux)

| Purpose | Path |
|---------|------|
| Config file | `~/.kimi/config.toml` |
| Extension install | `~/.vscode/extensions/moonshot-ai.kimi-code-*/` |
| Extension logs | `~/.kimi/logs/kimi.log` |
| Global storage | `~/.config/Code/User/globalStorage/moonshot-ai.kimi-code/` |

---

## License

This guide is provided as-is for educational purposes.
