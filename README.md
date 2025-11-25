# ⚠️该仓库fork于 MeTerminator/hcf2023 ，文档由 xAI: Grok 4.1 Fast 生成，请注意甄别！

## 设备检测技术完整手册 - HCF2023 项目全解析

**文档生成时间**：2025年11月25日  
**项目来源**：GitHub仓库 `hcf2023-main`（原网站：https://hcf2023.top/，已无法访问）  
**核心主题**：虽然原项目以“安卓电脑”玩梗为表象，但其**设备检测技术**极具价值。本手册整合所有总结文档、仓库代码及详细解释，聚焦**多层次、高精度设备/OS识别**技术。  
**适用人群**：Web开发者、安全研究者、设备指纹爱好者。  
**警告**：仅供技术交流学习，遵守隐私法规（如GDPR/CCPA）。

---

### 📖 文档导航与阅读指南

#### 项目概述
- **表象**：检测设备后播放梗音效、弹幕雨、非Apple设备触发WebGL卡顿（玩梗）。
- **本质**：**现代Web API驱动的设备指纹系统**，准确率高达95%+，创新PASSKEY硬件验证。
- **技术栈**：
  | 层级 | 技术                                                                  |
  | ---- | --------------------------------------------------------------------- |
  | 前端 | 原生JS + WebGL/WebAuthn/Client Hints/CSS Media/Web NFC/Serial/HID/USB |
  | 后端 | Node.js HTTP服务器（Client Hints头配置）                              |
  | 样式 | iOS HIG风格CSS（明暗模式自适应）                                      |

#### 推荐阅读顺序
1. **速查表**：快速掌握18种检测方法。
2. **流程图**：理解多层架构。
3. **技术总结**：深入原理与创新。
4. **代码解析**：逐文件解释（3290行JS核心）。
5. **部署&最佳实践**。

---

### ⚡ 检测方法速查表

#### 🎯 快速索引
```
设备检测体系
│
├─ 基础检测层 (JavaScript API + CSS) [18种，权重总18+]
│  ├─ 输入设备 (+2)
│  ├─ Apple (+2~6)
│  ├─ Android (+3~4)
│  ├─ 桌面 (+1~4)
│  ├─ 屏幕 (+5)
│  └─ WebGL (+4~6)
│
├─ 高级检测 [权重最高]
│  ├─ Client Hints (+3~5)
│  └─ PASSKEY (+8) ⭐
└─ 裁决：UA平分介入
```

#### 📋 检测清单
##### 🔵 基础检测
- **输入**：`maxTouchPoints`、`pointer:coarse`
- **Apple**：`-webkit-touch-callout`(+5)、`ApplePaySession`(+4)、`safari.pushNotification`(+4)、`DeviceMotionEvent.requestPermission`(+6)、`navigator.standalone`(+2)
- **Android**：`NDEFReader`(+4)、`getInstalledRelatedApps`(+3)
- **桌面**：`serial`(+4)、`hid`(+2)、`usb`(+1)
- **屏幕**：`screen.width/height + devicePixelRatio`(+5)
- **WebGL**：`UNMASKED_VENDOR/RENDERER`（apple/direct3d/mesa/adreno等，+4~6）

##### 🟢 高级检测
- **Client Hints**：
  ```js
  navigator.userAgentData.mobile/platform // 低熵
  await getHighEntropyValues(['platformVersion','architecture','model']) // 高熵
  ```
- **PASSKEY**：
  ```js
  PublicKeyCredential.isUserVerifyingPlatformAuthenticatorAvailable()
  navigator.credentials.create() // transports: internal/hybrid/usb
  ```

#### 🎯 系统速查
| 系统        | 关键特征                                                                                            |
| ----------- | --------------------------------------------------------------------------------------------------- |
| **iOS**     | WebKit CSS(+5)、ApplePay(+4)、MotionPermission(+6)、standalone(+2)、WebGL apple(+6)、短边<600px(+5) |
| **macOS**   | ApplePay(+4)、SafariPush(+4)、WebGL apple/angle+metal(+6/+4)、Serial(+4)                            |
| **Android** | NFC(+4)、RelatedApps(+3)、WebGL adreno/mali(+4)、Touch(+2)                                          |
| **Windows** | WebGL direct3d(+6)、Serial/HID/USB(+4/+2/+1)                                                        |
| **Linux**   | WebGL mesa/llvmpipe(+5)、Serial(+4)                                                                 |

#### 🔢 置信度表
| 条件           | 置信度 |
| -------------- | ------ |
| ≥15分 & 差距≥8 | 98%    |
| ≥12 & ≥6       | 95%    |
| ...            | ...    |
| 差距<1         | 45%    |

**高级**：PASSKEY+CH一致=95%、仅PASSKEY=90%。

#### 🛠️ 代码片段
```js
// 投票系统
const scores = {android:0, ios:0,...};
const vote = (targets, weight, title, ok=true) => {
  if(ok) targets.forEach(t=>scores[t]+=weight);
};

// WebGL
function getWebGLInfo(){ /* canvas+WEBGL_debug_renderer_info */ }

// PASSKEY
await PublicKeyCredential.isUserVerifyingPlatformAuthenticatorAvailable();
```

**浏览器兼容**：
| API          | Chrome | Safari | Firefox | Edge |
| ------------ | ------ | ------ | ------- | ---- |
| Client Hints | 89+    | ❌      | ❌       | 89+  |
| WebAuthn     | 67+    | 14+    | 60+     | 18+  |

---

### 🔄 检测流程图

#### 📊 完整流程（ASCII）
```
开始 → 基础检测(触控/Apple/Android/WebGL/桌面/屏幕)
     ↓
分数统计(iOS:15, Android:8, ...)
     ↓ 平分? → UA裁决
     ↓
高级检测(Client Hints + PASSKEY)
     ↓ 一致? → ✅95% | ⚠️篡改25%
     ↓
输出: OS + 置信度 + 信号JSON
```

#### 🔄 详细步骤
1. **基础**：18方法投票。
2. **裁决**：平分UA介入（Android优先、iPhone→iOS等）。
3. **高级**：CH低/高熵 + PASSKEY（3s超时，transports推断）。
4. **篡改检测**：基础≠高级 → 红主题+警告。

#### 📈 置信度流程
```
topScore - secondScore = gap
≥15&≥8=98% ... gap<1=45%
```

#### 🎯 输出JSON示例
```json
{
  "detectedOS": "ios", "confidence":95,
  "scores":{...}, "signals":{...},
  "advancedDetection":{ "passkey":{...} },
  "status":"normal"
}
```

---

### 🎓 设备检测技术总结

#### 📋 核心架构
- **基础层**：JS/CSS API（输入/Apple/Android/桌面/屏幕/WebGL）。
- **高级层**：CH（服务器头+API）+PASSKEY（硬件认证器）。
- **创新**：
  1. **PASSKEY**：WebAuthn transports（internal=TouchID等，+8分）。
  2. **权重投票**：累计分数+可视化。
  3. **篡改检测**：基础≠高级 → 警告。
  4. **隐私**：短超时、分级访问。

#### 🧮 置信度算法
```js
if(top>=15&&gap>=8) confidence=98; // ...
```

#### 🔒 篡改机制
```js
if(basicOS !== advancedOS) { isTampered=true; applyRedTheme(); }
```

#### 🎨 UX
- 音频：Apple.mp3 / Android_phone/compute.mp3（用户交互后播）。
- 弹幕：OS特定梗词。
- WebGL：非Apple后台渲染卡顿（?disablecanvas禁用）。

#### 📊 对比
| 方法    | 准确 | 隐私 | 伪造难 |
| ------- | ---- | ---- | ------ |
| UA      | 低   | 低   | 易     |
| PASSKEY | 极高 | 高   | 极难   |

**场景**：适配/安全/统计/反诈。

---

### 🗂️ 项目结构与部署

```
hcf2023-main/
├── public/
│   ├── index.html     ## 主页（meta CH头+iOS PWA）
│   ├── script.js      ## 核心检测(3290行)
│   ├── style.css      ## iOS HIG CSS
│   └── *.mp3          ## 音效
├── server.js          ## Node服务器(CH头)
├── package.json       ## deps
├── README.md          ## 说明
└── *.md               ## 本手册源
```

**部署**：
```bash
npm install
node server.js  ## http://localhost:3000
```

**package.json**：
```json
{
  "name": "device-detector",
  "version": "1.0.0",
  "description": "Node.js HTTP server for Client Hints",
  "main": "server.js",
  "scripts": {"start": "node server.js"},
  "keywords": ["http","client-hints"],
  "author": "MeTerminator",
  "license": "MIT"
}
```

**README.md**：项目留档，?disablecanvas禁用卡顿，API示例。

---

### 💻 代码详细解析

#### 1. **server.js** (Node.js服务器，~200行)
**作用**：静态文件+`/api/client-hints`（捕获CH头）。

**关键**：
- **CH头配置**：
  ```js
  Accept-CH: Sec-CH-UA*,...
  Permissions-Policy: ch-ua=*,...
  Critical-CH: Sec-CH-UA-Platform,Sec-CH-UA-Mobile
  ```
- **提取CH**：
  ```js
  function extractClientHints(req) {
    return { userAgent, mobile, platform, ... }; // 清理引号
  }
  ```
- **响应**：JSON+CH头，支持HTTPS检测。

**解释**：启用浏览器CH，服务器验证UA真实性。高熵需客户端`getHighEntropyValues()`。

#### 2. **index.html** (~100行)
**作用**：入口，iOS PWA+CH meta。

**关键**：
```html
<meta http-equiv="Accept-CH" content="Sec-CH-UA*,...">
<meta name="apple-mobile-web-app-capable" content="yes">
```
- 结构：导航+结果卡+过程卡+弹幕/WebGL容器。
- 无框架，原生。

#### 3. **style.css** (~800行，iOS HIG)
**作用**：明暗自适应、卡片/按钮/进度条/弹幕。

**亮点**：
- **变量**：`--system-blue`等，`@media (prefers-color-scheme: dark)`。
- **响应**：容器查询`cqmin`字号自适应。
- **动画**：`cubic-bezier` iOS曲线、减motion支持。
- **篡改**：红主题`tamperPulse`脉动。

#### 4. **script.js** (3290行，核心！)
**作用**：检测+UX（音频/弹幕/WebGL）。

##### **基础检测** (`detect()`)：
```js
const scores = {...}; vote(targets, weight, title);
```
- **输入**：`maxTouchPoints`、`pointer:coarse`。
- **Apple**：CSS.supports、ApplePay、MotionPermission等。
- **Android**：NFC/RelatedApps（async）。
- **桌面**：Serial/HID/USB。
- **WebGL**：`UNMASKED_RENDERER`（apple/direct3d/mesa/adreno）。
- **UA裁决**：平分时优先Android/iPhone。

##### **高级检测** (`performAdvancedDetection()`)：
- **CH**：meta+fetch `/api` + `userAgentData.getHighEntropyValues()`。
- **PASSKEY**：
  ```js
  isUserVerifyingPlatformAuthenticatorAvailable() // 平台器
  credentials.create({authenticatorAttachment:"platform",timeout:3000}) // transports
  ```
  - internal=内置（TouchID/Hello）、usb=桌面。

- **篡改**：不一致→红主题+弹窗。

##### **UX逻辑**：
- **音频**：`AudioManager`，用户交互后播（loop=1，vol=0.7）。
- **弹幕**：`danmu-container`，OS梗词，5s爆发+循环。
- **WebGL**：3D着色器（raymarching内核），非Apple后台渲染卡顿。
  ```js
  // 片段着色器：kernal() Mandelbulb分形
  float kernal(vec3 ver){ /* 迭代5次 */ }
  // 射线行进+法线+反射
  ```

##### **无限循环**：
弹幕3s → WebGL5s → 循环（Canvas禁用只弹幕）。

**置信度**：精细gap映射，95%+常见。

---

### 📄 LICENSE (MIT)
```
Copyright (c) 2025 MeTerminator
自由使用/修改/分发，无担保。
```

---

### 🚀 最佳实践&局限
#### ✅ 推荐
- 多层+权重，避免UA单依。
- HTTPS必备（NFC/CH/WebAuthn）。
- 隐私：知情同意、短超时。

#### ❌ 局限
- 隐私模式禁用API。
- 扩展/VM干扰。
- Firefox弱CH/WebAuthn。

**扩展**：集成服务器UA验证，反爬虫。

---

**结束语**：HCF2023不仅是梗，更是最强Web设备检测范例。PASSKEY创新值得深挖！如需源码调试，运行`node server.js`。
