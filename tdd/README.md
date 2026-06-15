# P2P Unity Sync — 设计文档

本目录是 p2p_unity_sync 的设计文档集,按子系统拆分。每份文档自包含,顶部声明依赖关系,使用 Obsidian `[[双向链接]]` 互联——在 Obsidian 中打开可点击跳转并查看关系图谱。

## 阅读入口

从 [[00-overview]] 开始,它串联所有子系统并给出整体架构、Phase 分界和依赖关系图。

## 文档索引

| 文档 | 内容 | 状态 |
|------|------|------|
| [[00-overview]] | 整体架构、Phase 分界、子系统依赖关系 | ✅ |
| [[01-bundle-format]] | Bundle 格式、内容寻址、CDC 块级去重 | 🚧 骨架 |
| [[02-p2p-transport]] | P2P 传输协议、分片、并行下载、选点 | 🚧 骨架 |
| [[03-svn-proxy]] | SVN Proxy:拦截、映射、缓存、commit、fallback | ✅ 已细化 |
| [[04-tracker]] | Tracker 服务:注册、索引、Revision 映射 | 🚧 骨架 |
| [[05-packing-infra]] | 打包与 seed 基础设施:coordinator/packer/seed | 🚧 骨架 |
| [[06-unity-integration]] | Phase 2:Unity Library/Accelerator 优化 | 🚧 骨架 |
| [[07-security-ops]] | 安全权限、部署运维、存储优化 | 🚧 骨架 |

## 当前进度

- **已细化**:[[03-svn-proxy]](拦截机制、三层映射、存储布局、commit 流程、fallback 降级链)
- **待细化**:其余子系统目前为骨架,记录了已敲定的决策与开放问题,后续逐个 brainstorm 展开。

## 约定

- 语言:简体中文;技术术语、代码标识符、命令、文件路径、API 名称保持英文原文。
- 脚本实现优先级:Python → bat → PowerShell。
