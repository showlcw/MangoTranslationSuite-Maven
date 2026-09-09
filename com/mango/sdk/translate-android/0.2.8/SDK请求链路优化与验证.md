# SDK 请求链路优化与验证

本文件描述 SDK 0.2.8 的请求链路优化。0.2.7 不包含下列新增 API，宿主需升级到 `com.mango.sdk:translate-android:0.2.8`。

## 执行流程

初始化配置 → 恢复用户选择 → 宿主调用预热 → 校验请求并检查内存缓存/相同在途请求 → 有界排队 → 创建本次请求独立的引擎控制器 → 校验会员、配置、冷却与连通性 → 异步 HTTP → 后台校验结果/执行回退 → 主线程通知宿主。

引擎 ID、语言代码、用户选择、各引擎配置和额度仍独立。跨平台 JSON 没有变化。预热仍为无凭据、无文本的 HTTPS HEAD，不占翻译额度。

## 默认行为

- SDK 创建的 Translator 共用请求链调度器：最多 8 条执行中的请求链、64 条等待请求。等待队列满时返回现有的 `RATE_LIMITED`，不发起 Provider 请求。
- HTTP 使用 Retrofit 异步适配器，OkHttp 最多 16 个异步请求、每域名 4 个，包含预热和探测；共享原连接池。
- 总预算从提交请求开始计算，包含排队和所有回退。每次 HTTP 使用引擎超时与剩余预算中的较小值，并保留绝对截止时间。截止后取消当前链；主线程繁忙时，错误通知只能在主线程恢复后送达。
- 结果校验和回退在后台完成，公开的翻译结果/错误回调在主线程执行。元数据钩子线程未指定，宿主必须快速返回。
- 同一 Translator 中，只有文本/批次顺序、源目标语言、引擎、配置快照、会员状态和超时配置完全一致的在途请求才合并。合并订阅共用原请求链的剩余预算，不延长原请求的生命期。
- 非 AI 引擎默认缓存 15 秒，最多 64 条，原文与译文字符总量最多 128 Ki 字符。缓存只在该 Translator 的内存中；不落盘。AI 不缓存完成结果，回退到其他引擎的结果也不缓存。
- 命中缓存/合并请求不新增真实 Provider 尝试，因此不额外扣 SDK 的日请求次数。缓存不会共享到另一个 Translator。配置和会员变化后不复用旧结果；过期配置的在途结果不会交付为成功。
- 网络切换使预热去重失效；旧网络的预热回调不能恢复新网络的预热记录。仅被标记为传输故障的冷却可在换网后失效，HTTP 限流、鉴权拒绝和 TLS 证书错误的冷却保留。

## 单请求取消和缓存控制

```java
Translator translator = TranslationSdk.createTranslator();
TranslationCall call = translator.translateCancellable(
        new TranslationRequest("Hello", LanguageConstant.ENGLISH, LanguageConstant.CHINESE), callback);

call.cancel();                // 取消这个订阅，其他请求不受影响
translator.clearResultCache(); // 清空这个 Translator 的内存结果
TranslationSdk.configureResultCache(0); // 全局关闭结果复用；可设置 0..60000 毫秒
translator.close();           // 取消所有订阅并清理该对象的缓存
```

示例语言 ID 需使用对应引擎公布的规范语言 ID（实际接入优先使用 `LanguageConstant`，不要自行从 Provider code 推导）。

取消一个合并订阅仅停止它的通知；最后一个订阅取消时才取消底层请求。取消后不再通知该订阅。连续截图场景可保存上一个 `TranslationCall`，新画面到达时主动取消旧画面请求；SDK 不自行推断不同请求是否属于同一画面。

缓存开关在下一次提交及请求完成时检查；需要立即清除对象中的已保存文本时调用 `clearResultCache()` 或 `close()`。要求每次都真实请求的宿主应关闭缓存。

## 耗时与连接观测

`TranslationHost.onNetworkRequestFinished(TranslationNetworkTiming timing)` 覆盖 `translation`、`preconnect`、`probe`。

| 字段 | 含义 |
| --- | --- |
| `chainId` | 翻译请求链关联号；预热、独立探测或无请求链上下文时为 0 |
| `connectionId` | 进程内连接编号，可关联预热与真实请求；没有获得连接时为 0 |
| `reusedConnection` | 该连接是否曾被已观测的 SDK 请求获得过 |
| `connectionAttempts` | 本次调用发起的建连次数 |
| `connectionWaitMs` | 从 HTTP callStart 到获得连接，含异步 HTTP 队列、DNS、建连等；不是独立可累加阶段 |
| `dnsMs` / `connectMs` / `tlsMs` | DNS、建连、TLS 耗时；重复阶段累计，`connectMs` 包含 TLS，不能再次相加 |
| `responseWaitMs` | 发送完请求到收到响应头事件的间隔，包含网络和服务端等待，不等于服务端处理时间 |
| `bodyMs` / `totalMs` | 响应体读取阶段与 HTTP 总耗时 |

没有发生的阶段为 `-1`。事件不含域名、完整 URL、原文、译文或凭据。

`onTranslationFinished(requestId, chainId, engineId, outcome, source, queueMs, callbackWaitMs, totalMs)` 用于公开订阅级别的耗时统计：

- `requestId` 区分每个宿主调用，`chainId` 关联同一次真实执行和回退；缓存命中的 `chainId` 为 0。
- `source` 为 `network`、`merged` 或 `cache`。
- `queueMs` 为 SDK 请求链排队时间，`callbackWaitMs` 为结果就绪后等待主线程的时间，`totalMs` 到主线程准备通知宿主为止。
- 成功/错误订阅上报，主动取消的订阅不再交付完成回调。

兼容保留 `onRequestFinished`、`onProviderAttemptFinished`、`onEngineSkipped` 和 `onFallback`。已有总耗时与新分段耗时不能直接相加。

## 验证口径

自动化覆盖请求合并、单订阅和整链取消、缓存命中/关闭/清空、配置与语言隔离、AI/回退不缓存、排队容量、绝对截止时间、后台处理、主线程交付，以及换网后只清除网络故障冷却。

本地回环 HTTP 服务验证实际 HEAD 与后续业务路径请求复用同一连接；真机测试复用同一类隔离服务，不调用生产翻译接口。它证明 SDK 连接池与调度行为，不代表生产网关的 TLS、上游初始化或翻译质量验收。

接入方报告中的额外约 700ms 尚需用新事件在同机、同网、相同文本上重测。应分别统计冷启动、连续请求、换网、弱网的成功率、P50/P95、实际请求数和缓存命中率；不得把缓存命中耗时混入真实网络请求延迟。

Google 连通性状态缺失时，应返回宿主渠道能力值，让 SDK 异步探测兜底；不要将缺失状态永久当作不可达。

## 验证记录

2026-09-09，最终源代码验证结果：

| 检查 | 结果 |
| --- | --- |
| 根验证 `scripts/verify.ps1` | 通过；引擎、语言、错误码、公开 API、边界检查全部通过 |
| SDK JVM/Robolectric 测试 | 142 项，0 失败、0 错误、0 跳过 |
| 平台测试 | 25 项，0 失败 |
| 自建服务测试 | 61 项，0 失败 |
| `assembleRelease` | 通过；AAR 不含 Voice base 类检查通过 |
| `connectedDebugAndroidTest` | vivo V2229A / Android 16，4 项通过 |
| `git diff --check` | 通过 |

真机四项覆盖：真实回环 HTTP 连接复用、换网预热代次隔离、剩余截止时间限制，以及后台执行/主线程交付/合并取消/缓存命中。合并与缓存测试中，两次相同在途订阅（取消其中一个）及一次后续缓存命中，只执行了一次底层请求。

真机运行器有 `androidx.test.services` 未安装导致的辅助 appops 提示，最终仪器测试 XML 和 Gradle 结果均通过；测试包已由运行器清理。没有运行宿主 App 的 R8 Release 完整翻译回归，没有改动生产服务；SDK 制品按 0.2.8 单独发布。

- [根验证日志](../build/release-0.2.8/verify.log)
- [真机验证日志](../build/sdk-optimization-device-tests.log)
- [SDK 单元测试报告](../mobile-sdk/mango-translate/build/reports/tests/testDebugUnitTest/index.html)
- [Release AAR](../mobile-sdk/mango-translate/build/outputs/aar/mango-translate-release.aar)

AAR SHA-256：`EA26268DA32C35E60A26D0F2BA43EAA2C31134020F8867318A3C477740FB8410`。对应 `translate-android-0.2.8.aar`，可与随包的 SHA256SUMS.txt 核对。
