# MangoPlatform 与 Android 翻译 SDK 统一接入文档

> 0.2.8 包含首次翻译延迟修复、SDK异步探测、分钟级失败冷却及完整兜底取消保护。0.2.6 不包含这些修复。宿主接入与验收见 [首次翻译延迟修复与验收](首次翻译延迟修复与验收.md)。

> 0.2.8 新增单请求取消、请求合并、可关闭的内存缓存、网络分段与完整链路耗时事件；详见 [SDK请求链路优化与验证](SDK请求链路优化与验证.md)。

> 对外接入的唯一正式文档。适用于所有接入 MangoPlatform 的业务 App 项目，不依赖任何
> 现有 App 的目录、项目代号或私有实现。接入方只需要交付本文件，无需组合其他文档。

## 1. 文档目标

完成本文后，一个新项目应具备：

1. 在 MangoPlatform 登记自己的 App。
2. 由自己的业务服务端安全同步该 App 的翻译引擎配置。
3. 由自己的移动端从业务服务端取得密文配置并初始化 Android SDK。
4. 展示后台允许用户选择的引擎和每个引擎自己的语言列表。
5. 直接展示 SDK 内置的官方品牌图标，不在 App 内重复映射。
6. 保存、恢复和主动切换整数引擎 ID。
7. 使用 SDK 内置 Provider、超时、冷却和失败切换完成单条或批量翻译。
8. 可选接入中台自建翻译调度和 App 互推。

本文中的 `{appId}` 必须替换为中台生成的真实 `app_xxx`，不能填写项目名称、显示名称、
Android 包名或自行约定的别名。

## 2. 最终架构与强制边界

```text
MangoPlatform
  │ HTTPS + X-App-Backend-Key（仅业务 App 服务端持有）
  ▼
业务 App 服务端
  │ 自己的 App/用户认证 + HTTPS；只下发本 App 的密文配置
  ▼
业务 App Android 客户端
  │ TranslationHost 提供 App 级解密、身份和会员状态
  ▼
Mango Translation Android SDK
  │ 每个 Translator 使用自己的语言表、Provider code、协议和凭据
  ▼
第三方翻译 Provider / 自建 Provider
```

强制规则：

- Android 客户端和 SDK 永远不能直接访问 MangoPlatform。
- `X-App-Backend-Key` 永远不能进入 APK、SDK、Git、日志或崩溃报告。
- 业务 App 服务端不解密 Provider 凭据，只保存和下发 App 绑定密文。
- SDK 封装 Google、百度、Qwen、RapidAPI 等具体协议，宿主 App 不重复实现 Provider。
- 中台“App 翻译配置”和“自建节点调度”是独立业务模块；自建翻译在 SDK 中仍是普通 Provider。
- 引擎 ID 是中台返回的整数 `id`。App 不能保存枚举序号、显示名称、别名或自行映射后的值。
- 每个引擎独立维护语言列表和 Provider code，禁止跨引擎统一转换或复制映射。
- 引擎名称、说明和官方品牌图标属于 SDK 展示契约；中台和宿主 App 不维护第二套映射。
- SDK 不接收 `versionCode`、HMAC、规则范围或规则包；版本规则只由 App 后台选择。

## 3. 四类值必须分清

| 值 | 示例 | 持有位置 | 用途 | 是否可恢复 |
| --- | --- | --- | --- | --- |
| App ID | `app_0123456789abcdef` | 中台、业务服务端 | 标识中台中的真实 App | 可在“App 管理”查看 |
| App 后台访问密钥 | `mat_...` | 仅业务服务端 | 业务服务端访问中台内部接口 | 完整值只显示一次，只能轮换 |
| 翻译密钥加密 Key | 32 UTF-8 字节 | 中台、App 受保护实现 | 加解密该 App 的 Provider 凭据 | 中台 App 管理可维护 |
| Provider 凭据 | STRING 或 JSON | 中台；SDK 内仅短时明文 | 调用某个翻译供应商 | 管理端维护，可配置多条 |

`MANGO_PLATFORM_BACKEND_KEY` 是业务服务端环境变量名称，对应中台页面的“App 后台访问密钥”。
它不是“翻译密钥加密 Key”，也不是 Google、百度或 Qwen 的 Provider Key。

## 4. 中台管理端准备

### 4.1 登记 App

进入“App 管理”并登记 App。保存后系统生成不可修改的：

```text
appId = app_xxxxxxxxxxxxxxxx
```

记录 App 卡片上的真实 `appId`。显示名称和包名只是元数据。

### 4.2 配置翻译密钥加密 Key

在 App 编辑界面配置32 UTF-8字节“翻译密钥加密 Key”。该值必须与移动端受保护解密实现
使用的值完全一致。修改它时，中台会重新加密该 App 已有的 Provider 凭据。

这项32字节限制只适用于 App 的翻译密钥加密 Key，不适用于翻译引擎 Provider 凭据。

### 4.3 生成 App 后台访问密钥

在 App 卡片右侧点击钥匙按钮“轮换后台密钥”。弹窗会显示一次完整 `mat_...`：

1. 立即复制到业务服务端 Secret 管理系统。
2. 不要截图、提交 Git 或发到客户端配置。
3. 关闭弹窗后中台只保留哈希和前缀，无法恢复完整值。
4. 再次点击会生成新值并立即使旧值失效。

### 4.4 配置 App 翻译引擎

进入“App 翻译配置”，选择目标 App 后必须先选择版本规则，再逐个配置引擎。V2 引擎和
新密钥只写入新版规则，不能写入 `legacy-v1`。例如实时语音翻译当前使用：

```text
legacy-v1       versionCode 0–208
sdk-v209-live   versionCode 209–∞
```

App 后台命中哪条规则，中台就只允许它向该客户端返回该规则的 `config`。规则之间的
Provider Key、连接地址、模型、每日次数和 App 加解密 Key 必须独立。

在选中的版本规则内配置：

- `enabled`：是否把该引擎下发给该 App；未启用时不会出现在配置响应中。
- `vipOnly`：是否只允许会员使用；下发字段名为 `vip`。
- `selectable`：是否出现在用户主动切换列表。
- `dailyLimit`：可选的单设备每日请求尝试次数；留空表示不限量。
- `credentials[]`：该 Provider 的一条或多条 STRING/JSON 凭据。
- `activeProfile`：管理端选中的 Provider 档案；中台只会把该档案的凭据放入客户端 `keys[]`。

`enabled=true, selectable=false` 表示引擎对用户隐藏，但 SDK 可按内置策略用于当前请求的失败兜底。
配置保存即生效，不存在客户端使用的“优先级”字段；自动降级顺序由 SDK 版本维护。

### 4.5 智谱官方与 OpenRouter 两套档案

`ZHIPU_FLASH(46)` 的 V2 配置不保留缺省地址或 STRING Key 兼容。两套 Profile 都必须是
完整 JSON，且中台只把 `activeProfile` 对应的密文放入最终 `keys[]`。

智谱官方：

```json
{
  "baseUrl": "https://open.bigmodel.cn/api/paas/v4/",
  "apiKey": "provider-key",
  "model": "glm-5.3-flash",
  "protocol": "openai",
  "reasoningEnabled": false
}
```

OpenRouter：

```json
{
  "baseUrl": "https://openrouter.ai/api/v1/",
  "apiKey": "provider-key",
  "model": "z-ai/glm-5.3-flash",
  "protocol": "openrouter",
  "reasoningEnabled": true,
  "providerRouting": {
    "sort": "latency",
    "max_price": {"prompt": 0.15, "completion": 0.50},
    "require_parameters": false,
    "allow_fallbacks": true,
    "data_collection": "deny"
  }
}
```

GLM 5.3 Flash 的 OpenRouter 端点强制启用推理；`reasoningEnabled=false` 会返回 HTTP 400。
切换供应商时只切 `activeProfile`，不修改引擎 ID、语言表或用户保存的选择。

## 5. 业务 App 服务端：必接

### 5.1 环境变量

```text
MANGO_PLATFORM_BASE_URL=https://platform.screenlate.com
MANGO_PLATFORM_APP_ID=app_中台生成的真实ID
MANGO_PLATFORM_BACKEND_KEY=mat_完整后台访问密钥
```

服务启动时必须校验 `APP_ID` 和 `BACKEND_KEY` 非空。生产环境不得为它们提供代码默认值。

### 5.2 拉取全部已发布版本规则

```http
GET {MANGO_PLATFORM_BASE_URL}/internal/v2/apps/{appId}/translation-rules
X-App-Backend-Key: mat_xxx
Accept: application/json
```

中台一次返回全部已发布规则；请求中不携带客户端版本：

```json
{
  "revision": 1788403524704,
  "rules": [
    {
      "ruleId": "app_xxx:legacy-v1",
      "minVersionCode": 0,
      "maxVersionCode": 208,
      "keyVersion": 1,
      "config": {"engines": []}
    },
    {
      "ruleId": "app_xxx:rule-xxx",
      "minVersionCode": 209,
      "maxVersionCode": null,
      "keyVersion": 2,
      "config": {"engines": []}
    }
  ]
}
```

App 后台把完整规则包按 `appId + revision` 原子缓存。每次自己的客户端请求到达时，从
App 自己可信的请求上下文读取 Android `versionCode`，匹配唯一规则：

```java
List<Rule> matches = cachedBundle.rules().stream()
        .filter(rule -> rule.minVersionCode() <= versionCode)
        .filter(rule -> rule.maxVersionCode() == null
                || versionCode <= rule.maxVersionCode())
        .toList();
if (matches.size() != 1) {
    throw new TranslationConfigUnavailableException("version rule mismatch");
}
TranslationConfig configForClient = matches.get(0).config();
```

零条或多条命中都必须返回配置不可用，不能合并规则、猜测最近版本或把整包发给客户端。
历史客户端没有版本字段时只能选择 `legacy-v1`。`versionCode` 不发送给中台，中台也不
接收 HMAC；其内部接口鉴权始终只有 `appId + X-App-Backend-Key`。

命中规则中的 `config` 格式：

```json
{
  "engines": [
    {
      "id": 8,
      "keys": ["app-bound-ciphertext"],
      "vip": false,
      "selectable": true,
      "dailyLimit": null
    },
    {
      "id": 36,
      "keys": ["ciphertext-one", "ciphertext-two"],
      "vip": true,
      "selectable": false,
      "dailyLimit": 100
    }
  ]
}
```

字段定义：

| 字段 | 类型 | 含义 |
| --- | --- | --- |
| `id` | integer | App 保存、恢复和切换使用的真实引擎 ID |
| `keys` | string[] | 按后台顺序排列的 App 绑定密文 Provider 值，允许多条 |
| `vip` | boolean | 是否要求实时会员资格 |
| `selectable` | boolean | 是否展示在用户可切换列表 |
| `dailyLimit` | integer/null | 可选的单设备本地自然日请求尝试上限；空值不限量，耗尽后 SDK 自动降级 |

响应不会包含引擎名称、语言表、节点、权重、QPS、RPM、并发、健康状态或优先级。
`dailyLimit` 是客户端成本分层限制，不是跨设备全局防刷额度。

SDK 对引擎目录采用前向兼容解析：未知整数 ID 不校验其新增字段、不解密其凭据并直接
跳过。已知引擎按自身校验凭据：某一条密文解密失败会继续检查该引擎的下一条凭据；
某个引擎最终没有有效凭据时只移除该引擎，不影响同一配置中的其他引擎、名称或图标。
只有整个非空配置没有任何可用引擎，或存在重复 ID、缺少必填结构字段等配置级错误时，
才拒绝新快照并保留最近一次成功缓存。涉及 App 加解密 Key 或 Provider Key换代时，App 后台必须
使用 V2 规则包按独立的 Android `versionCode` 选择一套配置，不能把多套规则交给 SDK；
`versionCode` 不发送给中台，也不进入中台鉴权。中台内部接口只使用
`appId + X-App-Backend-Key`。

### 5.3 服务端同步和存储规范

业务服务端必须保存“最近一次完整有效的密文规则包”，推荐实现：

```text
启动/定时/管理端主动同步
  → 调用中台接口
  → 校验 HTTP 200
  → 校验规则范围连续且不重叠、revision、config 结构
  → 原子写入最近成功规则包
  → 更新 syncedAt、revision 和 ruleCount
```

失败处理：

- 网络、DNS、超时、401、403、5xx 或格式错误：保留最近成功规则包，不覆盖为空。
- 命中规则明确包含 `{"engines":[]}`：这是有效配置，表示该版本没有可用引擎。
- 不允许把异常响应正文、请求 Header 或配置正文写入日志。
- 数据库保存的是 App 绑定密文，无需业务服务端再次解密。

建议同时提供：

- 每天一次定时同步。
- 服务启动后的首次同步或人工主动同步。
- 仅管理员可调用的主动同步入口。
- 只展示状态、不展示配置正文的管理页面。
- 同步状态：是否配置、是否有有效快照、引擎数量、最近成功时间、最近错误类型。

Java HTTP 客户端示例：

```java
String url = platformBaseUrl + "/internal/v2/apps/" + appId
        + "/translation-rules";
HttpRequest request = HttpRequest.newBuilder(URI.create(url))
        .timeout(Duration.ofSeconds(10))
        .header("X-App-Backend-Key", platformBackendKey)
        .header("Accept", "application/json")
        .GET()
        .build();

HttpResponse<String> response = httpClient.send(
        request, HttpResponse.BodyHandlers.ofString(StandardCharsets.UTF_8));
if (response.statusCode() != 200) {
    throw new TranslationConfigSyncException("platform HTTP " + response.statusCode());
}
TranslationRuleBundle bundle = objectMapper.readValue(
        response.body(), TranslationRuleBundle.class);
validatePublishedRangesAndConfigs(bundle);
ruleBundleRepository.replaceAtomically(
        appId, bundle.revision(), objectMapper.writeValueAsString(bundle));
```

### 5.4 提供给移动端的专用接口

每个业务 App 服务端应提供自己的接口，推荐：

```http
GET /api/translation/config
<App 自己的客户端签名或用户认证>
```

推荐响应：

```json
{
  "code": 200,
  "message": "success",
  "data": {
    "engines": [
      {"id": 8, "keys": ["app-bound-ciphertext"], "vip": false, "selectable": true, "dailyLimit": null}
    ]
  }
}
```

要求：

- 只向本 App 的合法客户端提供。
- App 后台必须先按该客户端 `versionCode` 选择唯一规则；接口只返回命中规则的 `config`。
- 不返回 `revision` 之外的规则元数据，不返回其他版本配置，也不让 SDK 再选择版本。
- 使用 HTTPS 和 `Cache-Control: no-store`。
- 不转发 `X-App-Backend-Key`。
- 不解密、打印或在公共缓存中保存 `keys[]`。
- 没有成功快照时返回503，不能伪造 `engines:[]`。
- 服务端真实同步到空数组时正常返回200和 `engines:[]`。

## 6. Android SDK 制品接入

### 6.1 版本要求

- `0.2.6` 增加基于现有 `vip + TranslationHost.isVip()` 的免费执行链。
- `0.2.6` 增加按任务指定引擎和 SDK 连接池预热。
- `0.2.6` 包含自建有道编号53、可配置超时及首请求延迟优化；已有编号22保持停用。
- 当前公开版本 `0.2.8`：Android `minSdk 24`
- 推荐 `compileSdk 37`
- Java 8字节码兼容
- 当前公开版本：`0.2.8`（`0.2.0` 停止使用）
- 运行时可通过 `TranslationSdk.VERSION` 读取制品自身版本号

`0.2.6` 包含逐引擎凭据隔离、兼容 VectorDrawable、智谱动态 Profile、最终备用顺序、
`getIconRes()` 图标契约及 V2 接入契约。

`0.2.6` 将每日限额的本地日期计算从 API 26 的 `java.time` 改为基于当前时间与
`TimeZone` 偏移的直接计算，不要求宿主启用 Core Library Desugaring。

发布状态（2026-09-09）：

- 公共 Maven POM：`https://showlcw.github.io/MangoTranslationSuite-Maven/com/mango/sdk/translate-android/0.2.8/translate-android-0.2.8.pom`
- 公共 Maven AAR：`https://showlcw.github.io/MangoTranslationSuite-Maven/com/mango/sdk/translate-android/0.2.8/translate-android-0.2.8.aar`
- Maven 坐标：`com.mango.sdk:translate-android:0.2.8`
- AAR 大小：`1,300,045 bytes`
- AAR SHA-256：`EA26268DA32C35E60A26D0F2BA43EAA2C31134020F8867318A3C477740FB8410`

正式制品同时发布到公开 GitHub Pages Maven 仓库和私有 GitHub Packages。普通接入
使用 Pages，无需 GitHub 登录或 Token。在项目 `settings.gradle.kts` 的依赖仓库中增加：

```kotlin
dependencyResolutionManagement {
    repositories {
        google()
        mavenCentral()
        maven("https://showlcw.github.io/MangoTranslationSuite-Maven/")
    }
}
```

GitHub Packages 仍作为需要访问控制的备用仓库；它要求 `read:packages` PAT。不要把
GitHub 用户名、PAT 或其他凭据写进项目仓库。

业务模块只需声明 Maven 坐标，POM 会带入运行时依赖：

```kotlin
dependencies {
    implementation("com.mango.sdk:translate-android:0.2.8")
}
```

无法访问 GitHub Pages Maven 仓库时，才回退到版本化 AAR：

```text
app/libs/translate-android-0.2.8.aar
```

GitHub Release 备用制品：

```text
https://github.com/showlcw/MangoTranslationSuite/releases/tag/v0.2.8
大小：1,300,045 bytes
SHA-256：EA26268DA32C35E60A26D0F2BA43EAA2C31134020F8867318A3C477740FB8410
```

接入方复制制品应核对 SHA-256，避免使用旧 AAR 或构建失败产生的不完整文件。

```kotlin
dependencies {
    implementation(files("libs/translate-android-0.2.8.aar"))

    // 直接使用 AAR 时没有 Maven POM，必须显式声明传递依赖。
    implementation("com.google.code.gson:gson:2.13.2")
    implementation("androidx.annotation:annotation:1.9.1")
    implementation("io.reactivex.rxjava2:rxjava:2.2.21")
    implementation("io.reactivex.rxjava2:rxandroid:2.1.1")
    implementation("com.squareup.retrofit2:retrofit:2.11.0")
    implementation("com.squareup.retrofit2:converter-gson:2.11.0")
    implementation("com.squareup.retrofit2:converter-scalars:2.11.0")
    implementation("com.squareup.retrofit2:adapter-rxjava2:2.11.0")
    implementation("com.squareup.okhttp3:okhttp:4.12.0")
}
```

AAR 已携带 Consumer ProGuard 规则。宿主不能删除或覆盖它们。

### 6.2 SDK 对外 API

SDK 只提供以下接入模型：

- `Translator`
- `TranslationRequest`
- `BatchTranslationRequest`
- `TranslationCallback`
- `TranslationResult`
- `TranslationSdk.createTranslator()` / `createTranslator(int legacyCode)`
- `TranslationSdk.preconnect()` / `preconnect(int legacyCode)`

Voice、翻译和 TTS 模块统一使用以下语言定义：

- `com.mg.translation.language.LanguageConstant`
- `com.mg.translation.language.LanguageVO`
- `com.mg.translation.language.CountryConstant`
- `com.mg.translation.language.LanguageSourceConstant`

SDK 只提供 `com.mg.translation.language.LanguageConstant` 一份语言标识定义，其中370个
实际语言标识符可供 Voice、翻译和 TTS 模块共同使用；每个标识符都能通过
`LanguageSourceConstant.getOriginalLanguage()` 取得对应的本地原生名称。SDK 只提供
`com.mg.translation.language.LanguageVO` 一个语言模型。Voice 实际使用并保留的扩展能力为
`ocrFlag`、`sourceLanguageIdName`、`topTitle`、`languageKey`、`languageCountry` 和 `clone()`；
没有真实读取链路的 `voiceValue`、识别质量、下载状态和语言级 VIP 已删除。

### 6.3 实现 TranslationHost

```java
final class AppTranslationHost implements TranslationHost {
    @Override
    public CredentialDecryptor getCredentialDecryptor() {
        return ciphertext -> ProtectedBridge.decryptTranslationCredential(ciphertext);
    }

    @Override
    public boolean isAppIdentityValid() {
        return ProtectedBridge.verifyCurrentPackageAndSignature();
    }

    @Override
    public boolean isVip() {
        return AccountSession.get().isVip();
    }

    @Override
    public boolean isGoogleBuild() {
        return BuildConfig.GOOGLE_SERVICES_AVAILABLE;
    }

}
```

SDK只接收解密函数，不接收或保存32字节原始 Key。

### 6.4 启动、刷新和轻缓存

冷启动推荐顺序：

```java
TranslationHost host = new AppTranslationHost();

try {
    // 先尝试 SDK 私有存储中的最近密文快照，保证弱网启动可用。
    TranslationSdk.initialize(applicationContext, "", host);
} catch (IllegalArgumentException noCache) {
    // 首次安装可以建立空运行时；不要把网络失败伪装成后台明确下发的空配置。
    TranslationSdk.initialize(applicationContext, "{\"engines\":[]}", host);
}

appBackendApi.getTranslationConfig().enqueue(new Callback<TranslationConfig>() {
    @Override public void onSuccess(TranslationConfig config) {
        // 传入业务服务端 data 对象对应的 JSON，不要包含外层 code/message。
        TranslationSdk.initialize(applicationContext, gson.toJson(config), host);
        restoreSavedSelection();
        refreshTranslationUi();
    }

    @Override public void onFailure(Throwable error) {
        // 已恢复缓存时继续使用；不得用空数组覆盖。
    }
});
```

SDK缓存规则：

- 只在 App 私有存储中保存完整密文配置 JSON。
- 新配置完成结构校验且至少保留一个可用引擎后，才原子替换运行时和缓存；无效凭据及其所属的
  无可用凭据引擎不会进入新快照。
- `null` 或空字符串表示“没有新响应，尝试使用缓存”。
- 明确的 `{"engines":[]}` 是有效新配置，会覆盖缓存。
- 新配置中的未知 ID直接跳过；单条密文或单个引擎凭据无效时隔离该项，其余有效引擎继续应用。
- JSON 格式错误、重复 ID、缺少配置级必填字段或全部引擎均不可用时抛出异常，并保留旧运行时与缓存。
- App 身份异常、安全事件或需要彻底清理时调用：

```java
TranslationSdk.clearCachedConfig(applicationContext);
```

App 级配置与用户会话相互独立；会员判断只读取 `TranslationHost.isVip()` 的实时结果。
中台和 App 后台不需要新增“免费模式”字段，也不需要为非 VIP 用户生成第二份引擎配置。
它们继续下发每个引擎已有的 `vip` 标识，由 SDK 在每个翻译任务开始时结合实时会员状态
决定本次可执行链路。

## 7. 引擎列表与选择状态

### 7.1 将 App 已保存的整数 ID 应用到 SDK

```java
int savedId = appPreferences.getInt(
        "translation_engine_id", EngineSelectionResult.NO_ENGINE);
EngineSelectionResult result = TranslationSdk.restoreSelectedEngine(savedId);
if (result.getOutcome() == EngineSelectionOutcome.REJECTED) {
    showEngineSelectionError(result.getReason());
}
```

- `appPreferences` 是宿主 App 自己的存储，不属于 SDK。
- `restoreSelectedEngine()` 只把传入 ID 应用到当前进程内存，绝不读取或写入任何持久化存储。
- 没有历史值时传 `NO_ENGINE(-1)`；SDK仅在当前进程选择首个健康、免费且可选的引擎，不会保存它。
- 已保存 ID 不可用、隐藏或退役时返回拒绝原因，不改写历史 ID；会员不足仍保留既有 ID，
  执行时自动进入免费链。
- 翻译失败后的自动兜底只影响当前请求，不能保存实际兜底引擎 ID。

### 7.2 展示后台允许选择的引擎

```java
List<TranslationEngineInfo> engines = TranslationSdk.getSelectableEngines();
for (TranslationEngineInfo engine : engines) {
    renderEngine(engine.getId(), engine.getName(), engine.getDescription(), engine.getIconRes(),
            engine.isLocked(), engine.isAi(), engine.isSlow());
}
```

该列表已经同时过滤：中台未下发、凭据无效、退役、`selectable=false` 和 SDK 不支持的引擎。

引擎名称、优势介绍、官方品牌图标和显示顺序完全由 SDK 本地
`TranslationEnginePresentationCatalog` 提供，后台不下发也不能覆盖展示文案：

- Google 与 Self-hosted Google 使用 Google 品牌展示；历史 `RAPID_*` 展示文案暂沿用
  既有文本，但图标必须使用中性翻译图标，不能冒充 Google 官方通道。
- `BAIDU`、`BD` 统一展示为百度翻译。
- `YOUDAO` 展示为有道翻译；已删除的 `YOUDAO_ME(22)` 仅保留墓碑 ID。
- Microsoft、MyMemory、DeepSeek、Hy-MT2、GPT、Gemma、Gemini、GLM、Qwen 使用各自品牌。
- 已删除的 `TIPS(21)`、`QWEN_TURBO(47)`、`ZHIPU(44)` 和 `YOUDAO_ME(22)`
  不进入可选列表、Factory或中台目录，整数 ID永久不得复用。

SDK 内置默认英文和简体中文品牌写法；没有可靠官方本地名称时保持官方英文品牌名。
引擎优势介绍已覆盖 SDK 的140个地区资源，不得由 App 自行拼接或覆盖。
`getIconRes()` 返回 SDK 自己的 `@DrawableRes`。宿主 ImageView 不得统一 tint 品牌图标，
不得拉伸、改色或把 OpenRouter 图标误当成模型品牌。没有可确认官方品牌的代理路线使用
中性翻译图标，不得借用 Google Logo；图标来源见
`mobile-sdk/品牌图标来源.md`。

### 7.3 用户主动切换

```java
EngineSelectionResult result = TranslationSdk.selectEngine(clickedEngineId);
if (result.isApplied()) {
    appPreferences.putInt("translation_engine_id", result.getSelectedId());
} else if (result.getReason() == EngineSelectionReason.VIP_REQUIRED) {
    openSubscriptionPage();
}
```

会员状态或后台配置变化后：

```java
EngineSelectionResult result = TranslationSdk.refreshSelection();
```

该调用只重新校验，不允许自动写入其他 ID。

`selectEngine()` 表示用户主动点击：非 VIP 点击 `vip=true` 引擎时仍返回
`VIP_REQUIRED`，便于 App 展示订阅入口。`restoreSelectedEngine()` 表示恢复历史选择：用户
会员过期后恢复到 `vip=true` ID 时仍保留该 ID，真正翻译时由 SDK 自动走免费链。两种行为
目的不同，App 不应在恢复存档时调用 `selectEngine()`。

## 8. 每个引擎自己的语言列表

```java
List<TranslationLanguageInfo> languages =
        TranslationSdk.getSupportedLanguages(engineId);

TranslationLanguageInfo selected = TranslationSdk.findLanguage(
        engineId, savedLanguageId);

boolean supported = TranslationSdk.supportsLanguage(engineId, savedLanguageId);
boolean supportsAuto = TranslationSdk.supportsAutoSource(engineId);
```

保存和展示规则：

- `languageId`/`selectionId` 是软件使用的稳定语言标识。
- `providerCode` 是当前引擎真实上游 code，只用于诊断，不应由 App 重新转换。
- 相同语言在不同引擎中可能分别使用 `ja`、`jp` 等 code，这是正常情况。
- 用户切换引擎后，必须用新引擎语言列表重新校验已保存源语言和目标语言。
- `displayName` 由 SDK 的 Android 资源返回；SDK 当前内置140个地区、每个地区376个语言名称。
- App 不应维护第二份语言名称映射，也不应直接展示 `providerCode`。

## 9. 执行翻译

### 9.1 单条

```java
Translator translator = TranslationSdk.createTranslator();
if (translator == null) {
    showEngineUnavailable();
    return;
}

TranslationRequest request = new TranslationRequest(
        text, sourceLanguageId, targetLanguageId);
translator.translate(request, new TranslationCallback() {
    @Override
    public void onSuccess(TranslationResult result) {
        showTranslation(result.getTranslatedText());
        // result.getActualEngineId() 只描述本次实际成功引擎，不能覆盖用户保存 ID。
    }

    @Override
    public void onError(TranslationError error) {
        if (error.getSuggestedAction() == TranslationErrorAction.PURCHASE_MEMBERSHIP) {
            showMessage(error.getMessage());
            openSubscriptionPage();
            return;
        }
        showTranslationError(error.getCode(), error.getMessage());
    }
});
```

### 9.2 批量

```java
BatchTranslationRequest request = new BatchTranslationRequest(
        texts, sourceLanguageId, targetLanguageId);
translator.translate(request, new TranslationCallback() {
    @Override public void onSuccess(TranslationResult result) {
        renderTranslations(result.getTranslations());
    }
    @Override public void onError(TranslationError error) {
        handleTranslationError(error);
    }
});
```

成功后通过 `TranslationResult.getTranslations()` 按原顺序读取结果。SDK 批量降级会跳过
内部标记为不支持批量的引擎。

Translator 应作为页面或翻译任务字段持有，任务结束或生命周期销毁时再调用 `close()`；
异步请求发出后立即关闭会取消请求。

自动翻译、字幕或 OCR需要按任务指定首选引擎时，0.2.6使用：

```java
Translator taskTranslator = TranslationSdk.createTranslator(engineId);
```

该调用不读取、修改或持久化用户选择，允许后台配置为 `selectable=false` 的隐藏执行引擎。
未知、退役、未配置或凭据无效的 ID返回 `null`。`taskTranslator.getEngineId()` 是首选引擎；
`TranslationResult.getActualEngineId()` 是本次经过自动备用链后真正成功的引擎。

### 9.3 会员校验和统一错误协议

SDK在每次实际请求前实时读取 `TranslationHost.isVip()`。当所选引擎
仍被配置为 `vip=true`、但当前用户不是会员时：

- 不向该付费 Provider 发起网络请求。
- 保留用户保存或任务指定的请求 ID，但自动从备用顺序中第一个已配置且支持当前语言的
  `vip=false` 引擎开始；后续备用继续排除所有 `vip=true` 引擎。
- 每个翻译任务开始时读取一次 VIP状态；本次非 VIP任务在整个备用链中固定仅免费，下一次
  新任务再读取最新会员状态。
- 成功时 `getActualEngineId()` 返回实际免费引擎，App不能用它覆盖用户选择。
- 没有可用免费引擎时返回 `ALL_ENGINES_FAILED`，绝不调用付费引擎。
- 用户主动点击锁定付费引擎仍通过选择结果返回 `VIP_REQUIRED`，用于打开订阅页面。
- `getMessage()` 已按系统地区国际化，可直接用于 Toast、Snackbar 或对话框。

执行矩阵：

| 用户状态 | 请求首选引擎 | 实际执行 |
| --- | --- | --- |
| VIP | `vip=false` | 先执行首选；失败后可使用完整的合资格备用链 |
| VIP | `vip=true` | 先执行首选；失败后可使用完整的合资格备用链 |
| 非 VIP | `vip=false` | 先执行首选；失败后备用链只考虑 `vip=false` |
| 非 VIP | `vip=true` | 不请求首选，直接从第一个合资格的 `vip=false` 备用引擎开始 |

这里的“免费”严格等于中台为该引擎配置的 `vip=false`。如果某个 Provider 会产生费用，即使
产品不展示会员锁，中台也必须把它标记为 `vip=true`，否则 SDK 无法把它与免费引擎区分。

`TranslationError` 同时提供：稳定的 `code`、错误 `category`、安全用户文案 `message`、
请求引擎 `engineId`、诊断用 `providerCode`、`retryable`、`purchaseRequired` 和
`suggestedAction`。`providerCode` 可能是原始 Provider 或 SDK 内部 code，仅供脱敏日志定位，
不得直接展示给用户；未调用上游时通常为0。

| Code | 常量 | 分类 | 建议处理 |
| ---: | --- | --- | --- |
| 1000 | `INVALID_REQUEST` | 请求校验 | 修正空文本、空语言等请求参数 |
| 1001 | `SDK_NOT_INITIALIZED` | SDK状态 | 完成初始化后重试 |
| 1002 | `ENGINE_NOT_CONFIGURED` | 引擎配置 | 刷新配置或选择其他引擎 |
| 1003 | `ENGINE_RETIRED` | 引擎配置 | 选择其他引擎 |
| 1004 | `UNSUPPORTED_LANGUAGE` | 语言能力 | 从该引擎语言列表重新选择 |
| 2001 | `VIP_REQUIRED` | 会员资格 | 提示并打开会员购买页，不自动重试 |
| 2003 | `QUOTA_EXHAUSTED` | 使用额度 | 提示购买会员或等待额度恢复 |
| 3001 | `NETWORK_UNAVAILABLE` | 网络 | 检查网络后重试 |
| 3002 | `REQUEST_TIMEOUT` | 网络 | 可稍后重试 |
| 3003 | `RATE_LIMITED` | 限流 | 延迟后重试，禁止紧密循环 |
| 4001 | `PROVIDER_AUTHENTICATION_FAILED` | Provider配置 | 不对用户展示凭据细节，记录并联系维护人员 |
| 4002 | `PROVIDER_UNAVAILABLE` | Provider状态 | 稍后重试或选择其他引擎 |
| 4003 | `INVALID_PROVIDER_RESPONSE` | Provider响应 | 可稍后重试 |
| 4004 | `ALL_ENGINES_FAILED` | 执行结果 | 所有内部候选均失败，稍后重试 |
| 5001 | `CANCELLED` | 生命周期 | 通常无需提示 |
| 9000 | `UNKNOWN` | 未分类 | 展示安全通用文案并记录诊断信息 |

失败统一通过 `onError(TranslationError)` 返回。SDK错误文案不包含密钥、请求原文、译文、
服务端响应正文或异常堆栈。App上报时也只能记录
错误 code、category、engineId、providerCode 和必要的安全摘要。

### 9.4 0.2.8 请求取消、缓存与耗时

旧的 `translate()` 保持可用，SDK 创建的 Translator 将结果和错误交付到主线程，结果校验与回退在后台执行。

```java
TranslationCall pending = translator.translateCancellable(request, callback);
pending.cancel(); // 仅取消这个订阅；最后一个合并订阅取消时才停止底层请求
translator.clearResultCache();
TranslationSdk.configureResultCache(0); // 关闭内存缓存；默认 15000 ms，允许 0..60000 ms
```

同一 Translator 中相同文本/批次、语言、配置、会员状态、引擎和超时配置的在途请求会合并。非 AI 引擎默认缓存 15 秒，最多 64 条、原译文合计 128 Ki 字符；AI 和回退结果不缓存。缓存不落盘，命中或合并不新增 Provider 请求次数。`close()` 取消该对象全部请求并清理缓存。合并订阅沿用原请求链的剩余预算。

SDK 最多同时执行 8 条请求链、等待 64 条；超出等待容量通过 `RATE_LIMITED` 提示稍后重试。总预算包含排队和回退，每次 HTTP 同时受引擎超时与剩余预算约束。网络切换后重新允许预热，并仅释放有网络故障标记的冷却；服务端限流和鉴权冷却保留。

宿主可覆盖 `onNetworkRequestFinished(TranslationNetworkTiming)` 读取 DNS、建连、TLS、响应等待、读取耗时和连接编号；覆盖 `onTranslationFinished(requestId, chainId, engineId, outcome, source, queueMs, callbackWaitMs, totalMs)` 统计公开请求总耗时。`chainId` 关联同一次执行与回退，`connectionId` 关联预热和业务请求。缺失网络阶段为 `-1`，`connectMs` 包含 TLS，`responseWaitMs` 不等于服务端处理时间。事件不含 URL、文本或凭据，元数据回调应快速返回。

## 10. SDK 超时、冷却和自动兜底

- AI Provider 整体超时10秒。
- 自建谷歌、自建百度（`BD`）、自建有道整体超时默认5秒；其他 Provider 整体超时3秒。
- 首选和全部备用引擎默认共享30秒调度预算；剩余时间不足下一个引擎完整请求窗口时停止备用，
  返回 `REQUEST_TIMEOUT`。预算只控制是否启动下一请求，不是到点强制取消的定时器。
- SDK 0.2.6支持 `TranslationSdk.configureTimeouts(5_000, 30_000)`；不影响已经创建的HTTP请求，
  不修改服务端超时。配置范围和网络结果回调见 [SDK超时与切换](../mobile-sdk/README.md#超时与切换)。
- `GOOGLE_CLIENT` 和 `GOOGLE` 使用免费接口请求间隔闸门，避免同一 App 并发触发限流。
- Provider 故障只影响对应引擎，不清空其他引擎状态。所有现役引擎失败冷却至少5分钟。
  Google / Google Client 直连的HTTP及网络错误统一90分钟；所有非自建引擎的429视为被封，停用90分钟。
  自建服务429按容量限流处理，但不能低于5分钟；其他网络、408、5xx故障按5分钟、10分钟、30分钟、
  各引擎上限递增，鉴权和证书错误取各自上限。短响应头不能缩短冷却底线，旧短冷却缓存也会提升。
  只有实际成功引擎才重置连续失败次数，首选和兜底执行都检查冷却。
- 自动兜底顺序由 SDK 版本和会员状态决定，不读取中台优先级。
- 自动兜底不修改用户保存的整数 ID。

当前 V2 完整顺序如下。用户主动选择的引擎总是先执行；此列表只用于当前请求失败后的
自动兜底：

```text
GOOGLE_CLIENT → SELF_HOSTED_GOOGLE → SELF_HOSTED_YOUDAO → GOOGLE
→ DEEPSEEK → HY_MT2 → QWEN → GEMINI → GPT_4O_MINI → ZHIPU_FLASH
→ MYMEMORY → RAPID_IRCTCAPI → RAPID_MULTI → RAPID_WEBSITE_PLUS
→ MICROSOFT → YOUDAO → BD → RAPID_AI → RAPID_PLUS → RAPID_DEEP
→ GOOGLE_V → MICROSOFT_VIP → BAIDU → GEMMA
```

批量请求跳过不支持批量的引擎；未启用、无有效凭据、不支持当前语言、VIP 不满足、
每日次数耗尽、处于冷却或本次已经尝试过的引擎都会被跳过。Gemma 因当前产品质量评估
放在整条链最后。

非 VIP任务不会建立另一条需要维护的硬编码顺序，而是在上述唯一顺序上动态过滤
`vip=true` 引擎。这样中台修改某个引擎的 `vip` 后，客户端下次配置刷新即可使用新费用
属性，不需要新增策略版本或重复配置。

SDK 会按“引擎 ID + 系统语言”缓存不可变的公开语言目录，配置刷新不会重复创建几百个
展示对象；清理 SDK 缓存时同时清除此内存目录。完整配置 JSON 最大2 MiB，超限时在
解析和解密前拒绝，避免异常配置造成内存与主线程压力。

每个具体 Translator 还在自己的源文件内复用本引擎不可变语言表；不同 Provider 之间不共享
语言目录、语言 code或 Translator。Provider JSON 凭据在配置加载时解析一次，运行时轮换不再
重复解析。传统 Provider Service按引擎独立懒加载，首次请求不会创建其他 Provider；Retrofit
与 Service缓存命中不获取全局创建锁。

宿主可实现 `TranslationHost.onProviderAttemptFinished(engineId, outcome, durationMs, httpStatus)`
采集所有 Provider 的匿名尝试耗时。事件不包含 URL、Key、原文、译文或响应体，回调线程不固定，
实现必须快速返回。用 `TranslationResult.getActualEngineId()` 对照请求引擎：两次实际引擎相同但
首次较慢通常是连接冷启动；实际引擎变化则说明首选失败后进入备用链。

可在用户进入相关页面时调用 `TranslationSdk.preconnect(engineId)`，或用无参数版本预热
当前选择。预热使用 SDK 自己的连接池异步发送不含鉴权和正文的 `HEAD /`，同一 Origin会去重；
成功收到响应后5分钟内不重复预热。它不消耗每日翻译次数、不执行备用翻译或写入失败冷却。
返回 `true` 仅表示已安排、执行中或近期执行过。
预热没有模型 Token费用，但普通 HTTP 请求是否计入第三方 QPS由 Provider决定。SDK 的连接池
原本就会复用连接，并非每次翻译都重新握手；该能力只优化首次、网络切换或空闲连接失效场景。
预热从指定引擎开始，按兜底顺序选择前两个健康且符合会员资格的引擎；跳过不可达、冷却、闸门未就绪或未配置的候选。

### 10.1 Factory 默认返回

内部 `TranslateFactory.createTranslate(context, engineId)` 不抛初始化异常且不返回 `null`。
遇到未初始化、未知 ID、退役 ID、未配置 ID 或未命中分支时，统一返回
`GoogleClientTranslate`。这是当前请求的执行层兜底，不表示请求 ID 有效，也不能把
`GOOGLE_CLIENT(8)` 覆盖写入用户保存的选择。

App 仍必须通过 `TranslationSdk.selectEngine()`、`restoreSelectedEngine()` 和
`createTranslator()` 接入，不能直接依赖内部 Factory 判断选择是否合法。

## 11. 可选：中台自建 Google 调度

这项功能与“App 翻译配置”独立。仅当项目需要通过中台调度自建 Google 节点时接入：

```http
POST {MANGO_PLATFORM_BASE_URL}/internal/v1/apps/{appId}/translate
X-App-Backend-Key: mat_xxx
Content-Type: application/json
```

单条请求：

```json
{
  "engineId": "SELF_HOSTED_GOOGLE",
  "sourceLanguage": "auto",
  "targetLanguage": "en",
  "text": "需要翻译的文本"
}
```

批量请求：

```json
{
  "engineId": "SELF_HOSTED_GOOGLE",
  "sourceLanguage": "auto",
  "targetLanguage": "en",
  "texts": ["第一条", "第二条"]
}
```

成功响应：

```json
{
  "requestId": "uuid",
  "engineId": "SELF_HOSTED_GOOGLE",
  "translations": ["First", "Second"],
  "durationMs": 238
}
```

错误处理：

- `429`：容量或限流，遵循 `Retry-After`，不要立即循环重试。
- `503`：没有健康节点。
- `502`：上游节点尝试失败。
- `401`：App ID 与后台访问密钥不匹配。

客户端仍不能直连这个入口；由业务 App 服务端代理，或把其服务端地址和所需凭据作为
对应 SDK Provider 的加密 JSON 配置下发。

## 12. 可选：App 互推

业务服务端调用：

```http
GET {MANGO_PLATFORM_BASE_URL}/internal/v1/apps/{appId}/promotions?limit=5
X-App-Backend-Key: mat_xxx
```

响应：

```json
{
  "sourceAppId": "app_0123456789abcdef",
  "version": 123456789,
  "apps": [
    {
      "appId": "app_fedcba9876543210",
      "name": "目标 App",
      "packageName": "com.example.target",
      "description": "介绍",
      "storeUrl": "https://example.com/store",
      "iconUrl": "https://example.com/icon.webp",
      "bannerUrl": "https://example.com/banner.webp",
      "priority": 1
    }
  ]
}
```

业务服务端缓存后提供给自己的客户端；客户端按返回顺序展示，不自行加入其他 App。

## 13. 密钥轮换与故障处理

### Provider 凭据轮换

1. 在中台更新对应 App/引擎凭据。
2. 触发业务服务端主动同步。
3. 确认服务端快照更新时间和引擎数量正确。
4. 客户端下一次刷新后验证单条和批量翻译。

### 翻译密钥加密 Key 轮换

1. 确认新客户端受保护解密实现已经准备好。
2. 在中台 App 管理修改 Key；中台会重新加密已有 Provider 凭据。
3. 主动同步业务服务端并验证客户端解密。
4. 未完成客户端兼容前不能提前轮换。

### App 后台访问密钥轮换

1. 先准备业务服务端 Secret 更新和重启窗口。
2. 中台点击“轮换后台密钥”并立即保存完整新值。
3. 立即更新 `MANGO_PLATFORM_BACKEND_KEY` 并重启/滚动发布业务服务端。
4. 主动同步验证成功。旧 Key 在中台生成新值时已经失效。

## 14. 发布顺序

首次接入严格按以下顺序：

1. 中台登记 App、配置加密 Key、生成后台访问密钥、配置引擎。
2. 发布业务 App 服务端的同步逻辑、管理入口和客户端专用配置接口。
3. 配置真实 `MANGO_PLATFORM_APP_ID` 与 `MANGO_PLATFORM_BACKEND_KEY`。
4. 管理端主动同步并确认存在有效快照。
5. 使用 App 自己的认证方式验证客户端专用配置接口。
6. 再发布集成 AAR 的 Android 客户端。

不能先发布依赖新接口的客户端，再等待业务服务端上线。

## 15. 完整验收清单

### 中台

- [ ] App ID 是中台生成的真实 `app_xxx`。
- [ ] 翻译密钥加密 Key 与 App 受保护解密实现一致。
- [ ] 后台访问密钥已安全保存，未进入客户端或 Git。
- [ ] 每个启用引擎至少有一条结构正确的凭据。
- [ ] `vipOnly` 和 `selectable` 与产品设计一致。
- [ ] V2 智谱两套 Profile 都包含完整地址、Key、模型、协议和推理开关。
- [ ] 已发布版本区间从0到无限连续覆盖且互不重叠。

### 业务服务端

- [ ] 启动时校验 App ID 和后台访问密钥非空。
- [ ] 支持主动同步和每日定时同步。
- [ ] 只保存最近一次完整有效密文规则包。
- [ ] 同步失败不覆盖旧规则包。
- [ ] 明确区分有效空数组与网络无响应。
- [ ] 客户端接口使用 App 自己的认证、HTTPS和 `no-store`。
- [ ] 日志、APM和反向代理不记录 Header或配置正文。
- [ ] App 后台按 `versionCode` 唯一命中规则，只返回该规则的 `config`。
- [ ] 客户端从未收到完整规则包、其他版本密文、HMAC 或中台后台访问 Key。

### Android

- [ ] AAR及全部直接依赖已加入。
- [ ] `TranslationHost` 使用 App 现有受保护解密和身份校验。
- [ ] 只保存整数引擎 ID。
- [ ] 只展示 `getSelectableEngines()`。
- [ ] 引擎图标只使用 `TranslationEngineInfo.getIconRes()`，App 没有第二套 ID 映射。
- [ ] 语言列表始终来自当前引擎。
- [ ] 用户切换成功后才持久化 ID。
- [ ] 自动兜底成功不覆盖用户选择。
- [ ] `TranslationHost.isVip()` 返回当前账号真实状态，不使用启动时永久缓存。
- [ ] 非 VIP + 付费首选、非 VIP + 免费首选、VIP + 付费首选三种链路均已验证。
- [ ] 业务仅使用 `result.getActualEngineId()` 做本次展示和诊断，不覆盖保存的请求引擎 ID。
- [ ] 新配置及时覆盖，接口失败才继续用缓存。
- [ ] 单条、批量、会员、非会员、弱网和R8构建均通过。

## 16. 常见问题

### 为什么客户端不能直接请求中台？

中台接口需要长期 `X-App-Backend-Key`，放进 APK 后任何人都能提取并获取该 App 的全部
翻译凭据密文。业务服务端是强制安全边界。

### 为什么后台只能看到 `mat_` 前缀？

完整后台访问密钥只显示一次，数据库只保存哈希。遗失后只能轮换，不能找回。

### 为什么配置接口没有 `enabled`？

引擎出现在 `engines[]` 中就表示中台已允许该 App 使用；未启用的引擎不会返回。

### 为什么配置接口没有语言和批量能力？

语言列表、Provider code、单条/批量能力属于 SDK 具体 Translator，不能由中台重新定义。

### 为什么配置接口没有优先级？

当前请求的默认和失败兜底顺序由 SDK 根据成本、速度、会员和冷却状态统一维护。

### 非 VIP 用户选过付费引擎，需要 App 后台改配置吗？

不需要。App 后台继续下发中台命中的原始配置，SDK 使用已有 `vip` 和
`TranslationHost.isVip()` 判断。恢复历史选择时保留原 ID；翻译任务执行时只走
`vip=false` 引擎。会员恢复后，下一次任务可重新执行原付费首选。

### 为什么非 VIP 翻译失败返回的不是 `VIP_REQUIRED`？

`VIP_REQUIRED` 用于用户主动点击锁定引擎时打开订阅入口。自动翻译、字幕和 OCR执行时，
SDK 会优先尝试免费链；只有没有免费候选或免费候选全部失败时才返回
`ALL_ENGINES_FAILED` 或相应网络错误，避免自动流程被订阅弹窗打断。

### 为什么新项目初始化后没有默认引擎？

没有历史选择时，SDK按兜底顺序寻找已下发、凭据有效、免费、允许选择且当前健康的引擎。
没有候选才返回 `NO_AVAILABLE_ENGINE`；已有用户选择不会因网络临时变化而被覆盖。

### 为什么引擎名称和介绍没有出现在后台配置中？

名称、介绍、官方品牌图标、排序和国际化资源属于 SDK 展示契约，由 SDK 版本统一维护。后台只控制 App
是否获得该引擎、凭据、VIP和 `selectable`，不能改变品牌名称或用户介绍。

### 为什么 SDK 不接收版本号或完整规则包？

版本号属于 App 自己的发布和认证上下文。中台只向 App 后台返回全部已发布规则，App 后台
负责唯一匹配并只返回一个 `config`。这样旧版只得到旧密文，新版只得到新密文，SDK 不会
接触其他版本的 Key，也不需要 HMAC 或规则选择逻辑。

### 如何重新生成语言国际化资源？

本仓库提供两个本地脚本：

```powershell
.\scripts\import_voice_language_resources.ps1
.\scripts\localize_sdk_resources.ps1
```

第一个脚本只从 Voice 复用 `language_*`；第二个脚本在本机调用中台
`SELF_HOSTED_GOOGLE` 补全缺失项，并在失败时使用千问兜底。脚本不会打印或保存明文密钥，
也不会导入 Voice 的 OCR、广告和界面业务文案。

## 17. 本仓库验证命令

```powershell
.\scripts\verify.ps1
```

SDK单独验证：

```powershell
cd mobile-sdk
.\gradlew.bat :mango-translate:testDebugUnitTest :mango-translate:assembleRelease
```

中台、业务服务端和 Android 是独立发布单元。验证命令不会自动部署或发布应用商店版本。
