# Python 后端工程化学习路线：FastAPI + Uvicorn + MySQL + Redis

## 结论

最适合你的学习路径不是先把 FastAPI、MySQL、Redis 各看一遍，而是围绕一个小型“待办事项 API”逐步加能力：

```text
Python 基础
  -> Web / HTTP 基础
  -> FastAPI 写接口
  -> Uvicorn 运行服务
  -> MySQL 持久化数据
  -> Redis 做缓存与限流
  -> 测试、日志、配置、Docker
```

最终目标：独立写出并部署一个具备注册登录、待办 CRUD、MySQL 持久化、Redis 缓存和限流、自动化测试的后端服务。

> 学习原则：每学一个组件，都让项目多解决一个真实问题。不要先背完全部概念再动手。

---

## 0. 先准备环境（第 0 天）

### 0.1 推荐组合

| 工具 | 推荐选择 | 用途 |
| --- | --- | --- |
| Python | 3.12 或 3.13 | 学习和运行服务 |
| 包管理 | `uv` | 创建虚拟环境、安装依赖，速度快 |
| 编辑器 | VS Code + Python 扩展 | 编写和调试 |
| 数据库 | Docker 中的 MySQL 8 | 避免污染本机环境 |
| 缓存 | Docker 中的 Redis 7 | 学习缓存、限流 |
| API 调试 | Bruno、Postman 或 VS Code REST Client | 调接口 |
| 版本管理 | Git | 提交代码、回退改动 |

Windows 下建议使用 **WSL2 + Docker Desktop**。如果暂时不想接触 WSL，也可以直接在 PowerShell 中运行 Python；但后面 Docker、Linux 命令和部署会更顺手。

### 0.2 验收命令

```bash
python --version
uv --version
git --version
docker --version
docker compose version
```

以上命令都能返回版本号，就可以进入下一阶段。

### 0.3 建立学习仓库

```bash
mkdir python-todo-api
cd python-todo-api
git init
uv init
uv venv
```

建议从第一天开始提交：每完成一个阶段，执行一次 `git add . && git commit -m "feat: 完成 xxx"`。出错时能回到上一个可运行版本，不会把项目炖成一锅代码粥。

---

## 1. Python 基础：先能写出干净的小程序（第 1～2 周）

### 1.1 必学清单

按这个顺序学，不需要一开始钻进元类、描述器等深水区：

1. 基础语法：变量、字符串、数字、`if`、`for`、`while`。
2. 常用容器：`list`、`dict`、`set`、`tuple`，以及推导式。
3. 函数：参数、返回值、默认参数、关键字参数、作用域。
4. 异常：`try / except / finally`，知道异常不能悄悄吞掉。
5. 文件与 JSON：读写文件、`json.loads` / `json.dumps`。
6. 模块和包：导入、自定义包、`__init__.py`。
7. 类型标注：`str`、`int`、`list[str]`、`dict[str, int]`、`str | None`。
8. 面向对象：类、实例、继承；重点理解“数据和行为放在一起”。
9. 异步基础：理解 `async` / `await` 的用途，不急着研究事件循环细节。

### 1.2 每个知识点都要做的小练习

用纯 Python 完成一个命令行待办程序：

```text
1. 添加待办：标题、截止时间、是否完成
2. 查看待办：支持按完成状态筛选
3. 完成待办
4. 删除待办
5. 将数据保存到 todos.json，重启后仍存在
```

### 1.3 阶段验收

- 能解释 `list` 和 `dict` 各适合存什么数据。
- 能把重复逻辑提取成函数。
- 能用类型标注描述函数输入和输出。
- 程序遇到非法输入不会直接崩溃。
- 能从 JSON 读取待办、修改后再写回。

### 1.4 推荐资料

- Python 官方教程：<https://docs.python.org/zh-cn/3/tutorial/>
- 类型标注指南：<https://typing.python.org/zh-cn/latest/guides/>

---

## 2. Web 与 HTTP：先弄明白接口在干什么（第 3 周前半）

在写框架前，先掌握以下概念：

| 概念 | 你需要理解的事情 |
| --- | --- |
| HTTP 请求 / 响应 | 客户端发什么，服务端回什么 |
| URL、路径、查询参数 | `/todos/1` 与 `/todos?done=true` 的区别 |
| 请求方法 | `GET` 查、`POST` 新增、`PUT/PATCH` 修改、`DELETE` 删除 |
| 状态码 | `200` 成功、`201` 创建、`400` 参数错、`401` 未登录、`404` 不存在、`500` 服务异常 |
| JSON | 前后端交换数据的常用格式 |
| 幂等 | 同一个请求重试多次，结果是否一致 |
| Cookie / Token | 服务端如何识别用户 |

### 阶段练习

用 `curl` 或 API 调试工具实际发请求，观察请求头、请求体和响应：

```bash
curl https://httpbin.org/get
curl -X POST https://httpbin.org/post -H "Content-Type: application/json" -d '{"title":"学习 FastAPI"}'
```

目标不是背 HTTP 状态码表，而是看到接口报错时，能判断是“路径错、参数错、权限错，还是服务端错”。

---

## 3. FastAPI + Uvicorn：把待办程序变成 API（第 3 周后半～第 4 周）

### 3.1 两者分别做什么

```text
FastAPI：定义路由、校验请求参数、编写业务逻辑、生成接口文档
Uvicorn：ASGI 服务器，负责真正监听端口并运行 FastAPI 应用
```

可以把 FastAPI 看作“餐厅的菜单和后厨流程”，Uvicorn 是“把餐厅门打开、接待顾客的人”。二者不是替代关系。

### 3.2 安装并跑通第一个接口

```bash
uv add fastapi "uvicorn[standard]"
```

新建 `app/main.py`：

```python
from fastapi import FastAPI

app = FastAPI(title="Todo API")


@app.get("/health")
async def health() -> dict[str, str]:
    return {"status": "ok"}
```

启动开发服务器：

```bash
uv run uvicorn app.main:app --reload
```

访问：

- 健康检查：`http://127.0.0.1:8000/health`
- Swagger 文档：`http://127.0.0.1:8000/docs`

### 3.3 按功能实现接口

按顺序完成，不要一次性把所有接口塞进一个文件：

1. `POST /todos`：创建待办。
2. `GET /todos`：分页查询待办，支持 `done` 筛选。
3. `GET /todos/{todo_id}`：查询单条待办。
4. `PATCH /todos/{todo_id}`：修改标题或完成状态。
5. `DELETE /todos/{todo_id}`：删除待办。

本阶段先用内存列表保存数据，目的是把重点放在 API 设计、Pydantic 请求校验和异常响应上。服务重启数据消失是故意的，下一阶段才解决它。

### 3.4 推荐目录

```text
python-todo-api/
├── app/
│   ├── main.py          # 应用入口、路由注册
│   ├── schemas/         # 请求和响应模型（Pydantic）
│   ├── api/             # 路由层：解析 HTTP 请求
│   ├── services/        # 业务规则
│   └── core/            # 配置、日志等基础能力
├── tests/
├── pyproject.toml
└── README.md
```

开始时目录可以少，但要守住一条边界：路由层不直接堆复杂业务和数据库 SQL。

### 3.5 阶段验收

- `/docs` 能自动展示五个待办接口。
- 标题为空时，接口返回清晰的 `400/422` 参数错误。
- 不存在的待办返回 `404`，不是裸露的 Python 报错。
- `GET /todos` 返回分页结构，而不是无限制地返回所有数据。
- 能说清 FastAPI 和 Uvicorn 的职责区别。

官方资料：<https://fastapi.tiangolo.com/zh/>、<https://www.uvicorn.org/>

---

## 4. MySQL：让数据可靠落库（第 5～6 周）

### 4.1 先补 SQL 和 MySQL 必要知识

重点学习：

1. `CREATE TABLE`、`INSERT`、`SELECT`、`UPDATE`、`DELETE`。
2. 主键、唯一索引、普通索引。
3. `WHERE`、`ORDER BY`、`LIMIT`、分页。
4. 表关联：`JOIN`。
5. 事务：原子性、提交、回滚。
6. 用 `EXPLAIN` 看索引是否生效。

已有笔记可配合复习：[[3-存储层/MySQL/1-MVCC]]、[[3-存储层/MySQL/2-事务]]。

### 4.2 用 Docker 启动 MySQL 和 Redis

在项目根目录新建 `compose.yaml`：

```yaml
services:
  mysql:
    image: mysql:8
    environment:
      MYSQL_ROOT_PASSWORD: root
      MYSQL_DATABASE: todo
      MYSQL_USER: todo
      MYSQL_PASSWORD: todo
    ports:
      - "3306:3306"
    volumes:
      - mysql_data:/var/lib/mysql

  redis:
    image: redis:7
    ports:
      - "6379:6379"

volumes:
  mysql_data:
```

启动：

```bash
docker compose up -d
docker compose ps
```

> 密码仅适用于本地学习环境。真实项目必须放入环境变量或密钥管理服务，绝不能提交到仓库。

### 4.3 接入 SQLAlchemy 和 Alembic

推荐学习异步数据库访问，安装依赖：

```bash
uv add sqlalchemy alembic asyncmy
```

需要掌握的顺序：

1. 用 SQLAlchemy 定义 `User`、`Todo` 两张表。
2. 配置连接串，例如 `mysql+asyncmy://todo:todo@127.0.0.1:3306/todo`。
3. 在 FastAPI 中通过依赖注入获取数据库会话。
4. 用 Alembic 生成并执行迁移，而不是手工改线上表。
5. 将第三阶段的内存列表替换为数据库读写。

建议的表关系：

```text
users (1) ──── (n) todos
```

`todos` 至少有：`id`、`user_id`、`title`、`completed`、`created_at`、`updated_at`。

### 4.4 阶段验收

- 重启 FastAPI 后，已创建的待办仍可查到。
- 为 `todos.user_id` 和常用查询条件设计索引，并能说明原因。
- 新增或修改表结构时，通过 Alembic 迁移完成。
- 数据库不可用时，日志有清晰错误；接口不泄露账号、连接串或 SQL 细节。
- 能解释为什么“查列表后循环查每个用户”会造成 N+1 查询问题。

官方资料：<https://docs.sqlalchemy.org/>、<https://alembic.sqlalchemy.org/>、<https://dev.mysql.com/doc/>

---

## 5. Redis：先解决缓存，再学习限流（第 7 周）

Redis 不是 MySQL 的替代品。简单分工如下：

```text
MySQL：真实、长期保存的业务数据
Redis：访问很频繁、允许短暂失效的数据；计数器、验证码、分布式协调
```

已有笔记可配合：[[3-存储层/Redis/1-Redis概念和基础]]、[[3-存储层/Redis/8-Redis-过期策略]]、[[3-存储层/Redis/10-Redis-典型实战-分布式锁]]。

### 5.1 接入 Redis

```bash
uv add redis
```

从最简单、最安全的缓存场景开始：缓存 `GET /todos/{todo_id}` 的查询结果。

缓存流程：

```text
请求详情
  -> Redis 有缓存？有：直接返回
  -> 没有：查 MySQL -> 写入 Redis（设置过期时间）-> 返回

更新或删除待办
  -> 先更新 MySQL
  -> 再删除该待办的 Redis 缓存
```

这叫 **Cache Aside（旁路缓存）**。初学阶段优先掌握它，别一上来就被十几种缓存模式包围。

### 5.2 再实现两个实用功能

1. **登录令牌**：登录成功后，把会话或 Token 的关联信息放到 Redis，并设置过期时间。
2. **限流**：限制单个 IP / 用户每分钟最多请求某个接口 60 次。

限流先实现固定窗口即可；理解后再学习滑动窗口、令牌桶。要注意 Redis 的“读-改-写”涉及并发时，使用原子命令或 Lua 脚本，别把限流计数写成竞态条件。

### 5.3 阶段验收

- 第一次查详情访问 MySQL；在 TTL 内再次查询命中 Redis。
- 更新待办后，下一次查询不会读到旧缓存。
- Redis 宕机时，核心查询可以降级为查 MySQL（性能下降，但服务不直接挂掉）。
- 超过限流阈值时返回 `429 Too Many Requests`。
- 能说明缓存穿透、缓存击穿、缓存雪崩分别是什么，并知道它们不是同一种问题。

官方资料：<https://redis.io/docs/latest/>

---

## 6. 工程化：让项目从“能跑”变成“能维护”（第 8～9 周）

### 6.1 配置管理

使用 `pydantic-settings` 读取环境变量。至少区分：

```text
开发环境：本地 MySQL / Redis、详细日志、自动重载
生产环境：真实连接配置、适度日志、不使用 --reload
```

不要把账号、密码、JWT 密钥写死在 Python 文件里；提供 `.env.example`，将 `.env` 加入 `.gitignore`。

### 6.2 日志与错误处理

完成这些基本能力：

- 每个请求带 `request_id`，方便串联排查。
- 记录请求方法、路径、状态码、耗时。
- 统一处理业务异常，给客户端稳定的错误 JSON。
- 500 错误在服务端记录堆栈，在客户端只返回通用信息。
- 提供 `/health`，检查服务基本存活；需要时再提供依赖 MySQL/Redis 的 `/ready`。

### 6.3 测试

```bash
uv add --dev pytest pytest-asyncio httpx ruff
```

最少写这些测试：

| 类别 | 示例 |
| --- | --- |
| API 测试 | 创建待办后能查询到；非法参数返回 422 |
| 业务测试 | 只能修改自己的待办 |
| 缓存测试 | 更新后缓存被删除 |
| 异常测试 | 查询不存在的待办返回 404 |

常用命令：

```bash
uv run pytest
uv run ruff check .
uv run ruff format --check .
```

### 6.4 Docker 化

学习并完成：

1. 写 `Dockerfile`，把 FastAPI 服务打成镜像。
2. 用环境变量传入配置。
3. 容器内用生产启动方式，例如 `uvicorn app.main:app --host 0.0.0.0 --port 8000`。
4. 使用 `docker compose` 一起启动 API、MySQL、Redis。
5. 用健康检查确认服务可用。

注意：开发时使用 `--reload`，生产环境不要使用它；生产并发模型需结合 CPU、容器和实际压测决定，先不要迷信“worker 数越多越好”。

### 6.5 阶段验收

- 一个新同学按 README 的步骤能在本地启动项目。
- `pytest` 和 Ruff 检查通过。
- `docker compose up` 后，API、MySQL、Redis 都可访问。
- 配置和密钥不在 Git 历史中。
- 日志足以定位一次失败请求。

---

## 7. 10 周执行表

| 周次 | 学什么 | 必须产出 |
| --- | --- | --- |
| 第 0 周 | 环境、Git、uv、Docker | 能创建虚拟环境并启动容器 |
| 第 1 周 | Python 语法、容器、函数 | 命令行待办程序 |
| 第 2 周 | 类型、异常、文件、异步基础 | JSON 持久化待办程序 |
| 第 3 周 | HTTP、FastAPI 入门 | `/health` 和待办 CRUD 接口 |
| 第 4 周 | Pydantic、依赖注入、分层 | Swagger 文档完整、异常规范 |
| 第 5 周 | SQL、MySQL、SQLAlchemy | 待办数据落 MySQL |
| 第 6 周 | Alembic、事务、索引 | 数据库迁移和用户-待办关系 |
| 第 7 周 | Redis 缓存、会话、限流 | 缓存详情、接口限流 |
| 第 8 周 | pytest、日志、配置 | 关键路径有测试和日志 |
| 第 9 周 | Docker、Compose、部署常识 | 一条命令启动完整服务 |
| 第 10 周 | 回顾、压测、文档 | 可演示的完整项目和 README |

每周建议投入 8～12 小时，其中约 70% 用来写代码和排错，30% 用来读文档与总结。看懂不等于会用，接口能跑才算过关。

---

## 8. 最终项目验收清单

- [ ] 用户可注册、登录，并且只能访问自己的待办。
- [ ] 待办 CRUD 数据保存在 MySQL，表结构由 Alembic 管理。
- [ ] FastAPI 自动文档可用，输入校验和错误码合理。
- [ ] 单条待办查询使用 Redis 缓存；写操作会失效缓存。
- [ ] 登录态或令牌具备过期能力；敏感配置不进入仓库。
- [ ] 某个高频接口存在 Redis 限流并返回 `429`。
- [ ] 核心接口有自动化测试，`pytest` 与 Ruff 检查均通过。
- [ ] Docker Compose 可一键启动 API、MySQL、Redis。
- [ ] README 说明环境变量、启动方法、测试方法和接口示例。

---

## 9. 学习中最常踩的坑

1. **只看视频不敲代码**：每节学习后必须有可运行提交，否则两周后几乎全忘。
2. **把异步当成性能魔法**：`async` 适合等待网络、数据库等 I/O；CPU 密集计算不会因此变快。
3. **直接在路由里写 SQL**：小 demo 看似快，需求一多就难测、难改；尽早分出路由、服务、数据访问职责。
4. **Redis 只写不设过期时间**：内存会一直涨；缓存键必须设计 TTL。
5. **把 Redis 当数据库**：Redis 重启、淘汰或故障时，不应丢失核心业务事实。
6. **手改数据库表却不写迁移**：团队和部署环境必然失控；所有结构变化都走 Alembic。
7. **把密码提交到 Git**：即使后来删除文件，Git 历史仍可能保留；泄露后要立即轮换密码。
8. **追求复杂架构**：目前单体 FastAPI + MySQL + Redis 足够，先把测试、日志和边界做好，再考虑消息队列、微服务、K8s。

---

## 10. 完成后再学什么

完成本路线后，再按目标选方向：

| 目标 | 下一步 |
| --- | --- |
| 后端开发 | JWT / OAuth2、任务队列（Celery / ARQ）、消息队列、性能分析 |
| 数据方向 | Pandas、SQL 优化、ETL、数据建模 |
| AI 应用 | PydanticAI / LangChain、向量数据库、RAG、异步任务 |
| 运维部署 | Linux、Nginx、CI/CD、监控（Prometheus / Grafana）、云服务 |

建议先把这个项目做完整，再进入下一层。一个能稳定启动、可测试、可排错的小项目，比十个“跟着教程写到一半”的仓库更有含金量。
