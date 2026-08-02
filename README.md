# Hedgehog

个人管理工具链。单用户自部署，PWA 前端 + 轻量后端，覆盖日程、习惯打卡、健康记录三类核心场景。所有数据保存在自己的服务器上，适合长期个人使用。

## 功能模块

| 模块 | 说明 | 入口 |
|------|------|------|
| 日历 | 对接 Radicale（CalDAV），列出日历、按时间/日历/关键词查日程、创建事件；后端只做 JSON ↔ iCalendar 翻译，不存数据 | `#/calendar` |
| 习惯打卡 | 每日习惯清单、完成/取消打卡、创建/编辑/归档/删除；按月 bitmap 存储，历史可追溯 | `#/habits` |
| 健康记录 | 统一指标模型（饮水/体重/饮食/运动），支持图片上传（自动生成缩略图）、按日查询、全局配置（目标饮水量、目标体重、营养系数等） | `#/health` |
| 认证 | 单用户 JWT 登录与校验，Go 无状态服务，由 nginx `auth_request` 保护外网入口 | 独立服务 |

## 架构

```
PWA 前端 (Vite + TypeScript)
        │ HTTPS
        ▼
cloudflared tunnel
        ▼
nginx ──auth_request──► auth-service (Go, :9091)
        │
        ▼
toolchain-service (Spring Boot, :9093)
   ├── calendar ──► Radicale (CalDAV)
   ├── habits
   ├── health (统一 metric_record + JSONB)
   └── PostgreSQL 16 (toolchain 库)
```

- 前后端分离：`web/` 构建为纯静态文件，由 nginx 托管；API 统一走 `/api/*`
- 业务后端只有一个：`services/toolchain-service`（Java 21 / Spring Boot 3.4）
- 认证独立：`services/auth-single-user-service`（Go），无状态、无持久化，外网鉴权 / 内网直连两种模式
- 健康数据统一存储：`metric_record` 单表 + JSONB，指标差异通过策略类处理，新增指标只需新增一个策略

## 目录结构

```
Hedgehog/
├── web/                                # PWA 前端（Vite + TypeScript，无框架）
├── services/
│   ├── toolchain-service/              # 业务后端（Java 21 / Spring Boot 3.4）
│   └── auth-single-user-service/       # 认证服务（Go）
├── doc/                                # 设计文档与部署手册
└── README.md
```

## 本地开发

前置条件：JDK 21、Maven 3.6.3+、Node.js 20+、PostgreSQL 16（本地或 Docker）。

1. 启动数据库并创建 `toolchain` 库，参考 [doc/deploy.md](doc/deploy.md) 与 [doc/sql](doc/sql)；
   默认连接串为 `jdbc:postgresql://localhost:5432/toolchain`，账号 `will` / `123456`（可在 `application.yml` 中修改）。

2. 启动后端：

```bash
cd services/toolchain-service
mvn spring-boot:run        # 默认端口 9093
```

3. 启动前端：

```bash
cd web
npm install
npm run dev                # http://localhost:5173，/api 代理到 :9093
```

4. （可选）启动认证服务，参考 `services/auth-single-user-service/docker-compose-example.yml`。

## 一键部署（docker-compose）

仓库提供根目录 [docker-compose.yml](docker-compose.yml)（nginx + auth + toolchain + postgres）与 [deploy/nginx.conf](deploy/nginx.conf)：

```bash
cd web && npm install && npm run build   # 构建前端静态资源
cd Hedgehog
# 先替换 docker-compose.yml 中的 AUTH_PASSWORD_HASH / JWT_SECRET
docker compose up -d --build
# 入口 http://<服务器>:8080
```

详细步骤与坑点见 [doc/deploy.md](doc/deploy.md)。

## 构建与测试

```bash
# 后端
cd services/toolchain-service && mvn test

# 认证服务
cd services/auth-single-user-service && go test ./...

# 前端
cd web && npm run build
```

当前状态：Java 后端 99 个测试、Go 认证服务 24 个用例均通过；前端 `npm run build` 通过。

## API 概览

| 模块 | 端点 | 说明 |
|------|------|------|
| 认证 | `POST /api/auth/login`、`GET /api/auth/verify` | 登录签发 JWT、校验 token |
| 日历 | `GET /api/calendar/calendars`、`GET /api/calendar/events`、`POST /api/calendar/calendars/{id}/events` | 日历列表、事件查询、事件创建 |
| 习惯 | `GET /api/habits/today`、`PATCH /api/habits/{id}/checkin`、`GET/POST /api/habits`、`PATCH/DELETE /api/habits/{id}`、`PATCH /api/habits/{id}/archive` | 今日清单、打卡、管理 |
| 健康 | `POST/GET/DELETE /api/health/metrics`、`GET/PUT /api/health/config` | 指标记录（WATER/DIET/EXERCISE/WEIGHT，支持图片）、全局配置 |

详细字段与示例见各设计文档。

## 文档索引

| 文档 | 内容 |
|------|------|
| [doc/idea.md](doc/idea.md) | 项目愿景、模块矩阵、整体架构 |
| [doc/toolchain-design.md](doc/toolchain-design.md) | 业务后端设计、日历模块 |
| [doc/habit-design.md](doc/habit-design.md) | 习惯打卡模块设计 |
| [doc/unified-metric-design.md](doc/unified-metric-design.md) | 健康统一指标模型设计与实现 |
| [doc/health-design.md](doc/health-design.md) | health 模块设计决策与全局配置 |
| [doc/auth-design.md](doc/auth-design.md) | 认证服务设计、部署与运维 |
| [doc/web-design.md](doc/web-design.md) | 前端设计与页面结构 |
| [doc/db-migration.md](doc/db-migration.md) | H2 → PostgreSQL 迁移记录 |
| [doc/deploy.md](doc/deploy.md) | 部署手册与已知坑点 |
| [doc/progress.md](doc/progress.md) | 开发进展记录 |

## 路线

- 第一阶段（当前）：模块独立自治，只做基础功能
- 第二阶段：AI Agent 跨模块数据聚合，生成分析与建议
