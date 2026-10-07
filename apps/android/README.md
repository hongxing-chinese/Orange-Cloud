# Orange Cloud — Android

原生 Kotlin + Jetpack Compose 客户端。设计与价值同源于 iOS 版，功能与交互以 Android 原生为先（不追求与 iOS 一一对应）。
最低 Android 8（API 26），目标 / 编译 API 36。

## 先决条件

- **JDK 17**
- **Android SDK**（platform `android-36` + 对应 build-tools）

用 Android Studio 打开 `apps/android/` 会自动配齐 SDK；或手动安装命令行工具：

```bash
# JDK 17（Homebrew 示例）
brew install --cask temurin@17
export JAVA_HOME="$(/usr/libexec/java_home -v 17)"

# Android SDK（命令行工具，无需 Android Studio）
brew install --cask android-commandlinetools
export ANDROID_HOME="$HOME/Library/Android/sdk"
sdkmanager "platforms;android-36" "build-tools;36.0.0" "platform-tools"
```

`local.properties` 不入库，首次需写入 SDK 路径（Android Studio 会自动生成）：

```properties
sdk.dir=/path/to/Android/sdk
```

## 构建

仓库已包含 Gradle Wrapper（`./gradlew`，Gradle 8.13），直接用即可：

```bash
cd apps/android
./gradlew :app:assemblePlayDebug      # 官方 Play 版（带 Billing）
./gradlew :app:assembleOssDebug       # 开源自编译版（无 Billing，isPro 恒真）
./gradlew :app:testPlayDebugUnitTest  # 单测
```

三个产品风味：

| 风味 | applicationId | 说明 |
|---|---|---|
| `play` | `jiamin.chen.orangecloud` | 官方版，Play Billing，内置官方 OAuth Client |
| `oss` | `jiamin.chen.orangecloud.oss` | 自用全解锁，无 Billing 依赖，默认个人 OAuth 配置 |
| `direct` | `jiamin.chen.orangecloud.direct` | 官网直发版，无 Play Billing，使用激活码 |

## OAuth Client 与回调地址

`OAUTH_CLIENT_ID` 与 `OAUTH_DOMAIN` 经 Gradle 注入到 `BuildConfig`。`oss` 默认使用个人 OAuth Client 和 `oauth.wideseek.de5.net` 回调中转；授权完成后，中转服务应将浏览器重定向到 `orangecloud://oauth/callback`。

- `play` 风味内置官方 Client ID——OAuth PKCE 下它是公开标识符而非机密，与 iOS `OAuthConfig.swift` 同值。
- `oss` 风味使用个人默认配置。自编译者可以在 `apps/android/local.properties` 覆盖 Client ID 和回调域名：
  ```properties
  OAUTH_CLIENT_ID=你自建的_client_id
  OAUTH_DOMAIN=oauth.example.com
  ```
- 也可以通过 Gradle 参数 `-POAUTH_CLIENT_ID=... -POAUTH_DOMAIN=...` 覆盖。`OAUTH_DOMAIN` 只填主机名，构建会生成 `https://<主机名>/oauth/callback`。
- GitHub Actions 的 OSS APK 工作流可用仓库 Secret `OAUTH_CLIENT_ID` 和变量 `OAUTH_DOMAIN` 覆盖默认值；官方 Client 与 `o-c.do` 中转不向第三方构建开放，详见根目录 [`CONTRIBUTING.md`](../../CONTRIBUTING.md)。

工作流签名需要配置仓库 Secrets：`ANDROID_KEYSTORE_BASE64`、`ANDROID_KEYSTORE_PASSWORD`、`ANDROID_KEY_ALIAS`、`ANDROID_KEY_PASSWORD`。OAuth Client ID 是 PKCE 公开标识符，不是密钥。

## 架构

Google 官方 App Architecture，分层无环：

```
ui/        Compose 屏 + ViewModel（每屏一个 UiState: StateFlow）
data/      Repository（单一可信源）
  ├─ remote/  CfApiClient(OkHttp) + DTO(@Serializable)
  └─ local/   Room(@Entity/@Dao) + DataStore
core/      auth / network / design（晨昏天景）/ di（Hilt）
```

- Composable 不直接发网络，只读 ViewModel 暴露的 `UiState`；ViewModel 不持有 OkHttp，只调 Repository。
- Token 只入 Keystore 包裹的加密 DataStore，绝不写明文。
- 数据模型为 `@Serializable` data class，`@SerialName` 映射 snake_case。
- 错误统一 `ApiError`（sealed class），在 ViewModel `catch` 后赋给 `UiState.error`。
- 用户可见文案一律入 `res/values/strings.xml`，禁止硬编码字面量。
