## ADDED Requirements

### Requirement: SVN 客户端经代理完成 update

本 spike 的代理 SHALL 夹在 svn 客户端(经 `--config-option servers:global:http-proxy-*` 配置)与 SVN 服务器之间。经代理执行的 `svn update` SHALL 成功完成并产出正确的工作副本。本 capability 为一次性可行性验证,非生产承诺。

#### Scenario: 通过代理更新到新 revision

- **WHEN** 客户端经代理对一个含文件变更的新 revision 执行 `svn update`
- **THEN** 工作副本文件内容 SHA-256 与期望一致,且 `svn status` 无异常

#### Scenario: 校验和通过无 E155017

- **WHEN** 代理从本地内容库服务文件字节(并已重写骨架 checksum)
- **THEN** svn 客户端不报 `E155017: Checksum mismatch`

### Requirement: 强制 send-all=false 分离元数据与内容

代理 SHALL 改写 `REPORT` update-report 请求体,将 `send-all` 属性设为 `"false"`,使服务器只返回骨架(元数据:path/rev/checksum,无内联 `<txdelta>`)。改写 SHALL 保持 XML 结构与命名空间(`S:` = `svn:`、`D:` = `DAV:`)完整。

#### Scenario: REPORT 请求被改写

- **WHEN** 代理收到 `REPORT` 请求
- **THEN** 转发给上游的请求体中 `send-all` 属性为 `"false"` 且 XML 仍可被解析器重新解析

#### Scenario: 骨架响应无内联内容

- **WHEN** 上游返回骨架响应
- **THEN** 响应体中文件条目不含内联 `<S:txdelta>` 窗口

### Requirement: 合成 GET 响应服务文件内容

对 `GET .../!svn/ver/<rev>/<path>`,代理 SHALL NOT 转发到上游;代理 SHALL 从本地内容库返回 `200 OK` 全文字节,`Content-Type` 为该文件 mime 类型(默认 `text/plain`),`Content-Length` 精确,且不含 `svndiff` content-encoding。

#### Scenario: GET 命中本地内容库

- **WHEN** 客户端发起 `GET .../!svn/ver/<rev>/<path>` 且对应文件存在于本地内容库
- **THEN** 代理返回该文件全文且不回源上游

#### Scenario: 响应头正确

- **WHEN** 代理合成 GET 响应
- **THEN** 响应 `Content-Type` 非 svndiff 类型、`Content-Length` 与字节长度精确一致

### Requirement: 上游零文件体字节回源

一次经代理的 update 会话期间,SVN 服务器 `access.log` SHALL 显示零个 `GET .../!svn/ver/...` 到达上游(全部被代理拦截);只有元数据请求(OPTIONS/PROPFIND/REPORT)到达服务器。

#### Scenario: 更新会话零文件体回源

- **WHEN** 一次 `svn update` 经代理完成
- **THEN** Ubuntu 服务器 `access.log` 在该会话窗口内对 `!svn/ver/` 的 GET 计数为 0

### Requirement: 骨架 checksum 与所服务字节一致

代理 SHALL 对本地内容库中每个文件的字节重算 `sha256`/`sha1`/`md5`,并覆写骨架中对应文件条目声明的 `sha256-checksum`/`text-content-sha1`/`text-content-md5`,使声明值等于所服务字节的实际哈希。

#### Scenario: 声明值与服务值一致

- **WHEN** 代理重写骨架中某文件条目的 checksum
- **THEN** 重写后的 checksum 等于本地内容库中该文件字节的重算哈希

### Requirement: 纯函数单元可测

XML 改写、checksum 重算、GET 响应合成等纯函数 SHALL 有单元测试覆盖:REPORT 请求改写保持 XML 有效性;骨架重算哈希与内容库字节一致;GET 响应头正确。

#### Scenario: REPORT 改写保持 XML 有效

- **WHEN** 单元测试对 `rewrite_report_request()` 输入带 `send-all="true"` 的请求体
- **THEN** 输出为 `send-all="false"` 且可被 XML 解析器重新解析

#### Scenario: 骨架重算哈希一致

- **WHEN** 单元测试对 `restamp_skeleton_checksums()` 输入骨架与已知字节
- **THEN** 输出骨架的 checksum 字段等于已知字节的哈希

### Requirement: 协议基线捕获

spike SHALL 产出 `docs/spike-capture/protocol-baseline.md`,记录所测 svn 客户端与服务器版本的真实 REPORT 请求/响应与 GET 请求报文形态,使 addon 实现基于观察而非假设。

#### Scenario: 基线文档存在且含真实报文

- **WHEN** spike 完成
- **THEN** `docs/spike-capture/protocol-baseline.md` 存在并记录所测 svn 客户端/服务器版本的 REPORT 与 GET 真实报文片段
