---
title: "适用于生产环境部署 open-webui 的 docker compose 配置"
date: 2025-05-05T00:00:00+08:00
draft: false
toc: false
images:
tags:
  - docker
  - docker compose
  - open-webui
  - open-webui pipelines
---

## 简介

open-webui 的默认 [docker-compose.yaml](https://github.com/open-webui/open-webui/blob/main/docker-compose.yaml) 其实并不太适合正式一点的环境使用．主要存在以下几个问题：

1. 数据库使用了默认的 sqlite，这个并不太适合，需要换成 postgresql．
2. redis 也没用上，正式一点的环境还是得用上吧．
3. open-webui pipelines 也没有，考虑到扩展性，还是可以先配置上的．
4. 使用默认的 docker volume，个人不太喜欢，还是喜欢直接挂载当前项目的指定目录到容器内直接使用，更加方便迁移和备份．

## 详情

废话少说，完整的 `docker-compose.yaml` 参考如下：

```yaml
name: open-webui
services:
  open-webui:
    image: ghcr.io/open-webui/open-webui:0.6.5
    volumes:
      - ./volumes/open-webui:/app/backend/data
    ports:
      - ${OPEN_WEBUI_PORT:-8080}:8080
    environment:
      # 禁用 Ollama API
      ENABLE_OLLAMA_API: ${ENABLE_OLLAMA_API:-false}
      # 启用 OpenAI API
      ENABLE_OPENAI_API: ${ENABLE_OPENAI_API:-true}
      OPENAI_API_BASE_URL: ${OPENAI_API_BASE_URL:-https://api.openai.com/v1}
      OPENAI_API_KEY: "${OPENAI_API_KEY:-openwebui123456}"

      # 使用 PostgreSQL 数据库
      DATABASE_URL: ${DATABASE_URL:-postgresql://postgres:openwebui123456@db:5432/openwebui}

      # 使用 Redis
      REDIS_URL: ${REDIS_URL:-redis://redis:6379/0}
      ENABLE_WEBSOCKET_SUPPORT: ${ENABLE_WEBSOCKET_SUPPORT:-true}
      WEBSOCKET_MANAGER: ${WEBSOCKET_MANAGER:-redis}
      WEBSOCKET_REDIS_URL: ${WEBSOCKET_REDIS_URL:-redis://redis:6379/0}

      # 其他环境变量（根据需要启用或配置）
      GLOBAL_LOG_LEVEL: ${GLOBAL_LOG_LEVEL:-INFO}
      UVICORN_WORKERS: ${UVICORN_WORKERS:-1}
      ENV: ${ENV:-prod}"
      # ENABLE_SIGNUP: ${ENABLE_SIGNUP:-false}
      # ENABLE_LOGIN_FORM: ${ENABLE_LOGIN_FORM:-false}
      # ENABLE_ADMIN_CHAT_ACCESS: ${ENABLE_ADMIN_CHAT_ACCESS:-true}
      # ENABLE_ADMIN_EXPORT: ${ENABLE_ADMIN_EXPORT:-true}
      # ENABLE_RAG_WEB_SEARCH: ${ENABLE_RAG_WEB_SEARCH:-false}
      # ENABLE_IMAGE_GENERATION: ${ENABLE_IMAGE_GENERATION:-false}
      ENABLE_API_KEY: ${ENABLE_API_KEY:-true}
      # RAG_EMBEDDING_ENGINE: ${RAG_EMBEDDING_ENGINE:-openai}
      OFFLINE_MODE: ${OFFLINE_MODE:-false}
    extra_hosts:
      - host.docker.internal:host-gateway
    restart: unless-stopped
    depends_on:
      - db
      - redis

  db:
    image: postgres:15-alpine
    restart: always
    environment:
      TZ: Asia/Shanghai
      PGUSER: ${PGUSER:-postgres}
      POSTGRES_PASSWORD: ${POSTGRES_PASSWORD:-openwebui123456}
      POSTGRES_DB: ${POSTGRES_DB:-openwebui}
      PGDATA: ${PGDATA:-/var/lib/postgresql/data/pgdata}
    command: >
      postgres -c 'max_connections=${POSTGRES_MAX_CONNECTIONS:-100}'
               -c 'shared_buffers=${POSTGRES_SHARED_BUFFERS:-128MB}'
               -c 'work_mem=${POSTGRES_WORK_MEM:-4MB}'
               -c 'maintenance_work_mem=${POSTGRES_MAINTENANCE_WORK_MEM:-64MB}'
               -c 'effective_cache_size=${POSTGRES_EFFECTIVE_CACHE_SIZE:-4096MB}'
    volumes:
      - ./volumes/db/data:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD", "pg_isready"]
      interval: 1s
      timeout: 3s
      retries: 30

  redis:
    image: valkey/valkey:8-alpine
    restart: always
    volumes:
      - ./volumes/redis/data:/data
    environment:
      REDISCLI_AUTH: ${REDIS_PASSWORD:-openwebui123456}
    entrypoint:
      ["valkey-server", "--requirepass", "${REDIS_PASSWORD:-openwebui123456}"]
    healthcheck:
      test: ["CMD", "valkey-cli", "ping"]
      start_period: 5s
      interval: 1s
      timeout: 3s
      retries: 5

  pipelines:
    image: ghcr.io/open-webui/pipelines:main
    volumes:
      - ./volumes/pipelines:/app/pipelines
      - ./volumes/data:/data
    restart: always
    environment:
      PIPELINES_API_KEY: ${PIPELINES_API_KEY:-openwebui123456}
```

在这个配置中：

1. 我们指定了 open-webui 需要使用的版本为 0.6.5，而不是直接写一个 `main`，不是每次都从本地构建．
2. 增加了 redis，这里使用 valkey．
3. 增加了数据库配置，这里使用的是 postgresql．
4. 这里没有使用 ollama，一般我们都直接使用一个兼容 openai 的服务来提供．
5. 增加了 open-webui pipelines 服务，如果不需要也可以不用，具体用法见官方文档．根据 [open-webui pipelines](https://github.com/open-webui/pipelines)，他没有版本号，对应的 docker 镜像也没有．

此外，我们还可以自己写一个 `.env` 文件来设置默认的环境变量，根据 `docker-compose.yaml` 文件的内容，你应该可以容易写出，这里不做赘述．还有其他的环境变量可以设置的，详情见 open-webui 的文档．
