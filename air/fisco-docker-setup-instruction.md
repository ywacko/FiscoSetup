# FISCO 4 节点 Docker 使用说明

这套交付物不再要求接收方手工一条条敲命令，直接执行脚本即可。

目标参数：

- 国密链
- 4 节点
- `group0`
- `chain0`
- `min_seal_time=2000`
- `block_tx_count_limit=4000`
- `txpool.limit=60000`
- 镜像：`ywackoo/fisco-air-gm-repro:3.0`

## 1. 交付物

把下面 7 个交付物交给对方：

- `setup_fisco_docker_4node.sh`
- `start_fisco_docker_4node.sh`
- `status_fisco_docker_4node.sh`
- `stop_fisco_docker_4node.sh`
- `build_chain.sh`
- `tassl-1.1.1b`
- `fisco-air-gm-repro-3.0.tar`

## 2. 接收方前置条件

接收方先安装 Docker、`curl`、`tar`，再安装可用的 Compose：

```bash
sudo apt-get update
sudo apt-get install -y docker.io curl tar
sudo apt-get install -y docker-compose-v2 || sudo apt-get install -y docker-compose
sudo systemctl enable --now docker
docker --version
docker compose version || docker-compose --version
```

说明：

- 有些 Ubuntu 默认仓库没有 `docker-compose-plugin`
- 这时改装 `docker-compose-v2` 或 `docker-compose` 都可以
- 当前脚本已经同时兼容 `docker compose` 和 `docker-compose`

如果当前用户没有 Docker 权限，执行：

```bash
sudo usermod -aG docker "$USER"
```

按当前 Ubuntu 测试机的实测结果，这一步之后直接开新终端还不够，先重启机器，再继续下面检查：

```bash
sudo reboot
```

重启并重新登录以后，先确认当前 shell 已经拿到 Docker 权限：

```bash
groups
docker info >/dev/null && echo "docker access ok"
```

正常情况下，`groups` 里应包含 `docker`。

如果这里仍然报：

```text
permission denied while trying to connect to the Docker daemon socket
```

说明当前会话仍然没有拿到新的用户组权限，先重新登录，再重复执行 `groups` 和 `docker info`。

## 3. 初始化整套 4 节点环境

先手动准备本地镜像。

当前统一使用离线方式，这条路径已经在 Ubuntu 测试机上实际验证过。

```bash
docker load -i fisco-air-gm-repro-3.0.tar
docker tag ywackoo/fisco-air-gm-repro:3.0 fisco-bcos-air-gm:3.0
```

确认本地镜像已经就绪：

```bash
docker image inspect fisco-bcos-air-gm:3.0 >/dev/null && echo "image ok"
```

然后再继续执行下面的初始化脚本。

给 4 个脚本和 `build_chain.sh` 加执行权限：

```bash
chmod +x setup_fisco_docker_4node.sh
chmod +x start_fisco_docker_4node.sh
chmod +x status_fisco_docker_4node.sh
chmod +x stop_fisco_docker_4node.sh
chmod +x build_chain.sh
```

执行初始化脚本：

```bash
bash setup_fisco_docker_4node.sh
```

不传参数时，默认工作目录是：

```text
$HOME/work/fisco-air-gm-3.0
```

如果要指定工作目录，如/data/fisco-air-gm-3.0，执行：

```bash
bash setup_fisco_docker_4node.sh /data/fisco-air-gm-3.0
```

初始化脚本会自动完成下面这些事情：

- 创建工作目录
- 优先使用随包下发的 `build_chain.sh`
- 如果当前用户目录下没有 `~/.fisco/tassl-1.1.1b`，优先使用随包下发的 `tassl-1.1.1b`
- 从镜像中提取 `fisco-bcos`
- 生成 4 节点国密链目录
- 把 `min_seal_time` 改成 `2000`
- 把 `block_tx_count_limit` 改成 `4000`
- 把 `txpool.limit` 改成 `60000`
- 生成 `docker-compose.yml`
- 生成 `container/entrypoint.sh`
- 生成 `container/up.sh`
- 生成 `container/down.sh`
- 生成 `container/status.sh`


## 4. 启动链

执行：

```bash
bash start_fisco_docker_4node.sh
```

不传参数时，默认使用：

```text
$HOME/work/fisco-air-gm-3.0
```

如果初始化时用了自定义目录，如/data/fisco-air-gm-3.0，执行：

```bash
bash start_fisco_docker_4node.sh /data/fisco-air-gm-3.0
```

## 5. 查看状态

执行：

```bash
bash status_fisco_docker_4node.sh
```

不传参数时，默认使用：

```text
$HOME/work/fisco-air-gm-3.0
```

如果初始化时用了自定义目录，执行：

```bash
bash status_fisco_docker_4node.sh /data/fisco-air-gm-3.0
```

也可以直接检查容器：

```bash
docker ps --format '{{.Names}}' | grep -E '^fisco-air-gm-node[0-3]$'
```

## 6. 停止链

执行：

```bash
bash stop_fisco_docker_4node.sh
```

不传参数时，默认使用：

```text
$HOME/work/fisco-air-gm-3.0
```

如果初始化时用了自定义目录，执行：

```bash
bash stop_fisco_docker_4node.sh /data/fisco-air-gm-3.0
```

## 7. 常用排查命令

如果启动阶段报下面这种错误：

```text
failed to resolve source metadata for docker.io/library/ubuntu:24.04
```

说明当前工作目录里的启动文件还是旧版本，还在尝试本地 build。使用最新的 `setup_fisco_docker_4node.sh` 重新执行一次初始化，再重新启动。

如果初始化阶段报下面这种错误：

```text
required local image not found: fisco-bcos-air-gm:3.0
```

说明当前机器还没有准备好本地镜像。先按上面的离线镜像导入方式完成镜像导入，再重新执行 `bash setup_fisco_docker_4node.sh`。

查看全部容器：

```bash
docker ps -a | grep fisco-air-gm
```

查看 `node0` 日志：

```bash
docker logs -f fisco-air-gm-node0
```

查看 `node1` 日志：

```bash
docker logs -f fisco-air-gm-node1
```

查看端口：

```bash
ss -ltn | grep -E '2020[0-3]|3030[0-3]'
```
