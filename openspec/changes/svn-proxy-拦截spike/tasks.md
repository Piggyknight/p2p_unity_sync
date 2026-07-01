# Tasks · svn-proxy 拦截 spike

> 一次性可行性验证(throwaway demo)。按依赖顺序:服务端 → 协议基线捕获 → 预置内容库 → mitmproxy addon 核心 → 单元测试 → 一键运行脚本 → 三方验证 → 结论归档。所有产物落在 `tdd-spike/` 与 `docs/` 下,不接入任何真实系统。

## 1. 环境搭建 / Ubuntu 服务器

- [ ] 1.1 在局域网 Ubuntu 服务器上安装 Apache + `mod_dav_svn`,并将 `SVNAllowBulkUpdates On` 显式开启(以此证明"强制关闭 bulk"动作真实有效);整理安装命令到 `tdd-spike/server-setup/README.md`。
- [ ] 1.2 创建微型测试仓库 `spike-repo`,导入内容固定的种子文件(若干文本 + 一个小二进制),种子文件实体放入 `tdd-spike/server-setup/repo-seed/files/`。
- [ ] 1.3 为每个种子文件生成 sha256/sha1/md5 并汇总写入 `tdd-spike/server-setup/repo-seed/checksums.json`(键为仓库内路径,值为各哈希)。
- [ ] 1.4 编写 Apache 站点配置 `tdd-spike/server-setup/apache-svn.conf`:明文 HTTP、仅局域网可达、详细 `access.log` 格式(记录方法/路径/状态/大小),供后续 grep `!svn/ver/` GET 计数。
- [ ] 1.5 启动站点并冒烟验证:`svn checkout` 直连(不经 proxy)能成功取到 `spike-repo` 全部文件。

## 2. 协议基线捕获

- [ ] 2.1 在 Windows 开发机以 mitmproxy **仅捕获模式(无 addon)**作为正向 proxy,svn 客户端经 `--config-option servers:global:http-proxy-*` 指向它,执行一次 `svn checkout` + 一次 `svn update`。
- [ ] 2.2 记录真实 svn 客户端版本与 `mod_dav_svn` 版本到 `docs/spike-capture/protocol-baseline.md`。
- [ ] 2.3 在该文档中落盘真实的 OPTIONS / PROPFIND / REPORT 请求与响应报文片段(头 + XML 体)。
- [ ] 2.4 在该文档中记录真实 `GET .../!svn/ver/...` 线上形态:请求头(含 `Accept-Encoding: svndiff*`)与响应 `Content-Type`/`Content-Length`/`Content-Encoding`。
- [ ] 2.5 钉死 skeleton 中确切字段名与属性:`sha256-checksum` / `text-content-sha1` / `text-content-md5`、`D:checked-in` href 格式、`send-all` 默认值——使后续 addon 实现基于观察而非文档猜测。

## 3. 预置内容库

- [ ] 3.1 编写 `tdd-spike/scripts/prepare_store.py`:读取 `repo-seed/checksums.json`,将每个文件字节写入 `tdd-spike/store/files/<file_sha256>`(扁平 key,无分块/无 manifest)。
- [ ] 3.2 运行脚本生成 sample store,并人工核对 `store/files/` 下每个文件名与内容数量正确。

## 4. mitmproxy addon 核心实现

- [ ] 4.1 创建 `tdd-spike/src/spike_addon.py`,实现 forward-proxy 请求路由:`OPTIONS`/`PROPFIND` 透传;`REPORT` 拦截;`GET` 且路径命中 `!svn/ver/` 时拦截并本地合成。
- [ ] 4.2 实现 `rewrite_report_request()`:强制 `send-all="false"`,重序列化后保持 XML 结构与命名空间(`S:`=`svn:`、`D:`=`DAV:`)完整(输入不含 `send-all` 时也能正确注入)。
- [ ] 4.3 实现 `restamp_skeleton_checksums()`:从每个 `add-file`/`open-file` 条目的 `D:checked-in` href 提取 `<rev>/<path>`,在 `store/files/` 定位字节,重算 sha256/sha1/md5 并覆盖 skeleton 声明值,使"声明值 == 将服务字节的实际哈希"。
- [ ] 4.4 实现 `serve_get()`:对命中 `GET .../!svn/ver/<rev>/<path>` 返回 `200 OK` + 全文字节,`Content-Type` 取文件 mime(默认 `text/plain`)、`Content-Length` 精确,不含 `svndiff` content-encoding;**不**回源上游。
- [ ] 4.5 加 logging hook:统计本会话内 served(本地合成)计数 vs upstream(转发)计数,便于后续核对零回源。

## 5. 单元测试

- [ ] 5.1 编写 `tdd-spike/tests/test_spike_addon.py`(pytest)覆盖 `rewrite_report_request()`:输出仍为合法 XML(可被 `xml.etree` 重新解析)、`send-all` == `"false"`、命名空间完好;输入无 `send-all` 亦可注入。
- [ ] 5.2 覆盖 `restamp_skeleton_checksums()`:给定样例 skeleton + 预置 store,改写后每条目 `sha256-checksum`/`text-content-sha1`/`text-content-md5` == 对应 store 文件字节真实哈希。
- [ ] 5.3 覆盖 `serve_get()`:对已知文件返回的 `Content-Type`/`Content-Length` 正确、body 字节与 store 一致且无 svndiff encoding。
- [ ] 5.4 运行 `pytest -q` 全绿,作为纯函数验证关口。

## 6. 一键运行脚本

- [ ] 6.1 编写 `tdd-spike/scripts/run_spike.ps1`(或 `.bat`):清理 workcopy → 经 proxy checkout 基线 rev → 服务器端 commit 制造一个新 rev → 客户端经 proxy `svn update` 到新 rev。
- [ ] 6.2 在脚本中编排 mitmproxy + `spike_addon.py` 的启动与退出,以及 svn 客户端 `http-proxy-*` 指向。

## 7. 三方验证

- [ ] 7.1 编写 `tdd-spike/scripts/verify.py`:(a) 校验 workcopy 各文件 SHA-256 与期望一致 + `svn status` 干净;(b) svn stderr/log 无 `E155017`;(c) grep Ubuntu `access.log` 中 update 窗口期 `GET .../!svn/ver/...` 计数 == 0,且抓到的 REPORT 响应无内联 `<txdelta>`。
- [ ] 7.2 `verify.py` 打印三条标准的逐条 PASS/FAIL 与汇总。
- [ ] 7.3 执行一次完整 `run_spike.ps1` + `verify.py` 端到端运行,确认三条标准同时成立(或在失败时如实记录负结果)。

## 8. 结论归档

- [ ] 8.1 填写 `docs/spike-result.md`:三条成功标准的实际结论(PASS/FAIL)、观察到的意外(如字段名差异、流式缓冲行为、`send-all` 是否被尊重)。
- [ ] 8.2 撰写对真实 SVN Proxy 设计的反馈,尤其指向 `tdd/03-svn-proxy.md` §2.4 / §7.1 的最高风险假设,说明 spike 结论对正式 change 提案的影响。
