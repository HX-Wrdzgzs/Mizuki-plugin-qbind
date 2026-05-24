---
description: "账号绑定(qbind)插件 — 全局 QQ 号绑定指令，用户必须先 qbind 才可使用官方 Bot 功能；后端 Gensokyo API，Markdown 格式消息与按钮交互"
---

# qbind — 全局账号绑定插件

## 概述

`qbind`（账号绑定）是一个基于 **NoneBot2 + OneBot V11** 开发的 **全局前置绑定插件**。

**核心问题**：QQ 官方 Bot 的 `event.get_user_id()` 返回的是**随机会话 ID**，而非用户的真实 QQ 号。这意味着：
- 用户若不绑定，Bot 无法识别其身份
- 其他功能（如查分器）因为无法获取真实 QQ 号而报错

**解决方案**：三步走机制
1. **强制绑定** — 用户**必须先执行 `qbind <真实QQ号>`** 完成绑定，否则无法使用任何 Bot 功能
2. **持久化存储** — 绑定关系 (`session_id → 真实QQ`) 存储在 `binds.json`，Bot 重启不丢失
3. **导出工具函数** — 其他插件可导入 `is_bound()`、`get_real_qq()`、`ensure_bound()` 进行访问控制

> ⚠️ 这是一个全局前置插件 — **未绑定的用户所有指令都会被拦截**，直到完成绑定。

---

## 后端 API（Gensokyo）

插件通过 **Gensokyo** 后端的 HTTP API 完成绑定操作。

| 项目 | 值 |
|------|-----|
| 基地址 | `http://127.0.0.1:15630/` |
| 端点 | `getid` |
| 方法 | HTTP GET |

### API 参数说明

| 参数 | 说明 |
|------|------|
| `oldRowValue` | 用户的**会话虚拟 ID**（`event.get_user_id()` 返回值） |
| `newRowValue` | 绑定值（真实 QQ 号）或 `0`（清除绑定） |
| `type` | `5` — 设置/清除绑定值 |

### 绑定/解绑 API

| 操作 | 请求 | 成功响应 | 失败响应 |
|------|------|---------|---------|
| **绑定** | `GET /getid?oldRowValue={会话ID}&newRowValue={QQ号}&type=5` | `{"status":"success"}` | `{"error":"{QQ} :已存在"}` |
| **解绑** | `GET /getid?oldRowValue={会话ID}&newRowValue=0&type=5` | `{"status":"success"}` | `{"error":"..."}` |
| **查询** | `GET /getid?oldRowValue={会话ID}&type=4` | `{"value":"当前绑定值"}` | 无 |

> ⚠️ **关键注意事项**：
> - `oldRowValue` 必须传 `event.get_user_id()` 返回的**虚拟会话 ID**（如 `20974018`），不能传用户输入的真实 QQ 号！
> - `newRowValue` 传用户输入的真实 QQ 号（绑定参数）
> - 解绑时 `newRowValue=0` 清除绑定记录
> - 若 QQ 号已被其他用户绑定，返回 `{"error":"{QQ} :已存在"}`，需先解绑

**代码实现**（`__init__.py`）：
```python
def _build_api_url(session_id: str, real_qq: str) -> str:
    """绑定"""
    return f"http://127.0.0.1:15630/getid?oldRowValue={session_id}&newRowValue={real_qq}&type=5"

def _build_unbind_api_url(session_id: str) -> str:
    """解绑"""
    return f"http://127.0.0.1:15630/getid?oldRowValue={session_id}&newRowValue=0&type=5"
```

---

## 指令详情

| 属性 | 值 |
|------|-----|
| 指令名 | `qbind` / `qunbind` |
| 触发方式 | `qbind <绑定参数>` / `qunbind [确认]` |
| 响应范围 | **全局** — 不限制 bot.self_id，支持群聊和私聊 |
| 匹配优先级 | `5`（所有指令） |
| 阻断后续匹配 | `False`（`block=False`，全部不阻断） |

### 全局处理前流程（所有指令共享）

每个指令处理前都会执行以下检查：

```
用户发送指令
    │
    ├─ 频率限制 ── 同一用户 5 秒内只能操作一次
    │                 ↓ 超限
    │            "⏱ 操作过于频繁，请 X 秒后再试。"
    │
    ├─ 输入校验（仅 qbind）── QQ 号格式：5-11 位纯数字，不能以 0 开头（`^[1-9]\\d{4,10}$`）
    │                            ↓ 不合法
    │                       "QQ 号格式不正确！请输入真实的 QQ 号（5-11 位数字）"
    │
    └─ 通过 → 执行后续逻辑
```

### 处理流程（二次确认机制）

为防止误绑定，采用**两步确认流程**：

#### 第一步：发起绑定（`qbind`）

1. 用户发送 `qbind <参数>`
2. 插件获取发送者的 QQ 号 (`event.get_user_id()`)
3. **参数检查**：如果参数为空 → 提示"请输入要绑定的参数！"
4. **输入校验**：正则 `^[a-zA-Z0-9_-]+$`，最长 64 字
5. 将绑定请求存入**待确认缓存**（key = `user:{QQ号}`），有效期 **120 秒**
6. 发送**二次确认消息**，附带两个按钮（**仅发起者可点击**）：
   - `✅ 确认绑定` → 触发 `confirm_bind` 指令
   - `❌ 取消绑定` → 触发 `cancel_bind` 指令

#### 第二步：确认/取消

| 操作 | 指令 | 行为 |
|------|------|------|
| 确认 | `confirm_bind` | 频率检查 → 检查缓存 → 检查超时 → 发送 Gensokyo API → 返回结果 |
| 取消 | `cancel_bind` | 清除缓存 → 提示已取消 |
| 超时 | — | 后台定时任务每分钟清理过期缓存 |

**confirm_bind 详细流程**：
1. 频率限制检查
2. 检查是否有待确认的绑定请求（无 → 提示错误）
3. 检查是否超时（超时 → 清除缓存，提示重新发起）
4. **调用 Gensokyo API**：`GET /getid?oldRowValue={session_id}&newRowValue={real_qq}&type=5`
5. **响应处理**：
   - 成功 → 绑定成功，保存持久化记录，清除缓存
   - 返回 `"已存在"` → 提示用户该 QQ 号已被绑定，引导先解绑
   - 返回其他 error → 显示后端错误信息，清除缓存
   - HTTP 错误 → 显示 HTTP 错误，清除缓存
   - 网络异常 → 显示连接失败提示，清除缓存

---

## 消息格式（Gensokyo Markdown + 按钮）

当前使用 `USE_MARKDOWN = True`，所有响应均采用 **Gensokyo Markdown** 格式。

### Markdown 语法

Gensokyo Markdown 支持以下格式：
- `### 标题` — 三级标题
- `> 内容` — 引用块
- `> - 列表项` — 引用内列表
- `` `代码` `` — 行内代码
- `**加粗**` — 加粗文本

### 按钮定义格式（扁平键名）

```python
# 最简形式
{"render_data.label": "按钮文字", "action.data": "指令内容"}
```

| 键名 | 类型 | 默认值 | 说明 |
|------|------|--------|------|
| `render_data.label` | str | 必填 | 按钮显示文字 |
| `render_data.visited_label` | str | 同 label | 点击后显示的文字 |
| `render_data.style` | int | 1 | 0=灰色线框, 1=蓝色线框 |
| `action.data` | str | 必填 | 指令文本或跳转 URL |
| `action.type` | int | 2 | 0=跳转, 1=回调, 2=指令 |
| `action.permission.type` | int | 2 | 0=指定用户, 1=仅管理者, 2=所有人 |
| `action.permission.specify_user_ids` | list/bool | [] | True=仅当前用户，或直接填 ID 列表 |
| `action.reply` | bool | False | 指令是否带引用回复 |
| `action.enter` | bool | False | 点击后直接发送指令 |
| `action.anchor` | int | 0 | 1=唤起选图器 |
| `action.unsupport_tips` | str | 默认提示 | 客户端不支持的提示 |

---

## 待确认绑定缓存

```python
_pending_binds: Dict[str, dict] = {}  # key: "user:{user_id}"
_PENDING_TIMEOUT = 120  # 有效期 120 秒

# 缓存结构
{
    "user:123456": {
        "value": str,           # 用户输入的绑定值
        "timestamp": float      # 发起绑定的时间戳
    }
}
```

- 缓存 key 为 `user:{QQ号}`，按用户隔离（不再含群号，支持跨群和私聊）
- 有效期 120 秒，超时后由后台定时任务自动清理（每分钟扫描一次）
- 确认/取消后立即从缓存中移除

### 定时清理机制（防内存泄漏）

```python
async def _cleanup_expired_pending():
    while True:
        await asyncio.sleep(60)
        now = time.time()
        expired = [k for k, v in list(_pending_binds.items())
                   if now - v["timestamp"] > _PENDING_TIMEOUT]
        for k in expired:
            _pending_binds.pop(k, None)
```

- 插件启动时通过 `@driver.on_startup` 自动启动后台任务
- 每分钟扫描一次，清理超时未确认的绑定请求
- 即使 Bot 长期运行也不会内存泄漏

---

## 强制绑定机制（全局拦截器）

插件通过 `run_preprocessor` 钩子实现全局消息拦截：

```python
@run_preprocessor
async def _global_bind_check(event: MessageEvent, bot: Bot):
    """全局预处理器：未绑定的用户只能使用 qbind 相关指令"""
    if not isinstance(event, MessageEvent):
        return
    session_id = event.get_user_id()
    if is_bound(session_id):
        return
    # 放行绑定相关指令
    text = event.get_plaintext().strip().split()[0] if event.get_plaintext().strip() else ""
    if text in {"qbind", "confirm_bind", "cancel_bind", "qunbind"}:
        return
    # 拦截并提示
    await bot.send(event, "🔒 请先完成绑定...")
    raise IgnoredException("用户未绑定，已拦截")
```

**拦截逻辑**：
- **已绑定用户** → 放行所有指令
- **未绑定用户** → 仅放行 `qbind` / `confirm_bind` / `cancel_bind` / `qunbind` 这四个指令
- **其他所有指令** → 拦截并发送「请先完成绑定」提示，抛出 `IgnoredException`
- **非消息事件**（如通知事件）→ 放行，不影响 Bot 正常运作

---

## 持久化绑定存储

```python
_BINDS_FILE = os.path.join(os.path.dirname(__file__), "binds.json")
_binds: Dict[str, str] = {}  # session_id → real_qq
```

绑定关系存储在插件同级目录的 `binds.json` 文件中：
```json
{
  "session_xxxx1": "123456789",
  "session_xxxx2": "987654321"
}
```

- 绑定成功后自动写入文件
- 解绑后自动从文件移除
- 插件加载时自动读取已有绑定
- Bot 重启不丢失绑定数据

---

## 导出函数（供其他插件使用）

其他插件通过以下方式导入使用：
```python
from plugins.qbind import is_bound, get_real_qq, ensure_bound
```

### `is_bound(session_id: str) → bool`
检查某个会话 ID 是否已完成绑定。

### `get_real_qq(session_id: str) → Optional[str]`
获取某个会话 ID 绑定的真实 QQ 号，未绑定返回 `None`。

### `ensure_bound(event: MessageEvent, bot: Bot) → bool`
在插件 handler 开头调用，未绑定时自动发送提示并返回 `False`。

**使用示例**：
```python
from plugins.qbind import ensure_bound

@matcher.handle()
async def handler(event: MessageEvent, bot: Bot):
    if not await ensure_bound(event, bot):
        return  # 未绑定，已发送提示，直接返回
    # 已绑定，继续处理...
```

---

## 安全机制

### 1. 频率限制（防滥用）

```python
_last_cmd_time: Dict[str, float] = {}
_RATE_LIMIT_SECONDS = 5
```

- 同一用户（按 QQ 号）**每 5 秒只能操作一次**
- 所有指令（`qbind`、`confirm_bind`、`cancel_bind`、`qunbind`）共享限流
- 超限时提示：`⏱ 操作过于频繁，请 X 秒后再试。`
- 防止恶意刷请求打爆后端 Gensokyo

### 2. 输入校验（防注入）

```python
def _validate_input(value: str) -> Optional[str]:
    if len(value) > 64:
        return "绑定参数过长，最多 64 个字符"
    if not re.match(r'^[a-zA-Z0-9_-]+$', value):
        return "绑定参数只能包含字母、数字、下划线和连字符"
    return None
```

- 只允许 `a-z`、`A-Z`、`0-9`、`_`（下划线）、`-`（连字符）
- 最长 64 个字符
- 防止特殊字符导致后端 SQL 注入或解析异常

### 3. 按钮权限控制

二次确认消息中的 `✅ 确认绑定` 和 `❌ 取消绑定` 按钮设置了：

```python
"action.permission.specify_user_ids": [user_id]  # 仅发起绑定的用户可点击
```

- 其他用户即使看到消息也无法点击按钮
- 防止恶意用户截胡确认

### 4. 解绑指令（防误绑无解）

提供 `qunbind` 指令，用户可主动解绑。同样需要二次确认：

```
qunbind → 确认消息 → qunbind 确认 → 调用 API 解绑
```

### 5. 定时清理缓存（防内存泄漏）

后台异步任务每分钟扫描一次，清除超时的待确认绑定记录。详见上方「待确认绑定缓存」章节。

---

## qunbind — 解绑指令

用户绑定错误或需要更换绑定值时，可使用 `qunbind` 解绑。

### 指令定义

| 属性 | 值 |
|------|-----|
| 指令名 | `qunbind` |
| 触发方式 | `qunbind`（确认）→ `qunbind 确认`（执行） |
| 响应范围 | 全局，支持群聊和私聊 |
| 优先级 | 5 |

### 处理流程

1. 用户发送 `qunbind`
2. 频率限制检查
3. 发送确认消息（按钮 `⚠️ 确认解绑`，仅发起者可点击）
4. 用户点击按钮 → 发送 `qunbind 确认`
5. 调用 Gensokyo API：`GET /getid?oldRowValue={session_id}&newRowValue=0&type=5`
6. 清除本地 `_binds` 持久化记录
7. 返回成功或失败提示

### 响应消息

**确认消息**：
```
### ⚠️ 确认解绑

> QQ：`{user_id}`

> 解绑后将无法使用 Bot 功能，需要重新绑定才能继续使用。

> 确认请发送：`qunbind 确认`
```
**按钮**：`[⚠️ 确认解绑]`

**解绑成功**：
```
### ✅ 已解绑

> QQ：`{user_id}`

> 账号已解绑，需要重新使用请发送 `qbind <绑定值>`。
```
**按钮**：`[重新绑定]`

**解绑失败（API错误）**：
```
### ❌ 解绑失败

> {错误信息}
```
**按钮**：`[重试解绑]`

**解绑失败（HTTP错误）**：
```
### ❌ 解绑失败

> 请求后端 API 时发生错误（HTTP {status_code}）
> 请稍后重试或联系管理员
```
**按钮**：`[重试解绑]`

**解绑失败（网络错误）**：
```
### ❌ 解绑失败

> 无法连接到后端 API（Gensokyo）
> 请确保后端服务正在运行

> 错误详情：`{e}`
```
**按钮**：`[重试解绑]`

---

## 响应消息内容

### 0. 频率限制（所有指令通用）
```
### ⏱ 操作过于频繁

> 请 {N} 秒后再试。
```
> 无按钮，纯提示

### 1. 参数为空时
```
### ❌ 绑定失败

> 请输入要绑定的参数！

> 格式：`qbind <绑定值>`
```
**按钮**：`[发起绑定]`

### 1b. 输入不合法
```
### ❌ 绑定失败

> 绑定参数只能包含字母、数字、下划线和连字符

> 格式：`qbind <绑定值>`
```
**按钮**：`[重新绑定]`

### 2. 二次确认消息（发起绑定后）
```
### ⚠️ 确认绑定

> QQ：`{user_id}`
> 绑定值：`{user_input}`

> ⏰ 请在两分钟内确认，超时自动取消
> 请确认无误后再点击「确认绑定」，误绑可能导致账号异常！
```
**按钮**：`[✅ 确认绑定]` `[❌ 取消绑定]`

### 3. 无待取消请求
```
### ❌ 无待取消请求

> 你当前没有待取消的绑定请求。

> 请先发送 `qbind <绑定值>` 发起绑定。
```
**按钮**：`[发起绑定]`

### 3b. 取消绑定成功
```
### ✅ 已取消绑定

> 绑定请求已取消，你可以随时重新发送 `qbind <绑定值>` 发起新的绑定。
```

### 4. 无待确认请求
```
### ❌ 无待确认请求

> 你当前没有待确认的绑定请求。

> 请先发送 `qbind <绑定值>` 发起绑定。
```
**按钮**：`[发起绑定]`

### 5. 确认超时
```
### ⏰ 确认已超时

> 绑定确认已超时（2分钟）。

> 请重新发送 `qbind <绑定值>` 发起绑定。
```
**按钮**：`[重新绑定]`

### 6. 绑定成功
```
### ✅ 绑定成功

> QQ：`{user_id}`
> 绑定值：`{user_input}`

> 绑定完成，现在你可以正常使用 Bot 的其他功能了！
```
**按钮**：`[查看帮助]`

### 3. 绑定失败（后端返回 error）
```
### ❌ 绑定失败

> {错误信息}

> 请检查参数后重试，格式：`qbind <绑定值>`
```
**按钮**：`[重新绑定]`

### 4. HTTP 错误
```
### ❌ 绑定失败

> 请求后端 API 时发生错误（HTTP {status_code}）
> 请稍后重试或联系管理员
```
**按钮**：`[重新绑定]`

### 5. 网络错误（无法连接后端）
```
### ❌ 绑定失败

> 无法连接到后端 API（Gensokyo）
> 请确保后端服务正在运行

> 错误详情：`{e}`
```
**按钮**：`[重新绑定]`

---

## 代码结构

```
qbind/
├── __init__.py       # 插件主逻辑 + Gensokyo Markdown 构建工具 + 安全机制
├── lib_msg.py        # [参考] 统一消息构建模块（其他插件使用的消息函数）
└── agent.md          # 本文档 — 插件说明文件
```

### `__init__.py` 内部数据

| 项目 | 类型 | 说明 |
|------|------|------|
| `USE_MARKDOWN` | `bool` | Gensokyo Markdown 开关（默认 `True`） |
| `_binds` | `Dict[str, str]` | 持久化绑定记录 `session_id → real_qq` |
| `_BINDS_FILE` | `str` | 绑定 JSON 文件路径 `binds.json` |
| `_pending_binds` | `Dict[str, dict]` | 待确认绑定缓存，key=`user:{QQ号}` |
| `_PENDING_TIMEOUT` | `int` | 确认有效期（120 秒） |
| `_last_cmd_time` | `Dict[str, float]` | 频率限制记录，key=`QQ号` |
| `_RATE_LIMIT_SECONDS` | `int` | 频率限制间隔（5 秒） |

### `__init__.py` 内部函数

| 函数 | 说明 |
|------|------|
| `_normalize_button(btn)` | 将扁平键名字典转换为嵌套字典 |
| `_build_button_row(buttons)` | 构建单行按钮（补全默认值） |
| `_build_keyboard(buttons_config)` | 构建完整键盘布局（多行按钮） |
| `_build_markdown_msg(content, buttons)` | 构建完整的 Gensokyo Markdown 消息（含可选按钮） |
| `_check_rate_limit(uid)` | 频率限制检查，返回 `None` 或等待秒数 |
| `_pending_key(user_id)` | 生成缓存 key `user:{QQ号}` |
| `_validate_qq(value)` | QQ 号格式校验（5-11 位数字） |
| `_build_api_url(session_id, real_qq)` | 构建绑定 API URL：`type=5` + 会话ID + QQ号 |
| `_build_unbind_api_url(session_id)` | 构建解绑 API URL：`type=5` + 会话ID + `newRowValue=0` |
| `_cleanup_expired_pending()` | 后台任务：每分钟清理超时缓存 |
| `_load_binds()` | 从 JSON 文件加载持久化绑定 |
| `_save_binds()` | 将绑定保存到 JSON 文件 |

### `__init__.py` 导出函数（供其他插件导入）

| 函数 | 说明 |
|------|------|
| `is_bound(session_id)` | 检查会话是否已绑定 → `bool` |
| `get_real_qq(session_id)` | 获取真实 QQ 号 → `Optional[str]` |
| `ensure_bound(event, bot)` | 未绑定则发提示并返回 `False` |

### `__init__.py` 指令处理器

| 指令变量 | 命令名 | 优先级 | 说明 |
|----------|--------|--------|------|
| `bind` | `qbind` | 5 | 发起绑定请求，存入缓存并发送二次确认 |
| `confirm` | `confirm_bind` | 5 | 确认绑定，执行实际 API 请求 |
| `cancel_cmd` | `cancel_bind` | 5 | 取消绑定，清除缓存 |
| `unbind` | `qunbind` | 5 | 解绑指令（两步确认，调用 API 清除记录） |

### `__init__.py` 全局钩子

| 钩子 | 说明 |
|------|------|
| `_global_bind_check` | `run_preprocessor` — 在所有 matcher 前拦截未绑定用户 |
| `_cleanup_expired_pending()` | 后台定时任务，每分钟清理过期确认缓存 |
| `_start_cleanup()` | `@driver.on_startup` — 启动时创建清理任务 |

### `lib_msg.py` 参考（其他插件使用）

### `lib_msg.py` 参考（其他插件使用）

`lib_msg.py` 提供了更多高级封装（位于 `qbind/` 目录下，供参考）：
- `send_message()` — 统一发送接口
- `_sorted_markdown_segment()` — 自动排序按钮
- `msg_help_main()` — 主帮助菜单
- `get_disclaimer_message()` — 服务条款消息

---

## 注意事项

1. **全局无限制响应**：不检测 `bot.self_id`，任何 Bot 实例均可响应 `qbind` 指令
2. **前置依赖**：用户必须先完成绑定，才能正常使用其他 Bot 功能
3. **后端依赖**：必须确保 Gensokyo 后端（`http://127.0.0.1:15630/`）正常运行
4. **消息格式**：推荐使用 Markdown 模式（`USE_MARKDOWN = True`）获得最佳展示效果
5. **按钮交互**：定义按钮时优先使用扁平键名格式，兼容性和可读性更好
6. **错误处理**：所有 API 请求均已在 `try-except` 中捕获，网络异常会给出友好提示

---

## 变更记录

### 2026-05-24 — 修正 API 调用方式（直接用 type=5，取消 type=1 两步流程）

**问题**：`type=1` 接口对所有用户都返回 `"unable to find a unique row ID"` HTTP 500

**根因**：Gensokyo 的 `type=1` 接口不可用，但 `type=5` **可以直接用虚拟会话 ID 调用**，无需先获取 row ID。

**修改**：
1. `confirm_handler` 直接调用 `type=5` + 虚拟会话 ID + QQ 号
2. `unbind_handler` 直接调用 `type=5` + 虚拟会话 ID + `newRowValue=0`
3. 移除所有 `type=1` 相关代码
4. 更新 `_build_api_url` 和 `_build_unbind_api_url` 为正确签名

### 2026-05-24 — 修复按钮格式错误

**问题**：`_build_markdown_msg` 报 `AttributeError: 'str' object has no attribute 'items'`

**根因**：按钮参数应为 `List[List[Dict]]`（行列表），传成了 `List[Dict]`（扁平列表）

**修改**：所有 `[btn]` 改为 `[[btn]]` 格式

### 2026-05-23（第二次）— 修复 `_build_api_url` 参数错误

**问题**：`confirm_bind` 时 Gensokyo 返回 HTTP 500（`{"error":"不存在:{QQ号}"}`）

**根因**：`_build_api_url()` 将 `oldRowValue` 和 `newRowValue` 都设为用户输入的 QQ 号，但 Gensokyo 要求 `oldRowValue` 必须是虚拟会话 ID。

**修改**：
1. `_build_api_url(session_id, real_qq)` — 将 `oldRowValue` 改为传会话虚拟 ID
2. `_build_unbind_api_url(session_id, real_qq)` — 同上
3. `confirm_handler` 和 `unbind_handler` 传入正确参数

### 2026-05-23 — 修复按钮权限问题

**问题**：确认/取消绑定按钮显示"无权限操作"

**根因**：按钮 `permission.type` 默认 `2`（所有人）与 `specify_user_ids` 冲突

**修改**：显式设置 `action.permission.type: 0`（指定用户模式），使权限声明一致
