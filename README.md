# Docs

个人技术文档与排障记录集。多数为「**现象 → 根因 → 修复 → 验证**」结构的实战笔记。

> **脱敏约定**：仓库为公开仓库，所有文档均已脱敏——不包含真实密钥、内部域名、私有 CA 名称或个人绝对路径；文中出现的均为占位符（如 `api.example.net`、`<GATEWAY_ENV>`）。

---

## 目录

### AI / 开发工具

| 文档 | 说明 |
| --- | --- |
| [DeepSeek Harness 接入第三方 OpenAI 兼容网关](deepseek-harness-third-party-gateway-zh.md) | 把 `dsh` 接到用私有 CA 签发证书的第三方网关时踩的四类问题：<br>① Node TLS 信任 —— `SELF_SIGNED_CERT_IN_CHAIN` 被吞成 `TRANSPORT` 重试<br>② 手工路由缺 reasoning 元数据导致思考级别不可选<br>③ 用探测而非照抄来定输出上限<br>④ `web_search` 为何要官方 key，以及自写 `ctx.web` provider 走网关 Responses API（**含完整插件源码**） |

### 移动端构建与集成

| 文档 | 说明 |
| --- | --- |
| [iOS 链接错误 `__swift_FORCE_LOAD_$_swiftCompatibility56`](ios-linker-error-swiftCompatibility56.md) | arm64 下 `libswiftCompatibility56` 未定义符号的根因、正确解法与临时绕过（常见于集成旧版 Firebase 组件） |
| [CocoaPods 私有 Specs 推送排障](cocoapods-private-specs-troubleshooting-zh.md) | `pod repo push` 的 `repo is not clean` / `Permission denied` 等问题；工作区脏点与 root 属主目录的处理 |
| [UnrealEngine 混编方案](UnrealEngine%E6%B7%B7%E7%BC%96%E6%96%B9%E6%A1%88%E8%A7%A3%E5%86%B3%E6%96%B9%E6%A1%88.docx) | UE 引擎在 iOS 平台不支持 OC/Swift 混编的改造方案：改 `XcodeProject.cs`、开启 `CLANG_ENABLE_MODULES` / `SWIFT_VERSION` 等（以 UE4.23 为例） |
| [跨端编译](%E8%B7%A8%E7%AB%AF%E7%BC%96%E8%AF%91.docx) | iOS / Android / Windows / PlayStation 各平台的交叉编译工具链选型 |

### 游戏引擎与运行时

| 文档 | 说明 |
| --- | --- |
| [团结引擎](%E5%9B%A2%E7%BB%93%E5%BC%95%E6%93%8E.docx) | 微信小游戏包体优化：引擎轻量化、托管代码精简（`ManagedStrippingLevel`）、引擎代码剔除、内建 Package 剔除 |
| [APM](APM.docx) | iOS 崩溃捕获原理：RunLoop 与事件循环、Mach 异常 → signal/NSException 的转换链路、挂起线程取栈的过程 |

### 网络与支付

| 文档 | 说明 |
| --- | --- |
| [苹果内购（IAP）支付失败排查报告](IAP_connectivity_report.md) | 根因是到苹果 IAP 节点 `p12-buy.itunes.apple.com:443` 的连接超时（`-1001`）；含日志证据链、直连/代理对比，以及监控值 `28999` 实为 `-1001 + 30000` 偏移的推断 |

### 开发环境

| 文档 | 说明 |
| --- | --- |
| [Homebrew shellenv 权限问题修复](homebrew-shellenv-fix.md) | macOS 受限环境下 `brew shellenv` 报 `/bin/ps: Operation not permitted` 的补丁与验证步骤 |
| [win-config](win-config.md) | Windows 开发机常用应用清单与配置路径（内容较零散，待补充） |

---

## 维护约定

- **命名**：`kebab-case`；中文文档在文件名末尾加 `-zh` 后缀（历史文档因命名不统一未强制回改）。
- **结构**：优先按 `背景 / 根因分析 / 修复步骤 / 验证` 组织，便于日后直接照做。
- **脱敏**：提交前检查真实密钥、内部域名、私有 CA 名称、个人绝对路径。
- **格式**：`.md` 可直接在 GitHub 预览；`.docx` 为二进制文件，需下载后查看。