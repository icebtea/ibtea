# Docker 学习笔记

## 一、核心概念

| 概念 | 说明 |
|------|------|
| **镜像（Image）** | 只读模板，包含运行应用所需的文件系统、依赖和配置。类似"安装包" |
| **容器（Container）** | 镜像的运行实例，可读写，可启动/停止/删除。类似"运行中的程序" |
| **Dockerfile** | 构建镜像的脚本文件，定义了从基础镜像到最终镜像的每一步操作 |
| **仓库（Registry）** | 存放和分发镜像的地方，如 Docker Hub、阿里云镜像仓库 |
| **Docker Compose** | 编排工具，用一个 YAML 文件定义和运行多容器应用 |

三者关系：**Dockerfile → 构建出镜像 → 镜像启动为容器**

---

## 二、底层原理

Docker 并不是虚拟机，它共享宿主机内核，通过 Linux 内核的三项机制实现隔离：

### 1. Namespaces（命名空间）—— 隔离"能看到什么"

每个容器拥有独立的：
- **PID**：进程空间，容器内 PID 1 是自己的 init 进程
- **NET**：网络栈，独立 IP、端口、路由表
- **MNT**：文件系统挂载点
- **UTS**：主机名
- **IPC**：进程间通信

效果：容器内进程看不到宿主机和其他容器的进程、网络、文件系统。

### 2. Cgroups（控制组）—— 限制"能用多少"

控制每个容器可使用的系统资源：
- CPU 份额
- 内存上限
- 磁盘 I/O
- 网络带宽

效果：防止单个容器耗尽宿主机资源。

### 3. UnionFS（联合文件系统）—— 分层存储

镜像由多个只读层（layer）叠加组成：
```
┌─────────────────────┐
│  可写层（容器运行时）  │  ← 容器修改写在这里
├─────────────────────┤
│  应用代码层          │
├─────────────────────┤
│  依赖包层            │
├─────────────────────┤
│  基础系统层（如 Ubuntu）│  ← 多个容器可共享此层
└─────────────────────┘
```

好处：
- 相同的基础层只存一份，节省磁盘空间
- 拉取镜像时已有层可以跳过，加快下载速度
- 容器删除后可写层消失，镜像不受影响

### Docker vs 虚拟机

| | Docker 容器 | 虚拟机 |
|--|-----------|--------|
| 隔离级别 | 进程级（共享内核） | 硬件级（独立内核） |
| 启动速度 | 秒级 | 分钟级 |
| 镜像大小 | MB 级 | GB 级 |
| 性能 | 接近原生 | 有虚拟化开销 |
| 安全性 | 较弱（共享内核） | 较强（完全隔离） |

---

## 三、常用命令

### 镜像操作

```bash
# 拉取镜像
docker pull nginx:latest
#        镜像名:标签（tag 不写默认 latest）

# 查看本地所有镜像
docker images
# 等同于 docker image ls

# 从 Dockerfile 构建镜像
docker build -t myapp:v1 .
#        -t 名称:标签   . Dockerfile 所在目录（当前目录）
# -f 指定 Dockerfile 路径：docker build -t myapp:v1 -f ./custom.Dockerfile .
# --no-cache 不使用缓存，全部重新构建

# 删除镜像
docker rmi nginx:latest
#        镜像名:标签（也可以用镜像 ID）
# -f 强制删除（即使有容器在使用）

# 给镜像打标签（用于推送到仓库前重命名）
docker tag myapp:v1 myrepo/myapp:v1
#       源镜像       新名称（通常包含仓库地址）

# 推送镜像到远程仓库
docker push myrepo/myapp:v1
# 需要先 docker login 登录仓库

# 保存镜像为 tar 文件（离线传输用）
docker save -o myapp.tar myapp:v1
#        -o 输出文件名

# 从 tar 文件加载镜像
docker load -i myapp.tar
#        -i 输入文件名

# 查看镜像构建历史（每一层执行了什么命令）
docker history myapp:v1
```

### 容器操作

`docker run` 是最常用的命令，参数较多，单独列出：

```bash
docker run [参数] 镜像名 [命令]
```

| 参数 | 说明 | 示例 |
|------|------|------|
| `-d` | 后台运行（detached 模式），不占用当前终端 | `docker run -d nginx` |
| `--name` | 给容器命名，不指定则自动生成随机名 | `docker run --name my-nginx nginx` |
| `-p` | 端口映射，格式 `宿主机端口:容器端口` | `docker run -p 8080:80 nginx` |
| `-P` | 大写 P，自动映射容器暴露的所有端口到宿主机随机端口 | `docker run -P nginx` |
| `-v` | 目录挂载，格式 `宿主机路径:容器路径` | `docker run -v /data:/app/data myapp` |
| `-e` | 设置环境变量 | `docker run -e MYSQL_ROOT_PASSWORD=123 mysql` |
| `-w` | 设置容器内工作目录 | `docker run -w /app myapp` |
| `-it` | `-i` 保持标准输入打开 + `-t` 分配伪终端，用于交互式操作 | `docker run -it ubuntu bash` |
| `--rm` | 容器停止后自动删除（适合一次性任务） | `docker run --rm python python script.py` |
| `--restart` | 重启策略（见下表） | `docker run --restart unless-stopped nginx` |
| `--network` | 指定容器加入的网络 | `docker run --network mynet nginx` |
| `--gpus` | 分配 GPU（需安装 nvidia-container-toolkit） | `docker run --gpus all pytorch/pytorch` |
| `--memory` | 限制内存使用 | `docker run --memory 512m nginx` |
| `--cpus` | 限制 CPU 核数 | `docker run --cpus 1.5 nginx` |
| `--entrypoint` | 覆盖镜像的 ENTRYPOINT | `docker run --entrypoint bash myapp` |

`--restart` 重启策略：

| 值 | 说明 |
|----|------|
| `no` | 不自动重启（默认） |
| `always` | 总是重启，包括 Docker 守护进程重启后 |
| `unless-stopped` | 类似 always，但手动 stop 后不会再重启 |
| `on-failure[:次数]` | 仅当容器异常退出（非 0 退出码）时重启 |

**容器管理命令：**

```bash
# 查看运行中的容器
docker ps
# -a 查看所有容器（含已停止的）
# -q 只显示容器 ID（常搭配其他命令使用，如 docker rm $(docker ps -aq)）
# --format 自定义输出格式，如 docker ps --format "table {{.Names}}\t{{.Status}}"

# 停止容器（发送 SIGTERM，10 秒后发送 SIGKILL）
docker stop <容器ID或名称>

# 强制停止容器（直接发送 SIGKILL）
docker kill <容器ID或名称>

# 启动已停止的容器
docker start <容器ID或名称>

# 重启容器
docker restart <容器ID或名称>

# 暂停 / 恢复容器（冻结/解冻进程）
docker pause <容器ID>
docker unpause <容器ID>

# 删除容器（需先停止，或加 -f 强制删除）
docker rm <容器ID>
# -f 强制删除运行中的容器
# -v 同时删除关联的匿名数据卷
```

### 调试与排错

```bash
# 查看容器日志
docker logs <容器ID>
# -f 实时跟踪输出（类似 tail -f）
# --tail 100 只显示最后 100 行
# --since 30m 显示最近 30 分钟的日志
# --timestamps 显示时间戳

# 进入运行中的容器（开启新的 bash 会话）
docker exec -it <容器ID> bash
#        -i 保持标准输入  -t 分配终端
# 也可以用 sh（某些精简镜像没有 bash）
# docker exec -it <容器ID> sh

# 在容器内执行单条命令（不进入交互模式）
docker exec <容器ID> ls /app

# 查看容器详细信息（IP 地址、挂载、环境变量等，输出 JSON 格式）
docker inspect <容器ID>
# 查看特定字段：docker inspect -f '{{.NetworkSettings.IPAddress}}' <容器ID>

# 实时监控容器资源占用（CPU、内存、网络 I/O、磁盘 I/O）
docker stats
# --no-trunc 不截断显示
# 指定容器：docker stats <容器ID>

# 查看容器内运行的进程
docker top <容器ID>

# 查看容器的端口映射
docker port <容器ID>

# 查看容器文件系统的变更（相比镜像新增/修改/删除了哪些文件）
docker diff <容器ID>

# 将容器内的文件复制到宿主机
docker cp <容器ID>:/app/log.txt ./log.txt
# 也可以反向复制：宿主机 → 容器
# docker cp ./config.yml <容器ID>:/app/config.yml
```

### 清理

```bash
# 删除所有已停止的容器
docker container prune
# -f 跳过确认提示

# 删除所有未使用的镜像（没有被任何容器引用的镜像）
docker image prune
# -a 删除所有未被运行中容器使用的镜像（更彻底）

# 删除未使用的数据卷
docker volume prune

# 删除未使用的网络
docker network prune

# 一键清理所有未使用的资源（容器、镜像、网络、构建缓存）
docker system prune
# -a 同时删除未被运行中容器使用的镜像
# --volumes 同时删除未使用的数据卷

# 查看 Docker 整体磁盘占用情况
docker system df
# -v 显示详细信息（每个镜像、容器、数据卷分别占多少）
```

### Docker Compose（多容器编排）

Docker Compose 通过 `docker-compose.yml` 文件定义多个容器服务，一键启动/管理。

```bash
# 启动所有服务（读取当前目录的 compose.yml 或 docker-compose.yml）
docker compose up
# -d 后台运行（不占用终端）
# --build 启动前重新构建镜像
# --force-recreate 强制重建所有容器（即使配置没变）

# 停止并删除所有容器和网络
docker compose down
# -v 同时删除数据卷（注意：会丢失数据）
# --rmi all 同时删除镜像
# --remove-orphans 删除 compose 文件中未定义但属于该项目的容器

# 查看服务状态
docker compose ps

# 查看日志
docker compose logs
# -f 实时跟踪
# <服务名> 只看某个服务的日志：docker compose logs -f web

# 重新构建镜像
docker compose build
# --no-cache 不使用缓存
# <服务名> 只构建某个服务

# 拉取最新镜像
docker compose pull

# 重启某个服务
docker compose restart <服务名>

# 停止某个服务（不删除）
docker compose stop <服务名>

# 启动已停止的服务
docker compose start <服务名>

# 扩缩容（启动多个实例）
docker compose up -d --scale worker=3
# 将 worker 服务启动 3 个实例

# 进入某个服务的容器
docker compose exec <服务名> bash

# 在某个服务内执行单条命令
docker compose exec <服务名> python manage.py migrate
```

---

## 四、数据持久化

容器删除后数据会丢失，需要用以下方式持久化：

### Volume（推荐）
```bash
# 创建数据卷
docker volume create mydata

# 挂载数据卷
docker run -v mydata:/var/lib/mysql mysql

# 查看数据卷
docker volume ls
```

### Bind Mount（绑定挂载）
```bash
# 直接映射宿主机目录
docker run -v /home/user/data:/app/data myapp
```

---

## 五、网络

```bash
# 查看网络列表
docker network ls

# 创建自定义网络
docker network create mynet

# 容器加入网络（同一网络内的容器可通过容器名互相访问）
docker run --network mynet --name app1 myapp
docker run --network mynet --name app2 myapp
# app1 和 app2 可以通过容器名互访
```

---

## 六、国内镜像加速

镜像源配置文件：`/etc/docker/daemon.json`

```json
{
  "registry-mirrors": [
    "https://docker.1ms.run",
    "https://hub.rat.dev"
  ]
}
```

修改后执行 `sudo systemctl restart docker` 生效。
