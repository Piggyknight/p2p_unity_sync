# 06 · Unity 集成(Phase 2)

> 🚧 骨架文档,Phase 2 范围。记录方向与开放问题,待 Phase 1 落地后展开。

**依赖**:复用 [[01-bundle-format]] · [[02-p2p-transport]]
**上层入口**:[[00-overview]]

---

## 目标

P2P 优化 Unity **Library** 与 **Accelerator**——当前 Accelerator 是集中式,与 SVN 服务器一样构成集中压力点。

## 方向(待细化)

- 拦截 Unity import / cache 流程(Asset Pipeline Hook)。
- Library/Accelerator 的导入产物按内容寻址,复用 Phase 1 的 chunk/bundle 机制做 P2P 分享。
- 与 [[03-svn-proxy]] 的资产同步联动:同一份源资产的导入结果在团队间共享,避免人人重复 import。

## 开放问题

- [ ] Unity import 产物的内容寻址 key(受 Unity 版本、平台、导入设置影响,确定性是难点)。
- [ ] 与官方 Unity Accelerator 协议的兼容/替代方式。
- [ ] Library 缓存的粒度与失效。
- [ ] 是否复用 Phase 1 的 [[04-tracker]] / [[05-packing-infra]],还是独立实例。
