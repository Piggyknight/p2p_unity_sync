# 00 · 整体架构总览

> 文档集入口。本文给出整体架构、Phase 分界和子系统依赖关系,各子系统细节见下方 `[[链接]]`。

**相关文档**:[[01-bundle-format]] · [[02-p2p-transport]] · [[03-svn-proxy]] · [[04-tracker]] · [[05-packing-infra]] · [[06-unity-integration]] · [[07-security-ops]]

---

## 项目背景

### 问题现状
- **团队规模**:200 开发者
- **资产规模**:SVN 资产 ~500GB,50-60 万散文件
- **开发场景**:开放世界游戏,单次地形修改 ~1GB(约 5,000 个小文件)
- **痛点**:
  - SVN 中心服务器网络压力过大
  - 更新速度慢,影响开发效率
  - Unity Accelerator 也是集中式,进一步加剧压力

### 解决目标
1. **Phase 1**:P2P 局域网资源同步,分担 SVN 服务器压力
2. **Phase 2**:P2P 优化 Unity Library 与 Accelerator

### 环境约束
- 所有开发者在同一办公室局域网(单一局域网)
- 当前工作流:SVN + 集中式 Unity Accelerator
- Git 用于代码,SVN 用于资产
- SVN 服务器**可配置 post-commit hook**(见 [[05-packing-infra]])

---

## 整体架构

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                         Developer Workflow Layer                            │
│                    (Unity Editor, SVN commands)                             │
└─────────────────────────────────────────────────────────────────────────────┘
                                       │
                                       ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                          Integration Layer                                   │
│  ┌──────────────────────┐              ┌──────────────────────────────────┐  │
│  │   SVN Proxy          │              │    Unity Asset Pipeline Hook    │  │
│  │   [[03-svn-proxy]]   │              │    [[06-unity-integration]]     │  │
│  └──────────────────────┘              └──────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────────────────────┘
                                       │
                                       ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                           Core Engine Layer                                  │
│  ┌──────────────────────────────────────────────────────────────────────┐  │
│  │  Bundle Manager [[01-bundle-format]]                                  │  │
│  │  Builder(打包) · Extractor(解包) · CDC 块级去重                       │  │
│  └──────────────────────────────────────────────────────────────────────┘  │
│  ┌──────────────────────────────────────────────────────────────────────┐  │
│  │  P2P Transport [[02-p2p-transport]]                                   │  │
│  │  Downloader(并行下载) · 选点 · hedged request                        │  │
│  └──────────────────────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────────────────────┘
                                       │
                                       ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                       Infrastructure / Storage Layer                         │
│  ┌──────────────────┐  ┌──────────────────┐  ┌───────────────────────────┐  │
│  │ Tracker          │  │ 打包与 seed 基础  │  │ 机器级全局 Store          │  │
│  │ [[04-tracker]]   │  │ [[05-packing-infra]]│ (chunk/file/manifest)     │  │
│  └──────────────────┘  └──────────────────┘  └───────────────────────────┘  │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## 子系统依赖关系

```
[[03-svn-proxy]]  ──消费──►  [[01-bundle-format]]  (Bundle/chunk 格式、Revision Manifest)
       │                            │
       │                            └──依赖──►  自研 CDC 库(已有)
       │
       ├──消费──►  [[02-p2p-transport]]  (拉取 chunk)
       ├──查询──►  [[04-tracker]]         (定位 chunk/manifest 持有者)
       └──触发──►  [[05-packing-infra]]   (commit 后接力打包)

[[05-packing-infra]]  ──产出──►  [[01-bundle-format]] 定义的 bundle/manifest
        │                ──announce──►  [[04-tracker]]
        └── seed 节点是 [[03-svn-proxy]] fallback 的可靠地板

[[06-unity-integration]]  (Phase 2)  ──复用──►  [[01-bundle-format]] · [[02-p2p-transport]]
[[07-security-ops]]  ──横切──►  所有子系统
```

---

## Phase 分界

| Phase | 组件 | 目标 |
|-------|------|------|
| **Phase 1** | [[01-bundle-format]] + [[02-p2p-transport]] + [[03-svn-proxy]] + [[04-tracker]] + [[05-packing-infra]] | 替代 SVN 文件下载流量 |
| **Phase 2** | [[06-unity-integration]] | 优化 Unity 导入和缓存 |

### Phase 1 的关键范围决策(分期上线策略)

- **seed 分发模式**:Phase 1 开发者机器**只下载、不作为上传源**。内容由专用打包/seed 基础设施分发。开发者之间的 dev-to-dev P2P 互传**推迟到后续阶段**。
- **本地复用保留**:开发者本地 Chunk Store 仍用于自身跨版本复用(CDC 增量),只是不对外分享。
- **commit 上传不优化**:Phase 1 只卸载 update 的读流量;commit 的写流量走原生 SVN(non-goal,见 [[03-svn-proxy]])。

---

## 核心设计决策(横切)

这些决策跨多个子系统,在此汇总,细节见各文档。

### D1 · 内容寻址 + CDC 块级去重
不按文件做 svndiff(50-60 万文件 → 噪音),而是把小文件合并成 bundle,对 bundle 字节流做**内容定义分片(CDC)**。插入/删除文件时块边界稳定,diff 数量小且有意义。复用团队**已有的自研 CDC 库**。详见 [[01-bundle-format]]。

### D2 · 机器级全局 Store + 跨分支共享
chunk 按 `chunk_hash`、文件按 `file_sha256` 存于**机器级全局目录**(非工作副本级)。不同分支、不同工作副本只要内容相同即自动复用同一份数据——内容寻址让跨分支共享天然发生。详见 [[03-svn-proxy]]。

### D3 · 配置随工程分发,Store 在工程之外
`.p2psync/config.yaml` 提交进 SVN,美术 `svn update` 即开箱即用;但 Store 路径默认指向工程之外的机器级全局目录,不破坏 D2 的共享。详见 [[03-svn-proxy]]。

### D4 · 打包职责在专用基础设施,客户端零负担
开发者机器 commit 后不打包。由 coordinator + packer + seed 组成的服务端基础设施接手,经 post-commit hook 触发。**chunk 先落 seed,coordinator 才 announce manifest**——这条原子规则保证"manifest 在 ⟹ seed 必有其 chunk"。详见 [[05-packing-infra]]。

### D5 · seed 作为 fallback 的可靠地板
[[03-svn-proxy]] 的降级链为 **本地 → (peer,后续阶段) → seed → SVN**。由 D4 的原子规则,seed 是有保证的下界,不靠"祈祷有源"。SVN 是最终兜底。

---

## 待细化部分

- [ ] [[01-bundle-format]]:Bundle 二进制格式详规、CDC 参数对齐、Revision Manifest 结构
- [ ] [[02-p2p-transport]]:传输协议、并行/hedged、选点策略
- [ ] [[04-tracker]]:数据结构、API、manifest 分发方式
- [ ] [[05-packing-infra]]:coordinator 调度、任务切分、高可用、角色↔机器配置
- [ ] [[06-unity-integration]]:Phase 2 Library/Accelerator 方案
- [ ] [[07-security-ops]]:安全权限、部署运维、`.svn/pristine` 双重存储优化
