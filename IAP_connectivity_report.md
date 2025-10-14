# 苹果内购（IAP）支付失败排查报告

## 结论
- 根因：到苹果 IAP 节点 `p12-buy.itunes.apple.com:443` 的网络连接超时，导致 StoreKit 购买失败（NSURLErrorDomain `-1001`）。
- 你方监控 `28999` 实为在上报时加了 `30000` 的偏移（`28999 - 30000 = -1001`）。
- 在挂海外代理时支付正常，说明是本地线路/策略到苹果 `p12-buy` 节点的路由/连通性问题。

## 关键证据
- 日志（文件：`log.txt`）
  - 发起支付：`log.txt:2051` `StoreKitServiceConnection processPayment`
  - 加入订单：`log.txt:2066` `Adding payment for com.tencent.lolm.ticket_10`
  - 网络超时：`log.txt:38900` `NSURLErrorDomain Code=-1001 请求超时`
  - 购买报错：`log.txt:39310` `Purchase completed with error … Code=-1001`
  - 支付失败：`log.txt:39317` `Payment failed … Code=-1001`
  - App回调：`log.txt:39333` `<SKPaymentQueue …> Payment completed with error … Code=-1001`

- 本环境连通性测试（直连）
  - DNS 解析：`p12-buy.itunes.apple.com -> 17.156.128.11`
  - 端口连通：`nc -vz -w 5 p12-buy.itunes.apple.com 443` → `Operation timed out`
  - HTTPS 探测：
    - `curl -svI --connect-timeout 5 --max-time 8 https://p12-buy.itunes.apple.com/WebObjects/MZBuy.woa/wa/inAppBuy`
    - 结果：`Failed to connect … port 443 … Timeout was reached`

## 代理 vs 直连对比
- 直连（本环境）：TCP 建连与 HTTPS 请求均超时（如上）。
- 代理（你方反馈）：挂海外代理后支付正常，说明代理路径可达苹果 IAP 节点，原网络路径存在路由/策略/链路问题。
- 备注：CLI 环境未继承系统代理环境变量，如需在本环境复测代理，请按下方“验证命令（代理）”设置并执行。

## 自检命令（建议直接复制执行）

### 基础连通性（直连）
- 端口：`nc -vz -w 5 p12-buy.itunes.apple.com 443`
- TLS：`echo | openssl s_client -connect p12-buy.itunes.apple.com:443 -servername p12-buy.itunes.apple.com -tls1_2 -brief`
- HTTP：`curl -svI --connect-timeout 5 --max-time 10 https://p12-buy.itunes.apple.com/WebObjects/MZBuy.woa/wa/inAppBuy`
- DNS：`dig +short p12-buy.itunes.apple.com` 或 `nslookup p12-buy.itunes.apple.com`

判读：
- “连接不成功”= `nc` 超时/拒绝，`openssl` 握手失败或卡住，`curl` 超时/握手失败。
- “连接成功但服务不允许”= `curl` 快速返回任意 HTTP 状态（即便 403/405/400 也代表网络/TLS 正常）。

### 验证命令（代理）
- HTTP/HTTPS 代理：
  - `export HTTPS_PROXY=http://<host>:<port>` 或 `export HTTPS_PROXY=http://user:pass@<host>:<port>`
  - `curl -svI --connect-timeout 5 --max-time 10 https://p12-buy.itunes.apple.com/WebObjects/MZBuy.woa/wa/inAppBuy`
- SOCKS5 代理：
  - `export HTTPS_PROXY=socks5h://<host>:<port>`
  - `curl -svI --connect-timeout 5 --max-time 10 https://p12-buy.itunes.apple.com/WebObjects/MZBuy.woa/wa/inAppBuy`
- 或临时指定：
  - `curl -x http://<host>:<port> -svI --connect-timeout 5 --max-time 10 https://p12-buy.itunes.apple.com/WebObjects/MZBuy.woa/wa/inAppBuy`

期望：代理路径应快速返回 HTTP 响应头（403/405/400 皆可），即证明“代理路径连通正常”。

## 修复建议（网络侧）
- 放通域名与端口（优先基于 SNI/域名）：
  - `p*-buy.itunes.apple.com:443`、`buy.itunes.apple.com:443`
  - 以及其 CNAME/别名：`*.itunes-apple.com.akadns.net`
- 不做 TLS 拦截（MITM）；确保到 Apple 前缀网段（如 `17.0.0.0/8`）无阻断/限速。
- 异常时对比其他 Apple 买单节点：`p1-buy.itunes.apple.com`、`p16-buy.itunes.apple.com` 等，定位是否单点路由问题。

## 客户端健壮性
- 对 `NSURLErrorDomain -1001`（请求超时）做指数退避重试与“稍后再试”提示；保留“恢复购买”。
- 采集 `URLSessionTaskMetrics`（DNS、Connect、TLS、TTFB）定位具体阶段超时。
- 上报同时携带错误域（Error Domain）与原始码，避免跨域码冲突；仅对负数网络错误做偏移或统一映射。

## 对网络团队的放通清单
- 目标：Apple IAP 买单服务
- 端口：`443/TCP`
- 域名：`p*-buy.itunes.apple.com`、`buy.itunes.apple.com`、及 `*.itunes-apple.com.akadns.net`
- 要求：允许直连，不做 TLS 检查/代理劫持；确保运营商/出口到 Apple 段的健康路由。

## 附：错误码映射说明
- 你方采集 `28999` = 原始 `-1001`（请求超时）+ 30000 偏移。
- 建议：同时记录错误域（如 `NSURLErrorDomain`、`SKErrorDomain`）；或统一映射常见网络错误：
  - `-1001` 超时、`-1009` 无网络、`-1005` 连接丢失 等。

