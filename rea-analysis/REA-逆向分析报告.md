# linkup（com.lhtg.spiritpavilion）逆向分析报告

> 分析对象：`/workspace/downloads/linkup.apk`
> 分析方式：REA 6.8.0 静态分析（JADX headless + Apktool + 原始 APK 解析），未运行目标 APK
> 分析日期：2026-11-08

---

## 0. 样本基本信息

| 项目 | 值 |
| --- | --- |
| 应用名 | linkup（`@string/app_name`） |
| 包名 | `com.lhtg.spiritpavilion` |
| 版本 | 4.3.1（versionCode 431） |
| 文件大小 | 146,365,699 字节（约 139.6 MiB） |
| SHA-256 | `289ed3ace9c2a81bc64b5d984b444853c974a0a2eae8593dac4d895de6a4933f` |
| minSdk / targetSdk | 28 / 36 |
| ABI | 仅 `arm64-v8a` |
| DEX | 7 个（classes.dex ~ classes7.dex） |
| 类总数 | 27,033（其中应用自有类 852） |
| 资源条目 | 4,180 |
| 签名方案 | v1 + v2 + v3（无 v3.1） |
| 证书主体 | `C=CN, ST=陕西省, L=上海, O=科技有限公司, OU=产品部, CN=lhtg` |
| 证书 SHA-256 | `ffdacb8010d5c76fe171d298c550c7309b8511bcbc12ef0a9d3ca2bfde960b9b` |
| 渠道信息 | `channel=10039`、`sub_channel_id=41803`（与分发路径 `/10039/41803/linkup.apk` 一致） |

### 服务端与网络

| 用途 | 地址 | 证据 |
| --- | --- | --- |
| 主 API | `https://ai-chat-api.tiankongshuyu.cn/api/` | `RetrofitConfig.createRetrofit()` |
| 更新/落地/分发 | `http://sm15.iwmls.com/` | `@string/url` |
| 资源/分享 | `https://pan.guaxu.fun` | `@string` 资源 |
| 网络安全配置 | `cleartextTrafficPermitted="true"`，信任 `system` + `user` CA，放行 `127.0.0.1` | `res/xml/network_security_config.xml` |

> 说明：允许明文与用户 CA，意味着该版本在抓包/中间人环境下不设防（对分析有利，对用户是风险）。

---

## 1. 应用画像

综合组件、API、布局与资源，判定为**成人向 AI 角色扮演 / 剧情聊天 + 后宫养成 + 用户共创广场 + 付费解锁 + 广告变现**的 Android 原生应用（非 Flutter / 非 uni-app；纯 Java/Kotlin + ViewBinding）。

主要功能域（由 ViewModel / API / 布局共同印证）：

- **AI 对话**：`chat/completions`（OpenAI 风格 messages + stream）、TTS、灵感回复、多人群聊、剧情聊天
- **角色养成/后宫**：角色中心、换装、私密相册、亲密度、侍寝/睡眠、改名、好友
- **广场共创**：创建角色 / 场景 / 剧情聊天，场景列表
- **商业化**：VIP、金币、充值、提现、新人福利、任务系统、邀请返利
- **榜单**：人气榜、亲密度榜、名人堂
- **广告变现**：开屏、插屏、激励视频、信息流、Banner，全局插屏计时器
- **风控/设备**：数美设备指纹 boxId、OAID、UMID、设备标识采集

### 内置第三方 SDK（按 dex 包名与 assets 证据）

| 类别 | SDK |
| --- | --- |
| 广告聚合/中介 | Taku（`com.cssky.adadapter`）、AnyThink、Zongliang（`com.zongliang.sdk`） |
| 广告平台 | 穿山甲 Pangle（`com.bytedance`、assets 内 pangle 加固库）、快手（`com.kwad`）、优量汇 GDT（`com.qq.e`、`gdtadv2.jar`）、BeiZi（`com.beizi`）、Sigmob（`com.sigmob`）、Mintegral（`com.mbridge`）、百度（`com.baidu`、`bdxadsdk.jar`）、华为 Ads（HMS ads-identifier） |
| 统计/归因 | 友盟 `com.umeng`（`UMENG_APPKEY` 已内置）、`com.google.android.gms.permission.AD_ID` |
| 设备风控 | 数美 `com.ishumei`（`SmAntiFraud`，boxId） |
| OAID | MSA（`com.asus.msa`、`freemme.permission.msa`） |
| 网络/图片/播放 | OkHttp + Retrofit + Gson、Glide（自定义 `FjGlideModule`）、ExoPlayer |
| 存储上传 | 七牛 `com.qiniu`（`QiniuUploader`） |
| 支付 | 支付宝 / 微信 / 抖音支付 scheme（`alipays://`、`weixin://`、`ttcjpay://`、`dypay://`） |

---

## 2. 运行逻辑

### 2.1 进程与启动链

`AndroidManifest.xml`：`android:name="com.lhtg.spiritpavilion.App"`，主题 `Theme.AppCompat.NoActionBar`，竖屏、`largeHeap=true`、`usesCleartextTraffic=true`。

入口 Activity 为 `com.lhtg.spiritpavilion.ui.activity.SplashActivity`（`MAIN` + `LAUNCHER` + `BROWSABLE`，exported）。

#### Application 初始化（`App.onCreate`）

```
App.onCreate():
    initCrashHandler()                              // 崩溃捕获
    ProcessLifecycleOwner.get().lifecycle.addObserver(this)
    registerActivityLifecycle()                     // 记录当前 Activity
    DeviceIdentifier.register(this)                 // 设备标识
    DeviceIDManager.startCollectDeviceId()          // 数美 boxId / OAID / UMID
    UMConfigure.preInit(this, UMENG_APPKEY, channel = "10039")
    if (processName != packageName)
        WebView.setDataDirectorySuffix(processName) // 多进程 WebView 隔离
    Thread { ... }.start()                          // 异步初始化（广告等）
```

#### 启动页逻辑（证据：`SplashActivity.initData / startLogin / goMain / selectGender`）

```
SplashActivity.initData():
    viewModel.loadAllConfiguration()                 // 全量配置（系统/广告/亲密度/解锁）
    handler.postDelayed(updatePBRunnable, 100ms)     // 进度条动画
    App.initWebAd(bgWebView)                         // 透明 WebView 初始化 Web 广告

SplashActivity.startLogin():
    DeviceIDManager.getBoxIdAsync { boxId ->
        handleLogin()                                // 携带设备信息登录
    }

SplashActivity.goMain():
    if (!NetworkUtil.isNetworkAvailable()) { toast("网络未连接"); return }
    if (!jumped) {
        jumped = true
        if (adData == null) adData = AdData(numberOfAds = 3, statusType = "3")
        selectGender()
        return
    }
    startActivity(MainActivity); finish()

SplashActivity.selectGender():                        // 实际是启动路由
    when {
        adData.statusType == "3" &&
        App.isNoRisk() &&                             // 数美风控通过
        AppDataManager.isShowFrontAd(false) &&        // 服务端开关
        !App.isVIP() ->
            startActivity(SplashAdsActivity, initAllConfigSuccess)
        SP.getInt(GUIDANCE, 0) == 0 ->
            startActivity(GuidanceActivity1)          // 新手引导
        else ->
            startActivity(MainActivity)
    }
    finish()
```

引导页之后由 `GuidanceActivity1/2` → `GenderSelectActivity`（性别/身份选择）进入主界面；`SpUtil` 用 `GUIDANCE` 标记是否已看过引导。

#### 主界面（`MainActivity.initData`）

```
setupBottomNavigation()        // 4 个 Tab，默认 HomeFragment
setupNewTaskDrag()             // 右侧“新人福利”悬浮按钮
handleExternalNavigation()     // 支持外部 INTENT: go_to_harem / go_to_square
NewUserGiftManage.showFirstEnterGiftDialog(...)
IntersTimer.start()            // 全局插屏计时
preLoadAd()
viewModel.versionUpdate(versionCode, versionName, adId, "android")
viewModel.referralConfig()     // 邀请返利配置
viewModel.initializeTasks()
```

### 2.2 登录与身份

`UserRepository.login(adId)` 构造 `NewLoginRequest`：

```
NewLoginRequest(
    boxId        = DeviceIDManager.boxId,        // 数美设备指纹
    oaid         = DeviceIDManager.oaid,         // 广告标识
    umid         = DeviceIDManager.umid,
    ?, manufacturer = Build.MANUFACTURER,
    channel      = App.channel,                  // "10039"
    model        = Build.MODEL,
    "0",
    adId,
    ...
)
POST user/login → LoginData { token, riskStatus, riskId, userInfo ... }
```

- 登录接口常量：`API_LOGIN = "user/login"`，注册：`API_REGISTER = "user/register"`（`ApiConstants`）
- 成功码 200，未授权 401
- Token 通过 `SpConstants` 持久化；`LoginViewModel`、`SplashViewModel` 共用该流程
- 登录返回的 `riskStatus/riskId` 决定后续是否展示广告、能否进入某些页面

### 2.3 全局配置缓存（`AppDataManager`）

服务端下发并由 SP 缓存，控制运行行为：

| 缓存键 | 内容 | 影响 |
| --- | --- | --- |
| `KEY_SYSTEM_INFO_DATA_CACHE` | 系统信息（含广告 ID） | 版本更新、全局开关 |
| `KEY_AD_DATA_CACHE` | 广告数据 | 开屏/插屏/激励/信息流/横幅开关 |
| `KEY_INTIMACY_CONFIG_CACHE` | 亲密度配置 | 养成数值 |
| `KEY_REFERRAL_APP_CONFIG_CACHE` | 邀请返利配置 | 拉新活动 |
| `KEY_UNLOCK_CONFIG_CACHE` | 解锁/付费配置 | 内容付费点 |

广告开关判定函数：`isShowFrontAd`、`isShowBanner`、`isShowNative`、`isShowReward`、`globalIntersEnable`、`pageIntersEnable`、`shouldShowIntersAd`、`shouldShowCombineIntersAd`、`shouldShowSpecialIntersAd`。

### 2.4 网络层与传输加密（核心）

`RetrofitConfig.createRetrofit()`：

```
Retrofit.Builder()
    .baseUrl("https://ai-chat-api.tiankongshuyu.cn/api/")
    .client(createOkHttpClient())
    .addConverterFactory(GsonConverterFactory.create())
    .build()
```

`createOkHttpClient()` 拦截器顺序：

1. 自定义拦截器（`createOkHttpClient$lambda$0`，日志/请求头处理）
2. `HttpLoggingInterceptor(Level.BODY)` —— **发布版仍打印完整请求体**
3. `CryptoInterceptor`
4. 超时均为 30s（connect / read / write）

#### 加密协议（完整恢复）

证据：`CryptoInterceptor`、`EncryptionUtils`、`DecryptUtils`。

- 仅对 **POST** 加密；带 `HEADER_NO_ENCRYPT` 头的请求跳过并移除该头
- 请求体 JSON → Map → 加密；响应按同一套规则解密
- 算法：
  - **AES-256-GCM**：每次请求随机 256-bit 密钥，随机 12 字节 IV，128-bit tag
  - **RSA/ECB/OAEPPadding**：`OAEPParameterSpec("SHA-256", "MGF1", MGF1ParameterSpec.SHA256, PSource.PSpecified.DEFAULT)`，用内置公钥加密 AES 密钥
  - **HMAC-SHA256**：对签名串做 hex 小写摘要，密钥为本次 AES 密钥
  - Base64 均为 `NO_WRAP`（flag=2）
  - `nonce`：16 字节 `SecureRandom` → 32 位 hex
  - `timestamp`：`System.currentTimeMillis()`

请求包结构（JSON）：

```json
{
  "encryptedKey": "<RSA-OAEP(base64) 加密后的 AES Key>",
  "iv": "<base64 12B>",
  "ciphertext": "<base64 AES-GCM(请求JSON)>",
  "timestamp": 1762xxxxxxxxx,
  "nonce": "<32 hex>",
  "sign": "<HMAC-SHA256 hex>"
}
```

签名原文：

```
path={requestPath}&encryptedKey={encryptedKey}&iv={iv}&ciphertext={ciphertext}&timestamp={timestamp}&nonce={nonce}
```

响应处理：

1. 若响应 JSON 不含 `{iv, ciphertext, timestamp, nonce, sign}` → 视作 `plain-json` 直接返回
2. 否则用本次 AES Key 重算 HMAC，比对 `sign`（忽略大小写）
3. 校验通过 → `AES-GCM-decrypt(iv, ciphertext, key)` → JSON
4. 失败分别返回 `encrypted-json-sign-mismatch` / `non-json`

关键常量（`EncryptionUtils`）：

- `GCM_IV_LENGTH = 12`
- `GCM_TAG_LENGTH = 128`
- `RELEASE_PUBLIC_KEY_PEM`：1024-bit RSA 公钥（正式签名）
- `DEBUG_PUBLIC_KEY_PEM`：2048-bit RSA 公钥（调试）
- 公钥直接硬编码在 DEX 静态字段中

> 还原后的可复现流程：
> `AES key = random(32B)` → `encryptedKey = RSA-OAEP-SHA256(pub, key)` → `(iv, ciphertext) = AES-GCM(key, JSON)` → `sign = HMAC-SHA256(key, path&...)`。
> 逆过程用 `key` 验签并解密响应即可，因此**只要拿到内置公钥对应的服务端私钥之外的任何一环都不需要**——客户端侧的加密不构成真正的机密性（公钥公开），仅防篡改与防抓包阅读。

### 2.5 AI 对话链路

证据：`ChatApiService`、`ChatViewModel.chatCompletions`、`ChatCompletionsRequest`。

`ChatCompletionsRequest` 字段：

```
messages: List<MessageListRequest>   // role/content
model: String
stream: Boolean                      // 流式输出
chatId, chatScene, currentStage, recordId, aiAdConfigId
```

`ChatViewModel.chatCompletions(aiAdConfigId, messages)` 先过滤空内容与特定 role，再发起 `chat/completions`（SSE 流式）。相关端点：

- `chat/completions`、`chat/tts`（语音）
- `chat/inspireReplies`（灵感回复）
- `aiChat/add`、`aiChat/detail`
- `multiChatRoom`、`multiChatSend`、`multiChatHistory`（群聊）
- `storyChatRoom`、`storyChatSend`、`storyChatHistory`（剧情）
- `exclusiveCallNameSave`、`getAiAdConfig`

> 说明：AI 生成在服务端完成，客户端只负责消息编排、流式渲染、付费/广告解锁与 TTS 播放。

### 2.6 广告逻辑

`App.initADSDK` 通过 Taku 聚合初始化：

```
AdManager.init(this, AdPlatformType.TAKU, AdConfig(
    appId, appKey,
    rewardPlacementID, insertAdId, feedPlacementID,
    bannerAdId, splashAdId, csjSplashAdId
), callback)
```

- 初始化成功置 `initAdSdkSuccess = true`
- `App.initWebAd(webView)`：每个 `BaseActivity` 在 DecorView 底层插入 **alpha=0、不可点击** 的 WebView（`setupBackgroundWebView`），用于 H5 广告 SDK
- `SplashAdsActivity` 承担开屏广告；`IntersTimer` 控制全局插屏节奏
- 广告平台 ID 由 `com.lhtg.spiritpavilion.ad.Config` 提供（应用内硬编码 + 服务端 `AdData` 配置）

### 2.7 WebView 与 H5 桥

`WebActivity`：

- 开启 JavaScript、允许混合内容，`loadUrl(url)` 后再次 `loadDataWithBaseURL(null, url, "text/html", "UTF-8", null)`
- `handlePayScheme`：识别 `ttcjpay://`、`dypay://`、`alipays://`、`alipay://`、`weixin://`、`wechat://`，以 `ACTION_VIEW + FLAG_ACTIVITY_NEW_TASK` 拉起外部应用完成支付

内置 H5 包（`assets/h5`，Vite 构建，标题 `SDK Web (App Production)`）：

- 页面：`index.html` + `home`（667 KB JS）+ `withdraw`（提现）+ `lottie/reward` 奖励动效
- 原生桥：`callNative(method, params)`，支持 `showToast`、`showLoading`、`hideLoading`、`showSimpleDialog`、`reportBehavior`
- 存在 `cb_<timestamp>_<random>` 回调 ID 机制与事件回调 Map（JS→Native→JS ack）

`BaseActivity.destroyBackgroundWebView()` 在销毁时释放该隐藏 WebView。

### 2.8 设备指纹与风控

`DeviceIDManager`：

- `SmAntiFraud.SmOption` 设置 `organization = @string/sm_appid`、`appId = "default"`、`publicKey = @string/sm_key`、`setNotCollect(emptySet)`
- `fetchBoxId()` 注册 `registerServerIdCallback`，回调得到 boxId 后写入 SP 与内存，并回调启动页继续登录
- 同时采集 OAID（MSA）、UMID、`DeviceIdentifier`
- `SmRiskUtils` 提供 boxId 的同步/异步读取
- 风控结果用于：开屏广告是否展示（`isNoRisk`）、登录 `riskStatus/riskId`

### 2.9 权限面（30 项，节选高风险）

`READ_PHONE_STATE`、`QUERY_ALL_PACKAGES`、`REQUEST_INSTALL_PACKAGES`、`READ_CLIPBOARD_IN_BACKGROUND`、`DETECT_SCREEN_RECORDING`、`WRITE_SETTINGS`、精确定位、相机、通知、`FOREGROUND_SERVICE_SPECIAL_USE`（保活服务 `KeepAliveService`）、存储读写、`GET_TASKS`、`REORDER_TASKS`、广告相关 `TT_PANGOLIN` / MSA 权限等。

---

## 3. UI 结构

### 3.1 主题与视觉规范

| 项 | 值 |
| --- | --- |
| 主题 | `Theme.AppCompat.NoActionBar`（深色） |
| 主背景 | `#15141A`（`activity_main`）/ `#151515`（`color_15`） |
| 强调色 | `#D647FF`（`@color/main`，紫红） |
| 未选中 | `#AAAAAA`（`@color/color_a`） |
| 底部导航文字 | 10sp、粗体、选中/未选中用 `menu_txt_color_selector` |
| 图标尺寸 | `itemIconSize=25dp`，`labelVisibilityMode=labeled` |
| 状态栏 | 沉浸式（`useImmersiveStatusBar` / `setupWindowInsetsForTopBar`） |

### 3.2 主界面骨架（`activity_main.xml`）

```
RelativeLayout (bg #15141A)
├── ConstraintLayout
│   ├── WebView (0dp, alpha=0)                 // 隐藏，用于 Web 广告
│   ├── FragmentContainerView (fragmentContainer)
│   └── BottomNavigationView (bottomNavigation)
│        菜单 bottom_navigation_menu：
│          推荐 navigation_home / 广场 navigation_square
│          后宫 navigation_harem / 我的 navigation_mine
└── 新人福利悬浮按钮 (newTask)
      ├── ic_zuan（52dp，圆角）
      ├── “新人福利”角标
      └── 红点 redDot
```

`MainActivity` 默认加载 `HomeFragment`；`EXTRA_GO_TO_HAREM` / `EXTRA_GO_TO_SQUARE` 可从外部直接切换 Tab。

### 3.3 页面与 Fragment 清单

Fragment（13）：`fragment_home`、`home_all`、`home_select`、`square`、`harem`、`ranking`、`intimacy`、`intimacy_ranking`、`mine`、`album`、`friend`、`info`、`splash`。

应用自有页面布局（40 个 `activity_*`，已排除广告 SDK 页面）：

| 域 | 页面 |
| --- | --- |
| 启动/引导 | `activity_splash`、`activity_splash_ads`、`activity_guidance1`、`activity_guidance2`、`activity_gender_select` |
| 主框架 | `activity_main`、`activity_main_chat` |
| 聊天 | `activity_chat`、`activity_plot_chat`、`activity_two_chat`、`activity_there_chat`、`activity_group_chat`、`activity_group_photo`、`activity_chat_record` |
| 角色 | `activity_role_center`、`activity_change_name`、`activity_friend`、`activity_image`、`activity_image_preview`、`activity_sleep` |
| 广场 | `activity_scene_list`、`activity_create_role`、`activity_create_scene`、`activity_create_plot_chat` |
| 商业化 | `activity_vip`、`activity_vip_pay`、`activity_currency`、`activity_currency_details`、`activity_bz_privilege` |
| 榜单 | `activity_ranking`、`activity_hall_of_fame` |
| 其他 | `activity_new_task_publish`、`activity_new_task_evaluate`、`activity_setting`、`activity_web`、`activity_in_app_browser`、`activity_sdk` |

### 3.4 关键布局结构

**聊天/剧情页（`activity_plot_chat.xml`）**

```
RelativeLayout
├── PlayerView (ExoPlayer, 全屏视频背景, resize_mode=zoom, use_controller=false)
├── ImageView (全屏封面, centerCrop)
├── TextView (底部 160dp 渐变遮罩)
└── LinearLayout (垂直)
    ├── top_layout：返回按钮 + 标题
    ├── RecyclerView (消息列表, 垂直渐隐边缘)
    ├── bottom_input_layout：输入框 + 灵感按钮 (ic_more_send)
    └── inspiration_layout：加载中 + 灵感建议 RecyclerView (默认 gone)
```

**角色中心（`activity_role_center.xml`）**：`NestedScrollView` + 横向滑动卡片区 + 27 个 TextView/15 个 ImageView（属性、亲密度、换装、相册入口）。

**VIP（`activity_vip.xml`）**：`RecyclerView` × 2 + 卡片式权益/支付方式列表。

**启动页（`activity_splash.xml`）**：全屏 `loadImg` → 开屏广告容器 `openAds` → 居中 logo（150dp）+ 应用名 → 右下 `ic_r18`（18+ 标识）→ 底部水平进度条。

### 3.5 UI 素材清单

资源总占用（解码后）：`res 96MB`、`assets 31MB`、`lib 14MB`。

| 类别 | 数量/大小 | 代表文件 |
| --- | --- | --- |
| 图片（png/webp/jpg） | 1,370 个，`drawable-xxhdpi` 36MB | `bg_harem_img.png`(3.85MB)、`bg_group_photo_top.png`、`ic_ranking_top.png`、`ic_love_record.png`、`bg_top_vip.png`、`bg_home.png`、`bg_new_task_dialog.png`、`bg_new_user_gift.png`、`bg_open_vip_dialog.png`、`bg_ranking_top.png`、`bg_unlock_dialog_vip.png` |
| 字体 | 39MB | `source_han_serif_cn_heavy.otf`(13.5MB)、`sourcehansanscnbold/medium/regular.otf`(各约 8.4MB)、`zihunhaimatiziti.ttf`(1.8MB)、`roboto_medium_numbers.ttf` |
| 视频 | 7.7MB | `assets/bg_harem_video.mp4`（后宫女主动态背景） |
| H5 资源 | 15MB | `assets/h5/*`：`index.html`、`home-COwuVOzb.js`(668KB)、`withdraw`、`vip-decoration*.svg`、`reward-*.png/svg`、`daka-DjEgNk2l.ttf` |
| Lottie | 3 个 | `lottie_json/shake_phone.json`、`swipe_right.json`、`twist_multi_angle.json`、`h5/lottie/reward.json` |
| 广告素材 | — | `ksad_data.3gp`、`ad_mate_error.png`、`zj_*.png`、`saio_res/*.png` |
| 图标规范 | — | `ic_*`（首页/广场/后宫/我的/聊天/亲密度/新人任务）、`bg_*`、`radius_*`、`selector_menu_*` |

素材命名可反推设计与模块划分：`ic_home_*`、`ic_square_*`、`ic_harem_*`、`ic_mine_*`、`ic_chat*`、`ic_intimacy*`、`ic_new_task*`、`bg_*`（大图背景）、`radius_*`（圆角背景）、`selector_*`（状态选择器）。

---

## 4. 关键逻辑的静态重建

以下为从字节码恢复的等价实现，可直接用于协议复现 / 流量分析 / 自有实现参考。

### 4.1 启动路由（Kotlin 伪代码）

```kotlin
class SplashActivity : BaseActivity() {
    private var jumped = false
    private var adData: AdData? = null

    override fun initData() {
        viewModel.loadAllConfiguration()
        handler.postDelayed(updatePBRunnable, 100L)
        App.initWebAd(bgWebView)
    }

    private fun startLogin() {
        DeviceIDManager.getBoxIdAsync { boxId -> handleLogin(boxId) }
    }

    private fun goMain() {
        if (!NetworkUtil.isNetworkAvailable(applicationContext)) {
            ToastUtil.showMessage("网络未连接"); return
        }
        if (!jumped) {
            jumped = true
            if (adData == null) {
                adData = AdData().apply { numberOfAds = 3; statusType = "3" }
            }
            routeAfterSplash()
            return
        }
        startActivity(Intent(this, MainActivity::class.java))
        finish()
    }

    private fun routeAfterSplash() {
        val d = adData ?: return
        if (d.statusType == "3") {
            when {
                App.isNoRisk() &&
                AppDataManager.isShowFrontAd(false) &&
                !App.isVIP() ->
                    startActivity(Intent(this, SplashAdsActivity::class.java)
                        .putExtra("initAllConfigSuccess", initAllConfigSuccess))
                SpUtil.getInt(this, SpConstants.GUIDANCE, 0) == 0 ->
                    startActivity(Intent(this, GuidanceActivity1::class.java))
                else ->
                    startActivity(Intent(this, MainActivity::class.java))
            }
            finish()
        }
    }
}
```

### 4.2 请求加解密（可直接复现）

```kotlin
object ReaCrypto {
    const val IV_LEN = 12
    const val TAG_BITS = 128
    val PUBLIC_KEY_PEM = """
        -----BEGIN PUBLIC KEY-----
        MIGfMA0GCSqGSIb3DQEBAQUAA4GNADCBiQKBgQDKmUgsljF+SPOJjQ/1f6Hi1rPs
        ...（见 evidence/class-EncryptionUtils.json）
        -----END PUBLIC KEY-----
    """.trimIndent()

    fun encryptPost(path: String, body: Any): Map<String, Any> {
        val aesKey = ByteArray(32).also { SecureRandom().nextBytes(it) }
        val iv = ByteArray(IV_LEN).also { SecureRandom().nextBytes(it) }
        val ts = System.currentTimeMillis()
        val nonce = ByteArray(16).also { SecureRandom().nextBytes(it) }
            .joinToString("") { "%02x".format(it) }

        // RSA/ECB/OAEPPadding, SHA-256 + MGF1(SHA-256)
        val cipherRsa = Cipher.getInstance("RSA/ECB/OAEPPadding").apply {
            init(Cipher.ENCRYPT_MODE, loadPublicKey(PUBLIC_KEY_PEM),
                 OAEPParameterSpec("SHA-256", "MGF1", MGF1ParameterSpec.SHA256,
                                   PSource.PSpecified.DEFAULT))
        }
        val encryptedKey = Base64.encodeToString(cipherRsa.doFinal(aesKey), Base64.NO_WRAP)

        // AES-256-GCM
        val cipherAes = Cipher.getInstance("AES/GCM/NoPadding").apply {
            init(Cipher.ENCRYPT_MODE, SecretKeySpec(aesKey, "AES"), GCMParameterSpec(TAG_BITS, iv))
        }
        val ciphertext = Base64.encodeToString(
            cipherAes.doFinal(Gson().toJson(body).toByteArray()), Base64.NO_WRAP)

        val signPayload = "path=$path&encryptedKey=$encryptedKey&iv=${
            Base64.encodeToString(iv, Base64.NO_WRAP)}&ciphertext=$ciphertext&timestamp=$ts&nonce=$nonce"
        val sign = hmacSha256Hex(signPayload, aesKey)

        return linkedMapOf(
            "encryptedKey" to encryptedKey,
            "iv" to Base64.encodeToString(iv, Base64.NO_WRAP),
            "ciphertext" to ciphertext,
            "timestamp" to ts,
            "nonce" to nonce,
            "sign" to sign
        )
    }

    fun decryptResponse(path: String, json: Map<String, Any>, aesKey: ByteArray): String {
        if (!json.keys.containsAll(setOf("iv", "ciphertext", "timestamp", "nonce", "sign")))
            return Gson().toJson(json)                       // plain-json

        val payload = "path=$path&iv=${json["iv"]}&ciphertext=${json["ciphertext"]}" +
                      "&timestamp=${json["timestamp"]}&nonce=${json["nonce"]}"
        check(hmacSha256Hex(payload, aesKey).equals(json["sign"].toString(), ignoreCase = true)) {
            "sign mismatch"
        }
        val cipher = Cipher.getInstance("AES/GCM/NoPadding").apply {
            init(Cipher.DECRYPT_MODE, SecretKeySpec(aesKey, "AES"),
                 GCMParameterSpec(TAG_BITS, Base64.decode(json["iv"].toString(), Base64.NO_WRAP)))
        }
        return String(cipher.doFinal(Base64.decode(json["ciphertext"].toString(), Base64.NO_WRAP)))
    }

    fun hmacSha256Hex(content: String, key: ByteArray): String =
        Mac.getInstance("HmacSHA256").run {
            init(SecretKeySpec(key, "HmacSHA256"))
            doFinal(content.toByteArray()).joinToString("") { "%02x".format(it) }
        }
}
```

> 注意：响应签名原文**不含 `encryptedKey`**，请求签名原文**含 `encryptedKey`**；两者 path 均使用 `Request.url().encodedPath`。

### 4.3 登录请求（还原）

```kotlin
data class NewLoginRequest(
    val boxId: String,          // 数美 boxId
    val oaid: String?,
    val umid: String?,
    val type: Int = 0,
    val manufacturer: String,   // Build.MANUFACTURER
    val channel: String,        // "10039"
    val model: String,          // Build.MODEL
    val platform: String = "0",
    val adId: String
)

// POST https://ai-chat-api.tiankongshuyu.cn/api/user/login
// → { token, riskStatus, riskId, userInfo ... }
```

### 4.4 AI 对话请求（还原）

```json
POST /chat/completions
{
  "messages": [ { "role": "user", "content": "..." } ],
  "model": "<服务端指定>",
  "stream": true,
  "chatId": "...",
  "chatScene": "...",
  "currentStage": "...",
  "recordId": "...",
  "aiAdConfigId": "..."
}
```

### 4.5 广告初始化（还原）

```kotlin
AdManager.init(
    context = app,
    type = AdPlatformType.TAKU,
    config = AdConfig(
        appId, appKey,
        rewardPlacementID, insertAdId, feedPlacementID,
        bannerAdId, splashAdId, csjSplashAdId
    ),
    callback = object : AdSdkInitCallback {
        override fun onAdInitSuccess() { App.initAdSdkSuccess = true }
        override fun onAdInitFail(code: Int, msg: String) { /* 仅日志 */ }
    }
)
```

---

## 5. 安全与风险观察

| 观察 | 影响 |
| --- | --- |
| 允许明文 HTTP + 信任用户 CA | 可被中间人/抓包分析；用户流量存在被篡改风险 |
| Release 包启用 `HttpLoggingInterceptor(Level.BODY)` | 明文请求/响应写入 Logcat（含 token、聊天内容） |
| 加密为“客户端内置 RSA 公钥 + 每请求 AES” | 公钥公开，协议可完全离线复现；只能防阅读，不能防伪造（服务端仍需鉴权） |
| RSA 正式公钥仅 1024-bit | 低于现代推荐强度 |
| 明文保存 Umeng/数美等 SDK key、广告位 ID | SDK 维度可被冒用/刷量 |
| 30 项权限，含 `QUERY_ALL_PACKAGES`、`READ_CLIPBOARD_IN_BACKGROUND`、`DETECT_SCREEN_RECORDING`、`REQUEST_INSTALL_PACKAGES` | 隐私面较大，具备安装包下发与广告归因能力 |
| `KeepAliveService` + `FOREGROUND_SERVICE_SPECIAL_USE` | 保活行为 |
| 大量广告 SDK 硬编码 + 动态下发 | 广告内容合规性依赖服务端配置 |

---

## 6. 证据文件索引

所有原始 Evidence 位于 `/workspace/rea-analysis/`：

| 文件 | 内容 |
| --- | --- |
| `evidence/package.json` | APK 声明、权限、264 个组件、27,033 类 |
| `evidence/signature.json` | v1/v2/v3 签名、证书、摘要 |
| `evidence/app-classes.json` / `app-classes.txt` | 852 个应用类清单 |
| `evidence/class-*.json` | 关键类成员清单（App、ApiConstants、AppDataManager、SplashActivity、MainActivity、WebActivity、BaseActivity、RetrofitConfig、CryptoInterceptor、EncryptionUtils、DecryptUtils、ChatViewModel、LoginViewModel、UserRepository 等） |
| `evidence/meth-*.json` | 关键方法反编译源码（启动、登录、加解密、导航、支付、WebView、广告） |
| `apktool-res/` | Apktool 解码后的完整资源（`res/layout` 1,142 个、`assets`、`lib`、`AndroidManifest.xml`） |
| `dexstrings/classes*.dex` | 从 APK 提取的 DEX，用于字符串/包名检索 |

---

## 7. 分析限制

- 全程静态分析，未安装/运行 APK，未连接真实后端；接口真实响应结构与字段以运行时为准。
- Kotlin 协程请求体构造位于编译器生成的匿名类中，JADX 部分显示为 `AnonymousClass1`，本报告以 `UserRepository`/请求模型为依据还原。
- 广告聚合（Taku）与各广告平台内部实现未逐一展开，仅确认集成与初始化路径。
- 服务端公钥对应私钥、后端鉴权与风控策略不可从客户端静态获得。
