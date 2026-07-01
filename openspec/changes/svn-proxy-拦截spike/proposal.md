## Why

本变更是为 `tdd/03-svn-proxy.md` §2.4 / §7.1 中设计的 SVN Proxy 进行的**技术可行性验证 spike(throwaway demo)**。仓库目前仍是文档阶段(greenfield,零实现)。在投入任何真实实现之前,spike 用于验证设计中单一最高风险假设:能否用一个本地 HTTP proxy 拦截 `svn update`,强制 `send-all=false` 以将元数据与文件内容分离,并从本地内容缓存合成 `GET !svn/ver/REV/path` 响应——最终让 SVN 服务器在整次 update 会话中返回**零文件体字节**(仅元数据)。

为何现在做:设计文档 §2.4 明确将 REPORT 改写 / send-all 强制 / 合成 GET 响应格式标记为"落地前必须 spike 验证"。这些结论取决于真实的 svn client + server 版本及 ra_serf 行为,属于未知风险而非既定事实。spike 将该风险在动工前转化为一个已知答案。

## What Changes

这是一个 spike,不是生产代码。具体新增内容:

- **`tdd-spike/` 目录**:一个自包含、一次性的可行性验证 demo,**不接入任何真实系统**。
  - **服务端**:在 Ubuntu 服务器上部署 Apache + `mod_dav_svn` 的搭建指南,附带一个内容固定、已知的微型测试仓库,并刻意将 `SVNAllowBulkUpdates On` 设为开启——以此证明"强制关闭 bulk"的动作真实有效。
  - **客户端**:一个 mitmproxy Python addon(`spike_addon.py`),作为正向 HTTP proxy。svn client 通过 `--config-option servers:global:http-proxy-*` 指向它。该 addon 的工作:(a) 透传 `OPTIONS`/`PROPFIND`;(b) 改写 REPORT 请求体,强制 `send-all="false"`;(c) 改写 skeleton REPORT 响应中的 checksum(`sha256-checksum`/`text-content-sha1`/`text-content-md5`),使其匹配将要提供的文件字节;(d) 在 `GET !svn/ver/<rev>/<path>` 时,从预置的本地内容库(`files/<file_sha256>`)提供完整文件字节,而不回源上游。
  - **预置内容库**:模拟"已从 seed/P2P 拉取到文件字节"的场景(`prepare_store.py` + `store/files/`)。
  - **验证脚本与一次性运行器**:`verify.py` + `run_spike.ps1`。
- **协议基线捕获文档**:`docs/spike-capture/protocol-baseline.md`,记录所用真实 svn client+server 版本下的真实 REPORT/GET 线上交互形态。

**成功标准(三者必须同时成立)**:
1. 经由 proxy 的 `svn update` 成功,working copy 内容与预期一致,`svn status` 干净。
2. 不出现 `E155017: Checksum mismatch`。
3. Ubuntu 服务器 `access.log` 显示整次 update 会话期间**零文件体 GET** 到达上游(全部被 proxy 拦截);REPORT 响应中不含内联 `<txdelta>`。

## Attribution

- Contributors:
  - Piggyknight
- Redmine:
  - None

## Capabilities

### New Capabilities
- `svn-interception-spike`: 限定该一次性可行性验证 demo 的范围。**注意:此 capability 是 throwaway spike,不构成任何生产 spec 承诺;spike 完成后不产生生产级契约。**

### Modified Capabilities
无(仓库尚无任何 spec)。

## Impact

- **代码范围**:仅新增 `tdd-spike/` 与 `docs/spike-capture/` 下的内容。不修改任何现有 `tdd/` 设计文档。
- **依赖**:mitmproxy(Python)、一台 LAN 内可达的 Ubuntu 服务器(Apache + `mod_dav_svn`)、Windows 开发机上可用的 svn client。
- **非目标(明确排除)**:CDC 分块、bundles、P2P 传输、tracker、commit-path 改写、真实 seed 抓取、`.svn/pristine` 去重——上述全部由预置内容库替代。spike **仅**验证拦截机制本身。
- **语言选型**:Python(mitmproxy addon + 脚本),遵循 `tdd/README.md` 中"脚本实现优先级:Python → bat → PowerShell"。
