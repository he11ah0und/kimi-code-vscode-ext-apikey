# Использование собственного API-ключа в расширении Kimi Code для VS Code

[English](./README.md) | [中文](./kimi-code-cn-endpoint-guide.zh.md)

По умолчанию расширение **Kimi Code** для VS Code использует встроенную **подписку Kimi** (управляется Moonshot AI). Однако вы можете настроить его на использование собственного **API-ключа** из [platform.moonshot.cn](https://platform.moonshot.cn) или [platform.moonshot.ai](https://platform.moonshot.ai).

Это руководство объясняет, как переключиться с управляемой подписки на собственный API-ключ, с поддержкой как **глобальной** (`api.moonshot.ai`), так и **китайской** (`api.moonshot.cn`) точек входа.

---

## Зачем использовать собственный API-ключ?

- **Пользователи из Китая**: аккаунты, зарегистрированные на `platform.moonshot.cn`, имеют API-ключи, которые работают только с точкой входа `api.moonshot.cn`. Эти ключи **не действительны** на глобальной точке входа `api.moonshot.ai`.
- **Глобальные пользователи**: если вы зарегистрированы на `platform.moonshot.ai`, ваш ключ работает с `api.moonshot.ai`.
- **Гибкость**: использование собственного API-ключа позволяет независимо от встроенной подписки расширения контролировать биллинг, лимиты запросов и доступ к моделям.

---

## Справка по точкам входа

| Регион | Платформа | Точка входа API |
|--------|-----------|-----------------|
| Материковый Китай | [platform.moonshot.cn](https://platform.moonshot.cn) | `https://api.moonshot.cn/v1` |
| Глобальная/Международная | [platform.moonshot.ai](https://platform.moonshot.ai) | `https://api.moonshot.ai/v1` |

> **Важно**: API-ключи **не взаимозаменяемы** между точками входа. Ключ `.cn` вернёт ошибку `401 Invalid Authentication` на точке входа `.ai`, и наоборот.

---

## Настройка

Расширение Kimi Code читает конфигурацию из файла `~/.kimi/config.toml`. Этот файл позволяет определять собственных провайдеров и модели.

### Шаг 1: Создайте директорию конфигурации

```bash
mkdir -p ~/.kimi
```

### Шаг 2: Напишите `config.toml`

Замените `YOUR_API_KEY_HERE` на ваш реальный API-ключ с соответствующей платформы.

#### Для пользователей из материкового Китая (`api.moonshot.cn`)

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

#### Для глобальных пользователей (`api.moonshot.ai`)

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

### Шаг 3: Перезагрузите VS Code

```bash
code --restart
```

Или используйте палитру команд (`Ctrl+Shift+P`) → **Developer: Reload Window**.

---

## Ключевые детали конфигурации

### Ключи таблиц TOML и точки

TOML интерпретирует точки в именах таблиц как пути вложенности. Например, `[models.kimi-k2.6]` распарсится как `[models.kimi-k2]` с подключом `6`, что вызывает ошибки валидации.

**Решение**: используйте чистый псевдоним без точек для имени таблицы TOML (например, `k2_6`), а реальное имя модели укажите в поле `model`:

```toml
[models.k2_6]          # <-- имя таблицы TOML (без точек)
provider = "moonshot-cn"
model = "kimi-k2.6"    # <-- реальный ID модели для API (точки здесь допустимы)
```

Поле `default_model` должно точно совпадать с именем таблицы TOML:

```toml
default_model = "k2_6"  # <-- соответствует [models.k2_6]
```

### Схема провайдера

Расширение валидирует `config.toml` по следующей схеме (извлечено из `dist/extension.js`):

```
providers.<name>.type       = "kimi" | "openai_legacy" | "anthropic" | ...
providers.<name>.base_url   = URL кастомной точки входа
providers.<name>.api_key    = ваш API-ключ
providers.<name>.env        = опциональные переменные окружения
providers.<name>.custom_headers = опциональные HTTP-заголовки

models.<alias>.provider     = имя провайдера выше
models.<alias>.model        = реальный ID модели, отправляемый в API
models.<alias>.max_context_size = размер контекстного окна
models.<alias>.capabilities = ["thinking", "always_thinking", "image_in"]
```

### Разрешение пути к конфигу

Расширение определяет путь к файлу конфигурации по следующей логике:

```js
function getConfigPath(customDir) {
  let base = customDir || process.env.KIMI_SHARE_DIR || path.join(os.homedir(), ".kimi");
  return path.join(base, "config.toml");
}
```

Это означает:
- **По умолчанию**: `~/.kimi/config.toml`
- **Переопределение**: задайте переменную окружения `KIMI_SHARE_DIR=/custom/path`

---

## Устранение неполадок

| Симптом | Причина | Исправление |
|---------|---------|-------------|
| `401 Invalid Authentication` | API-ключ от другой точки входа (например, `.cn` ключ отправлен на `.ai`) | Убедитесь, что `base_url` соответствует платформе, на которой выдан ключ |
| `Field required` для `kimi-k2` | TOML распарсил точку в имени модели как вложенный ключ | Используйте `[models.k2_6]` вместо `[models.kimi-k2.6]` |
| `Default model ... not found` | строка `default_model` не совпадает с ключом таблицы TOML | Убедитесь, что `default_model = "k2_6"` соответствует `[models.k2_6]` |
| Файл конфигурации не найден | `~/.kimi/config.toml` отсутствует или в неправильном месте | Проверьте путь командой `cat ~/.kimi/config.toml` |
| Расширение всё ещё использует встроенную подписку | провайдер `managed:kimi-code` имеет приоритет | Удалите или переопределите управляемого провайдера в конфиге |

---

## Справка по путям файлов (Linux)

| Назначение | Путь |
|------------|------|
| Файл конфигурации | `~/.kimi/config.toml` |
| Директория установки расширения | `~/.vscode/extensions/moonshot-ai.kimi-code-*/` |
| Логи расширения | `~/.kimi/logs/kimi.log` |
| Глобальное хранилище | `~/.config/Code/User/globalStorage/moonshot-ai.kimi-code/` |

---

## Лицензия

Это руководство предоставляется как есть в образовательных целях.
