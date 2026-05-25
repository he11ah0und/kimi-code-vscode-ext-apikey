# 在 Kimi Code VS Code 扩展中使用自定义 API 密钥

[English](./README.md) | [Русский](./kimi-code-cn-endpoint-guide.ru.md)

默认情况下，**Kimi Code** VS Code 扩展使用内置的 **Kimi 订阅**（由 Moonshot AI 管理）。但你可以将其配置为使用自己在 [platform.moonshot.cn](https://platform.moonshot.cn) 或 [platform.moonshot.ai](https://platform.moonshot.ai) 上获取的 **API 密钥**。

本指南说明如何从托管订阅切换到自定义 API 密钥，并支持**全球**（`api.moonshot.ai`）和**中国大陆**（`api.moonshot.cn`）两种接入点。

---

## 为什么要使用自定义 API 密钥？

- **中国用户**：在 `platform.moonshot.cn` 注册的账户，其 API 密钥仅适用于 `api.moonshot.cn` 接入点。这些密钥在**全球** `api.moonshot.ai` 接入点上**无效**。
- **全球用户**：如果你在 `platform.moonshot.ai` 注册，你的密钥适用于 `api.moonshot.ai`。
- **灵活性**：使用自己的 API 密钥可以让你独立于扩展内置订阅来控制计费、速率限制和模型访问权限。

---

## 接入点参考

| 区域 | 平台 | API 接入点 |
|------|------|-----------|
| 中国大陆 | [platform.moonshot.cn](https://platform.moonshot.cn) | `https://api.moonshot.cn/v1` |
| 全球/国际 | [platform.moonshot.ai](https://platform.moonshot.ai) | `https://api.moonshot.ai/v1` |

> **重要**：API 密钥在两种接入点之间**不可互换**。`.cn` 密钥在 `.ai` 接入点上会返回 `401 认证无效`，反之亦然。

---

## 配置方法

Kimi Code 扩展从 `~/.kimi/config.toml` 读取配置。该文件允许你定义自定义提供商和模型。

### 步骤 1：创建配置目录

```bash
mkdir -p ~/.kimi
```

### 步骤 2：编写 `config.toml`

将 `YOUR_API_KEY_HERE` 替换为你在对应平台上获取的实际 API 密钥。

#### 中国大陆用户（`api.moonshot.cn`）

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

#### 全球用户（`api.moonshot.ai`）

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

### 步骤 3：重新加载 VS Code

```bash
code --restart
```

或使用命令面板（`Ctrl+Shift+P`）→ **Developer: Reload Window**。

---

## 关键配置细节

### TOML 表名与点号

TOML 将表名中的点号解析为嵌套路径。例如，`[models.kimi-k2.6]` 会被解析为 `[models.kimi-k2]` 加子键 `6`，从而导致验证错误。

**解决方案**：为 TOML 表名使用不含点号的简洁别名（例如 `k2_6`），并将实际模型名称放在 `model` 字段中：

```toml
[models.k2_6]          # <-- TOML 表名（无点号）
provider = "moonshot-cn"
model = "kimi-k2.6"    # <-- 实际 API 模型 ID（此处点号可用）
```

`default_model` 字段必须与 TOML 表名完全匹配：

```toml
default_model = "k2_6"  # <-- 与 [models.k2_6] 匹配
```

### 提供商 Schema

扩展根据以下 schema 验证 `config.toml`（从 `dist/extension.js` 中提取）：

```
providers.<name>.type       = "kimi" | "openai_legacy" | "anthropic" | ...
providers.<name>.base_url   = 自定义接入点 URL
providers.<name>.api_key    = 你的 API 密钥
providers.<name>.env        = 可选环境变量
providers.<name>.custom_headers = 可选 HTTP 请求头

models.<alias>.provider     = 上述提供商的名称
models.<alias>.model        = 发送至 API 的实际模型 ID
models.<alias>.max_context_size = 上下文窗口大小
models.<alias>.capabilities = ["thinking", "always_thinking", "image_in"]
```

### 配置文件解析路径

扩展通过以下逻辑解析配置文件：

```js
function getConfigPath(customDir) {
  let base = customDir || process.env.KIMI_SHARE_DIR || path.join(os.homedir(), ".kimi");
  return path.join(base, "config.toml");
}
```

这意味着：
- **默认路径**：`~/.kimi/config.toml`
- **覆盖路径**：设置环境变量 `KIMI_SHARE_DIR=/custom/path`

---

## 故障排除

| 现象 | 原因 | 解决方法 |
|------|------|----------|
| `401 Invalid Authentication` | API 密钥与接入点不匹配（例如 `.cn` 密钥发送到 `.ai`） | 确保 `base_url` 与你获取密钥的平台一致 |
| `Field required` for `kimi-k2` | TOML 将模型名中的点号解析为嵌套键 | 使用 `[models.k2_6]` 而非 `[models.kimi-k2.6]` |
| `Default model ... not found` | `default_model` 字符串与 TOML 表名不匹配 | 确保 `default_model = "k2_6"` 与 `[models.k2_6]` 匹配 |
| 找不到配置文件 | `~/.kimi/config.toml` 缺失或位置错误 | 用 `cat ~/.kimi/config.toml` 验证路径 |
| 扩展仍使用内置订阅 | `managed:kimi-code` 提供商优先级更高 | 在配置中移除或覆盖托管提供商 |

---

## 文件路径参考（Linux）

| 用途 | 路径 |
|------|------|
| 配置文件 | `~/.kimi/config.toml` |
| 扩展安装目录 | `~/.vscode/extensions/moonshot-ai.kimi-code-*/` |
| 扩展日志 | `~/.kimi/logs/kimi.log` |
| 全局存储 | `~/.config/Code/User/globalStorage/moonshot-ai.kimi-code/` |

---

## 许可证

本指南按原样提供，仅供教育用途。
