# FISCO 3.7.3 4 节点 Docker 离线部署说明

目标是让接收方在一台新机器上，仅凭当前目录中的随包文件，按文档命令离线拉起 `FISCO BCOS 3.7.3` 国密 4 节点。

固定参数：

- `FISCO BCOS 3.7.3`
- 国密链
- 4 节点
- `group0`
- `chain0`
- 原生端口：`20200-20203`、`30300-30303`
- `min_seal_time=2000`
- `block_tx_count_limit=4000`
- `txpool.limit=60000`
- 镜像：`fisco-bcos-air-gm:3.7.3`

## 1. 交付物

把下面 9 个文件整体交给对方：

- `setup_fisco_docker_4node.sh`
- `start_fisco_docker_4node.sh`
- `status_fisco_docker_4node.sh`
- `stop_fisco_docker_4node.sh`
- `build_chain.sh`
- `get_gm_account.sh`
- `tassl-1.1.1b`
- `fisco-air-gm-repro-3.7.3.tar`

## 2. 接收方前置条件

接收方机器只需要准备 Docker、Compose、`tar`。

```bash
sudo apt-get update
sudo apt-get install -y docker.io tar
sudo apt-get install -y docker-compose-v2 || sudo apt-get install -y docker-compose
sudo systemctl enable --now docker
docker --version
docker compose version || docker-compose --version
```

如果当前用户没有 Docker 权限，执行：

```bash
sudo usermod -aG docker "$USER"
```

然后重新登录，确认当前 shell 已经拿到 Docker 权限：

```bash
groups
docker info >/dev/null && echo "docker access ok"
```

## 3. 先确保宿主机没有旧 FISCO 占用原生端口

当前包固定使用原生端口，所以启动前先确认宿主机上没有旧 FISCO：

```bash
docker ps -a | grep fisco-air-gm || true
ss -ltn | grep -E '2020[0-3]|3030[0-3]' || true
```

如果旧容器还在，先停掉再继续。

## 4. 导入离线镜像

```bash
docker load -i fisco-air-gm-repro-3.7.3.tar
docker image inspect fisco-bcos-air-gm:3.7.3 >/dev/null && echo "image ok"
```

## 5. 初始化整套 4 节点环境

给脚本加执行权限：

```bash
chmod +x setup_fisco_docker_4node.sh
chmod +x start_fisco_docker_4node.sh
chmod +x status_fisco_docker_4node.sh
chmod +x stop_fisco_docker_4node.sh
chmod +x build_chain.sh
chmod +x get_gm_account.sh
chmod +x tassl-1.1.1b
```

执行初始化：

```bash
bash setup_fisco_docker_4node.sh
```

默认工作目录：

```text
$HOME/work/fisco-air-gm-3.7.3
```

如果要指定工作目录，例如 `/data/fisco-air-gm-3.7.3`：

```bash
bash setup_fisco_docker_4node.sh /data/fisco-air-gm-3.7.3
```

初始化脚本会自动完成：

- 把随包下发的 `build_chain.sh` 和 `get_gm_account.sh` 复制到工作目录
- 把随包下发的 `tassl-1.1.1b` 挂载到容器内
- 在 `fisco-bcos-air-gm:3.7.3` 容器内执行 `build_chain.sh`
- 生成 4 节点国密链目录
- 把 `min_seal_time` 改成 `2000`
- 把 `block_tx_count_limit` 改成 `4000`
- 把 `txpool.limit` 改成 `60000`
- 生成 `docker-compose.yml`
- 生成 `container/up.sh`
- 生成 `container/down.sh`
- 生成 `container/status.sh`

## 6. 启动链

```bash
bash start_fisco_docker_4node.sh
```

如果初始化时用了自定义目录，例如 `/data/fisco-air-gm-3.7.3`：

```bash
bash start_fisco_docker_4node.sh /data/fisco-air-gm-3.7.3
```

## 7. 查看状态

```bash
bash status_fisco_docker_4node.sh
```

也可以直接看容器：

```bash
docker ps --format '{{.Names}}' | grep -E '^fisco-air-gm-node[0-3]$'
```

看端口：

```bash
ss -ltn | grep -E '2020[0-3]|3030[0-3]'
```

## 8. 停止链

```bash
bash stop_fisco_docker_4node.sh
```

如果初始化时用了自定义目录，例如 `/data/fisco-air-gm-3.7.3`：

```bash
bash stop_fisco_docker_4node.sh /data/fisco-air-gm-3.7.3
```

## 9. 常用排查

查看全部相关容器：

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

如果初始化时报：

```text
required local image not found: fisco-bcos-air-gm:3.7.3
```

说明接收方机器还没有完成离线镜像导入，先重新执行第 4 步。
