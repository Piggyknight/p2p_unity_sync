# 04 · Tracker 服务

> 🚧 骨架文档。记录已敲定的决策与开放问题,待后续 brainstorm 展开为完整规范。

**被依赖**:[[03-svn-proxy]](查询 chunk/manifest 持有者) · [[05-packing-infra]](announce)
**依赖**:无(独立服务)
**上层入口**:[[00-overview]]

---

## 职责

- Peer / Seed 注册与心跳(在线状态)
- chunk / bundle 分布索引(谁有哪个 chunk)
- Revision 映射(哪些 seed 有完整版本、manifest 在哪)

## 数据结构(来自初版设计,待细化)

```
Peer/Seed Registry : Peer ID / IP / Last Seen / Available chunks·bundles / 角色(peer|seed)
Chunk Index        : chunk_hash → 持有者列表
Revision Map       : (branch, rev) → manifest 指针 + 拥有完整版本的 seed
```

## API(来自初版设计,待细化)

```
POST /api/peer/register     # 上线注册
POST /api/peer/heartbeat    # 心跳(每 30s)
POST /api/peer/locate       # 查询:谁有这些 chunk/manifest?
POST /api/bundle/announce   # 通知:我有新内容(由 coordinator 调用)
```

---

## 已敲定的相关决策

- **announce 由服务端基础设施发起**:不是开发者客户端,而是 [[05-packing-infra]] 的 coordinator 在 chunk 落 seed 后 announce manifest(原子提交点)。
- **seed 是 fallback 地板**:Revision Map 须能回答"哪些 seed 持有某 rev 的完整内容",供 [[03-svn-proxy]] 降级使用。

## 开放问题

- [ ] **Manifest 分发方式**:Tracker 只存指针 + manifest 内容寻址走 P2P(推荐,Tracker 轻量) vs Tracker 直接存 manifest 实体。与 [[01-bundle-format]] 对齐。
- [ ] **调度职责归属**:[[05-packing-infra]] 的 coordinator/scheduler 是否与 Tracker 同进程?二者关注点不同(Tracker=谁有什么,scheduler=谁该打包什么),需厘清边界。
- [ ] **高可用**:Tracker 单点的主备 / 数据持久化。
- [ ] **心跳与失效**:peer 掉线判定、索引清理。
- [ ] **规模**:200 peer + 多 seed + 海量 chunk_hash 的索引规模与查询性能。
