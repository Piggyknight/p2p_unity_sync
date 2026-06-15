# 07 · 安全、运维与存储优化

> 🚧 骨架文档,横切关注点。记录已知议题与开放问题,待后续展开。

**横切**:涉及所有子系统
**上层入口**:[[00-overview]]

---

## 议题清单

### 安全与权限
- [ ] **访问控制**:P2P 内容是否需鉴权(局域网内部,但资产有保密要求?)。chunk 内容寻址天然不暴露目录结构,但能拿到 hash 即能拉取内容。
- [ ] **与 SVN 权限的一致性**:SVN 有路径级权限,P2P 绕过 SVN 直取内容是否会泄露无权访问的资产 → 需评估 manifest/chunk 的可见性控制。
- [ ] **完整性**:已有内容寻址 + 双重校验(chunk_hash + file_sha256,见 [[03-svn-proxy]] §3),防篡改/损坏。
- [ ] **Tracker/coordinator 鉴权**:防止恶意 announce 污染索引。

### 部署与运维
- [ ] **客户端分发**:proxy 安装/升级方式(200 台开发机);配置随工程走(见 [[03-svn-proxy]] §4.4)但二进制本体如何分发。
- [ ] **基础设施部署**:Tracker / coordinator / packer / seed 的部署拓扑与扩容(见 [[04-tracker]] · [[05-packing-infra]])。
- [ ] **监控告警**:seed 可用性、打包延迟、fallback 率、Tracker 健康。
- [ ] **脚本实现**:运维/部署脚本按项目约定优先 Python(Python → bat → PowerShell)。

### 存储优化
- [ ] **`.svn/pristine` 双重存储**(来自 [[03-svn-proxy]] §4.5):同一文件存于 `.svn` pristine + Chunk Store + 工作副本实体,500GB 规模下浪费明显。缓解手段涉及干预 SVN 客户端行为,有风险。Phase 1 接受冗余,此处评估优化方案。
- [ ] **Chunk Store 容量管理**:Tier1 持久 store 的上限、淘汰策略(冷 chunk 清理 vs 跨版本复用价值)。
- [ ] **seed 全量历史**:是否保留所有历史 revision,还是只保最近 N 个 + 按需回源 SVN。
