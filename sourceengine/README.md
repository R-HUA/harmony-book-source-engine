# @legado/sourceengine

书源解析内核，以 HAR 形式供应用模块依赖，最初是[开源轻页](https://github.com/wip3l/legado-harmony)的一部分。它包含：

- 兼容书源（阅读格式）的规则解析：`AnalyzeUrl`、`AnalyzeRule`、`JsonPathEvaluator`、`JavaRegexCompat` 与 JS 兼容层。
- 请求层：`HttpClient`、`CookieStore`、`TlsTrustStore`、`HttpAuth`、限流与网页验证支持。
- 原生书源：`NativeSourceSpec`、`NativeExpr`、`NativeSourceRunner`、`NativeSourceService`。
- 脚本运行时：QuickJS 运行、迁移与诊断，以及需要 ArkWeb 的阶段脚本运行时 `BookSourceStageWebRuntime` 和网页抓取 `WebBookFetchRuntime`。
- 数据模型：`model/data/Book.ts` 里的 `Book`、`BookChapter`、`BookSource` 等。

搜索、发现以及详情/目录/正文的调度（`SearchCoordinator`、`ExploreCoordinator`、`WebBookService`）不在本包里，由宿主应用实现；可参考开源轻页 `entry/src/main/ets/core/book/` 下的同名文件。

## 引入

```json5
// 应用模块的 oh-package.json5
"dependencies": {
  "@legado/sourceengine": "file:../sourceengine"
}
```

`build-profile.json5` 的 `modules` 里还要注册 `sourceengine`。

`Index.ts` 是主入口，只导出 `.ts` 文件，`.ts` 和 `.ets` 都可以从这里导入。`QuickJsDiagnostics` 和 `QuickJsShadowExecutor` 用到了 `@Concurrent`，必须是 `.ets` 文件，所以单独从子入口导入：

```ts
import { QuickJsDiagnostics } from '@legado/sourceengine/Diagnostics';
```

新增公开类型时，把它追加到 `Index.ts`（或 `Diagnostics.ets`）对应的文件分组里。

## 宿主应用需要接入的部分

内核不访问应用数据库，也不包含界面。下面这些由宿主提供：

| 接入点 | 做法 |
| --- | --- |
| 书源运行时状态持久化 | 实现 `BookSourceRuntimePersistence`（读取书源、写回 `variable` / `loginHeader` / `loginInfo`），在启动时调用 `BookSourceRuntimeStore.install(...)`。不安装时读到 `null`，写入被跳过。 |
| ArkWeb 宿主 | `BookSourceStageWebRuntime.get().attach(controller)`、`WebBookFetchRuntime` 需要页面里挂一个 `Web` 组件提供控制器；参考开源轻页的 [`BookSourceStageRuntimeHost.ets`](https://github.com/wip3l/legado-harmony/blob/main/entry/src/main/ets/components/BookSourceStageRuntimeHost.ets)、[`WebBookFetchRuntimeHost.ets`](https://github.com/wip3l/legado-harmony/blob/main/entry/src/main/ets/components/WebBookFetchRuntimeHost.ets)。 |
| Context | `TlsTrustStore.load(context)`、`QuickJsDiagnostics.initialize(context)` 等 Preferences 存储需要传入应用上下文。 |
| 网页验证 | 遇到验证页时 `VerificationSupport` 写入 AppStorage：`pendingVerificationUrl`、`pendingVerificationTitle`、`pendingVerificationTime`、`pendingVerificationSourceUrl`、`pendingVerificationLoginUrl`、`pendingVerificationUserAgent`。宿主的验证页读取这些值，使用对应 User-Agent 打开验证资源。 |
| QuickJS 诊断刷新 | 存储变化时更新 AppStorage 的 `quickJsMigrationRevision`、`quickJsValidationRevision`、`quickJsRegressionArtifactRevision`，界面可以监听这些值来刷新。 |

## 可选宿主端口

另一个应用复用本包时，可以通过 `SourceEngineHost.install(adapter)` 替换下面这些能力。这个调用在启动时、执行任何书源之前做一次，任务执行期间不要更换 adapter。每个端口都可以不提供；开源轻页本身不安装任何端口，行为与没有这一层时相同。

端口都是可信的宿主代码，绝不能暴露给书源脚本。

| 端口 | 作用 | 不提供时 |
| --- | --- | --- |
| `cookies(sourceUrl)` | 按书源返回 Cookie 存储（`SourceCookieJar`），实现按书源或按账号隔离 | 全局 `CookieStore` |
| `persistence(source)` | 按书源读写运行时登录状态 | `BookSourceRuntimeStore.install` 安装的实现 |
| `scripts` | 用宿主的执行器运行阶段脚本。只有每次都在独立、有预算限制的上下文里执行时才能声明 `concurrent: true`；`forceFullScripts` 让所有脚本都走完整脚本层；`allowSynchronousQuickJs: false` 禁止同步执行 QuickJS | 隐藏 ArkWeb 串行执行 |
| `requests` | 决定同一书源下等待者的先后和取消状态。请求频率窗口仍由本包维护，优先级只改变排队顺序，不会额外放出额度 | 按到达顺序等待窗口 |
| `web` | 网页会话租约，WebView 抓取与听书音频嗅探在租约内执行 | 直接执行 |
| `state` | 按书源记录最近一次失败原因 | 写入 AppStorage 的 `bookSourceStageLastError` |
| `diagnostics` | 接收分段耗时（HTTP 传输与首字节、限流等待、脚本拼装等） | 不记录，也不拼接耗时明细 |
| `convertChinese(text, converterType)` | 为规则兼容层的 `java.t2s` / `java.s2t` 提供同步转换；方向 `1` 为繁转简、`2` 为简转繁，词典由宿主加载 | 原样返回文本 |

`NativeSourceImport.importObject` / `toBookSource` 只做定义转换，不清缓存、不访问网络或数据库；应用内导入仍用 `NativeSourceService.importObject`，它会清掉该书源的缓存。原生书源的声明式缓存按“定义 + 请求头 + 登录头 + 登录信息”分开，所以只用 `NativeSourceImport` 换了定义也不会读到旧值；登录站点的 Cookie 不参与分区，网页登录与退出登录时各自清缓存。

## 宿主兼容性回归

把 `DEVECO_HOME` 设为 DevEco Studio 的安装目录后执行 `node scripts/source-engine-host-check.cjs`。它覆盖默认串行与注入并发执行、Cookie、持久化、取消和原生书源导入；平台 API 用测试替身，不能代替真机验证。

## 回归检查

`scripts/` 里的 Node 回归脚本通过 `scripts/lib/ets-module-staging.mjs` 加载本模块的源码。这个工具把 `entry` 和 `sourceengine` 两个源码根合并成同一棵目录树，并把 `@legado/sourceengine` 的导入按名字改写成指向定义文件的相对导入。改动内核后至少运行：

```bash
node --experimental-strip-types scripts/native-source-check.mjs
node scripts/neutral-source-engine-check.mjs
```

改动 `AnalyzeRule`（选择器、属性提取、旧式写法）后再运行 `node --experimental-strip-types scripts/analyze-rule-check.mjs`。它用 `scripts/lib/load-analyze-rule.mjs` 把 AnalyzeRule 及其依赖加载进 Node（系统模块换成空桩，脚本会自动加上 `--experimental-transform-types` 重跑），把一组页面 × 规则的结果与 `scripts/fixtures/analyze-rule-golden.json` 比对，并打印 HTML 目录取字段的参考耗时。确认是有意改变选择器行为时，加 `--update` 重写基准。
