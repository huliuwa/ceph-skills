# ceph-docs-reference

> 面向 AI 编程助手（如 Claude Code、Cursor）的 Ceph 分布式存储实操参考技能与防坑知识库。

English | 简体中文

## The Problem

大语言模型在回答 Ceph 分布式存储问题时，往往依赖泛化的通用知识。它们经常会在配置示例中编造虚假的 IP 地址，缺乏对特定操作系统（如 OpenCloudOS）及底层内核网络抖动的排障经验，更危险的是，它们可能会在未给出任何警告的情况下，直接提供 `ceph-volume lvm zap` 等毁灭性的指令。

## The Solution

**ceph-docs-reference** 提供了一份经过真实虚拟化环境（OpenCloudOS 8.10 + Ceph Pacific 16.2.15，16 个实验验证）严格验证的技能文档。将此文档注入 AI 的上下文后，利用"8 条铁律"和"27 项已知坑数据库"，将其从一个只会背官方文档的应答机，改造为一位严谨、懂底层、知敬畏的 Ceph 运维专家。

## The Details

本项目不仅包含从官方提取的核心概念和 25+ 章节的 URL 映射，更重要的是沉淀了从节点初始化到 CephFS/RBD 高阶应用的全生命周期实操经验。

### 核心内容

| 模块 | 描述 | 关键内容 |
|------|------|---------|
| AI 交互铁律 | 规范 AI 的输出行为与排障逻辑 | 8 条铁律：实操验证优先于推测、参数变量化、高危操作阻断、排障先查日志 |
| 已知坑数据库 | 27 项生产环境与特定系统的排障记录 | 安装/Monitor(9)、网络/OVS(9)、OSD(4)、Auth(2)、环境(3) |
| 系统底层适配 | 针对 OpenCloudOS 8.10 及底层组件的调优 | YUM repo 404、内核 5.4.119 virtio_net 抖动、MGR 无声退化、SELinux audit2allow |
| 部署与配置实战 | Phase A-H 标准部署流水线及配置模板 | ceph.conf、CephFS fstab、rsyslog 8 级拆分、OVS systemd、灾备恢复 |
| 实验与文献 | 学习路径索引与官方出处对照 | 16 个递进式实验索引；references/ 含 19 份官方文档 + 3 份实操经验 |

### 已知坑数据库分类

| 分类 | 数量 | 典型问题 |
|------|------|---------|
| 安装/Monitor | 9 | EPEL 依赖、镜像源 404、时钟偏移、systemd 启动限制 |
| 网络/OVS | 9 | NM-ovs 插件失败、OVS db 锁残留、防火墙规则丢失 |
| OSD 管理 | 4 | Bootstrap keyring 路径不匹配、LVM 写零死锁 |
| Auth | 2 | insecure_global_id 阻塞、osd purge vs osd rm 机制差异 |
| 环境 | 3 | Paramiko 超时、Windows GBK 编码 |

## Key Insight

> AI 不缺官方文档的通用理论，缺的是特定系统内核下的网络抖动处理经验、SELinux 拦截规则，以及对 `ceph osd pool delete` 等高危破坏指令的敬畏之心。

## 安装指南

### 方式一：CLI Agent（Claude Code / Aider）

```bash
git clone https://github.com/huliuwa/ceph-skills.git
cd ceph-skills
# 在提示词中引入文档
claude "Read ceph-skills/SKILL.md and help me configure my Ceph cluster."
```

### 方式二：IDE 规则注入（Cursor / Windsurf）

将 `SKILL.md` 的核心内容复制到项目的 `.cursorrules` 或 `.windsurfrules` 文件中。

### 方式三：Web UI 自定义指令（ChatGPT / Claude Web）

将 `SKILL.md` 完整内容粘贴到"自定义指令（Custom Instructions）"或作为 Project Knowledge 上传。

## 使用示例

**场景 A — 配置生成：**

> 你："帮我生成一份包含挂载 CephFS 的 fstab 配置，需要防网络卡死。"
>
> AI 会使用变量占位符：`<monIPs>:/ <挂载点> ceph name=<用户>,secretfile=<密钥路径>,noatime,_netdev,x-systemd.requires=network-online.target 0 0`

**场景 B — 排障引导：**

> 你："集群新加了 OSD，但遇到了 insecure global_id 告警，怎么办？"
>
> AI 检索已知坑 #23，告诉你设置 `auth_allow_insecure_global_id_reclaim true`，并提示这是实验环境降级方案，生产应启用 msgr2。

**场景 C — 高危拦截：**

> 你："帮我执行 ceph-volume lvm zap 销毁这个 OSD。"
>
> AI 不会直接给命令，而是先输出警告要求二次确认。

## 验证标准

当此知识库生效时，你会观察到 AI 行为的本质转变：

- **拒绝编造 IP** — 配置文件中所有 IP、路径、节点名都用 `<管理IP>`、`<节点名>` 等变量占位，不再出现硬编码
- **触发安全拦截** — 执行 `ceph-volume lvm zap`、`ceph osd pool delete` 等破坏性命令时，先输出警告要求二次确认
- **"老兵式"排障** — 遇到报错不再让你盲目重启，而是索要 `ceph -s` 状态或 `journalctl -u ceph-xxx` 底层日志

## 项目结构

```
ceph-docs-reference/
├── SKILL.md                              ← 核心技能文档（750+ 行）
├── README.md                             ← 本文件
└── references/                           ← 参考文献目录
    ├── architecture.txt                  ← Ceph 架构 [官方]
    ├── install_manual.txt                ← 手动安装 [官方]
    ├── rados_config.txt                  ← RADOS 配置 [官方]
    ├── rbd_cmds.txt                      ← RBD 命令 [官方]
    ├── cephfs_create.txt                 ← CephFS 创建 [官方]
    ├── cephfs_admin.txt                  ← CephFS 管理 [官方]
    ├── cephfs_mds.txt                    ← MDS 增删 [官方]
    ├── rgw_admin.txt                     ← RGW 管理 [官方]
    ├── security.txt                      ← 安全配置 [官方]
    ├── glossary.txt                      ← 术语表 [官方]
    ├── known_pitfalls.txt                ← 27 项已知坑数据库（实操验证）
    ├── opencloudos_tips.txt              ← OpenCloudOS 8.10 适配指南
    └── disaster_recovery.txt             ← 灾备恢复指南
    └── ... (共 19 份官方文档 + 3 份实操经验)
```

## License

MIT License.
