---
name: ceph-docs-reference
description: >
  Ceph 实操参考技能 — 基于真实实验环境的完整经验库。
  回答 Ceph 问题、规划部署、排查故障、进行实验、编写配置时使用。
  官方文档来源: https://docs.ceph.net.cn/en/latest/
  触发词: Ceph, 手动部署, ceph-volume, OSD, PG, pool, RBD, CephFS, MDS,
  CRUSH, SELinux, firewall, OVS, KVM, libvirt, rsyslog, 灾备恢复, 已知坑,
  HEALTH_WARN, clock skew, mon probing, OSD down, PG unknown
---

# Ceph 实操参考 Skill

## 铁律

**回答优先级：本地实验验证 > 官方文档 > 推测。**

1. **实操验证优先**：本 skill 包含在 OpenCloudOS 8.10 + Ceph Pacific 16.2.15 上经过实际部署验证的知识。当用户环境匹配时，优先使用这些经过验证的方案。
2. **官方文档为底线**：本地经验未覆盖的场景，查阅 `references/` 目录或用 `WebFetch` 获取 https://docs.ceph.net.cn/en/latest/ 官方原文。
3. **不编造**：不允许编造、猜测或引用文档/实验中不存在的内容。
4. **标注来源**：回答末尾标注信息来源。实操经验标注 `[已验证-实验N]`，官方内容标注 `[官方文档 - 章节名](URL)`。
5. **环境感知**：回答时先确认用户系统（OpenCloudOS/RHEL/Ubuntu）和 Ceph 版本（Pacific/Quincy/Reef），不同环境下命令和行为可能不同。
6. **参数变量化**：回答中涉及用户环境时，IP、节点名、路径、版本号等必须用变量/占位符表示，提示用户替换（见下方原则）。
7. **高危操作阻断**：遇到 `ceph-volume lvm zap`（销毁 OSD）、`ceph osd pool delete`（删除存储池）、`ceph osd purge`（清除 OSD）等破坏性命令时，必须在输出前使用显眼警告提示用户二次确认。
8. **排障先查日志**：排查故障时不要盲目推测。应优先要求用户提供 `ceph -s` 状态，并查看组件日志（`journalctl -u ceph-mon@<节点名> -e` 或 `/var/log/ceph/ceph-osd.<ID>.log`）。

---

## 参数变量化原则

回答中涉及用户环境时，必须遵循：

| 参数类型 | ❌ 错误示例 (硬编码) | ✅ 正确示例 (变量化) |
|---------|-------------------|-------------------|
| IP 地址 | `192.168.48.129` | `<管理IP>` 或 `{mon_ip}` |
| 节点名 | `node129` | `<节点名>` 或 `{node1}` |
| 路径 | `/mnt/kvm_logs/` | `<日志目录>` 或 `{log_path}` |
| 版本号 | `16.2.15` | `<Ceph 版本>` 或 `{ceph_version}` |
| OSD 数量 | `6 OSD` | `<OSD 总数>` 或 `{osd_count}` |
| 网段 | `192.168.48.0/24` | `<管理网络 CIDR>` 或 `{public_network}` |
| FSID | `a8aa9e2a-...` | `<集群 FSID>` 或 `{fsid}` |

**格式约定：**
- 首次出现用中文描述：`<管理IP>`
- 同一参数再次出现时可用变量名：`{mon_ip}`
- 在回答末尾汇总所有变量，方便用户对照替换

---

## 用户实验环境（参考示例）

> 本 skill 基于以下环境验证。**所有 IP、节点名、FSID 均为实际值展示，你在使用时必须替换为自己的参数。**

```
宿主机: Windows/Linux + VMware/VirtualBox/Proxmox
OS:     Linux 发行版 (如 OpenCloudOS/RHEL/Rocky/Ubuntu, 内核 5.4+)
Ceph:   Pacific 16.2.x / Quincy / Reef / Squid (手动部署)
OVS:    3.x LTS (源码编译)

集群: N 节点，M 个网段（管理网 + 存储私有网）
┌─────────┬────────────────┬─────────────────┬──────────────────────────────────┐
│  节点   │  管理 IP       │  存储私网 IP    │  角色                            │
├─────────┼────────────────┼─────────────────┼──────────────────────────────────┤
│ node1   │ <管理网段.1>   │ <存储网段.1>    │ mon, mgr, osd.0/1, (可选 KVM)    │
│ node2   │ <管理网段.2>   │ <存储网段.2>    │ mon, osd.2/3                     │
│ node3   │ <管理网段.3>   │ <存储网段.3>    │ mon, osd.4/5                     │
│ ...     │ ...            │ ...             │ 更多节点...                      │
└─────────┴────────────────┴─────────────────┴──────────────────────────────────┘

磁盘: 每节点 N 块数据盘 (HDD/SSD) + 可选 NVMe 做 BlueStore WAL/DB
网络: 管理桥 + 存储/VM 桥 (Linux bridge / OVS)
池:   根据用途规划 PG 数量 (建议 (OSD总数 × 100) / 副本数)
CephFS: 可选用于日志/共享存储
```

---

## 文档结构速查

官方文档顶级章节及对应 URL：

| 章节 | URL |
|------|-----|
| Ceph 简介 | https://docs.ceph.net.cn/en/latest/start/ |
| 安装 Ceph | https://docs.ceph.net.cn/en/latest/install/ |
| 手动安装 | https://docs.ceph.net.cn/en/latest/install/index_manual/ |
| Cephadm | https://docs.ceph.net.cn/en/latest/cephadm/ |
| Cephadm 安装 | https://docs.ceph.net.cn/en/latest/cephadm/install/ |
| Cephadm 运维 | https://docs.ceph.net.cn/en/latest/cephadm/operations/ |
| Cephadm 故障 | https://docs.ceph.net.cn/en/latest/cephadm/troubleshooting/ |
| 存储集群(RADOS) | https://docs.ceph.net.cn/en/latest/rados/ |
| RADOS 配置 | https://docs.ceph.net.cn/en/latest/rados/configuration/ |
| RADOS 运维 | https://docs.ceph.net.cn/en/latest/rados/operations/ |
| RADOS 故障 | https://docs.ceph.net.cn/en/latest/rados/troubleshooting/ |
| RBD 块设备 | https://docs.ceph.net.cn/en/latest/rbd/ |
| RBD 运维 | https://docs.ceph.net.cn/en/latest/rbd/rbd-operations/ |
| RBD 命令 | https://docs.ceph.net.cn/en/latest/rbd/rados-rbd-cmds/ |
| CephFS 文件系统 | https://docs.ceph.net.cn/en/latest/cephfs/ |
| CephFS 创建 | https://docs.ceph.net.cn/en/latest/cephfs/createfs/ |
| CephFS 管理 | https://docs.ceph.net.cn/en/latest/cephfs/administration/ |
| MDS 增删 | https://docs.ceph.net.cn/en/latest/cephfs/add-remove-mds/ |
| RADOSGW 网关 | https://docs.ceph.net.cn/en/latest/radosgw/ |
| RGW 管理 | https://docs.ceph.net.cn/en/latest/radosgw/admin/ |
| Mgr 管理器 | https://docs.ceph.net.cn/en/latest/mgr/ |
| Dashboard | https://docs.ceph.net.cn/en/latest/mgr/dashboard/ |
| 监控 | https://docs.ceph.net.cn/en/latest/monitoring/ |
| 安全 | https://docs.ceph.net.cn/en/latest/security/ |
| 架构 | https://docs.ceph.net.cn/en/latest/architecture/ |
| ceph-volume | https://docs.ceph.net.cn/en/latest/ceph-volume/ |
| 版本说明 | https://docs.ceph.net.cn/en/latest/releases/ |
| 术语表 | https://docs.ceph.net.cn/en/latest/glossary/ |

### RADOS 子章节 (rados/operations/)

| 子页面 | URL |
|--------|-----|
| 监控器 | https://docs.ceph.net.cn/en/latest/rados/operations/monitoring/ |
| OSD | https://docs.ceph.net.cn/en/latest/rados/operations/osd/ |
| PG | https://docs.ceph.net.cn/en/latest/rados/operations/placement-groups/ |
| 纠删码 | https://docs.ceph.net.cn/en/latest/rados/operations/erasure-code/ |
| CRUSH | https://docs.ceph.net.cn/en/latest/rados/operations/crush-map/ |
| 添加/删除 OSD | https://docs.ceph.net.cn/en/latest/rados/operations/add-or-rm-osds/ |
| 数据均衡 | https://docs.ceph.net.cn/en/latest/rados/operations/balancer/ |
| 数据校验 | https://docs.ceph.net.cn/en/latest/rados/operations/data-check/ |
| 用户管理 | https://docs.ceph.net.cn/en/latest/rados/operations/user-management/ |
| 缓存分层 | https://docs.ceph.net.cn/en/latest/rados/operations/cache-tiering/ |
| 清洗 | https://docs.ceph.net.cn/en/latest/rados/operations/scrub/ |
| 蓝屏/烈焰 | https://docs.ceph.net.cn/en/latest/rados/operations/blue-store-migration/ |
| 控制 | https://docs.ceph.net.cn/en/latest/rados/operations/control/ |

---

## 架构核心概念 (摘自官方文档)

**Ceph 存储集群** 基于 **RADOS**（Reliable Autonomic Distributed Object Store）。

**核心组件:**
- **Ceph Monitor (mon)** — 维护集群映射主副本，集群仲裁（奇数节点，防脑裂）
- **Ceph OSD Daemon (osd)** — 数据存储、复制、恢复、再均衡
- **Ceph Manager (mgr)** — 监控、编排、插件模块端点
- **Ceph Metadata Server (mds)** — CephFS 元数据管理

**CRUSH 算法:** 客户端和 OSD 使用 CRUSH 计算数据位置，无需中心查找表。CRUSH 使用智能数据复制确保弹性。

**数据存储:** 数据以 RADOS 对象存储在 OSD 上。默认 BlueStore 后端。对象 ID 集群唯一。

**集群映射:** 包含 5 个映射：Monitor Map、OSD Map、PG Map、CRUSH Map、MDS Map

**BlueStore 架构 (NVMe WAL/DB 加速):**
```
HDD (数据盘)          NVMe (高速盘)
┌──────────────┐      ┌──────────────┐
│  Object Data │      │  RocksDB DB  │ ← 元数据索引
│  (100-400KB) │      │  (WAL 日志)  │ ← 写前日志
└──────────────┘      └──────────────┘
     sda/sdb              nvme0n2
```

**DB 设备空间建议 [官方]:** 当将 WAL+DB 卸载到高速设备时，官方推荐 `block.db` 大小 **至少为块设备（HDD）容量的 2.5%**。具体负载类型有所差异：
- **RGW 负载**: 建议至少 4%（大量 omap 元数据）
- **RBD 负载**: 通常 1%–2% 即够
- 示例：1TB HDD → block.db ≥ 25GB（通用）/ ≥ 10GB（纯 RBD）/ ≥ 40GB（RGW）

**RocksDB 层级尺寸 [官方]:** 官方说明内部层级对应约 3GB、30GB、300GB 的 DB 尺寸可完全利用。Pacific 版本开始支持实验性动态层级（dynamic level），可更好地利用任意大小的 DB 设备。
> **实验参考 [已验证]:** 本实验环境每节点 2 块 25G HDD + 10G NVMe，以每 OSD 约 5G DB 空间运行。此配置适用于小规模验证环境，**生产环境请按百分比计算**。

常用部署命令：
```bash
ceph-volume lvm batch /dev/sda /dev/sdb --db-devices /dev/nvme0n2
```

---

## OpenCloudOS 8.10 适配指南

OpenCloudOS 8.10 是国产 Linux 发行版，基于 CentOS 8/RHEL 8 兼容内核（5.4.119），与标准 CentOS 8 有以下关键差异：

### YUM 仓库

| 问题 | 解决方案 |
|------|---------|
| 默认 repo `mirrors.opencloudos.tech` 返回 404 | `sed -i s/opencloudos.tech/scmu.edu.cn/g /etc/yum.repos.d/OpenCloudOS.repo` |
| 缺少 Plus 仓库 | 手动追加 `[Plus]` 段到 OpenCloudOS.repo |
| 修改后必须刷新缓存 | `dnf clean all && dnf makecache` |

### 包名差异

| 组件 | CentOS 8 名称 | OpenCloudOS 8.10 名称 |
|------|-------------|---------------------|
| NM-ovs 插件 | NetworkManager-ovs | NetworkManager-ovs（同） |
| SELinux 管理 | policycoreutils-python-utils | policycoreutils-python-utils（同） |
| 内核版本 | 4.18.x | 5.4.119-20.0009.34 |
| Python | python3 | python3.12（系统默认） |

### SELinux 注意事项

- OpenCloudOS 8.10 的 SELinux 策略与 RHEL 8 基本兼容
- Ceph 的 `ceph-selinux` 包提供 `ceph_t` 域，标准路径开箱即用
- OVS 安装后需手动 `restorecon -Rv /etc/openvswitch`（上下文从 `etc_t` → `openvswitch_rw_t`）
- rsyslog 写 CephFS 需自定义策略模块（`audit2allow` 工作流，见下方 SELinux 章节）

### 内核 5.4.119 已知问题

- `virtio_net` + `vhost-net` 在长连接 TCP 场景下可能出现 keepalive 异常
- KVM 虚拟机内运行 ceph-mgr 时，6-20 分钟后可能出现无声退化（PG unknown）
- **建议**：active mgr 始终运行在物理机上，VM 内 mgr 仅做 standby

---

## 手动部署快速参考

> 完整步骤见 `ceph-ovs-deploy` skill。此处为精简速查卡。

### 部署顺序

```
Phase A: 节点初始化（主机名、hosts、SELinux、防火墙、EPEL、Ceph repo、SSH 免密）
Phase B: OVS 部署（如需要）→ br0 + ovs-vm 双桥
Phase C: Monitor 集群（ceph.conf、密钥、monmap、mon 启动、mgr 启动）
Phase D: 处理告警（insecure global_id、msgr2、时钟同步）
Phase E: OSD 创建（ceph-volume lvm batch + NVMe WAL/DB）
Phase F: 存储池 + RBD 初始化
Phase G: KVM/libvirt 集成（可选）
Phase H: CephFS 日志架构（可选）
```

### 关键命令速查

```bash
# === 安装前提 ===
dnf install epel-release -y                          # 必须先装 EPEL
# Ceph Pacific 仓库配置（OpenCloudOS 8.10 示例，其他发行版请替换）
# dnf install -y centos-release-ceph-pacific
dnf install ceph-mon ceph-mgr ceph-osd lvm2 -y       # 主节点（含 dashboard）
dnf install ceph-mon ceph-osd lvm2 -y                 # 从节点

# === Monitor 初始化（主节点） ===
# 创建 bootstrap-osd keyring（必须先于 monmaptool 导入）
ceph-authtool --create-keyring /var/lib/ceph/bootstrap-osd/ceph.keyring --gen-key \
  -n client.bootstrap-osd --cap mon 'profile bootstrap-osd' --cap osd 'allow *'
ceph-authtool --create-keyring /tmp/ceph.mon.keyring --gen-key -n mon. --cap mon 'allow *'
ceph-authtool --create-keyring /etc/ceph/ceph.client.admin.keyring --gen-key -n client.admin \
  --cap mon 'allow *' --cap osd 'allow *' --cap mds 'allow *' --cap mgr 'allow *'
ceph-authtool /tmp/ceph.mon.keyring --import-keyring /etc/ceph/ceph.client.admin.keyring
ceph-authtool /tmp/ceph.mon.keyring --import-keyring /var/lib/ceph/bootstrap-osd/ceph.keyring
chown ceph:ceph /tmp/ceph.mon.keyring /tmp/monmap
monmaptool --create --add <节点名1> <管理IP1> --add <节点名2> <管理IP2> \
  --add <节点名3> <管理IP3> --fsid <FSID> /tmp/monmap
sudo -u ceph ceph-mon --mkfs -i <主节点主机名> --monmap /tmp/monmap --keyring /tmp/ceph.mon.keyring
systemctl enable --now ceph-mon@<主节点主机名>

# === ceph-mgr 启动 ===
ceph auth get-or-create client.mgr.<主节点主机名> mon 'allow *' osd 'allow *' mds 'allow *'
sudo -u ceph mkdir -p /var/lib/ceph/mgr/ceph-<主节点主机名>
ceph auth get-or-create client.mgr.<主节点主机名> | \
  sudo -u ceph tee /var/lib/ceph/mgr/ceph-<主节点主机名>/keyring
systemctl enable --now ceph-mgr@<主节点主机名>

# === OSD 创建 ===
ln -sf /var/lib/ceph/bootstrap-osd/ceph.keyring /etc/ceph/ceph.client.bootstrap-osd.keyring
ceph-volume lvm batch <数据盘1> <数据盘2> --db-devices <NVMe盘> --bluestore --yes

# === 存储池（实验示例，请根据实际调整 PG 数）===
# 实验环境 6 OSD × 3 副本，使用 32 PG 精简配置（每 OSD ~5 PG）
# 生产环境推荐启用 pg_autoscale_mode on，或按公式计算 PG 数
ceph osd pool create ds_ceph_vm 32 --autoscale-mode on
ceph osd pool application enable ds_ceph_vm rbd
rbd pool init ds_ceph_vm

# === CephFS（实验示例）===
ceph osd pool create kvm_log_data 32 --autoscale-mode on
ceph osd pool create kvm_log_metadata 32 --autoscale-mode on
ceph osd pool application enable kvm_log_data cephfs
ceph osd pool application enable kvm_log_metadata cephfs
ceph fs new cephfs_kvm_log kvm_log_metadata kvm_log_data
dnf install -y ceph-mds
sudo -u ceph mkdir -p /var/lib/ceph/mds/ceph-<主节点主机名>
ceph auth get-or-create mds.<主节点主机名> mon 'profile mds' mgr 'profile mds' osd 'allow *' mds 'allow *' \
  -o /var/lib/ceph/mds/ceph-<主节点主机名>/keyring
systemctl enable --now ceph-mds@<主节点主机名>

# === 状态检查 ===
ceph -s                          # 总览
ceph osd tree                    # OSD 拓扑
ceph df                          # 存储使用
ceph osd pool ls detail          # 池详情
ceph health detail               # 详细健康
ceph fs status cephfs_kvm_log    # CephFS 状态
```

---

## 已知坑完整数据库（27 项，全部实操验证）

> 来源：CLAUDE.md 已知坑汇总 + 实验 1-16 排错记录

### 安装/Monitor 类

| # | 问题 | 触发条件 | 解决方案 | 出现阶段 |
|---|------|---------|----------|---------|
| 1 | 缺 EPEL → `libtcmalloc.so.4` 缺失 | `dnf install ceph-mon` 报 nothing provides | `dnf install epel-release -y` **必须先执行** | Phase A |
| 2 | 镜像源不可用/404 | `dnf makecache` 报连接超时或 BaseOS 404 | OpenCloudOS: `sed -i s/opencloudos.tech/scmu.edu.cn/g`；通用: 改用阿里云 `mirrors.aliyun.com` | Phase A |
| 3 | mon keyring SCP 后属主 root | `ceph-mon --mkfs` 报 Permission denied | `chown ceph:ceph /tmp/ceph.mon.keyring /tmp/monmap` | Phase C |
| 4 | 单 Monitor 启动后 `ceph -s` 无反应 | 一直卡住无输出 | **正常！** 需 >= 2 mon 形成 quorum | Phase C |
| 5 | MGR/MON 启动 `start request repeated too quickly` | systemd StartLimitBurst 触发 | `systemctl reset-failed ceph-mgr@<节点名>` 再 start。**必须用 `journalctl -u ceph-xxx -e` 查根因**，否则重启依然会挂 | Phase C |
| 6 | Monitor 时钟偏移 > 0.05s | `ceph -s` 报 clock skew | 三节点 `systemctl restart chronyd` | Phase D |
| 7 | msgr2 未启用 + insecure global_id reclaim | `ceph -s` 告警 | `ceph config set mon auth_allow_insecure_global_id_reclaim false` + `ceph mon enable-msgr2` | Phase D |
| 8 | VMware 重启后时间漂移 → mon 卡 Paxos | `osd_failure` 卡住 | 配置上游 NTP + 重启全部 3 个 mon | 任何时候 |
| 9 | MGR 延迟注册 | `systemctl is-active` 显示 active 但 `ceph -s` 显示 no daemons | 等待 30-60 秒自动恢复 | Phase C |

### 网络/OVS 类

| # | 问题 | 触发条件 | 解决方案 | 出现阶段 |
|---|------|---------|----------|---------|
| 10 | NM-ovs 插件不可用 | `nmcli con up ovs-vm` 报 "plugin unavailable" | `systemctl restart NetworkManager` + sleep 5 重试 | Phase B |
| 11 | OVS systemd exit code 1 | `ovs-ctl start` 返回 1（system-id 警告） | Type=oneshot + wrapper 脚本 + `--system-id=random` | Phase B |
| 12 | OVS db 锁文件残留 | "failed to lock lockfile" | `rm -f /etc/openvswitch/.conf.db.~lock~` | Phase B |
| 13 | 切换 br0 后 SSH 断连 | 终端卡死 | **正常！** 用 br0 新 IP 重新 SSH | Phase B |
| 14 | ens224 DHCP 同网段 IP 冲突 | 路由表冲突，`ceph -s` 卡住 | `nmcli con modify "有线连接 1" ipv4.method disabled` | Phase B |
| 15 | 防火墙规则 NM 重启后丢失 | Monitor 间无法通信，卡在 probing | **重新 apply 全部防火墙规则** + 重启 ceph-mon | Phase B→C |
| 16 | OVS bond 第二网卡防环 | 配置 bond 时引发二层广播风暴 | 第二网卡加 `autoconnect no`，全部配置完再激活 | OVS bond |
| 17 | SELinux 阻止 OVS 写 `/etc/openvswitch` | `avc: denied { write } for comm="ovsdb-tool"` | `restorecon -Rv /etc/openvswitch` | Phase B |
| 18 | OVS tar 解压后文件不完整 | `make` 执行时 configure 不存在 | `rm -rf && tar -zxf` 重新解压 | Phase B |

### OSD 类

| # | 问题 | 触发条件 | 解决方案 | 出现阶段 |
|---|------|---------|----------|---------|
| 19 | Bootstrap OSD keyring 路径不匹配 | `ceph-volume lvm batch` 报 unable to find keyring | `ln -sf /var/lib/ceph/bootstrap-osd/ceph.keyring /etc/ceph/ceph.client.bootstrap-osd.keyring` | Phase E |
| 20 | LVM 写零操作导致阵列卡死锁 | Dell H710 阵列卡 | `lvm.conf` 设置 `zero = 0`，`lvcreate` 加 `-W n -Z n` | Phase E |
| 21 | 系统重装后 OSD 恢复 | 系统盘坏，OSD 数据盘完好 | 重装系统 → 配 IP → 装依赖 → `ceph-volume lvm activate --all` | 灾备 |
| 22 | LVM 锁残留导致 ceph-volume 挂死 | `ceph-volume lvm batch` 无响应 | `killall -9 lvm; rm -f /var/lock/lvm/*_global*` 或改用 `ceph-volume raw prepare` | Phase E |

### auth 类

| # | 问题 | 触发条件 | 解决方案 | 出现阶段 |
|---|------|---------|----------|---------|
| 23 | `auth_allow_insecure_global_id_reclaim=false` 阻塞新 OSD 认证 | OSD auth 失败 | `ceph osd purge <id>` 清理旧 OSD + `ceph config set mon auth_allow_insecure_global_id_reclaim true` 后重试（**实验环境降级方案，生产应启用 msgr2**） | Phase E |
| 24 | OSD 删除残留 `entity exists but key does not match` | `ceph osd rm` 后残留 auth 条目 | 用 `ceph osd purge <id>`（**非** `ceph osd rm`）彻底清除。原因：`osd rm` 仅从 OSD map 移除，遗留 CRUSH map 位置信息和 cephx auth 密钥；`osd purge` 一并清理 OSD map + CRUSH map + auth 密钥，是安全下线 OSD 的标准做法 | Phase E |

### 通用/环境类

| # | 问题 | 触发条件 | 解决方案 | 出现阶段 |
|---|------|---------|----------|---------|
| 25 | SSH 免密需通过宿主机 paramiko 完成 | 节点间 ssh-copy-id 需要密码交互 | 在宿主机用 paramiko 直接写 authorized_keys | Phase A |
| 26 | Windows GBK 编码导致 UnicodeEncodeError | Python `print()` 含 emoji 等非 BMP 字符 | 日志中不使用 emoji，用纯文本标记 | 全程 |
| 27 | paramiko exec_command 长命令超时 | `dnf install` ~96MB 下载无输出 | 拆分为独立短命令执行，或流式读取 stdout.channel.recv() | 全程 |

---

## SELinux 实操指南（OpenCloudOS 8.10）

### 基本原则

- 部署阶段 `setenforce 0` 临时关闭，**不永久禁用**
- 部署完成后恢复 Enforcing，遇到拦截用 `audit2allow` 精确放通

### audit2allow 工作流（标准三步）

```bash
# 1. 安装工具（首次）
dnf install -y policycoreutils-python-utils

# 2. 分析 AVC 拦截日志，自动生成策略模块
#    注意：grep 模式要精确匹配 domain_t → target_t，路径必须明确
grep 'syslogd_t.*cephfs_t' /var/log/audit/audit.log | audit2allow -M /tmp/rsyslogallow
#    -M 参数可含路径前缀（模块名取自 basename），输出 .te/.pp 到指定目录

# 3. 加载策略
semodule -i /tmp/rsyslogallow.pp
#    注意：semodule -i 触发 SELinux 策略全量重建，耗时数秒至数十秒（视策略大小和硬件而定）
```

### 实战案例：SELinux 阻止 rsyslog 写 CephFS（#30）

**现象**：宿主机重启后 rsyslog 日志写入中断，`/var/log/ceph/` 下无新日志。
**根因**：SELinux Enforcing 模式下，`syslogd_t` 域无权写入 `cephfs_t` 类型的挂载点。
**修复**：
```bash
grep 'syslogd_t.*cephfs_t' /var/log/audit/audit.log | audit2allow -M /tmp/rsyslogallow
semodule -i /tmp/rsyslogallow.pp
```

### 常见拦截场景

| 源域 | 目标域 | 场景 | 所需权限 |
|------|--------|------|---------|
| `syslogd_t` | `cephfs_t` | rsyslog 写 CephFS 挂载点 | 文件: `{ append getattr ioctl open }`, 目录: `{ search }` |
| `syslogd_t` | `cephfs_t` | rsyslog imfile 读取 QEMU 日志写 CephFS | 同上 |
| `virsh_t` | `cephfs_t` | libvirt 访问 CephFS | 文件: `{ read open }`, 目录: `{ search }` |
| `httpd_t` | `cephfs_t` | Apache/Nginx 读 CephFS | 文件: `{ read open getattr }`, 目录: `{ search }` |

### 诊断命令

```bash
getenforce                                    # 当前模式
ausearch -m avc -ts boot                      # 本次启动以来的 AVC 拦截
ls -Z /path/                                  # 检查目录 SELinux 上下文
restorecon -Rv /path/                         # 恢复默认上下文
semodule -l | grep ceph                       # 查看已加载的 Ceph 相关策略
semodule -l | grep rsyslog                    # 查看 rsyslog 相关策略
```

---

## 防火墙配置

> 来源: [Ceph 官方文档 - 网络配置参考](https://docs.ceph.net.cn/en/latest/rados/configuration/network-config-ref/)

### 端口清单

| 端口 | 用途 | 协议 |
|------|------|------|
| 3300 | Monitor v2 (msgr2 默认) | TCP |
| 6789 | Monitor v1 (兼容) | TCP |
| 6800-7568 | OSD/MGR/MDS 数据面（默认范围）| TCP |

> OSD 绑定端口行为是非确定性的：重启后可能绑定到不同端口。因此必须打开整个 `6800-7568` 范围。

### 方案一（官方推荐）：iptables + 子网源过滤

Ceph 官方文档使用 iptables 配合 CIDR 子网作为源地址过滤。适用于 CentOS/RHEL/OpenCloudOS 等所有发行版。

```bash
# Monitor — 放行 6789 和 3300
sudo iptables -A INPUT -i <公网接口> -p tcp -s <管理子网/CIDR> --dport 6789 -j ACCEPT
# OSD/MGR/MDS — 放行完整端口范围
sudo iptables -A INPUT -i <公网接口> -m multiport -p tcp -s <管理子网/CIDR> --dports 6800:7568 -j ACCEPT

# 如有独立集群网络（复制流量），为集群接口添加相同规则
sudo iptables -A INPUT -i <集群接口> -m multiport -p tcp -s <存储子网/CIDR> --dports 6800:7568 -j ACCEPT
```

### 方案二（实验可选）：firewall-cmd + ipset

适用于使用 firewalld 的发行版（如 OpenCloudOS 8.10/CentOS 8）。ipset 方式可以精确控制每个节点 IP 而非整个子网。

```bash
# 创建 IP 集合（包含所有节点的管理 IP + 存储私网 IP）
firewall-cmd --permanent --new-ipset=ceph_pool --type=hash:ip
firewall-cmd --permanent --ipset=ceph_pool --add-entry=<管理IP1>
firewall-cmd --permanent --ipset=ceph_pool --add-entry=<管理IP2>
firewall-cmd --permanent --ipset=ceph_pool --add-entry=<管理IP3>
firewall-cmd --permanent --ipset=ceph_pool --add-entry=<存储IP1>
firewall-cmd --permanent --ipset=ceph_pool --add-entry=<存储IP2>
firewall-cmd --permanent --ipset=ceph_pool --add-entry=<存储IP3>

# 放行 Ceph 端口
firewall-cmd --permanent --add-rich-rule='rule family="ipv4" source ipset="ceph_pool" port port="3300" protocol="tcp" accept'
firewall-cmd --permanent --add-rich-rule='rule family="ipv4" source ipset="ceph_pool" port port="6789" protocol="tcp" accept'
firewall-cmd --permanent --add-rich-rule='rule family="ipv4" source ipset="ceph_pool" port port="6800-7568" protocol="tcp" accept'
firewall-cmd --reload
```

**已知坑 [已验证]**: NetworkManager 重启后 iptables/firewall-cmd 规则可能丢失，必须重新 apply。

---

## 常用配置片段（已验证可用）

### ceph.conf 最小化配置

```ini
[global]
fsid = <你的集群FSID>
mon_initial_members = <node1>, <node2>, <node3>
mon_host = <管理IP1>, <管理IP2>, <管理IP3>
public_network = <管理网段CIDR>
cluster_network = <存储网段CIDR>    # 独立集群网络用于复制流量，避免与客户端流量争抢带宽
auth_cluster_required = cephx
auth_service_required = cephx
auth_client_required = cephx
osd_pool_default_size = <副本数>
osd_pool_default_min_size = <最小副本数>
# size=<副本数>（含 primary），min_size=<最小副本数>（允许故障时仍可 I/O）
# 例如: size=3, min_size=2 允许 1 OSD 故障
```

### CephFS 挂载（/etc/fstab）

```
<管理IP1>:6789,<管理IP2>:6789,<管理IP3>:6789:/ <CephFS挂载点> ceph name=<用户名>,secretfile=<密钥文件路径>,noatime,_netdev,x-systemd.requires=network-online.target 0 0
```

**关键**: 必须包含 `x-systemd.requires=network-online.target`，否则宿主机重启后挂载可能失败。
**多 Mon IP 格式**: 多个 Mon 地址用英文逗号分隔且不能有空格，例如 `10.0.0.1,10.0.0.2,10.0.0.3:6789:/`。

### rsyslog 直写 CephFS（极简 1 template + 1 action）

利用 `%syslogseverity-text%` 动态生成文件路径，按 syslog 标准 8 级（emerg, alert, crit, err, warning, notice, info, debug）自动拆分为独立日志文件，无需 if-else 判断。

```conf
template(name="CephFSLog" type="string"
  string="<CephFS挂载点>/logs/<本机管理IP>/%$year%-%$month%-%$day%/%syslogseverity-text%.log")

*.* action(
    type="omfile"
    dynaFile="CephFSLog"
    dirCreateMode="0755"
    fileCreateMode="0644"
    queue.type="LinkedList"
    queue.filename="cephfs_fallback"
    queue.maxdiskspace="1g"
    queue.saveonshutdown="on"
    action.resumeRetryCount="-1"
)
```

**已知坑**: rsyslog 模板中使用 `%fromhost-ip%` 对 imfile 本地消息为空，建议用固定标识 `<本机标识>` 替代。

### OVS systemd 服务（Type=oneshot + wrapper）

```ini
[Unit]
Description=Open vSwitch (LTS 3.3.9 Pure Build)
DefaultDependencies=no
After=local-fs.target
Before=network.target

[Service]
Type=oneshot
RemainAfterExit=yes
ExecStart=/usr/local/bin/ovs-start-wrapper
ExecStop=/usr/share/openvswitch/scripts/ovs-ctl stop
ExecReload=/usr/share/openvswitch/scripts/ovs-ctl restart

[Install]
WantedBy=multi-user.target
```

wrapper 脚本 `/usr/local/bin/ovs-start-wrapper`:
```bash
#!/bin/bash
exec /usr/share/openvswitch/scripts/ovs-ctl start --system-id=random 2>&1
```

### libvirt RBD 存储池 XML

```xml
<pool type='rbd'>
  <name><libvirt池名称></name>
  <source>
    <name><RBD池名称></name>
    <host name='<MON节点管理IP>' port='6789'/>
    <auth type='ceph' username='<ceph用户名>'>
      <secret uuid='<UUID>'/>
    </auth>
  </source>
</pool>
```

**权限铁律**: 绝不把 `client.admin` 给虚拟化引擎。应使用专属降权账户：
```bash
ceph auth get-or-create client.libvirt mon 'profile rbd' osd 'profile rbd pool=<RBD池名>'
```

---

## 故障诊断速查

### 集群状态类

| 现象 | 诊断命令 | 可能原因 | 处理 |
|------|---------|---------|------|
| `ceph -s` 卡住无响应 | `timeout 5 ceph -s` | 单 monitor 未形成 quorum | 启动第 2 个 monitor |
| `HEALTH_WARN clock skew` | `ceph health detail` | 时钟偏差 > 0.05s | 三节点 `systemctl restart chronyd` |
| `HEALTH_WARN 97 pgs unknown` | `ceph pg stat` | mgr 到 mon 订阅连接断开 | 确认 mgr active 位置，重启 mgr |
| OSDs down | `ceph osd tree` | OSD 进程崩溃或磁盘故障 | `systemctl start ceph-osd@N` 或检查磁盘 |
| Mon probing | `ceph mon stat` | 防火墙阻断或网络不通 | 检查端口 6789/3300、防火墙规则 |
| `entity exists but key does not match` | `ceph auth list` | OSD 删除后 auth 残留 | `ceph osd purge <id>` |

### OSD 管理类

| 操作 | 命令 | 注意事项 |
|------|------|---------|
| 查看 OSD 状态 | `ceph osd tree` / `ceph osd df` | — |
| 完全清除 OSD | `ceph osd purge <id> --yes-i-really-mean-it` | 比 `osd rm` 更彻底，清除 auth + map |
| 停止 OSD | `systemctl stop ceph-osd@N` | 清除前必须先停 |
| OSD 数据盘重置 | `ceph-volume lvm zap /dev/sda --destroy` | OSD 必须已停止 |
| 查看 LVM 映射 | `ceph-volume lvm list` | — |

### PG 相关

| 操作 | 命令 |
|------|------|
| 查看 PG 状态 | `ceph pg stat` |
| 查看 PG 详细 | `ceph pg <pgid>` |
| 查看池的 PG 分布 | `ceph pg ls-by-pool <pool_name>` |
| 调整 PG 数 | `ceph osd pool set <pool> pg_num <N>` (只能增不能减) |
| PG 数量设置 | **官方推荐**: 启用 `pg_autoscale_mode on` 自动调节；**手动计算（官方参考）**: `(OSD_count × 100) / pool_size`（pool_size = 含 primary 的总副本数），每 OSD 约 50-100 个 PG，最终取最近的 2 的幂 |

### RBD 操作

| 操作 | 命令 |
|------|------|
| 创建 RBD 镜像 | `rbd create <pool>/<image> --size <size>` |
| 查看镜像信息 | `rbd info <pool>/<image>` |
| 查看池使用量 | `rbd -p <pool> du` |
| RBD 快照 | `rbd snap create <pool>/<image>@<snap_name>` |
| RBD 快照回滚 | `rbd snap rollback <pool>/<image>@<snap_name>` (需先停止 VM) |
| RBD 镜像克隆 | `rbd clone <pool>/<image>@snap <pool>/<clone>` |

---

## 灾备恢复

### 场景一：系统盘故障，OSD 数据盘完好

```bash
# 1. 重装 OpenCloudOS 8.10，保持相同主机名和 IP
# 2. 配置网络（br0 + ovs-vm）
# 3. 安装依赖
dnf install epel-release -y
dnf install ceph-mon ceph-osd lvm2 -y

# 4. 复制集群配置（从其他节点 SCP）
scp <其他节点>:/etc/ceph/ceph.conf /etc/ceph/
scp <其他节点>:/etc/ceph/ceph.client.admin.keyring /etc/ceph/
scp <其他节点>:/var/lib/ceph/bootstrap-osd/ceph.keyring /var/lib/ceph/bootstrap-osd/

# 5. 修正权限（SCP 过来的文件属主是 root，必须改回 ceph）
chown -R ceph:ceph /etc/ceph/

# 6. 激活 OSD（自动读取 OSD 数据盘上的 LVM 元数据）
ceph-volume lvm activate --all

# 7. 验证
ceph -s
```

### 场景二：VM 在节点间迁移（Ceph RBD）

```bash
# RBD 镜像在 Ceph 集群中，迁移只需拷贝 VM 配置
# 1. 关闭 VM
virsh shutdown <vm_name>

# 2. 导出 VM XML
virsh dumpxml <vm_name> > /tmp/<vm_name>.xml

# 3. 在目标节点定义 VM
virsh define /tmp/<vm_name>.xml

# 4. 启动
virsh start <vm_name>
```

**libvirt 快照限制**: 不支持 RBD 网络磁盘的内部快照。使用 `rbd snap create` 代替。

### 关键备份文件

```
/etc/ceph/ceph.conf                       # 集群配置
/etc/ceph/ceph.client.admin.keyring       # admin 密钥
/var/lib/ceph/bootstrap-osd/ceph.keyring  # OSD bootstrap 密钥
/tmp/ceph.mon.keyring                     # Monitor 密钥
/tmp/monmap                                # Monitor map
/etc/ceph/admin.secret                     # CephFS 挂载密钥
/etc/rsyslog.d/99-cephfs-logs.conf        # rsyslog 配置
/etc/rsyslog.d/97-kvm-qemu-imfile.conf    # QEMU imfile 配置
/etc/fstab                                 # CephFS 挂载条目
```

---

## MGR 与 Dashboard 配置

### Dashboard 启用

官方文档建议每个 mgr 实例单独配置自身 IP（默认绑定 `::`，即所有地址）：

```bash
ceph mgr module enable dashboard
ceph dashboard create-self-signed-cert
ceph config set mgr mgr/dashboard/<mgr_id>/server_addr <本机管理IP>
# 注意：SSL 启用时端口由 ssl_server_port 控制，非 SSL 时由 server_port 控制
ceph config set mgr mgr/dashboard/<mgr_id>/ssl_server_port 8443
ceph dashboard ac-user-create admin -i <(echo "admin123") administrator
```

访问: `https://<mgr_node_ip>:8443/`

> **实验可选**: 如果 mgr 可能在节点间迁移，可将 `server_addr` 设为 `0.0.0.0` 避免 IP 缓存问题。但此方式非官方标准，可能带来安全风险。详见 [已验证-实验12]。

### MGR 在 KVM VM 中的稳定性问题

**现象**: ceph-mgr 在 KVM VM 内启动成功，6-20 分钟后 `ceph -s` 显示 "97 pgs unknown"，HEALTH_WARN。

**根因**: VM 的 virtio 网卡 + OVS 用户态转发路径增加延迟抖动，导致 mgr 到 mon 的 TCP 订阅连接不稳定。mgr 的 status 模块从缓存读取过时数据，不报错不抛异常（"无声退化"）。

**建议**: active mgr 始终运行在物理机上，VM 内 mgr 仅做 standby。

---

## 实验记录索引

> 完整实验记录位于 `Ceph/实验记录/` 目录，均为 Xshell 终端实录格式。

### 推荐阅读顺序

```
新集群部署(v4) → 实验6(PG基础) → 实验9(池+PG) → 实验10(副本策略)
  → 实验7(HDD池+条带) → 实验8(SELinux+防火墙) → 实验1(VLAN+OVS)
  → 实验2(ACL+漂移) → 实验3b/5(故障恢复) → libvirt集成
  → 实验12(mgr迁移) → 实验13(镜像精简) → 实验16(CephFS日志架构)
```

### 关键实验摘要

| 实验 | 主题 | 核心收获 |
|------|------|---------|
| 实验1 | VLAN + OVS 双上行网络 | nmcli 四层模型，br0 + ovs-vm 双桥 |
| 实验6 | PG 基础参数 | PG 是 2 的幂，只能增不能减 |
| 实验8 | SELinux + 防火墙 | audit2allow 工作流，ipset 精确放通 |
| 实验9 | 池 + PG 管理 | PG 计算公式，pool application enable |
| 实验10 | 副本策略 | size/min_size 三副本 |
| 实验12 | mgr 迁移 | VM 内 mgr 不稳定，物理机优先 |
| 实验13 | mgr 最小化 + VM 根因 | ceph-mgr RPM 依赖链，无声退化机制 |
| 实验16 | CephFS 日志架构 | 1 template + 1 action 极简方案，SELinux 策略 |

---

## 使用方式

当用户提问时，按以下流程操作：

1. **判断环境**: 用户系统是 OpenCloudOS 8.10 吗？Ceph 版本是 Pacific 吗？
2. **查实操经验**: 如果匹配，优先查看本 skill 的已知坑数据库和配置片段
3. **查官方文档**: 实操经验不足时，查 `references/` 本地文件或用 `WebFetch` 获取在线文档
4. **组织回答**: 结合实操经验和官方文档，给出经过验证的方案
5. **标注来源**: 实操经验标注 `[已验证-实验N]`，官方内容标注 `[官方文档 - 章节名](URL)`

## 引用格式

```
> 来源: [已验证-实验16] CephFS 分布式日志架构
> 来源: [Ceph 官方文档 - 手动部署](https://docs.ceph.net.cn/en/latest/install/manual-deployment/)
```

## 本地参考文件索引

`references/` 目录下包含从官方文档提取的文本内容：

| 文件 | 内容 | 大小 |
|------|------|------|
| `architecture.txt` | Ceph 架构 — RADOS、CRUSH、核心组件 [官方] | 44KB |
| `install_manual.txt` | 手动安装步骤 [官方] | 4KB |
| `cephadm_install.txt` | Cephadm 安装指南 [官方] | 20KB |
| `cephadm_ops.txt` | Cephadm 运维操作 [官方] | 28KB |
| `cephadm_trouble.txt` | Cephadm 故障排查 [官方] | 16KB |
| `rados_config.txt` | RADOS 配置参考 [官方] | 4KB |
| `rados_ops.txt` | RADOS 运维操作概览 [官方] | 4KB |
| `rados_trouble.txt` | RADOS 故障排查 [官方] | 4KB |
| `rbd_cmds.txt` | RBD 命令参考 [官方] | 8KB |
| `rbd_ops.txt` | RBD 运维操作 [官方] | 4KB |
| `cephfs_create.txt` | CephFS 创建指南 [官方] | 8KB |
| `cephfs_admin.txt` | CephFS 管理维护 [官方] | 16KB |
| `cephfs_mds.txt` | MDS 添加/删除 [官方] | 8KB |
| `rgw_admin.txt` | RADOSGW 管理操作 [官方] | 32KB |
| `security.txt` | 安全配置 [官方] | 4KB |
| `glossary.txt` | 术语表 [官方] | 16KB |
| `known_pitfalls.txt` | 已知坑数据库（实操经验）| 8KB |
| `opencloudos_tips.txt` | OpenCloudOS 8.10 适配指南（实操经验）| 4KB |
| `disaster_recovery.txt` | 灾备恢复指南（实操经验）| 8KB |
