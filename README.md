# ShowShow 🕵️

> 训练慢了？中断了？ShowShow一下，端网一起查，根因直接给你秀出来。

GPU Fabric 网络端到端诊断工具，针对已经使用锐捷TH5设备以及ONC运维管理平台的用户，支持 AILB 路径还原、PFC/ECN 全链路指标分析、GPU/NIC 亲和关系检查。

---

## 快速开始

### 部署（离线环境）

```bash
# 在有网络的机器上打包
docker build -t showshow:latest .
docker save showshow:latest | gzip > showshow.tar.gz

# 传到目标机器
scp showshow.tar.gz root@<target>:/root/
scp config.yaml root@<target>:/root/showshow/

# 目标机器上加载
docker load < showshow.tar.gz

# 设置别名（必须加 COLUMNS 参数，否则输出会被截断）
echo "alias showshow='docker run --rm -it --network host \
  -e COLUMNS=\$(tput cols) \
  -e TERM=xterm-256color \
  -v /root/showshow/config.yaml:/root/.showshow/config.yaml \
  -v /root/showshow/showshow:/app/showshow \
  showshow:latest'" >> ~/.bashrc
source ~/.bashrc
```

### 验证安装

```bash
showshow --help
```

---

## 命令说明

### showshow path — 路径查询

查看两台 GPU 服务器之间的 AILB 转发路径，验证路径是否符合预期。

```bash
showshow path --src <src_roce_ip> --dst <dst_roce_ip>
```

**输出示例：**
```
📍 完整路径：
  YKJJ-33 (→ FH0/18:1)
  ↓
  LEAF1 (FH0/18:1 → FH0/49) [AILB member-id=17]
  ↓
  SPINE5 (FH0/1 → FH0/6)
  ↓
  LEAF2 (FH0/50 → FH0/17:2) [AILB member-id=18]
  ↓
  YKJJ-44 (← FH0/17:2)
```

**AILB 异常时**会同时显示理论路径（配置期望）和实际路径（路由表），帮助定位哪条链路 DOWN 了。

---

### showshow diagnose — 端到端故障诊断

最核心的命令。输入两个节点 IP，输出完整的端到端诊断报告，包含：

- AILB 路径还原
- GPU/NIC 亲和关系（PCIe 拓扑）
- 全链路 PFC/ECN/丢包指标（NIC → LEAF → SPINE → LEAF → NIC）

```bash
# 查最近1小时
showshow diagnose --src <src_roce_ip> --dst <dst_roce_ip>

# 查历史故障时间点（前后30分钟）
showshow diagnose --src <src_roce_ip> --dst <dst_roce_ip> --time "2026-04-25 14:00"

# 不查PCIe拓扑（更快）
showshow diagnose --src <src_roce_ip> --dst <dst_roce_ip> --no-pcie
```

**输出包含三部分：**

**① GPU/NIC 拓扑**
```
🔌 GPU/NIC拓扑 [172.18.11.45]：
  GPU0 Gen4x16 ✓ → mlx5_2 [✓ 优(PXB)]
  GPU1 Gen4x16 ✓ → mlx5_2 [✓ 优(PXB)]
  ...
  NIC↔GPU亲和关系：
  mlx5_2: GPU0:PXB GPU1:PXB GPU2:NODE ...
```

**② 完整路径（含网卡信息）**
```
  YKJJ-33 (→ FH0/18:1) (mlx5_2-ens24np0 GPU0 PXB-Good OK)
  ↓
  LEAF1 (FH0/18:1 → FH0/49) [AILB member-id=17]
  ...
```

**③ 全链路指标表**
```
┃ 节点    ┃ 方向 ┃ 端口             ┃ PFC发送率 ┃ PFC接收率 ┃ ECN速率 ┃ TX丢包 ┃ 状态 ┃
│ node33  │ OUT→ │ mlx5_2(ens24np0) │         0 │         0 │       0 │      0 │  ✓   │
│ LEAF1   │ ←IN  │ FH0/18:1         │         0 │         0 │       0 │      0 │  ✓   │
│ LEAF1   │ OUT→ │ FH0/49           │         0 │         0 │       0 │      0 │  ✓   │
│ SPINE5  │ ←IN  │ FH0/1            │         0 │         0 │       0 │      0 │  ✓   │
│ SPINE5  │ OUT→ │ FH0/6            │         0 │         0 │       0 │      0 │  ✓   │
│ LEAF2   │ ←IN  │ FH0/50           │         0 │         0 │       0 │      0 │  ✓   │
│ LEAF2   │ OUT→ │ FH0/17:2         │         0 │         0 │       0 │      0 │  ✓   │
│ node44  │ ←IN  │ mlx5_3(ens25np0) │         0 │         0 │       0 │      0 │  ✓   │
```

---

### showshow inspect — 集群巡检

预防性检查主机和网络配置，包括：CPU 高性能模式、iommu、nouveau 驱动、GPU ECC 跳变、PCIe 降速。

```bash
# 全量巡检
showshow inspect

# 指定节点
showshow inspect --nodes 172.18.11.45,172.18.11.46

# 只查主机侧
showshow inspect --scope host

# 只查网络侧
showshow inspect --scope network
```

---

### showshow tools get-server-roce-ip — 扫描服务器RoCE IP

自动 SSH 到所有服务器，获取 RoCE 网卡 IP，更新到 config.yaml 的 servers 映射。

```bash
# 扫描并打印
showshow tools get-server-roce-ip

# 扫描并自动更新 config.yaml
showshow tools get-server-roce-ip --update

# 保存到指定文件
showshow tools get-server-roce-ip --output /tmp/servers.yaml
```

---

## 配置文件说明

配置文件位于 `~/.showshow/config.yaml`（或通过挂载指定路径）。

```yaml
onc:
  host: 172.18.1.245     # ONC控制器IP
  port: 443              # HTTPS用443，HTTP用18080
  timeout: 30

ssh:
  default_user: root
  default_password: ""
  default_port: 22
  nodes:
    # 交换机凭证（覆盖默认值）
    172.18.11.45:
      user: admin
      password: "your_password"
    # 服务器凭证
    172.18.11.246:
      user: root
      password: "your_password"

network:
  roce_priority: 3        # RoCE流量的PFC Priority（P3或P4）
  config_cache_ttl: 300   # 交换机配置缓存TTL（秒）

  leaf_ips:               # LEAF交换机管理IP列表
    - 172.18.11.145
    - 172.18.12.145

  spine_ips:              # SPINE交换机管理IP列表
    - 172.18.13.245
    - 172.18.14.245

  mgmt_servers:           # 服务器管理IP → ONC hostname映射
    172.18.20.45: YKJJ-20-45.YYos.org
    172.18.21.46: YKJJ-21-46.YYos.org

  servers:                # 服务器网卡CX6/7/8 RoCE IP → ONC hostname映射
    # 用 showshow tools get-server-roce-ip --update 自动填充
    172.18.10.45: YKJJ-33-45.YYos.org
    172.18.11.45: YKJJ-33-46.YYos.org
```

**三套IP区分：**

| 配置项 | IP类型 | 用途 |
|--------|--------|------|
| `ssh.nodes` | 管理IP（172.18.13.x） | SSH连接执行命令 |
| `mgmt_servers` | 管理IP（172.18.20.x） | get-server-roce-ip扫描 |
| `servers` | RoCE IP（172.18.10.x） | path/diagnose路径查询 |

---

## 技术原理

### showshow path 数据来源

| 步骤 | 数据来源 | 接口/命令 |
|------|---------|-----------|
| 找server连的leaf | ONC API | `GET /api/topo-srv/v1/topology/...` |
| 解析AILB member-id | 交换机SSH | `show run` |
| 找spine端口 | 交换机SSH | `show lldp neighbors` |
| 验证实际路由 | 交换机SSH | `show ip route <dst>` |
| 找spine管理IP | ONC API | InternalLink反查 |

### showshow diagnose 额外数据来源

| 步骤 | 数据来源 | 接口/命令 |
|------|---------|-----------|
| GPU/NIC亲和 | 服务器SSH | `nvidia-smi topo -m` |
| PCIe带宽 | 服务器SSH | `nvidia-smi --query-gpu=...` |
| ens→mlx5映射 | 服务器SSH | `rdma link` |
| leaf端口→网卡名 | ONC API | InternalLink |
| 交换机PFC/ECN指标 | ONC API | `POST /api/ne-monitor/v1/deviceIndicatorData/...` |
| NIC PFC/ECN指标 | ONC API | 同上，port=ens24np0 |

---

## 常见问题

**Q: 显示"ONC拓扑中找不到 xxx 连接的Leaf"**
A: config.yaml 的 `servers` 段没有该RoCE IP，运行 `showshow tools get-server-roce-ip --update` 更新。

**Q: 显示"路由黑洞"**
A: 源leaf上没有到目标服务器的路由，可能是服务器离线或网络故障。

**Q: 显示"AILB主路径异常，已fallback到ECMP"**
A: AILB计算的理论路径和路由表实际路径不一致，通常是某条spine上行链路DOWN了，查指标表里对应端口是否有异常。

**Q: 重启后dockerd没了**
A: 该环境/etc/group只读，无法用systemd管理docker，每次重启后需要手动运行 `dockerd &`。

**Q: 输出被截断**
A: alias里缺少 `-e COLUMNS=$(tput cols)`，重新设置alias。
