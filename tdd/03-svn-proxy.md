# 03 · SVN Proxy

> Phase 1 的客户端集成点。拦截 SVN 操作,把文件内容的下载从 SVN 服务器卸载到 P2P/seed,对开发者无感。

**依赖**:[[01-bundle-format]](Bundle/chunk 格式、Revision Manifest) · [[02-p2p-transport]](拉取 chunk) · [[04-tracker]](定位持有者) · [[05-packing-infra]](commit 后接力、seed 兜底)
**被依赖**:无(最上层集成组件)
**上层入口**:[[00-overview]]

---

## 1. 定位与职责

SVN Proxy 是运行在**每台开发者机器**上的本地 HTTP 代理。它夹在 SVN 客户端与 SVN 服务器之间,做三件事:

1. **update**:拦截更新会话,把文件内容下载重定向到 P2P/seed,SVN 服务器只提供元数据。
2. **commit**:仅监测 commit 成功并拿到新 revision,**不改写写路径**,成功后触发 [[05-packing-infra]] 接力。
3. **fallback**:P2P/seed 不可用时,透明降级回原生 SVN,保证正确性。

设计原则:**proxy 只搬运内容字节,绝不重新发明 SVN 的版本逻辑**。版本号、变更集、校验和一律以 SVN 为权威来源,正确性风险最低。

---

## 2. 拦截机制(update 会话)

### 2.1 背景:SVN over HTTP 的两种内容传输模式

`svn update` 走 WebDAV/DeltaV(ra_serf)。内容传输方式由 `REPORT` 请求中的 `send-all` 决定:

- `send-all=true`(现代客户端默认):服务器把变更文件内容(svndiff)**内联**在 `update-report` 的大 XML 响应里,内容与元数据焊死在同一流。
- `send-all=false`:服务器只返回**骨架**(每个变更文件的 path / 目标 rev / 期望校验和),内容由客户端随后逐个 `GET !svn/ver/REV/path` 拉取。

### 2.2 策略:强制 `send-all=false`,分离元数据与内容

proxy 改写客户端的 REPORT 请求,强制 `send-all=false`,把会话拆成两条路:

```
svn update
   │
   ▼
┌──────────────────────────────────────────────────────────┐
│  SVN Proxy (本地 HTTP 代理)                                 │
│                                                            │
│  ① OPTIONS / PROPFIND  ──── 原样转发 ────►  SVN Server     │
│     (能力协商、版本协商,流量小)                              │
│                                                            │
│  ② REPORT (update-report)                                  │
│     改写请求: 强制 send-all=false  ──────►  SVN Server     │
│     ◄──── 骨架响应: 变更集 ──────                           │
│     [path, action, target-rev, expected-checksum] × N      │
│                                                            │
│  ③ 解析变更集 → 查 Revision Manifest → 映射 chunk 集合      │
│                                                            │
│  ④ 拉取缺失 chunk(本地→seed→SVN,见 §6)→ 校验 → 解包      │
│                                                            │
│  ⑤ GET !svn/ver/REV/path × N                               │
│     ◄──── 从本地内容缓存命中,合成 GET 响应 ────            │
│     (不再回源 SVN;校验和与骨架里的一致)                     │
│                                                            │
│  ⑥ 客户端正常落地工作副本 + 元数据                          │
└──────────────────────────────────────────────────────────┘
```

**设计理由**:
- 元数据(谁变了、目标版本、校验和)由 SVN 权威给出,proxy 不碰版本逻辑。
- 占带宽的批量文件字节走 P2P/seed,正是要卸载的 SVN 压力。
- 服务 GET 时返回**全文**(见 §2.3),SVN 客户端完全接受全文 GET(全新 checkout 即如此)。

### 2.3 取舍:GET 返回全文,放弃 svndiff

服务 update 时返回**完整文件**,不做 SVN 的 against-base 增量(svndiff)。理由:

- svndiff 是**按单文件**算增量的;50-60 万散文件逐个算 = 海量噪音与开销,绝大多数文件未变。
- bundle 内存的本来就是完整文件,解包即得,无需再算 delta。
- 算 delta 要维护"客户端当前版本"状态并生成 SVN 私有 svndiff 格式,徒增复杂度与风险。
- 多出的带宽(全文 vs 增量)本就是要从 SVN 卸载到局域网 P2P 的那部分——局域网带宽是项目想利用的廉价资源,SVN 服务器压力才是瓶颈。

> 真正的增量优化在更底层:bundle 的 **CDC 块级去重**(见 [[01-bundle-format]] D1),让"全文"在传输时只搬变化的块。

### 2.4 实现风险(落地前必须 spike 验证)

REPORT 改写、`send-all` 强制、合成 GET 响应的 header/校验和格式,依赖具体的 **SVN 客户端版本与服务器版本**,不同 ra_serf 行为有差异。**这是 SVN Proxy 落地前的前置技术验证项**,不是已确定事实。

---

## 3. 三层映射:从"文件清单"到"要拉的块"

SVN 变更清单只给 `path + 目标 rev + 期望校验和`,但 P2P 网络流动的是 **chunk**。中间靠 **Revision Manifest** 这座桥(由 [[05-packing-infra]] 打包时生成,结构详见 [[01-bundle-format]])。

```
SVN 变更清单                Revision Manifest              本地块存储 / P2P
─────────────              ──────────────────             ──────────────────
path@rev          ──①──►   file → (bundle_id,
+ expected_sha              offset, size, file_sha)
                                    │
                            ──②──►  bundle 字节范围
                                    → 覆盖的 chunk_hash 列表
                                              │
                                       ──③──► chunk_hash → bytes
                                              (本地有? 复用 : seed 拉)
```

**关键点**:Revision Manifest 中 `files` 视图与 `chunks` 视图是**同一段 bundle 字节流的两个 overlay**——`files` 用于切出完整文件给 SVN,`chunks` 用于按内容去重与传输。

**收益:部分 bundle 物化**。第 ② 步只取"变化文件覆盖的块",无需重组整个 bundle。改地形那 1GB 只触及对应 bundle 的少数块。

---

## 4. 存储布局

### 4.1 两层缓存

```
┌─────────────────────────────────────────────────────────────┐
│  Tier 1 · Chunk Store(持久,内容寻址)                        │
│    key = chunk_hash → chunk bytes                            │
│    跨 revision 共享 → CDC 去重在这里发生(增量优势的本体)     │
│    r100→r200 时,绝大多数 chunk 已在,只缺变化的几个          │
├─────────────────────────────────────────────────────────────┤
│  Tier 2 · File Content Cache(临时,服务 GET,LRU 可丢)       │
│    key = file_sha256 → 完整文件 bytes                        │
│    用完可淘汰,随时能从 Chunk Store + Manifest 重建           │
│    按内容寻址 → 跨路径相同文件天然再去重                       │
└─────────────────────────────────────────────────────────────┘
```

### 4.2 机器级全局 Store + 跨分支共享(D2)

存储是**机器级全局**,非工作副本级。这是与 `.svn/pristine` 的本质区别:

```
.svn 模式:每个工作副本各存一份 pristine → 同文件跨分支存多遍
本模式:  所有工作副本共享机器上唯一的全局 Store
          内容相同 → hash 相同 → 只存一份,所有分支复用
```

**分支维度落在哪一层**:Revision Manifest **按 (branch, rev) 区分**(它描述树的形状);chunk 与 file **按内容全局共享**(树的叶子内容)。分支切换、分支间相同文件,在底层 Store 完全复用,无需任何"跨分支去重"逻辑。

### 4.3 目录布局

```
%LOCALAPPDATA%\p2p_unity_sync\        ← 机器上唯一一份,所有工作副本共用
  chunks\          chunk_hash → bytes      (持久, Tier1)
  files\           file_sha256 → bytes     (临时, Tier2, LRU)
  manifests\       (branch, rev) → RevisionManifest
  index.db         本地索引: 我有哪些 chunk / file / bundle
```

一台机器跑**一个 proxy + 一个 Store**,服务该机器上所有工作副本。proxy 按请求里的 SVN URL(分支路径)路由到对应 Manifest,取数据都落到同一个 Store。

### 4.4 配置随工程分发(D3)

区分两样东西:

- **配置**(进 SVN,随 update 分发,美术零配置开箱即用):
  ```yaml
  # <工程根>\.p2psync\config.yaml  ← 提交进 SVN,团队统一
  tracker_url: http://tracker.internal:9000
  store:
    path: "${LOCALAPPDATA}/p2p_unity_sync"   # 默认机器级全局,保住跨分支共享
    max_size: 50GB
  bundle:
    max_size: 100MB
  fallback:
    seed_first: true
    svn_fallback: true
  ```
  可选 `config.local.yaml`(svn:ignore,不提交)覆盖个别机器的特殊路径。
- **Store 数据**(走 P2P/seed,不进 SVN):chunk/file/bundle 实体,大,内容寻址。

> Store 路径默认指向工程**之外**的机器级目录;若设在工程内会退回"每个工作副本各存一份"的老问题,破坏 §4.2 的共享。

### 4.5 开放问题:与 `.svn/pristine` 的双重存储

SVN 客户端自身在 `.svn\pristine\` 留一份原始副本。同一文件可能存于三处:`.svn` pristine + Chunk Store + 工作副本实体。500GB 规模下是真实空间浪费。缓解手段都涉及干预 SVN 客户端行为,有风险。**Phase 1 接受此冗余**,优化议题移交 [[07-security-ops]]。

---

## 5. commit 流程

### 5.1 不对称性:commit 必须先经过 SVN

update 可绕开 SVN 走 P2P,但 commit **不能**——SVN 是 revision 的唯一权威分配者。commit 内容字节仍上传 SVN(此流量 Phase 1 不优化,见 §5.3)。P2P 在此的角色是**发布**,不是替代上传。

### 5.2 proxy 只监测,不改写

commit 走 `MERGE`/`PUT`/`CHECKIN` 等 WebDAV 写请求。proxy **不改写写路径**(让内容原样上传 SVN,保证版本正确),只:

1. 监测 commit 是否成功 + 拿到新 revision 号;
2. 成功后,把"内容进入 P2P"的接力棒交给 [[05-packing-infra]]——经 SVN **post-commit hook** 通知 coordinator(SVN 服务器已确认可配 hook)。

**开发者机器 commit 后零负担**。打包/切块/announce 全在服务端基础设施完成。这修正了"本地打包"方案——后者会把基础设施级可靠性职责压在不可控的客户端上,引发失败/重试/同步问题。详见 [[05-packing-infra]]。

```
svn commit ──► SVN Server (分配 r200,内容字节上传)
                    │ post-commit hook 推送
                    ▼
            [[05-packing-infra]] coordinator 接手
            (export → CDC 切块 → 推 seed → announce manifest)
                    │
                    ▼
   其他人 update r200 → 从 seed(及后续阶段的 peer)拉取
```

### 5.3 Phase 1 非目标:commit 上传流量

500GB 仓库日常瓶颈是 200 人**反复 update**(读多),不是 commit(写少)。Phase 1 只卸载 update 读流量;commit 上传走原生 SVN,逻辑简单且版本绝对正确。commit 上传优化若将来成瓶颈再单独立项——**明确列为 Phase 1 non-goal**,非遗漏。

---

## 6. Fallback 降级链

### 6.1 关键不变量

由 [[05-packing-infra]] 的原子规则(**chunk 先落 seed,coordinator 才 announce manifest**):

> **只要 manifest 已 announce,它引用的所有 chunk 必定在某个持久 seed 上。**

因此 fallback 不是"祈祷有源",而是"**seed 是有保证的地板,除非基础设施宕机**"。

### 6.2 降级链(Phase 1)

```
需要某个 chunk
   │
   ├─① 本地 Chunk Store ──命中──► 用它(免费,瞬时;自身跨版本复用)
   │        │ 缺失
   │        ▼
   │   〔② P2P Peers —— 后续阶段开启,Phase 1 跳过〕
   │        │
   │        ▼
   ├─③ Seed ───────────命中──► 拉取(可靠地板:manifest 在 ⟹ seed 必有)
   │        │ 所有 seed 都宕机(罕见)
   │        ▼
   └─④ SVN Server ─────────► 原生 GET 拿完整文件(最终兜底)
```

> **Phase 1 = seed 分发模式**:开发者机器只下载、不作为上传源,故②(dev-to-dev peer)暂不启用。本地 Chunk Store 仍用于自身跨版本复用。dev-to-dev P2P 互传推迟到后续阶段(届时②成为分担 seed 负载的优化层,seed 仍是地板)。

### 6.3 降级粒度

- **seed 层:chunk 级**——可混源(chunk7 本地、chunk8 seed)。
- **SVN 层:file 级**——SVN 只给完整文件。只要某文件有任一 chunk 在 seed 也拿不到(seed 宕机),该**文件**整体走 SVN:proxy 以真实 SVN 客户端身份发原生 `GET !svn/ver/REV/path` 拿全文回给下游。

### 6.4 特殊情况:manifest 不存在 → 整体透传 SVN

打包窗口期(刚 commit,基础设施未处理完)无 manifest = 无块布局图 = 无法 P2P:

```
查不到 RevisionManifest(branch, rev)
   → 该 revision 整体退化为原生 SVN 透传(不强制 send-all=false)
   → 正确,只是无 P2P 加速(窗口很短,可接受)
```

这是"清晨第一人 / 刚 commit 未扩散"场景的兜底。

### 6.5 失败处理与超时

| 触发 | 动作 |
|------|------|
| seed 单块超时(局域网,秒级阈值) | 切到另一个 seed / 升到 SVN |
| seed 失败 | 该文件升到 ④ SVN |
| **SVN 也不可达** | 硬失败,如实报错给 svn 客户端,**绝不伪造内容**(校验和对不上会污染工作副本) |

> 后续阶段开启 peer 层后,可对同块向 2 个 peer 并发请求取最快返回(hedged request),避免慢 peer 拖累。详见 [[02-p2p-transport]]。

### 6.6 推迟的优化:fallback 内容机会性再分享

proxy 走 ④ 从 SVN 拿到全文后,理论上可再切块、加入本地 store、分享给后来者。但这会把"客户端打包"从后门引回(已在 §5.2 移除,理由充分)。**Phase 1 推迟**:fallback 是纯读路径——拿到、服务、结束。内容进入 P2P 的权威路径只有 [[05-packing-infra]] + seed。记为 future optimization。

---

## 7. 落地前置项小结

1. **[必须 spike]** REPORT 改写 / `send-all` 强制 / 合成 GET 响应格式 —— 针对实际 SVN 客户端+服务器版本验证(§2.4)。
2. **[依赖]** Revision Manifest 结构与 CDC 参数 —— 对齐 [[01-bundle-format]]。
3. **[依赖]** seed 拉取协议 —— 对齐 [[02-p2p-transport]]。
4. **[依赖]** post-commit hook → coordinator 通信 —— 对齐 [[05-packing-infra]]。
5. **[配置]** SVN 服务器 post-commit hook 配置权限 —— 已确认可配。
