# powerxio-etcd

3 节点 etcd 集群，使用静态引导。每个节点的数据保存在独立的 Docker volume 中。

## 两台机器部署（A 上两个节点，B 上一个节点）

将此目录放到两台机器，在两台机器上都复制 `.env.example` 为 `.env`，把 `ETCD_HOST_A_IP`、`ETCD_HOST_B_IP` 换成真实、互相可访问的宿主机 IP。两台机器的 IP 配置必须相同。无需修改 `/etc/hosts`，节点使用 IP 互连。

`ETCD_DATA_ROOT` 指定各机器上的宿主机数据根目录，示例为 `/data/powerxio-etcd`。机器 A 使用其下的 `etcd-1`、`etcd-2`，机器 B 使用 `etcd-3`；容器内统一挂载到 `/etcd-data`。可以在两台机器分别设置不同的数据根目录。Docker Compose 会创建尚不存在的宿主机目录。

| 机器 | 节点 | 宿主机客户端端口 | 宿主机节点通信端口 |
| --- | --- | --- | --- |
| A | `powerxio-etcd-1` | `2379` | `2380` |
| A | `powerxio-etcd-2` | `12379` | `12380` |
| B | `powerxio-etcd-3` | `2379` | `2380` |

机器 A 执行：

```sh
cp .env.example .env
# 将 .env 中的两个示例 IP 和 ETCD_DATA_ROOT 改为实际值
docker compose -f compose.multi-host.yaml --profile host-a up -d --build
docker compose -f compose.multi-host.yaml --profile host-a ps
```

机器 B 执行（使用与 A 相同的 `.env` 内容）：

```sh
cp .env.example .env
# 将 .env 中的两个示例 IP 和 ETCD_DATA_ROOT 改为实际值
docker compose -f compose.multi-host.yaml --profile host-b up -d --build
docker compose -f compose.multi-host.yaml --profile host-b ps
```

集群检查（在机器 A 上执行，IP 替换为实际值）：

```sh
docker compose -f compose.multi-host.yaml --profile host-a exec powerxio-etcd-1 /usr/local/bin/etcdctl \
  --endpoints=http://192.168.1.11:2379,http://192.168.1.11:12379,http://192.168.1.12:2379 \
  endpoint health
```

两台机器之间需允许 TCP `2380` 和 `12380` 通信；客户端按需访问 TCP `2379` 和 `12379`。此示例使用明文 HTTP，只适合受信任的内网，请用防火墙限制访问来源。

3 个 etcd 成员的多数派是 2。机器 B 故障时，A 上的 2 个成员仍能提供服务；机器 A 故障时，集群失去多数派，无法继续写入。若要容忍任意一台机器故障，需要第三台独立机器运行一个成员。

上述 IP 仅为示例。各节点首次启动时应使用相同的集群成员列表；已有数据卷的节点重启时会读取原有成员信息。

## 单台机器部署（同机三节点）

```sh
docker compose up -d --build
docker compose ps
docker compose exec powerxio-etcd-1 /usr/local/bin/etcdctl \
  --endpoints=http://powerxio-etcd-1:2379,http://powerxio-etcd-2:2379,http://powerxio-etcd-3:2379 \
  endpoint health
```

在同一个 Docker 网络 `powerxio-network` 中，客户端使用以下地址连接：

```text
http://powerxio-etcd-1:2379,http://powerxio-etcd-2:2379,http://powerxio-etcd-3:2379
```

宿主机仅在本机监听 `32379`、`12379`、`22379`，可用于本机调试。节点对外通告的是 Docker 网络内的名称；需要从其他机器连接时，应另行配置可从客户端和节点互相访问的地址及 TLS。

`docker compose down` 会保留数据卷。只有在确定要删除所有集群数据时才使用 `docker compose down -v`。
