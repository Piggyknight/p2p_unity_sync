# 05 · 打包与 seed 基础设施

> 🚧 骨架文档。记录已敲定的决策与开放问题,待后续 brainstorm 展开为完整规范。
> (原计划名 `05-seed-node`,因角色扩展为 coordinator/packer/seed 而改名。)

**被依赖**:[[03-svn-proxy]](commit 后接力、seed 兜底)
**依赖**:[[01-bundle-format]](产出 bundle/manifest) · [[04-tracker]](announce) · SVN post-commit hook
**上层入口**:[[00-overview]]

---

## 核心定位

把"内容进入 P2P 网络"这一**可靠性关键路径**从不可控的开发者客户端上移到**专用服务端基础设施**。开发者机器 commit 后零负担。

修正的反模式:让客户端打包/announce 会遇到机器关机、网络抖动、重试导致状态不一致、并发 commit race、无单一事实来源等问题。

---

## 角色架构(规模化)

角色与机器解耦,数量按负载配置伸缩:

```
SVN Server ──post-commit hook──► Coordinator/Scheduler (中心调度, 1 主+备)
                                        │ 派发(按 bundle 切分)
                          ┌─────────────┼─────────────┐
                          ▼             ▼             ▼
                       Packer 1     Packer 2  …   Packer N   (多, 伸缩, 可临时)
                          │ push chunk                │
                          └──────────────┬────────────┘
                                         ▼
                                   Seed 1..K   (少, 持久, 兜底源)
```

| 角色 | 数量 | 职责 | 可靠性 |
|------|------|------|--------|
| Coordinator | 1 主+备 | 收 commit、派活、汇总、announce | 关键,高可用 |
| Packer | 多,伸缩 | 拉任务、CDC 切块、推 seed | 可临时,失败重派 |
| Seed | 少,固定 | 持久存全量,fallback 地板 | 关键,持久 |

> 角色↔机器映射通过**配置**决定:小规模时可同机兼任;量上来后增加 packer、保持少量 seed + 中心调度。

---

## 已敲定的边界保障

1. **幂等**:内容寻址 → 重复打包同一 rev 产出完全相同的 chunk/manifest,重试绝对安全。
2. **原子性**:**chunk 先落持久 seed,coordinator 最后才 announce manifest**。一个 rev "可用" ⟺ manifest 已 announce 且所有 chunk 在位。绝不 announce 半成品。这条规则是 [[03-svn-proxy]] §6 "seed 是可靠地板"不变量的来源。
3. **断点恢复**:coordinator 记录 `last_announced_revision`,重启后从 `last_announced+1` 追到 SVN HEAD;崩在半路按未 announce 处理,重打即可(幂等)。
4. **顺序与积压**:按 revision 顺序处理;宕机期间积压恢复后顺序补齐,补齐期间开发者走 SVN 兼底。
5. **manifest 汇总单点**:大 commit 的多 bundle 可拆给多 packer 并行,各自回报 chunk 列表,**coordinator 汇总成完整 RevisionManifest 再一次性 announce**,原子点唯一。

## 通信机制

| 链路 | 机制 | 备选 |
|------|------|------|
| SVN → coordinator | post-commit hook 推送(低延迟,已确认可配) | 轮询 `svn log` |
| coordinator → Tracker | announce(manifest hash + chunk 列表 + 持有者) | —— |
| packer → seed | chunk push | —— |

---

## 开放问题

- [ ] **任务切分策略**:整 rev 给一个 packer(简单) vs 按 bundle 拆多 packer 并行(快,需 manifest 汇总)。
- [ ] **chunk 交接**:packer→seed 推送协议;packer 是否短期也作为源。
- [ ] **coordinator 高可用**:主备切换、`last_announced_revision` 持久化与一致性。
- [ ] **seed 容量与淘汰**:全量 500GB+ 历史版本的存储策略。
- [ ] **多 coordinator/多 seed 冗余**:内容寻址产出一致,如何配置互为冗余。
- [ ] **与 [[04-tracker]] 的进程边界**:coordinator 是否与 Tracker 合并部署。
- [ ] **脚本实现语言**:按项目约定优先 Python(Python → bat → PowerShell)。
