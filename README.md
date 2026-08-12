# Odoo 19

Odoo 19 源码项目，基于 Python 开发的企业管理软件套件。

## 环境要求

- Python >= 3.10
- Docker Desktop（提供 PostgreSQL 数据库）
- Windows / Linux / macOS

## 快速启动

### 1. 启动 PostgreSQL 容器

```powershell
docker run -d `
  --name odoo19-pg `
  -e POSTGRES_USER=odoo `
  -e POSTGRES_PASSWORD=odoo `
  -p 5432:5432 `
  postgres:17-alpine
```

> 如果 5432 端口被占用，将 `-p 5432:5432` 改为 `-p 5434:5432`，启动 Odoo 时加上 `--db_port 5434`。

### 2. 创建虚拟环境并安装依赖

```powershell
python -m venv venv
.\venv\Scripts\Activate.ps1
pip install -r requirements.txt
```

### 3. 启动 Odoo

```powershell
python odoo-bin -d odoo19 --db_user odoo --db_password odoo --dev reload
```

首次启动会自动创建 `odoo19` 数据库并安装 base 模块。

### 4. 访问

浏览器打开 `http://localhost:8069`，首次需创建管理员账号。

> 如果已有 `admin` 用户但忘记密码，执行：
> ```powershell
> docker exec odoo19-pg psql -U odoo -d odoo19 -c "UPDATE res_users SET password='admin' WHERE login='admin';"
> ```

## 项目结构

```
odoo19/
├── odoo-bin          # 启动入口
├── odoo/             # 核心源码
│   ├── cli/          # CLI 命令（server, shell, db 等）
│   ├── http.py       # WSGI 应用入口
│   ├── modules/      # 模块管理
│   ├── orm/          # ORM 实现
│   ├── service/      # 服务层（HTTP/Cron/进程管理）
│   └── tools/        # 工具库（配置、日志等）
├── addons/           # 内置模块（crm, sale, hr, mrp 等）
├── setup.py          # pip 安装配置
└── requirements.txt  # Python 依赖
```

## 常用命令

```powershell
# 安装/更新模块
python odoo-bin -d odoo19 --db_user odoo --db_password odoo -i sale,crm
python odoo-bin -d odoo19 --db_user odoo --db_password odoo -u sale

# 交互式 shell
python odoo-bin shell -d odoo19 --db_user odoo --db_password odoo

# 运行测试
python odoo-bin -d odoo19 --db_user odoo --db_password odoo --test-enable --stop-after-init
```

## PostgreSQL 容器管理

```powershell
# 查看数据库列表
docker exec odoo19-pg psql -U odoo -d postgres -c "\l"

# 停止/启动容器
docker stop odoo19-pg
docker start odoo19-pg
```

## 启动流程

详见 [odoo-bin](odoo-bin) → [odoo/cli/command.py](odoo/cli/command.py) → `main()`：

```
python odoo-bin
  ├── import odoo.init              # 版本检查、GC、猴子补丁
  ├── odoo.cli.main()               # CLI 命令分发
  │     └── 默认命令 'server'
  ├── config.parse_config()         # 加载 CLI/环境变量/配置文件
  ├── netsvc.init_logger()          # 初始化日志
  ├── server.start()
  │     ├── load_server_wide_modules()    # ['base', 'rpc', 'web']
  │     ├── import odoo.http              # WSGI 应用 root = Application()
  │     └── server.run()
  │           ├── ThreadedServer           # 默认模式
  │           ├── preload_registries()     # Registry.new() 加载模块
  │           ├── cron_spawn()             # 定时任务线程
  │           └── HTTP 服务就绪，进入事件循环
```
