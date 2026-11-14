# 百度网盘自动转存

> **声明**：本项目是使用 Cursor AI 辅助编写的。本人是一名剪辑师，非专业程序员，精力有限。如遇到使用问题，请先查阅文档并尝试自行解决，谢谢理解。

一个基于 Flask + Vue 3 的百度网盘自动转存系统，支持多用户管理、定时任务调度、订阅同步和通知推送。

## 主要特性

- 🔄 自动转存：支持自动转存百度网盘分享链接到指定目录
- 👥 多用户管理：支持添加多个百度网盘账号
- ⏰ 定时任务：支持全局定时和单任务定时规则
- 📥 订阅同步：自动读取「订阅分享管理」中的历史订阅，一键批量补全任务
- 📱 消息推送：支持 25+ 种通知方式和自定义 WEBHOOK
- 🎯 任务分类：支持对任务进行分类管理
- 📊 状态监控：实时显示任务执行状态和进度
- 🔍 智能去重：自动跳过已转存的文件
- 💾 容量监控：监控网盘容量并在超过阈值时发送通知
- 📋 链接复制：支持一键复制分享链接到剪贴板
- 🤖 智能填充：自动获取分享链接的文件夹名称并填充任务名称
- 🔍 正则处理：支持文件过滤和重命名的正则表达式功能
- 🎨 美观界面：Vue 3 + Element Plus，响应式设计，支持移动端访问

## 系统要求

- **Python 3.10**（`baidupcs-py==0.7.6` 官方声明支持 `>=3.7,<3.11`，推荐使用 3.10；3.11 实测也可运行）
- **Node.js 18+**（仅构建/开发前端时需要）
- **uv**（Python 包与环境管理）
- Windows / Linux / macOS

## 项目结构

```
baidu-autosave/
├── config/
│   ├── config.json              # 运行时配置（自动生成，不提交）
│   └── config.template.json     # 配置模板
├── frontend/                    # Vue 3 前端源码
│   ├── src/
│   ├── public/
│   ├── package.json
│   └── vite.config.ts
├── frontend-new/                # API 接口文档
├── static/                      # 前端构建产物（由 frontend/dist 复制而来，不提交）
├── log/                         # 运行日志（自动生成）
├── web_app.py                   # Web 应用主程序（Flask + gevent）
├── storage.py                   # 网盘存储与配置管理
├── scheduler.py                 # 定时任务调度
├── notify.py                    # 通知推送
├── utils.py                     # 通用工具函数
├── pyproject.toml               # uv 项目依赖定义
├── requirements.txt             # pip 依赖清单（可选）
├── Dockerfile
└── docker-compose.yml
```

> ⚠️ **重要**：`static/` 目录被 `.gitignore` 忽略，克隆仓库后里面是空的。**必须先构建前端**，否则访问 `http://localhost:5656` 只会看到空白页或 404。

## 安装与运行（本地部署）

### 1. 克隆仓库

```bash
git clone https://github.com/LittleOrange-i/baidu-autosave.git
cd baidu-autosave
```

### 2. 安装 uv（已安装可跳过）

```bash
# Windows (PowerShell)
powershell -ExecutionPolicy ByPass -c "irm https://astral.sh/uv/install.ps1 | iex"

# macOS / Linux
curl -LsSf https://astral.sh/uv/install.sh | sh
```

### 3. 安装后端依赖

```bash
# 自动创建 .venv 并按 pyproject.toml / uv.lock 安装依赖（已配置阿里云镜像源）
uv sync

# 如需指定 Python 版本（推荐 3.10）
uv sync --python 3.10
```

> 也可继续使用传统方式：`uv venv --python 3.10` 然后 `uv pip install -r requirements.txt`。

### 4. 构建前端（关键步骤，不可跳过）

在**项目根目录**执行，一行命令完成「安装依赖 → 构建 → 复制到 `static/`」，Windows / macOS / Linux 通用（PowerShell 5.1 也支持）：

```bash
npm --prefix frontend run build:static
```

执行完在项目根目录生成 `static/`（含 `index.html`、`assets/`、`favicon/`），后端才能正常返回页面。

验证：

```powershell
Test-Path static\index.html        # Windows PowerShell，应返回 True
ls static/index.html               # macOS / Linux
```

### 5. 启动应用

```bash
uv run python web_app.py
```

看到以下输出表示启动成功：

```
Server started at http://0.0.0.0:5656
```

### 6. 访问并登录

浏览器打开 <http://localhost:5656>，使用默认管理员账号登录：

| 账号 | 密码 |
| --- | --- |
| `admin` | `admin123` |

> **安全提示**：首次登录后请立即在 **系统设置 → 修改密码** 中修改默认密码。
> 登录成功后密码会自动以哈希形式保存到 `config/config.json` 的 `auth.password` 字段。

**修改账号/密码的两种方式：**

1. Web 界面：登录后进入「系统设置」→「修改密码」，需输入旧密码。
2. 直接编辑配置：修改 `config/config.json` 中的 `auth` 段（明文口令会在首次登录校验通过后自动升级为哈希）：

```json
"auth": {
    "users": "admin",
    "password": "admin123",
    "session_timeout": 3600
}
```

- `users`：登录用户名
- `password`：登录密码（支持明文或哈希）
- `session_timeout`：会话超时时间（秒），默认 3600

**端口修改**：通过环境变量覆盖，默认 5656。

```powershell
# Windows PowerShell
$env:WEB_APP_PORT="8080"; uv run python web_app.py
```

```bash
# macOS / Linux
WEB_APP_PORT=8080 uv run python web_app.py
```

## 开发模式（前端热更新）

生产部署用上面的构建方式即可；需要改前端代码时使用开发模式，需**同时启动两个服务**。

### 1. 启动后端

```bash
uv run python web_app.py
# 后端：http://localhost:5656
```

### 2. 启动前端开发服务器

```bash
cd frontend
npm install        # 首次运行
npm run dev
# 前端：http://localhost:3001
```

> 端口以 [frontend/vite.config.ts](frontend/vite.config.ts) 中的 `server.port` 为准，当前为 **3001**。
> `/api`、`/login`、`/logout` 请求会由 Vite 自动代理到后端 5656 端口。

开发时访问 <http://localhost:3001>；改动前端代码自动热更新，改动后端代码需手动重启。

## Docker 部署

Dockerfile 会自动完成前端构建并放入 `static/`，无需手动构建。

### 使用 docker-compose（推荐）

1. 创建 `docker-compose.yml`：

```yaml
services:
  baidu-autosave:
    image: kokojacket/baidu-autosave:latest
    container_name: baidu-autosave
    restart: unless-stopped
    ports:
      - "5656:5656"
    volumes:
      - ./config:/app/config
      - ./log:/app/log
    environment:
      - TZ=Asia/Shanghai
```

2. 创建必要目录并启动：

```bash
mkdir -p config log
docker-compose up -d
docker-compose logs -f
```

3. 访问 <http://localhost:5656>，默认账号 `admin` / `admin123`。

### 使用 Docker CLI

```bash
mkdir -p config log

docker run -d \
  --name baidu-autosave \
  --restart unless-stopped \
  -p 5656:5656 \
  -v $(pwd)/config:/app/config \
  -v $(pwd)/log:/app/log \
  -e TZ=Asia/Shanghai \
  kokojacket/baidu-autosave:latest
```

## 使用说明

### 1. 添加用户

1. 登录百度网盘网页版
2. 按 F12 打开开发者工具获取 Cookies
3. 在系统「用户管理」中添加用户，填入 Cookies

### 2. 添加任务

1. 点击「添加任务」按钮
2. 填写任务信息：
   - 任务名称（可选，支持自动获取分享链接的文件夹名称）
   - 分享链接（必填，输入完成后会自动获取文件夹名称）
   - 保存目录（必填，会根据任务名称智能同步）
   - 定时规则（可选）
   - 分类（可选）
   - 过滤表达式（可选，正则表达式过滤需要转存的文件）
   - 重命名表达式（可选，正则表达式重命名转存的文件）

**智能功能说明：**

- **自动填充任务名称**：输入分享链接后，系统会自动获取分享内容的文件夹名称并填充到任务名称
- **保存目录同步**：任务名称变化时会自动更新保存目录的最后一级文件夹名
- **编辑检测**：一旦手动编辑保存目录，当前任务将停止自动同步；新建任务时重新启用
- **分享链接复制**：任务创建后，可在任务列表中一键复制分享链接
- **正则文件过滤**：使用正则表达式筛选需要转存的文件，如 `^(\d+)\.mp4$` 只转存以数字开头的 mp4 文件
- **正则文件重命名**：使用正则表达式重命名转存的文件，如 `第\1集.mp4` 将匹配的数字替换为「第X集」格式

**⚠️ 重命名功能注意事项：**
百度网盘 API 对重命名操作有严格的频率限制，频繁重命名可能导致失败或触发风控。建议优先通过调整分享源文件名来满足需求，谨慎使用重命名功能。

### 3. 定时设置

- 全局定时规则：适用于所有未设置自定义定时的任务
- 单任务定时：可为每个任务设置独立的定时规则
- 定时规则使用 cron 表达式，例如：
  - `*/5 * * * *`：每 5 分钟执行一次
  - `0 */1 * * *`：每小时执行一次
  - `0 8,12,18 * * *`：每天 8 点、12 点、18 点执行

### 4. 订阅同步

在「订阅同步」页面开启后，可自动读取百度网盘「订阅分享管理」中的历史订阅并批量创建任务：

- 可设置自动同步开关与同步频率
- 可指定保存位置（每个订阅保存到「保存位置 / 订阅名」）
- 已转存文件自动跳过，只补新文件
- 支持失效提醒，接口异常时通过已配置的通知渠道告知

### 5. 通知设置

系统支持 25+ 种通知方式，包括但不限于：

- **PushPlus**：访问 [pushplus.plus](http://www.pushplus.plus) 获取 Token
- **Bark**：iOS 平台推送服务
- **钉钉机器人**、**飞书机器人**、**企业微信**
- **Telegram**、**SMTP 邮件**
- **自定义 WEBHOOK**：支持任意 HTTP 接口推送
- **其他**：Gotify、iGot、ServerJ、PushDeer 等

**配置方法：**

1. 在「系统设置」的「通知设置」部分启用通知功能
2. 添加对应通知服务的字段配置（如 `PUSH_PLUS_TOKEN`）
3. WEBHOOK 配置支持「添加 WEBHOOK 配置」按钮一键添加全部字段
4. 点击「测试通知」验证配置是否正确

**WEBHOOK 配置说明：**

- `WEBHOOK_BODY` 支持简化输入格式：`title: "$title"content: "$content"source: "项目名"`
- 保存时系统会自动转换为标准多行格式，无需手动处理换行符和转义
- 支持自定义 HTTP 方法、请求头和请求体格式

**通知延迟合并：**

1. 多个任务在短时间内执行完成时，通知会合并为一条发送
2. 可在「通知设置」中配置延迟时间（默认 30 秒，设为 0 关闭合并）
3. 容量警告等重要通知不会被合并，会立即发送

### 6. 网盘容量监控

1. 在系统设置中启用「网盘容量提醒」
2. 设置容量提醒阈值（默认 90%）
3. 设置检查时间（默认每天 00:00）
4. 超过阈值时通过已配置的通知渠道发送警告

## 配置文件说明

`config/config.json` 主要配置项（示例）：

```json
{
    "baidu": {
        "users": {},          // 用户信息
        "current_user": "",   // 当前用户
        "tasks": []           // 任务列表
    },
    "retry": {
        "max_attempts": 3,    // 最大重试次数
        "delay_seconds": 5    // 重试间隔
    },
    "cron": {
        "default_schedule": [   // 默认定时规则，支持多个 cron 表达式
            "0 10 * * *"
        ],
        "auto_install": true    // 自动启动定时
    },
    "notify": {
        "enabled": true,      // 启用通知
        "notification_delay": 30, // 通知延迟合并时间（秒），设置为 0 禁用合并
        "direct_fields": {     // 直接映射到通知库的字段（推荐）
            "PUSH_PLUS_TOKEN": "",
            "PUSH_PLUS_USER": "",
            "BARK_PUSH": "",
            "DD_BOT_TOKEN": "",
            "DD_BOT_SECRET": "",
            "TG_BOT_TOKEN": "",
            "TG_USER_ID": "",
            "SMTP_SERVER": "",
            "SMTP_EMAIL": "",
            "SMTP_PASSWORD": "",
            "WEBHOOK_URL": "",
            "WEBHOOK_METHOD": "POST",
            "WEBHOOK_CONTENT_TYPE": "application/json",
            "WEBHOOK_HEADERS": "Content-Type: application/json",
            "WEBHOOK_BODY": "title: \"$title\"\ncontent: \"$content\"\nsource: \"项目名\""
        },
        "custom_fields": {}
    },
    "quota_alert": {
        "enabled": true,          // 是否启用容量提醒
        "threshold_percent": 90,  // 容量提醒阈值百分比
        "check_schedule": "0 0 * * *" // 检查时间
    },
    "scheduler": {
        "max_workers": 1,         // 最大工作线程数
        "misfire_grace_time": 3600, // 错过执行的容错时间
        "coalesce": true,         // 合并执行错过的任务
        "max_instances": 1        // 同一任务的最大并发实例数
    },
    "share": {
        "default_password": "1234", // 默认分享密码
        "default_period_days": 7    // 分享有效期（天）
    },
    "subscription": {
        "enabled": false,         // 启用订阅同步
        "auto_sync": false,       // 自动同步
        "schedule": "0 * * * *",  // 同步频率
        "session_id": "",
        "save_dir_prefix": "/订阅",
        "max_pages": 10,
        "alert_enabled": true
    },
    "auth": {
        "users": "admin",         // 登录用户名
        "password": "admin123",   // 登录密码（登录后自动升级为哈希）
        "session_timeout": 3600   // 会话超时（秒）
    }
}
```

## 常见问题

### 1. 访问 5656 端口页面空白 / 404 / 只有加载动画

**原因**：没有构建前端，或构建产物没有复制到 `static/`。

**解决**：按「安装与运行 → 第 4 步」，在项目根目录执行一行命令：

```bash
npm --prefix frontend run build:static
```

然后重启后端。

可以用下面命令确认 `static/` 是否有内容：

```powershell
Test-Path static\index.html   # 应返回 True
```

### 2. 无法登录 / 忘记密码

- 默认账号：`admin`，默认密码：`admin123`
- 自定义过的请查看 `config/config.json` 的 `auth` 段
- 忘记密码时，停止服务后把 `auth.password` 改回明文 `admin123`，重启即可登录，之后再在界面中修改
- 若提示「用户名或密码错误」，注意 `auth.users` 是**用户名**而非中文昵称

### 3. 前端开发服务器端口

实际端口为 **3001**（见 `frontend/vite.config.ts`），不是 3000。若被占用，改 `server.port` 即可。

### 4. 任务执行失败

- 检查分享链接是否有效、提取码是否正确
- 确认百度网盘账号 Cookie 未过期（在用户管理中切换/更新 Cookie）
- 查看 `log/web_app_*.log` 了解详细原因

### 5. 定时任务不执行

- 确认 cron 表达式格式正确
- 检查系统时间是否准确（容器内注意 `TZ=Asia/Shanghai`）
- 查看调度器日志

### 6. 通知推送失败

- 验证 Token / 设备码等配置是否正确，使用「测试通知」功能验证
- 检查网络连接和防火墙设置
- 查看日志中的具体错误信息

### 7. WEBHOOK 配置问题

- 使用「添加 WEBHOOK 配置」按钮自动添加所有必需字段
- `WEBHOOK_BODY` 支持简化格式输入，系统会自动转换
- 确认目标服务器能接收 JSON 格式的 POST 请求，且 `WEBHOOK_URL` 可访问

### 8. 通知未合并

- 确认通知延迟时间设置合理（默认 30 秒）
- 任务执行间隔超过延迟时间时不会合并
- 容量警告等重要通知不会合并

### 9. 正则表达式使用问题

- 确认语法正确，可先用在线正则测试工具验证
- 过滤表达式用于筛选文件，重命名表达式用于重命名
- 示例：过滤 `\.mp4$` 只转存 mp4 文件，重命名 `(.+)\.mp4$` → `\1_renamed.mp4`

### 10. uv 相关错误

- `error: No pyproject.toml found`：请确认当前目录是项目根目录（存在 `pyproject.toml`），再执行 `uv sync`
- 依赖安装慢：`pyproject.toml` 已默认使用阿里云镜像，可自行调整 `[[tool.uv.index]]`
- Python 版本不符：使用 `uv sync --python 3.10`

## 开发说明

### 主要模块

- **web_app.py**：Web 应用核心，处理 HTTP 请求、认证与 SPA 路由
- **storage.py**：管理百度网盘 API 调用、用户与任务数据存储
- **scheduler.py**：处理定时任务的调度和执行
- **notify.py**：实现各种通知方式
- **utils.py**：提供通用工具函数

### 前后端交互

- 前端为 Vue 3 SPA，构建后由后端从 `static/` 目录提供
- 前端使用**轮询**获取任务状态（未启用 WebSocket）
- 主要接口前缀：`/api/tasks`、`/api/task/*`、`/api/users`、`/api/user/*`、`/api/config*`、`/api/auth/*`、`/api/subscriptions/*`、`/api/notify/*`

## 更新日志

### v1.1.5
- 版本号改为从 `frontend/package.json` 自动注入
- 新增 `pyproject.toml` / `uv.lock`，支持 `uv sync` 一键安装依赖
- 文档补充前端构建步骤、默认管理员账号与端口说明

### v1.1.3
- 修复移动端任务卡片样式不一致的问题，统一间距、字体和对齐方式
- 修复移动端分享链接按钮换行和被截断的布局问题
- 优化 PC 端分享链接显示，恢复可点击链接和内联复制按钮功能
- 移除版本信息获取失败时的错误通知，改为静默处理
- 改进响应式设计，确保移动端和桌面端的一致性体验

### v1.1.2
- 修复正则表达式字段二次编辑保存失败的问题
- 修复定时任务运行时手动执行任务可能出现提取码错误的并发问题
- 优化分享链接失效时的错误信息显示，将复杂的 API 错误转换为用户友好的提示
- 改进错误处理机制，提升用户体验

### v1.1.1
- 新增 WEBHOOK 配置自动格式化功能，简化用户配置过程
- 添加「添加 WEBHOOK 配置」快速按钮，一键添加所有必需的 WEBHOOK 字段
- `WEBHOOK_BODY` 字段支持简化输入格式，系统会自动转换为标准多行格式
- 后端自动处理 `WEBHOOK_BODY` 字段的格式转换，确保兼容性

### v1.1.0
- 新增正则表达式功能，支持文件过滤和重命名处理
- 添加通知延迟合并功能，解决同一时间多个任务执行时发送多条通知的问题
- 添加通知延迟时间配置选项到系统设置页面
- 添加分享链接一键复制功能
- 优化任务名称与保存目录单向同步逻辑，支持智能编辑检测
- 完善通知功能集成，支持 25+ 种通知方式和自定义 WEBHOOK
- 添加重命名频率限制提醒，避免触发百度网盘 API 风控

### v1.0.9
- 添加网盘容量监控功能，支持在容量超过阈值时发送通知
- 优化前端界面交互体验
- 修复配置加载和保存的问题

### v1.0.2
- 修复任务编辑时提取码丢失的问题
- 优化任务更新逻辑，确保密码信息正确保存
- 改进任务调度更新机制

### v1.0.1
- 优化界面交互体验，修复已知问题，提升系统稳定性

### v1.0.0
- 初始版本发布
- 实现基本功能：任务管理、定时执行、通知推送

## 许可证

MIT License

## 致谢

- [Flask](https://flask.palletsprojects.com/)
- [Vue 3](https://vuejs.org/) / [Element Plus](https://element-plus.org/)
- [APScheduler](https://apscheduler.readthedocs.io/)
- [baidupcs-py](https://github.com/PeterDing/BaiduPCS-Py)
- [quark-auto-save](https://github.com/Cp0204/quark-auto-save) - 夸克网盘自动转存项目，提供了很好的参考