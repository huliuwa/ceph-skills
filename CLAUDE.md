# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## 仓库用途

ceph-skills 是一个 Claude Code Skill 仓库，提供 Ceph 实操参考技能。Skill 文件 `SKILL.md` 被 Claude Code 自动加载，用于回答 Ceph 相关问题、规划部署、排查故障。

## 文件结构

```
SKILL.md              # 主 Skill 文件 — Ceph 实操参考（核心）
references/           # 从 docs.ceph.net.cn 提取的官方文档参考文本
├── architecture.txt  # Ceph 架构
├── install_manual.txt
├── cephadm_*.txt     # Cephadm 相关
├── rados_*.txt       # RADOS 存储集群
├── rbd_*.txt         # RBD 块设备
├── cephfs_*.txt      # CephFS 文件系统
├── rgw_admin.txt     # RADOSGW 对象网关
├── disaster_recovery.txt   # 灾备恢复
├── known_pitfalls.txt      # 已知坑数据库
├── opencloudos_tips.txt    # OpenCloudOS 适配
├── security.txt
└── glossary.txt
```

第三方 skill（通过 `npx skills` 安装）以 symlink 形式存在，不纳入版本控制。

## 核心原则

### 回答铁律
- **本地实验验证 > 官方文档 > 推测** — 实操经验优先，官方文档为底线
- 不允许编造或猜测文档中不存在的内容
- 标注来源：`[已验证-实验N]` 或 `[官方文档 - 章节名](URL)`
- 所有 IP、节点名、路径等参数**必须变量化**（`<管理IP>`、`{mon_ip}`），不得硬编码

### 修改 SKILL.md 的规则
- Description 中的 `触发词` 决定此 skill 何时被加载，新增 Ceph 相关术语时同步更新
- 新增内容必须标注来源（实验或官方文档）
- 不保留用户的实际 IP、主机名、FSID 等敏感信息
- `references/` 中的文件仅保存官方文档原文提取文本，不写入实验经验

## 常用命令

```bash
# 推送更新到 GitHub
git add . && git commit -m "说明改动内容" && git push

# 查看改动
git status
git diff

# 拉取远程更新并合并
git pull --rebase origin master
```

## 文档验证

本仓库所有技术声明需与 https://docs.ceph.net.cn/en/latest/ 官方文档保持一致。验证方式：

1. 在 "文档结构速查" 表中找到对应 URL
2. 使用 `curl -sL <URL>` 获取文档原文
3. 对比关键参数和命令语法
4. 命令行示例用 `--help` 验证参数是否存在
