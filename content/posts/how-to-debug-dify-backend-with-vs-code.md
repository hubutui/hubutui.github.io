---
title: "如何使用 vs code 调试 dify 的后端"
date: 2025-04-19T00:00:00+08:00
draft: false
toc: false
images:
tags:
  - dify
  - vs code
  - docker
  - python
---

## 简介

本文将会介绍如何调试 [dify](https://github.com/langgenius/dify) 的后端代码，前端代码的调试不在本文的讨论范围内．本文写作的时候使用的是 dify 0.15.5，但是对于其他版本应该也是通用的．

## 最简单的打印法

如果你只需要简单的打印日志来进行调试的话，可以：

1. 修改 `api` 目录下的代码，将其挂载到容器内的对应位置．也就是修改 [docker/docker-compose.yaml](https://github.com/langgenius/dify/blob/0.15.5/docker/docker-compose.yaml) 中 `api` 的 `volumes`，把修改后端代码对应挂载到容器内的 `/app/api` 目录下．特别注意，不要直接设置 `../api:/app/api`，因为容器内的 Python 环境就在 `/app/api` 目录下，直接这样子挂载会导致无法启动项目．最简单省事的应该是：

```yaml
- ../api/app_factory.py:/app/api/app_factory.py:ro
- ../api/app.py:/app/api/app.py:ro
- ../api/commands.py:/app/api/commands.py:ro
- ../api/configs:/app/api/configs:ro
- ../api/constants:/app/api/constants:ro
- ../api/contexts:/app/api/contexts:ro
- ../api/controllers:/app/api/controllers:ro
- ../api/core:/app/api/core:ro
- ../api/dify_app.py:/app/api/dify_app.py:ro
- ../api/events:/app/api/events:ro
- ../api/extensions:/app/api/extensions:ro
- ../api/factories:/app/api/factories:ro
- ../api/fields:/app/api/fields:ro
- ../api/libs:/app/api/libs:ro
- ../api/migrations:/app/api/migrations:ro
- ../api/models:/app/api/models:ro
- ../api/schedule:/app/api/schedule:ro
- ../api/services:/app/api/services:ro
- ../api/tasks:/app/api/tasks:ro
- ../api/templates:/app/api/templates:ro
- ../api/tests:/app/api/tests:ro
```

2. 设置 [docker/.env](docker/.env) 中的环境变量 `DEBUG=true` 和 `FLASK_DEBUG=true`
3. 注意修改代码打印日志的时候不能直接用 `print` 函数，应该用 `logging` 模块，记得添加一些标记，方便查找．
4. 此外还可以在抛出异常的部分使用 `traceback` 来输出更加详细的信息：

```python
import traceback

traceback.print_exc()
```

## 使用 vs code 断点调试

使用 vs code 断点调试则稍微复杂一些，但是也非常值得．

1. 修改 `api` 服务的启动脚本 [api/docker/entrypoint.sh](https://github.com/langgenius/dify/blob/0.15.5/api/docker/entrypoint.sh)：

```bash
# 修改前
# exec flask run --host=${DIFY_BIND_ADDRESS:-0.0.0.0} --port=${DIFY_PORT:-5001} --debug
# 修改后
/app/api/.venv/bin/python -Xfrozen_modules=off -m debugpy --listen 0.0.0.0:5678 -m flask run --host=${DIFY_BIND_ADDRESS:-0.0.0.0} --port=${DIFY_PORT:-5001} --debug
```

也就是改用 `debuggpy` 来启动，这样就可以让我们的 vs code 连接到要调试的程序了．将修改后的启动脚本挂载到 `api` 服务容器内的 `/entrypoint.sh`．
如果 `api` 服务所使用的容器还没有安装 `debugpy` 这个包，我们可以在已有的容器基础上安装这个包并构建镜像，也可以直接修改以上的启动脚本，在启动程序之前使用 `pip install debugpy` 命令安装．新的 dify 1.x 版本改用 uv 进行包管理，这里可以替换为 `uv pip install debugpy`．

2. 注意到我们调试需要用 5678 端口，因此需要修改 `api` 服务的端口映射，将 5678 端口映射出来．出于安全原因，建议绑定到 `127.0.0.1`，也就是写 `- 127.0.0.1:5678:5678`．如果宿主机的端口 5678 被占用了，你可以用其他的端口．
3. 在 [.vscode/launch.json](.vscode/launch.json) 中添加调试配置：

```json
{
  "name": "Python: Flask (Docker)",
  "type": "debugpy",
  "request": "attach",
  "connect": {
    "host": "localhost",
    "port": 5678
  },
  "pathMappings": [
    {
      "localRoot": "${workspaceFolder}/api",
      "remoteRoot": "/app/api"
    }
  ],
  "subProcess": true
}
```

注意，这里的 5678 是宿主机的端口，请根据实际情况修改．

4. 在 `docker` 目录下使用 `docker compose up -d` 启动项目．曾经尝试过将这个命令用 vs code 的 tasks 来执行，然后在调试配置里设置 `preLaunchTask`，实际并不好用，不建议使用，还是直接手动 `docker compose up -d` 启动吧．
5. 使用 `docker compose logs -f --tail 200 api` 查看日志，看到 `Debugger is active!` 的提示之后，即可启动 vs code 的调试．当然，你需要在合适的位置打上断点．
6. 根据日志提示，我们还需要设置环境变量：`PYDEVD_DISABLE_FILE_VALIDATION: 1`．否则实际上会无法调试．
7. 你可以随时修改代码并保存，由于 flask 是以 debug 模式启动的，代码更新后会自动重启 flask 的．

### 额外补充

注意到 `api` 服务和 `worker` 服务其实是用的同一个镜像，以上的修改，包括代码和镜像本身的修改，需要这两个服务都同步，以免出错．

### 详细的来源解释

使用 vs code 来调试 dify 后端，实际上我们需要了解 dify 这个项目是如何启动运行的．

1. 从 [docker/docker-compose.yaml](https://github.com/langgenius/dify/blob/0.15.5/docker/docker-compose.yaml) 文件中，我们可以容易知道，后端服务 `api` 是通过容器启动的，对应的容器镜像则是通过 [api/Dockerfile](https://github.com/langgenius/dify/blob/0.15.5/api/Dockerfile) 构建的．
2. 简单阅读该 Dockerfile 文件，我们可以容易知道其 python 代码在容器内的路径为 `/app/api`，对应着源码目录下的 `api` 路径．但是同时也要注意到 `/app/api/.venv` 也包含着整个 Python 虚拟环境，我们不能直接将源码目录的 `api` 目录挂载进去，那样就覆盖掉了虚拟环境了．
3. 其次，我们注意到整个服务使用的启动脚本是 [api/docker/entrypoint.sh](https://github.com/langgenius/dify/blob/0.15.5/api/docker/entrypoint.sh)，我们可以在这个脚本里看到实际的启动逻辑．易知，当我们设置环境变量 `DEBUG=true` 的时候就会执行 `exec flask run --host=${DIFY_BIND_ADDRESS:-0.0.0.0} --port=${DIFY_PORT:-5001} --debug` 来启动服务了．
4. 但是这个只是 flask 自己的调试模式，它的好处在于修改代码并且保存之后 flask 会自动重新启动．我们需要的是 vs code 的调试，此时就需要改为使用 debugpy 来启动．
5. 至此，我们就基本上掌握了使用 vs code 调试所需的信息了，简单修改之后，再生成一份调试配置，做好源码的映射，即可启动调试了．

类似的方法也可以应用到其他的项目中．关键点在于：

1. 检查 `docker-compose.yaml`，确认哪一个是后端服务．
2. 检查 Dockerfile，了解对应的服务镜像是如何构建的，确认其启动脚本．
3. 修改启动脚本的内容，安装上 debugpy，并且以 debugpy 来启动程序，以便进行调试．

另外一个例子，比如 [ragflow](https://github.com/infiniflow/ragflow)，也可以这么调试．但是 ragflow 有一个问题，它的 flask 服务并不是用 flask run 来启动的，仔细看它的 Python 脚本 [api/ragflow_server.py](https://github.com/infiniflow/ragflow/blob/v0.17.2/api/ragflow_server.py#L121)，它是用 `from werkzeug.serving.run_simple` 来启动的，它甚至都不太适合用在生产环境，ragflow 至今也没有改进这个点．此外，它的 [docker/entrypoint.sh](https://github.com/infiniflow/ragflow/blob/v0.17.2/docker/entrypoint.sh) 甚至是先后启动了 nginx, worker 和 ragflow 服务三个程序．按理来说，这些应该拆分为 3 个服务，nginx 用一个容器启动，worker 用一个容器启动，ragflow 服务用另外一个容器启动，而且应该改用 gunicorn 来启动．

如果想要支持 gunicorn 来启动服务，以及使用 flask run 来启动，应该需要修改 [api/ragflow_server.py](https://github.com/infiniflow/ragflow/blob/v0.17.2/api/ragflow_server.py#L121)，这个不是本文的重点，问问 GPT 应该就可以拿到一个合适的答案了．然后在 `entrypoint.sh` 改成使用 debugpy 启动，应该就可以按照前述方法使用 vs code 来调试了．
