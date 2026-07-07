<h1>AI跨境电商系统(Toutiao E-commerce)</h1>

一个基于FastAPI和SQLAlchemy构建的跨境电商系统,支持用户注册登录、商品浏览、收藏和历史
记录等功能。

<h2>1-1 项目概述</h2>

这是一个跨境电商系统后端API服务,采用异步Python框架FastAPI开发,使用MySQL作为数据
库存储,通过SQLAlchemy ORM进行数据访问。系统提供完整的用户管理、商品浏览、收藏和历史记录
功能。

<h2>1-2 技术栈</h2>

· 后端框架:FastAPI

· 数据库:MySQL

· ORM:SQLAlchemy(异步)

· 数据库驱动:aiomysql

· 密码加密:passlib+bcrypt

· 缓存系统:Redis

· 异步支持:Python asyncio

<h2>1-3 功能模块</h2>

· 1.后端框架:FastAPI

· 2.用户管理模块

· 3.新闻模块

· 4.收藏模块

· 5.缓存模块

<h2>1-4 工程结构</h2>

- crud：数据库增删改查逻辑(封装数据库操作)

- models：数据库模型(SQLAlchemy ORM)

- routers：路由层(按模块划分)

- schemas：数据验证模型(Pydantic)

- utils：工具函数

- config：配置相关

- main.py：主函数入口

- cache：缓存读取和写入

<h2>1-5 环境配置</h2>

· 前端驱动：node.js

· 缓存驱动：Redis 1.5
