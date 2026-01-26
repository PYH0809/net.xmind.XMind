# electron-webauthn-mac 库研究报告

## 概述

**仓库地址**: https://github.com/vault12/electron-webauthn-mac
**许可证**: MIT
**版本**: 1.0.0

这是一个为 macOS 上的 Electron 应用提供原生 WebAuthn/Passkey 支持的库。由于 Electron 在 macOS 上的标准 Web Authentication API (`navigator.credentials`) 存在问题无法正常工作，该库通过 Apple 的 AuthenticationServices 框架提供了一个替代方案。

## 为什么需要这个库？

在 Electron 应用中，标准的 `navigator.credentials` API 在 macOS 平台上是损坏的/不可用的。这个库通过直接调用 Apple 原生的 AuthenticationServices 框架来绕过这个限制，实现 WebAuthn 功能。

## 支持的功能

### 认证器类型
- **平台认证器 (Platform Authenticators)**
  - Touch ID
  - iCloud 钥匙串 (iCloud Keychain)
  - 跨设备 QR 码配对
- **安全密钥 (Security Keys)**
  - 外部 FIDO2 硬件令牌

### 扩展功能
- **PRF (Pseudo-Random Function)**: 用于密钥派生，仅支持平台认证器，需要 macOS 15.0+
- **LargeBlob**: 数据存储功能，仅支持平台认证器，需要 macOS 14.0+

## 技术架构

```
┌─────────────────────────────────────────────────────────┐
│                    JavaScript/TypeScript                 │
│                      (js/index.js)                       │
├─────────────────────────────────────────────────────────┤
│                    C++ Node-API 绑定                     │
│                   (.node 原生模块)                        │
├─────────────────────────────────────────────────────────┤
│                 Objective-C/Swift 桥接                   │
│                (PasskeyManager.swift)                    │
├─────────────────────────────────────────────────────────┤
│            Apple AuthenticationServices 框架             │
└─────────────────────────────────────────────────────────┘
```

### 文件结构
```
electron-webauthn-mac/
├── dev-mac-app/                # 原生 macOS 开发/测试应用
├── example-electron-app/       # 示例 Electron 应用
├── include/                    # C++ 头文件
├── js/                         # JavaScript 包装器 + TypeScript 定义
├── src/                        # Swift/Objective-C/C++ 源代码
│   └── PasskeyManager.swift    # 核心 WebAuthn 逻辑
├── binding.gyp                 # node-gyp 构建配置
└── package.json
```

## 安装

```bash
npm install electron-webauthn-mac
```

## 核心 API

### 1. createCredential() - 创建/注册凭证

用于注册新的 Passkey。

**必需参数**:
- `rpId`: 依赖方 ID (域名)
- `userId`: 用户 ID
- `name`: 用户名
- `displayName`: 显示名称

**示例**:
```typescript
import webauthn from 'electron-webauthn-mac';

const credential = await webauthn.createCredential({
  rpId: 'example.com',
  userId: 'user-123',
  name: 'john@example.com',
  displayName: 'John Doe',
  authenticatorType: 'platform' // 或 'securityKey'
});
```

### 2. getCredential() - 获取/验证凭证

用于使用现有凭证进行身份验证。

**必需参数**:
- `rpId`: 依赖方 ID

**示例**:
```typescript
const assertion = await webauthn.getCredential({
  rpId: 'example.com',
  allowCredentials: [{ id: 'credential-id-base64' }]
});
```

### 3. managePasswords() - 管理密码

打开系统密码管理界面。

```typescript
webauthn.managePasswords();
```

## 重要配置要求

### 1. 域名关联 (Domain Association)

macOS 要求证明你的应用拥有作为依赖方 ID 使用的域名。这是为了防止恶意应用冒充银行等机构窃取凭证。

**步骤**:
1. 在服务器上托管 `apple-app-site-association` 文件:
   ```
   https://yourdomain.com/.well-known/apple-app-site-association
   ```

2. 文件内容示例:
   ```json
   {
     "webcredentials": {
       "apps": ["TEAM_ID.com.yourcompany.yourapp"]
     }
   }
   ```

### 2. 代码签名

应用必须是经过签名的 `.app` 包，并包含嵌入的 entitlements。未签名的进程会因 "application identifier" 错误而失败。

**Entitlements 示例**:
```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
<dict>
    <key>com.apple.developer.associated-domains</key>
    <array>
        <string>webcredentials:yourdomain.com</string>
    </array>
</dict>
</plist>
```

### 3. Provisioning Profile

需要包含 associated-domains 能力的 provisioning profile。

## 与浏览器 WebAuthn 的区别

| 特性 | 浏览器 WebAuthn | electron-webauthn-mac |
|------|----------------|----------------------|
| Challenge | 手动提供 | 自动内部生成 |
| rp.name | 支持 | 不支持 |
| 超时配置 | 可配置 | 不可配置 |
| 算法 | 多种 | 仅 ES256 |

## 跨平台兼容性

该库仅在 macOS 上工作。建议的跨平台策略:

```typescript
let webauthn;

if (process.platform === 'darwin') {
  webauthn = require('electron-webauthn-mac');
} else {
  // 使用标准 Web Authentication API 或其他平台特定实现
  webauthn = navigator.credentials;
}
```

## 依赖

**开发依赖**:
- node-gyp (^11.5.0) - 原生模块构建工具
- node-addon-api (^8.5.0) - Node.js 扩展 API
- electron-rebuild (^3.2.9) - Electron 模块重建工具

**运行依赖**: 无

## 系统要求

- macOS (Darwin)
- PRF 扩展: macOS 15.0+
- LargeBlob 扩展: macOS 14.0+
- 需要代码签名和域名关联配置

## 适用场景

1. Electron 应用需要实现 Passkey/WebAuthn 登录
2. 需要支持 Touch ID 认证
3. 需要与 iCloud 钥匙串集成
4. 需要支持 FIDO2 安全密钥

## 潜在集成考虑

对于 XMind Electron 应用集成此库需要考虑:

1. **域名关联**: 需要在 xmind.net 或相关域名配置 apple-app-site-association
2. **代码签名**: 确保 macOS 版本已正确签名
3. **Entitlements**: 添加 associated-domains entitlement
4. **条件加载**: 仅在 macOS 平台加载此模块
5. **后端支持**: 服务端需要实现 WebAuthn 验证逻辑

## 参考链接

- GitHub 仓库: https://github.com/vault12/electron-webauthn-mac
- Apple AuthenticationServices: https://developer.apple.com/documentation/authenticationservices
- WebAuthn 规范: https://www.w3.org/TR/webauthn/
