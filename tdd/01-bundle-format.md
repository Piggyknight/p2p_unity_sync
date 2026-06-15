# 01 · Bundle 格式

> 🚧 骨架文档。记录已敲定的决策与开放问题,待后续 brainstorm 展开为完整规范。

**被依赖**:[[03-svn-proxy]](消费 Bundle/chunk 格式与 Revision Manifest) · [[05-packing-infra]](产出 bundle/manifest) · [[02-p2p-transport]](传输 chunk)
**依赖**:团队自研 CDC 库(已有)
**上层入口**:[[00-overview]]

---

## 已敲定的决策

### D1 · 内容寻址 + CDC 块级去重
不按文件做 svndiff(50-60 万文件 → 海量噪音),而是把小文件合并成 bundle,对 bundle 字节流做**内容定义分片(Content-Defined Chunking, CDC)**:

- **必须用 CDC,不能用固定偏移切块**:在大文件中插入/删除小文件时,固定偏移会让后续所有块错位("全变了"),CDC 用滚动哈希按内容定边界,插入/删除只影响局部块,边界稳定。这是"合并成大文件做 diff"能成立的前提。
- **小文件拼接顺序必须确定**(如按路径排序):同一文件集合每次拼出字节完全相同的大文件,否则 hash 对不上、无法跨版本去重。
- **按 bundle 分桶,不是全仓合一**:500GB 合成单一文件不现实(任一改动要重算整个流)。沿用"按目录 + 100MB 上限"切多个 bundle,在每个 bundle 内部做合并 + CDC。改地形那 1GB 只触及对应少数 bundle。
- **复用团队已有的自研 CDC 库**,不重新实现。

### 同一字节流的两个视图
Revision Manifest 中,`files` 视图与 `chunks` 视图描述**同一段 bundle 字节流**:`files` 用于切出完整文件(给 [[03-svn-proxy]] 服务 GET),`chunks` 用于内容去重与传输。

---

## Bundle 格式(来自初版设计,待细化)

```
Bundle Header   : Bundle ID(SHA256) / Name / File Count / Total Size / Compression
File Manifest   : [File Path, File Size, Offset, SHA256] × N
File Data       : 压缩后的文件数据(顺序拼接)
Bundle Trailer  : Bundle SHA256(校验)
```

- 压缩算法:zstd(速度与压缩率平衡)候选,待与 CDC 配合方式确认。

---

## 开放问题

- [ ] **CDC 参数对齐**:自研库的 chunk 大小范围、滚动哈希算法、API 接口 → 据此确定 Revision Manifest 的 chunk 字段。
- [ ] **Revision Manifest 完整结构**:`files` / `chunks` 两视图的精确字段、序列化格式。
- [ ] **压缩与 CDC 的顺序**:先压缩后分片 vs 先分片后压缩(影响去重率)。
- [ ] **Bundle 分桶策略细节**:目录映射规则、100MB 上限的拆分方式、跨目录小文件归桶。
- [ ] **Manifest 分发方式**:内容寻址走 P2P vs 存 Tracker(与 [[04-tracker]] 对齐)。
