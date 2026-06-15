# 02 · P2P 传输

> 🚧 骨架文档。记录已敲定的决策与开放问题,待后续 brainstorm 展开为完整规范。

**被依赖**:[[03-svn-proxy]](拉取 chunk) · [[05-packing-infra]](packer→seed 推送、seed 分发)
**依赖**:[[01-bundle-format]](chunk 单位) · [[04-tracker]](定位持有者)
**上层入口**:[[00-overview]]

---

## 已敲定的决策

### 传输单位是 chunk
传输与去重的单位是 CDC 切出的 **chunk**(按 `chunk_hash` 内容寻址),不是文件、不是整 bundle。支持**部分 bundle 物化**:只拉变化文件覆盖的块。

### Phase 1 = seed 分发模式
- 开发者机器**只下载、不作为上传源**;内容由 seed/打包基础设施分发。
- dev-to-dev P2P 互传**推迟到后续阶段**;届时 peer 层作为分担 seed 负载的优化,seed 仍是可靠地板(见 [[03-svn-proxy]] §6)。

### 拉取时校验
每个 chunk 拉取后按 `chunk_hash` 校验;切出文件后再按 `file_sha256` 与 SVN 给的 expected checksum 双重校验(见 [[03-svn-proxy]] §3)。

---

## 开放问题

- [ ] **拉取协议**:HTTP / 自定义 TCP / QUIC;chunk 批量请求格式。
- [ ] **并行策略**:跨 chunk 并行度、单 chunk 的 hedged request(后续阶段多 peer 时取最快返回)。
- [ ] **选点策略**(后续阶段):peer 优选(就近、负载、历史成功率);seed 多实例的负载均衡。
- [ ] **断点续传**:大 bundle / 大量 chunk 中断后的恢复。
- [ ] **超时与重试**:局域网秒级阈值的具体取值;失败升级到下一 tier 的判定(与 [[03-svn-proxy]] §6.5 对齐)。
- [ ] **限流**:避免单机拉取压垮 seed 或占满局域网带宽。
