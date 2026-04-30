# CLAUDE.md

此文件为 Claude Code (claude.ai/code) 在此仓库中工作时提供指导。

计划模式下输出的计划文档和git commit信息使用简体中文编写。

## 概述

基于 Docker Compose 的本地开发基础设施，提供 PostgreSQL 17 和 Nacos（阿里巴巴服务发现与配置管理）。

## 常用命令

```bash
# 启动所有服务
docker-compose up -d

# 启动单个服务
docker-compose up -d postgres
docker-compose up -d nacos

# 重新构建并重启服务（例如 Dockerfile 变更后）
docker-compose build nacos && docker-compose up -d nacos

# 查看日志
docker-compose logs -f nacos
docker-compose logs -f postgres

# 停止所有服务
docker-compose down

# Nacos 健康检查
curl -sf -o /dev/null -w "%{http_code}" http://localhost:8080/

# 连接 PostgreSQL
docker exec -it dev-postgres psql -U postgres -d dev-nacos
```

## 架构

### Nacos 使用 PostgreSQL 作为后端存储

Nacos 默认使用内嵌 Derby 数据库。本配置将其替换为 PostgreSQL：

1. **Dockerfile**（[nacos/Dockerfile](nacos/Dockerfile)）— 基于 `nacos/nacos-server:latest` 构建，下载 PostgreSQL JDBC 驱动（42.7.5）到 Nacos 插件目录。
2. **custom.properties**（[nacos/conf/custom.properties](nacos/conf/custom.properties)）— 挂载到容器的 `/home/nacos/conf/application.properties`，配置 Spring 数据源通过 Docker 网络 DNS 连接到 `postgres:5432/dev-nacos`。Nacos 3.x 使用 `spring.sql.init.platform=postgresql` 声明数据库平台（非旧版 `spring.datasource.platform`）。
3. **Schema**（[nacos/schema/nacos-pg.sql](nacos/schema/nacos-pg.sql)）— 所有 Nacos 表的 PostgreSQL DDL（配置、用户、角色、权限、租户）。首次部署时需手动导入 `dev-nacos` 数据库：
   ```bash
   docker exec -i dev-postgres psql -U postgres -d dev-nacos < nacos/schema/nacos-pg.sql
   ```

### 服务依赖

Nacos 依赖 PostgreSQL 健康检查通过后才会启动（`depends_on` + `condition: service_healthy`）。

### 端口

| 服务       | 端口 | 用途                |
| ---------- | ---- | ------------------- |
| PostgreSQL | 5432 | 数据库              |
| Nacos      | 8848 | API 服务端          |
| Nacos      | 8080 | 管理控制台 Web UI   |
| Nacos      | 9848 | gRPC 客户端         |
| Nacos      | 9849 | gRPC 服务端内部通信 |

### 数据持久化

所有数据卷映射到宿主机的 `/Users/cloudswhite/Personal/SoftwareData/docker/`：

- PostgreSQL 数据：`.../postgresql`
- Nacos 数据：`.../nacos/data`
- Nacos 日志：`.../nacos/logs`

### 认证

Nacos 已开启认证（`NACOS_AUTH_ENABLE=true`）。默认凭据：`nacos` / `nacos`。Schema 中预置了该用户并赋予 `ROLE_ADMIN` 角色。

### .env 文件

包含 PostgreSQL 凭据（`POSTGRES_USER`、`POSTGRES_PASSWORD`、`POSTGRES_DB`）。`.gitignore` 已排除 `.env`，当前值为默认值（`postgres`/`postgres`/`postgres`）。
