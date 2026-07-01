# Design · svn-proxy 拦截 spike

> 本文件是 OpenSpec 变更 `svn-proxy-拦截spike` 的设计文档。它为提案中定义的**一次性技术可行性验证 spike(throwaway demo)**给出架构/方案,而非生产架构。设计必须满足 `proposal.md` 的三条成功标准,并为 `tdd/03-svn-proxy.md` §2.4 / §7.1 标记的最高风险假设(REPORT 改写 / send-all 强制 / 合成 GET 响应格式)提供一个可观察、可复现的答案。

---

## Context

仓库当前处于 greenfield 阶段(仅文档,零实现)。`tdd/03-svn-proxy.md` 设计了一台运行在每台开发者机器上的本地 HTTP proxy,用以拦截 `svn update`,把文件内容下载从 SVN 服务器卸载到 P2P/seed,使服务器在一次 update 会话中只返回元数据(skeleton),零文件体字节。

该设计的核心假设——即"能强制 `send-all=false`、能改写 skeleton 中的 checksum、能从本地缓存合成 `GET !svn/ver/REV/path` 全文响应"——并非既定事实,而是**强依赖具体 svn 客户端版本、`mod_dav_svn` 版本与 `ra_serf` 行为**的未知项。§2.4 与 §7.1 明确要求:在投入任何真实实现之前,必须先 spike 验证这些机制在真实版本组合下是否成立。本 spike 的唯一职责,就是把这个未知风险在动工前转化为一个已知答案。

本 spike 是 throwaway demo:**不接入任何真实系统**,不构成生产 spec 承诺,完成后不产生生产级契约。它的运行环境是用户真实硬件:Windows 开发机(svn 客户端 + mitmproxy)+ 局域网内的 Ubuntu 服务器(Apache + `mod_dav_svn`)。

spike 的三条成功标准(提案 §成功标准,三者必须同时成立):
1. 经由 proxy 的 `svn update` 成功,working copy 内容与预期一致,`svn status` 干净。
2. 不出现 `E155017: Checksum mismatch`。
3. Ubuntu 服务器 `access.log` 显示整次 update 会话期间**零文件体 GET 到达上游**(全部被 proxy 拦截);REPORT 响应中不含内联 `<txdelta>`。

设计全程贯彻 **"spike simplicity"**:最小化活动部件,接受生产代码不会接受的捷径。所有生产级机制(CDC 分块、bundles、Revision Manifest、P2P 传输、tracker、seed、commit-path 改写、`.svn/pristine` 去重、机器全局 store、配置分发、多工作副本)一律用一个平凡的预置内容库 `store/files/<file_sha256>` 替代。

---

## Goals / Non-Goals

### Goals

1. **证明提案的三条成功标准可达成**:update 成功、无 `E155017`、零文件体 GET 到达上游。这是 spike 的全部产出。
2. **产出 `protocol-baseline.md`**:在动手写 addon **之前**,捕获所用真实 svn 客户端 + `mod_dav_svn` 版本下 REPORT/GET 的真实线上交互形态——请求/响应头、XML 元素与属性名(尤其是 skeleton 中 `sha256-checksum` / `text-content-sha1` / `text-content-md5` 的确切字段形态)、`send-all` 的实际默认值、GET 的实际请求头。使 addon 代码基于**观察**而非文档猜测。
3. **在用户真实硬件上可运行**:Windows 客户端(svn + mitmproxy + Python addon)+ Ubuntu 服务器(Apache + `mod_dav_svn` + 详细 `access.log`)。

### Non-Goals(明确排除)

下列全部由 trivial 的预置内容库替代或干脆不做,生产化时再处理:

- CDC 分块、bundles、Revision Manifest、P2P 传输、tracker、seed 抓取。
- commit-path 改写(本 spike **只**验证 update 读路径,不碰 commit)。
- `.svn/pristine` 去重、机器级全局 store、配置随工程分发、多工作副本支持。
- 生产级可靠性、并发、错误处理、HTTPS、可观测性、可重入安装。
- 验证 commit 路径(显式排除)。

> 这些是 spike 有意放弃的维度,不是遗漏。spike 的价值在于"用最小代价回答一个 yes/no 问题",而非逼近生产形态。

---

## Decisions

每个决策都给出选型理由与被否定的备选。

### D1. mitmproxy 作为 proxy 基质

**选 mitmproxy。**

理由:(a) 它是 Python 实现,符合 `tdd/README.md` 中"脚本实现优先级:Python → bat → PowerShell";(b) 成熟的 HTTP/1.1 + 绝对 URI(absolute-URI)forward-proxy 支持,svn 客户端能原生指向它;(c) 提供 addon API(`request` / `response` hook)用于请求/响应体改写,正是改写 REPORT 与合成 GET 所需;(d) 自带 flow 日志,免去自己造抓包。

**备选(被否定):**
- **手搓 asyncio HTTP proxy**:控制力更强,但要重新发明 HTTP 解析、`CONNECT` 隧道、chunked transfer-encoding——对一次性 spike 是错误的投入。
- **nginx / Caddy 反向代理**:无法逐文件改写 REPORT body、无法按 `!svn/ver/...` 合成 GET 响应,根本不具备所需的可编程性。

### D2. Forward-proxy 模式,而非 reverse-proxy

**用 forward proxy。** svn 客户端通过 `--config-option servers:global:http-proxy-host=... http-proxy-port=...` 原生支持 forward-proxy 配置。客户端发出的是 absolute-form URI(`GET http://host/!svn/ver/...`),proxy 直接知道真实 upstream origin,read-only update 不需要改写 `Destination` 等 WebDAV 头。

**备选(被否定):**
- **reverse-proxy + 主机名改写**:需要更多配置,且要正确处理 WebDAV 方法透传、`Host` 头、`Destination` 头中的 absolute URI 重写,对纯读路径无收益。spike 优先选择最少活动部件的方案。

### D3. REPORT 请求改写:强制 `send-all="false"`

解析 update-report 请求体,把 `send-all` 属性显式设置/覆盖为 `"false"`,再重新序列化。**必须保留 XML 命名空间**(`S:` = `svn:`、`D:` = `DAV:`),避免破坏客户端/服务器的解析。

**理由:** 现代 `ra_serf` 通常已默认 skeleton 模式,但测试仓库**刻意**开启 `SVNAllowBulkUpdates On`。在 bulk 被允许的前提下,显式强制 `send-all=false` 才是真正"证明这个机制可控"的动作——否则无法区分"成功"是因为我们改写了,还是因为客户端恰好没要 bulk。protocol-baseline 会先记录默认行为,addon 再据此覆盖。

### D4. REPORT skeleton 响应改写:按提供的字节重新盖章 checksum(关键技术)

skeleton 为每个 `add-file` / `open-file` 条目声明 `sha256-checksum` / `text-content-sha1` / `text-content-md5`。客户端随后对通过 GET 拿到的字节重算这些哈希,并与 skeleton 声明值比较——**不一致即 `E155017`**。

由于 proxy 服务的是**自己预置 store 里的字节**(而非服务器字节),它**必须**用 store 字节的哈希覆盖 skeleton 声明的 checksum,使"声明值 == 实际服务值"。流程:从每个条目的 `D:checked-in` href 提取 `<rev>/<path>` → 在 store 中定位文件 → 重算三哈希 → 改写 skeleton。

**备选(被否定):**
- **让 store 字节与服务器字节完全相同**(则无需改写 skeleton)。被否定,因为这**违背 spike 目的**:我们要证明的是"能服务**任意**缓存字节"——即模拟一次真实的 cache hit(命中内容可能来自任意来源)。若字节强制与服务器一致,等于绕过了待验证的核心能力。

> 取舍说明:改写 checksum 意味着我们**有意允许**"服务字节 ≠ 服务器字节"。这是 spike 故意的(证明 cache-hit 路径),代价是无法再断言"字节与服务器一致";改为断言"我们声明的 checksum == 我们服务的字节"。

### D5. GET 合成:返回全文,不用 svndiff

命中 `GET .../!svn/ver/<rev>/<path>` 时,返回 `200 OK` + `Content-Type: <文件 svn:mime-type,默认 text/plain>` + 精确 `Content-Length` + body = store 中该文件完整字节。**绝不**返回 `Content-Encoding: svndiff*`。

客户端在 GET 请求中会声明 `Accept-Encoding: svndiff1;q=0.9,svndiff;q=0.8`,但当响应 `Content-Type` 不是 svndiff 类型时,客户端接受全文。这正面验证了 `tdd/03-svn-proxy.md` §2.3 的设计取舍(GET 返回全文,放弃 svndiff)。

### D6. 预置内容库(`prepare_store.py` + `store/files/<sha256>`)

`prepare_store.py` 读取 `checksums.json`(由测试仓库种子文件生成),把每个文件字节写入 `store/files/<file_sha256>`。这模拟"内容已从 seed/P2P 拉取到位"的初始状态。**扁平 key = sha256,无分块、无 manifest、无 bundle**——spike 不需要任何内容布局复杂度。

### D7. Ubuntu 上的测试仓库

在 Ubuntu 服务器上建一个内容固定、已知的微型仓库:若干文本文件 + 一个小二进制文件。Apache + `mod_dav_svn`,`SVNAllowBulkUpdates On`,开启详细 `access.log`。

**spike 运行流程:** 客户端先 checkout 基线 rev → 在服务器上 commit 一个新 rev(制造一次变更,让 update 有活干)→ 客户端**经由 proxy** update 到新 rev。

### D8. 验证(`verify.py`)

`verify.py` 检查三件事,逐条对应成功标准:
1. working copy 各文件 SHA-256 与期望一致,`svn status` 干净;
2. svn stderr 不含 `E155017`;
3. grep Ubuntu `access.log`:update 窗口期内到达 upstream 的 `GET .../!svn/ver/...` 请求数 == 0;且抓到的 REPORT 响应不含内联 `<txdelta>`。

### 关于测试驱动(TDD)的声明

**本 spike 不以 TDD 为主导方法。** 理由:这是一次针对外部系统(svn 客户端 + `mod_dav_svn`)行为的可行性验证——spike 本身**就是**对"外部系统是否会按假设行为"的测试。red-green-refactor 的 TDD 节奏适用于我们拥有并迭代的代码,而 spike 的核心问题(真实版本下的线上形态)只能通过观察真实流量回答,无法由先写的单元测试驱动出来。

**但少数纯函数单元仍会有小型单元测试**(见下方 Unit Tests),因为这些函数有明确的输入输出契约,适合 TDD 式先写测试。核心验证则是一次性的端到端手动测试(verify.py 的三标准运行)。

#### Unit Tests

针对 addon 中三个纯函数,用 `pytest` 写小单元测试:

- **(a) `rewrite_report_request()`**:`send-all` 强制改写后,输出仍是合法 XML(可被 `xml.etree` 重新解析),`send-all` 属性值 == `"false"`,且 `svn:`/`DAV:` 命名空间保持完好;输入不含 `send-all` 时也能正确注入。
- **(b) `restamp_skeleton_checksums()`**:给定一份样例 skeleton + 一个预置 store,改写后每个条目声明的 `sha256-checksum` / `text-content-sha1` / `text-content-md5` == 对应 store 文件字节的真实哈希。
- **(c) `serve_get()`**:对一个已知文件,返回的 `Content-Type` / `Content-Length` 正确,body 字节与 store 一致。

#### Manual Tests

端到端 spike 运行即为手动测试计划,直接以提案三条成功标准为验收项:

1. 在 Windows 客户端经由 proxy 执行 `svn update` 到新 rev,**人工确认** update 成功完成、working copy 文件内容与预期一致、`svn status` 干净(成功标准 1)。
2. 人工检查 svn 客户端 stderr / 输出中**无 `E155017`**(成功标准 2)。
3. 在 Ubuntu 服务器 `access.log` 中人工/grep 确认:update 窗口期内到达 upstream 的 `GET .../!svn/ver/...` 请求数 == 0,且抓到的 REPORT 响应不含内联 `<txdelta>`(成功标准 3)。

运行入口为 `run_spike.ps1`(一键:起 mitmproxy + addon → 触发 update → 跑 verify.py → 汇总三标准结论)。

---

## Risks / Trade-offs

格式:[风险] → [缓解]。所有针对外部不确定性的风险,都靠 **protocol-baseline 先捕获再编码** 这一前置步骤缓解。

- **[skeleton 中 checksum 字段名/属性随 `mod_dav_svn` 版本变化]** → [protocol-baseline.md 的捕获步骤在写 addon **之前**钉死确切字段名;addon 按观察到的字段实现,而非按文档猜测。]
- **[mitmproxy 可能破坏大型 REPORT 响应的 chunked/XML 流式传输]** → [使用 mitmproxy 的 `response` hook 并完整缓冲 body 后再改写;若流式确实被破坏,降级为仅在 `request` hook 改写(强制 send-all)、放行服务器 skeleton 原样,然后单独验证 GET 拦截是否仍成立。]
- **[svn 客户端可能发起带条件/Range 的 GET,或用 ETag 绕过我们的合成]** → [baseline 捕获会暴露真实 GET 请求头;addon 只处理观察到的头,忽略未观察到的。]
- **[在当前客户端/服务器组合上强制 `send-all=false` 不被尊重]** → [这正是 spike 要测的;若确实不被尊重,这本身就是一个**有效的负结果**,如实写入 `docs/spike-result.md`。]
- **[HTTPS 会让 mitmproxy 陷入证书信任麻烦]** → [spike 在局域网 Ubuntu 服务器上**只用明文 HTTP**,显式回避 HTTPS。]

**Trade-off:** 改写 checksum 意味着服务字节可以**有意不同于**服务器字节——这是为了证明 cache-hit 路径;代价是我们不能再断言"字节与服务器一致",改为断言"我们声明的 checksum == 我们服务的字节"。

---

## Migration Plan

**N/A。** 本 spike 为一次性验证,**无部署、无回滚、无迁移**。验证结束后 `tdd-spike/` 目录可整体删除或归档;结论写入 `docs/spike-result.md`,供正式 SVN Proxy change 提案引用。不产生任何需要在生产环境上线或回滚的产物。

---

## Open Questions

- **Windows 上 svn 客户端的确切版本、Ubuntu 上 `mod_dav_svn` 的确切版本** —— 由 baseline 捕获步骤解决,记录进 `protocol-baseline.md`。
- **客户端是否对未变更的 `<open-file>` 条目跳过 GET(复用 pristine)** —— 若是,会影响 spike 是否需要一个 `add-file` 密集的测试 rev;baseline 捕获会回答。
- **mitmproxy 是否能干净缓冲 REPORT 响应以原地改写 XML,还是只能退到 request-hook-only 方案** —— 运行时验证;若缓冲不可行,触发 D4 风险中记录的降级路径。
