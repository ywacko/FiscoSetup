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

把下面 8 个文件整体交给对方：

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

## 10. TeeGateway 对接 FISCO 所需配置

如果后续要在同一台机器上部署 `TeeGateway`，安装完 FISCO 后，至少需要把下面这些配置提供给网关。

以下默认假设当前 FISCO 部署路径为：

```text
/home/ywacko/work/fisco-air-gm-3.7.3
```

### 10.1 网关需要的 FISCO 配置项

#### 10.1.1 非路径配置

- `FISCO_GROUP_ID`
  说明：FISCO 群组标识。
  示例值：`group0`
- `FISCO_PEER`
  说明：建议填写 `node0` 的 RPC 地址。
  示例值：`127.0.0.1:20200`
- `FISCO_USE_SM_CRYPTO`
  说明：是否启用国密模式。
  示例值：`true`
- `FISCO_DISABLE_SSL`
  说明：是否关闭 SSL。
  示例值：`false`
- `FISCO_TDS_UUPS_ADDRESS`
  说明：不是装链时自动生成的值，需要在后续部署 `TDS_UUPS` 合约后填写实际地址。
  示例值：`0x435407f2be59a102edc42077027b92093349d4c7`
- `FISCO_MESSAGE_TIMEOUT_MS`
  说明：网关侧链调用超时时间，通常可直接使用默认值。
  示例值：`10000`

#### 10.1.2 宿主机路径变量

- `HOST_FISCO_SDK_DIR`
  说明：如果后续通过 `docker-compose` 方式部署 `TeeGateway`，这里填写宿主机上的 SDK 证书目录。
  示例值：`/home/ywacko/work/fisco-air-gm-3.7.3/nodes/127.0.0.1/sdk`
- `HOST_FISCO_ACCOUNT_DIR`
  说明：如果后续通过 `docker-compose` 方式部署 `TeeGateway`，这里填写宿主机上的账户 `PEM` 目录。
  示例值：`/home/ywacko/work/fisco-air-gm-3.7.3/nodes/ca/accounts_gm`

#### 10.1.3 容器内读取路径变量

当前这些容器内路径和宿主机路径看起来很像，这是刻意为之：`docker-compose.yml` 现在把 SDK 证书目录直接挂到容器内同名路径，而账户 `PEM` 则挂到当前约定的 `console/account/gm` 读取路径，目的是减少网关内额外的路径换算。

- `FISCO_CA_CERT`
  说明：当前 `docker-compose.yml` 挂载后，容器内读取的 CA 证书路径。
  示例值：`/home/ywacko/work/fisco-air-gm-3.7.3/nodes/127.0.0.1/sdk/sm_ca.crt`
- `FISCO_SSL_CERT`
  说明：当前 `docker-compose.yml` 挂载后，容器内读取的 SDK 证书路径。
  示例值：`/home/ywacko/work/fisco-air-gm-3.7.3/nodes/127.0.0.1/sdk/sm_sdk.crt`
- `FISCO_SSL_KEY`
  说明：当前 `docker-compose.yml` 挂载后，容器内读取的 SDK 私钥路径。
  示例值：`/home/ywacko/work/fisco-air-gm-3.7.3/nodes/127.0.0.1/sdk/sm_sdk.key`
- `FISCO_EN_SSL_CERT`
  说明：当前 `docker-compose.yml` 挂载后，容器内读取的国密加密证书路径。
  示例值：`/home/ywacko/work/fisco-air-gm-3.7.3/nodes/127.0.0.1/sdk/sm_ensdk.crt`
- `FISCO_EN_SSL_KEY`
  说明：当前 `docker-compose.yml` 挂载后，容器内读取的国密加密私钥路径。
  示例值：`/home/ywacko/work/fisco-air-gm-3.7.3/nodes/127.0.0.1/sdk/sm_ensdk.key`
- `FISCO_ACCOUNT_PEM`
  说明：当前 `docker-compose.yml` 挂载后，容器内读取的账户 `PEM` 路径。
  示例值：`/home/ywacko/fisco-console-3.7.0/console/account/gm/<账户地址>.pem`

### 10.2 安装完成后去哪里找这些文件

SDK 证书目录：

```bash
ls -lh /home/ywacko/work/fisco-air-gm-3.7.3/nodes/127.0.0.1/sdk
```

应能看到：

- `sm_ca.crt`
- `sm_sdk.crt`
- `sm_sdk.key`
- `sm_ensdk.crt`
- `sm_ensdk.key`
- `sm_sdk.nodeid`

GM 账户目录：

```bash
ls -lh /home/ywacko/work/fisco-air-gm-3.7.3/nodes/ca/accounts_gm
```

这里的 `*.pem` 就是可供网关使用的宿主机侧账户材料。

例如：

```text
/home/ywacko/work/fisco-air-gm-3.7.3/nodes/ca/accounts_gm/0xfb4f7a22706f05a9b004cf28fdb362b9bff04cab.pem
```

### 10.3 安装完成后如何确认端口

当前 4 节点默认 RPC 端口固定为：

- `node0 -> 127.0.0.1:20200`
- `node1 -> 127.0.0.1:20201`
- `node2 -> 127.0.0.1:20202`
- `node3 -> 127.0.0.1:20203`

可以直接检查：

```bash
ss -ltn | grep -E '2020[0-3]'
```

也可以分别查看各节点配置：

```bash
sed -n '/\[rpc\]/,/^\[/p' /home/ywacko/work/fisco-air-gm-3.7.3/nodes/127.0.0.1/node0/config.ini
sed -n '/\[rpc\]/,/^\[/p' /home/ywacko/work/fisco-air-gm-3.7.3/nodes/127.0.0.1/node1/config.ini
sed -n '/\[rpc\]/,/^\[/p' /home/ywacko/work/fisco-air-gm-3.7.3/nodes/127.0.0.1/node2/config.ini
sed -n '/\[rpc\]/,/^\[/p' /home/ywacko/work/fisco-air-gm-3.7.3/nodes/127.0.0.1/node3/config.ini
```

如果后续已把 4 节点正式缩容成 1 节点，网关仍建议继续指向：

```text
127.0.0.1:20200
```
