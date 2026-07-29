# 头条新闻后端 — 开发文档

## 一、项目概述

一个基于 **FastAPI + SQLAlchemy 2.0 异步 + MySQL + Redis** 的新闻资讯类 App 后端服务。提供用户认证、新闻分类浏览、新闻详情、收藏管理、浏览历史等核心 API，采用经典的分层架构。

**技术栈**：Python 3.13、FastAPI、SQLAlchemy 2.0 Async、aiomysql、Redis、bcrypt、Pydantic v2、Uvicorn

---

## 二、项目结构

```
toutiao_backend/
├── main.py                          # FastAPI 应用入口 + CORS + 路由注册
├── config/
│   ├── db_config.py                 # MySQL 异步引擎 + 会话工厂 + 依赖项
│   └── cache_config.py              # Redis 连接 + 缓存读写工具
├── models/                          # ORM 模型层 (SQLAlchemy 2.0 Mapped)
│   ├── news.py                      # Category 分类表 + News 新闻表
│   ├── users.py                     # User 用户表 + UserToken 令牌表
│   ├── favorite.py                  # Favorite 收藏表 (联合唯一约束)
│   └── history.py                   # History 浏览历史表
├── schemas/                         # Pydantic 数据校验层
│   ├── users.py                     # UserRequest/UserInfoResponse/UserAuthResponse/...
│   ├── favorite.py                  # FavoriteCheckResponse/FavoriteListResponse/...
│   └── history.py                   # HistoryAddRequest/HistoryListResponse/...
├── crud/                            # 数据访问层 (业务逻辑)
│   ├── users.py                     # 用户 CRUD + Token 管理 + 密码验证
│   ├── news.py                      # 新闻分类/列表/详情/浏览计数/推荐
│   ├── news_cache.py                # 新闻缓存层 (Redis 优先 → DB)
│   ├── favorite.py                  # 收藏 CRUD (检查/添加/删除/列表/清空)
│   └── history.py                   # 历史 CRUD (添加/列表/删除单条/清空)
├── routers/                         # 路由层 (API 接口定义)
│   ├── users.py                     # /api/user/* (注册/登录/信息/修改)
│   ├── news.py                      # /api/news/* (分类/列表/详情)
│   ├── favorite.py                  # /api/favorite/* (收藏操作)
│   └── history.py                   # /api/history/* (历史操作)
├── cache/
│   └── cache_news.py                # Redis 缓存封装 (分类/列表)
├── utils/                           # 工具层
│   ├── security.py                  # bcrypt 密码加解密
│   ├── auth.py                      # Token 认证依赖 (get_current_user)
│   ├── response.py                  # 统一 JSON 响应格式
│   ├── exception.py                 # 4 个全局异常处理器
│   ├── exception_handle.py          # 异常处理器注册函数
│   └── base.py                      # 公共 Pydantic 基类 (NewsItemBase)
└── .venv/                           # 虚拟环境
```

---

## 三、分层架构

```
┌──────────────────────────────────────────────────────┐
│                    main.py (入口)                     │
│  FastAPI() → CORS → 异常注册 → 路由挂载 → uvicorn    │
└───────────────────────┬──────────────────────────────┘
                        │
          ┌─────────────┼─────────────┐
          ▼             ▼             ▼
┌──────────────┐ ┌──────────────┐ ┌──────────────┐
│   routers/   │ │   routers/   │ │   routers/    │
│   users.py   │ │   news.py    │ │favorite/history│
│  /api/user/* │ │  /api/news/* │ │   /api/fav/*   │
└──────┬───────┘ └──────┬───────┘ └──────┬────────┘
       │                │                │
       ▼                ▼                ▼
┌──────────────────────────────────────────────────────┐
│                    crud/ (数据访问层)                   │
│  users.py  │  news.py  │  news_cache.py  │            │
│  favorite.py  │  history.py                           │
│                                                      │
│  news_cache.py: 先查 Redis → 未命中再查 MySQL          │
└──────┬───────────────┬───────────────────────────────┘
       │               │
       ▼               ▼
┌──────────────┐ ┌──────────────┐ ┌──────────────────┐
│  models/     │ │  cache/      │ │  schemas/        │
│  SQLAlchemy  │ │  Redis封装   │ │  Pydantic v2     │
│  ORM 模型    │ │              │ │  请求/响应校验     │
└──────┬───────┘ └──────────────┘ └──────────────────┘
       │
       ▼
┌──────────────────────────────────────────────────────┐
│                   config/ (基础设施)                    │
│  db_config.py (MySQL 异步引擎)                          │
│  cache_config.py (Redis 连接)                          │
└──────────────────────────────────────────────────────┘
```

---

## 四、开发流程

### 阶段一：项目初始化 + 基础设施配置

#### 1.1 数据库配置 (`config/db_config.py`)

```python
ASYNC_DATABASE_URL = "mysql+aiomysql://root:root@localhost:3306/news_app?charset=utf8mb4"
async_engine = create_async_engine(ASYNC_DATABASE_URL, pool_size=10, max_overflow=20)
AsyncSessionLocal = async_sessionmaker(bind=async_engine, class_=AsyncSession)
```

**关键技术选择**：

| 选择 | 理由 |
|------|------|
| `mysql+aiomysql` | 纯 Python 异步 MySQL 驱动 |
| `async_sessionmaker` | SQLAlchemy 2.0 推荐方式 |
| `pool_size=10` | 连接池大小，适合中等并发 |
| `create_db()` 依赖项 | FastAPI `Depends` 注入，自动提交/回滚/关闭 |

#### 1.2 Redis 配置 (`config/cache_config.py`)

```python
redis_client = redis.Redis(decode_responses=True)
async def get_cache(key)          # 读取字符串缓存
async def get_cache_json(key)     # 读取 JSON 缓存
async def set_cache(key, value)   # 写入缓存（自动序列化 dict/list）
```

---

### 阶段二：ORM 模型定义

**SQLAlchemy 2.0 新语法**：使用 `Mapped` + `mapped_column` 声明式映射。

#### 2.1 数据库 E-R 图

```
┌──────────┐     ┌──────────────┐     ┌──────────┐
│   User   │     │   Favorite   │     │   News   │
│──────────│     │──────────────│     │──────────│
│ id (PK)  │◄────│ user_id (FK) │     │ id (PK)  │
│ username │     │ news_id (FK) │────►│ title    │
│ password │     │ created_at   │     │ content  │
│ nickname │     └──────────────┘     │ image    │
│ avatar   │                          │ views    │
│ gender   │     ┌──────────────┐     │ ...      │
│ bio      │     │   History    │     └────┬─────┘
│ phone    │◄────│──────────────│          │
│ ...      │     │ user_id (FK) │     ┌────▼─────┐
└──────────┘     │ news_id (FK) │────►│ Category │
                 │ view_time    │     │──────────│
     ┌───────────┴──────────────┘     │ id (PK)  │
     │            UserToken           │ name     │
     │ user_id (FK) → User            │ sort_order│
     │ token (UNIQUE)                 └──────────┘
     │ expires_at
     └──────────────
```

#### 2.2 模型文件设计要点

| 模型 | 关键设计 |
|------|----------|
| `Category` | `Base` 包含 `created_at`/`updated_at` 公共字段 |
| `News` | 复合索引 `(category_id)` + `(publish_time)` 优化查询；`views` 自增更新 |
| `User` | `username`/`phone` 唯一索引；默认头像/性别/简介 |
| `UserToken` | UUID token + 7天过期；唯一索引防重复 |
| `Favorite` | `UniqueConstraint(user_id, news_id)` 防止重复收藏 |
| `History` | 按 `view_time` 索引，支持按时间排序 |

---

### 阶段三：Pydantic Schema 定义

**关键设计**：
- 使用 `alias` 实现 **Python 蛇形命名 ↔ JSON 驼峰命名** 的自动转换
- 使用 `ConfigDict(from_attributes=True)` 支持 ORM 对象直接转 Pydantic

| Schema | 用途 |
|--------|------|
| `UserRequest` | 注册/登录请求体 |
| `UserInfoResponse` | 用户信息响应 |
| `UserAuthResponse` | 登录响应（token + userInfo） |
| `FavoriteCheckResponse` | 收藏状态检查（`isFavorite: bool`） |
| `FavoriteListResponse` | 收藏列表（分页 + hasMore） |
| `HistoryListResponse` | 浏览历史列表（分页 + hasMore） |

---

### 阶段四：CRUD 数据访问层

**核心模式**：每个 CRUD 函数接收 `db: AsyncSession` 作为第一个参数（由 FastAPI `Depends` 注入）。

#### 4.1 用户模块

| 函数 | 说明 |
|------|------|
| `get_user_by_username()` | 用户名查用户 |
| `create_user()` | 注册 → bcrypt 加密密码 → 写入 DB |
| `create_token()` | UUID token + 7天过期 → 已有则更新，无则新建 |
| `authenticate_user()` | 用户名+密码验证 |
| `get_user_by_token()` | Token 查用户（含过期校验） |
| `update_user()` | 部分字段更新（`exclude_unset=True`） |
| `update_user_password()` | 旧密码验证 → 新密码哈希 → 更新 |

#### 4.2 新闻模块

| 函数 | 说明 |
|------|------|
| `get_categories()` | 分页查询分类 |
| `get_news_list()` | 按分类ID分页查询新闻 |
| `get_news_total()` | 统计分类下新闻总数 |
| `get_news_detail()` | 单条新闻详情 |
| `news_view_update()` | 浏览量自增（`News.views + 1`） |
| `get_related_news()` | 同分类下按浏览量+时间排序的推荐 |

#### 4.3 新闻缓存层 (`crud/news_cache.py`)

```
用户请求
  ├─ get_categories()  → 先查 Redis (news:categories)
  │                       └─ 未命中 → 查 DB → 写入 Redis (expire: 7200s)
  └─ get_news_list()   → 先查 Redis (news_list:{category}:{page}:{size})
                          └─ 未命中 → 查 DB → 写入 Redis (expire: 1200s)
```

**缓存 Key 设计**：

| Key 模板 | 示例 | 过期 |
|----------|------|------|
| `news:categories` | `news:categories` | 7200s |
| `news_list:{cat}:{p}:{s}` | `news_list:3:1:10` | 1200s |

#### 4.4 收藏模块

| 函数 | 说明 |
|------|------|
| `is_new_favorite()` | 检查是否已收藏 |
| `add_new_favorite()` | 添加收藏（`db.flush()` 触发唯一约束检查） |
| `remove_news_favorite()` | 取消收藏 |
| `get_news_favorite_list()` | 联表查询收藏列表（`Favorite JOIN News`） |
| `clear_all_favorite()` | 清空所有收藏 |

#### 4.5 历史记录模块

| 函数 | 说明 |
|------|------|
| `add_history()` | 已有记录更新 `view_time`，无则新建 |
| `get_news_history_list()` | 联表查询历史列表 |
| `delete_one_history()` | 删除单条记录 |
| `clear_all_history()` | 清空所有历史 |

---

### 阶段五：路由层

**统一前缀**：

| 路由模块 | 前缀 | 接口数 |
|----------|------|--------|
| `users` | `/api/user` | 5 |
| `news` | `/api/news` | 3 |
| `favorite` | `/api/favorite` | 5 |
| `history` | `/api/history` | 4 |

#### 5.1 认证流程

```
注册：POST /api/user/register
  → UserRequest(username, password)
  → 检测用户是否存在 → 创建用户(bcrypt hash) → 创建 Token(UUID)
  → 返回 {token, userInfo}

登录：POST /api/user/login
  → UserRequest(username, password)
  → authenticate_user() → bcrypt.verify() → create_token()
  → 返回 {token, userInfo}

鉴权：Header Authorization: Bearer <token>
  → get_current_user() 依赖项
  → 兼容 "Bearer xxx" / "Bearerxxx" / 无前缀 三种格式
  → get_user_by_token() → 验证 Token + 过期检查
  → 返回 User ORM 对象注入到路由函数
```

#### 5.2 API 清单

| 方法 | 路径 | 说明 | 认证 |
|------|------|------|------|
| POST | `/api/user/register` | 注册 | 否 |
| POST | `/api/user/login` | 登录 | 否 |
| GET | `/api/user/info` | 获取用户信息 | 是 |
| PUT | `/api/user/update` | 修改用户信息 | 是 |
| PUT | `/api/user/update/password` | 修改密码 | 是 |
| GET | `/api/news/categories` | 获取分类列表 | 否 |
| GET | `/api/news/list` | 获取新闻列表（分页） | 否 |
| GET | `/api/news/detail` | 获取新闻详情 + 推荐 | 否 |
| GET | `/api/favorite/check` | 检查收藏状态 | 是 |
| POST | `/api/favorite/add` | 添加收藏 | 是 |
| DELETE | `/api/favorite/remove` | 取消收藏 | 是 |
| GET | `/api/favorite/list` | 收藏列表 | 是 |
| DELETE | `/api/favorite/clear` | 清空收藏 | 是 |
| POST | `/api/history/add` | 添加浏览记录 | 是 |
| GET | `/api/history/list` | 浏览历史列表 | 是 |
| DELETE | `/api/history/delete/{id}` | 删除单条 | 是 |
| DELETE | `/api/history/clear` | 清空历史 | 是 |

---

### 阶段六：工具层

#### 6.1 安全 (`utils/security.py`)

```
加密：password.encode('utf-8')[:72] → bcrypt.hashpw(bcrypt.gensalt())
验证：bcrypt.checkpw(plain_bytes, hashed_bytes)
```

> 从 `passlib` 迁移到 `bcrypt` 原生库，减少依赖。

#### 6.2 认证 (`utils/auth.py`)

```
get_current_user()
  ├─ 从 Header 读取 Authorization
  ├─ 兼容 "Bearer xxx" / "Bearerxxx" / "xxx" 三种 token 格式
  ├─ get_user_by_token(db, token) 查询
  └─ 返回 User 或 raise 401
```

#### 6.3 统一响应 (`utils/response.py`)

```json
{"code": 200, "message": "成功", "data": {...}}
```

使用 `jsonable_encoder()` 自动处理 ORM/Pydantic 对象的 JSON 序列化。

#### 6.4 全局异常处理 (`utils/exception.py` + `exception_handle.py`)

| 处理器 | 捕获 | 状态码 |
|--------|------|--------|
| `http_exception_handler` | `HTTPException` | 保持原样 |
| `integrity_error_handler` | `IntegrityError` | 400 (识别重复/外键错误) |
| `sqlalchemy_error_handler` | `SQLAlchemyError` | 500 |
| `general_exception_handler` | `Exception` | 500 |

开发模式 (`DEBUG_MODE=True`) 返回详细 traceback，生产模式隐藏细节。

---

### 阶段七：应用组装 (`main.py`)

```python
app = FastAPI()

# 1. 全局异常处理
register_exception_handlers(app)

# 2. CORS 中间件（白名单模式）
app.add_middleware(CORSMiddleware, allow_origins=[...], allow_credentials=True)

# 3. 路由挂载
app.include_router(news.router)      # /api/news/*
app.include_router(users.router)     # /api/user/*
app.include_router(favorite.router)  # /api/favorite/*
app.include_router(history.router)   # /api/history/*

# 4. 启动
uvicorn.run(app, host="127.0.0.1", port=8000)
```

---

## 五、数据流全景

```
客户端请求：GET /api/news/list?categoryId=1&page=1&pageSize=10
    │
    ▼
FastAPI 路由匹配 → routers/news.py:get_news()
    │  提取参数: category_id=1, page=1, page_size=10
    │  注入 db: AsyncSession = Depends(create_db)
    ▼
crud/news_cache.py:get_news_list()
    │
    ├─→ 构造 Redis Key: news_list:1:1:10
    ├─→ await redis_client.get(key)
    │     ├─ 命中 → json.loads → 返回 ORM 对象
    │     └─ 未命中 ↓
    ├─→ SELECT * FROM news WHERE category_id=1 LIMIT 10 OFFSET 0
    ├─→ jsonable_encoder → Redis SET (expire 1200s)
    └─→ 返回新闻列表

总数量查询 → get_news_total() → SELECT COUNT(*) FROM news WHERE category_id=1

hasMore = (offset + len(list)) < total → True/False

响应：
{
  "code": 200, "message": "Success",
  "data": {
    "list": [...],
    "total": 50,
    "hasMore": true
  }
}
```

---

## 六、关键设计决策

| 决策 | 选择 | 原因 |
|------|------|------|
| ORM 风格 | SQLAlchemy 2.0 Mapped | 类型安全，IDE 友好 |
| 数据库驱动 | aiomysql | 纯 Python 异步，跨平台 |
| 密码加密 | bcrypt | 自动加盐，抗彩虹表 |
| Token 方案 | UUID (非 JWT) | 简单，无需 JWT 解析库 |
| 缓存策略 | Cache-Aside | 先读 Redis → 未命中再查 DB |
| 异常处理 | 全局注册 + DEBUG 模式 | 统一 JSON 格式，开发/生产差异化 |
| 响应格式 | 统一 `{code, message, data}` | 前端统一处理 |
| 命名转换 | Pydantic `alias` + `populate_by_name` | Python 蛇形 ↔ JSON 驼峰 |

---

## 七、运行方式

```bash
# 1. 激活虚拟环境
.venv\Scripts\activate

# 2. 启动 MySQL + Redis（确保运行中）

# 3. 创建数据库
mysql -u root -p -e "CREATE DATABASE IF NOT EXISTS news_app DEFAULT CHARSET utf8mb4;"

# 4. 创建表（在 Python 中执行）
python -c "from models.news import Base; from config.db_config import async_engine; import asyncio; asyncio.run(async_engine.run_sync(Base.metadata.create_all))"

# 5. 启动服务
python main.py
# → http://127.0.0.1:8000
```

**依赖清单**：
```
fastapi
uvicorn[standard]
sqlalchemy[asyncio]
aiomysql
redis[hiredis]
bcrypt
pydantic
```

---

## 八、已修复/已知问题

| # | 问题 | 文件 | 状态 |
|---|------|------|------|
| 1 | `discriminator` 误用于 `int` 类型字段 | `schemas/history.py` | 注释标记 |
| 2 | `UserToken` 更新逻辑：已有 Token 更新、无则新建 | `crud/users.py` | 已确认正确 |
| 3 | 部分 `db.commit()` 被注释，改用 `db.flush()` | 多处 | 依赖 `create_db()` 的 `session.commit()` |
| 4 | `jsonable_encoder` 处理复合查询结果 | routers/ | 手动 dict 拼接 |

---

## 九、扩展方向

1. **JWT 替换 UUID Token**：支持 token 内嵌用户信息，免查 DB
2. **新闻搜索**：接入 Elasticsearch 实现全文搜索
3. **API 限流**：接入 `slowapi` 防刷
4. **日志系统**：结构化日志 + 审计日志
5. **单元测试**：pytest + httpx 异步测试
6. **Docker 部署**：MySQL + Redis + FastAPI 容器编排
