# NodeOnly 订阅功能修改记录

## 📋 功能概述

**目标**：在 BPB Panel 中新增一个 "NodeOnly" 订阅类型，仅包含 xray 核心的基础节点（v2rayNG、MahsaNG、v2rayN、v2rayN-PRO、Streisand），每个节点生成独立的分享链接（VLESS/Trojan），以 BASE64 编码格式提供，每行一条链接。

**创建时间**：2026-03-11  
**基于版本**：BPB-Worker-Panel v4.1.3  
**最后更新**：2026-03-11

---

## 📝 更新日志

### 2026-03-11 - 后续更新

#### 🔧 修复 1：支持 non-TLS 端口链接生成
**问题**：初始版本只生成 HTTPS 端口（TLS）的链接，non-TLS 端口的链接被过滤掉了。

**修复位置**：`src/cores/xray/configs.ts` - `getNodeOnlyLinks()` 函数

```typescript
// 修复前：
const totalPorts = ports.filter(port => isHttps(port));

// 修复后：
const totalPorts = ports; // Include all ports (both TLS and non-TLS)
```

**同时修复了 Trojan 链接格式**：
```typescript
// 修复前：Trojan 强制使用 TLS
const params = new URLSearchParams({
    security: 'tls',  // ❌ 硬编码
    // ...
});

// 修复后：根据端口类型动态设置
const params = new URLSearchParams({
    security: isTLS ? 'tls' : 'none',  // ✅ 动态
    // ...
});
```

#### 🔄 修复 2：响应内容改为 BASE64 编码
**原因**：有些客户端不支持明文订阅，BASE64 编码后兼容性更好。

**修复位置**：`src/cores/xray/configs.ts` - `getNodeOnlyLinks()` 函数

```typescript
// 添加 Base64 编码
const plainText = links.join('\n');
const base64Content = btoa(plainText);

return new Response(base64Content, {
    status: 200,
    headers: {
        'Content-Type': 'text/plain;charset=utf-8',
        'Cache-Control': 'no-store',
        'CDN-Cache-Control': 'no-store'
    }
});
```

**效果对比**：
- **修改前（明文）**：客户端直接读取 `vless://...` 和 `trojan://...`
- **修改后（BASE64）**：客户端读取编码后的字符串，自动解码后得到相同内容

---

## 🔧 修改文件清单

### 1. `src/cores/xray/configs.ts`

#### 修改位置 1：导入语句（第 13-24 行）
```typescript
import {
    getConfigAddresses,
    generateRemark,
    isDomain,
    isHttps,
    getProtocols,
    parseHostPort,
    toRange,
    generateWsPath,      // ← 新增
    selectSniHost,       // ← 新增
    randomUpperCase      // ← 新增
} from '@utils';
```

**目的**：引入生成分享链接所需的工具函数

---

#### 修改位置 2：文件末尾添加新函数（第 418 行之后）

**初始版本代码**（已过时，仅供参考）：
```typescript
function generateShareLink(...) { ... }

export async function getNodeOnlyLinks(): Promise<Response> {
    const { outProxy, ports } = globalThis.settings;
    const hasChain = !!outProxy;

    const addresses = await getConfigAddresses(false);
    const totalPorts = ports.filter(port => isHttps(port));  // ❌ 只包含 TLS 端口
    const protocols = getProtocols();

    const links: string[] = [];
    let index = 1;

    // ... 循环生成链接 ...

    return new Response(links.join('\n'), {  // ❌ 明文输出
        status: 200,
        headers: {
            'Content-Type': 'text/plain;charset=utf-8',
            'Cache-Control': 'no-store',
            'CDN-Cache-Control': 'no-store'
        }
    });
}
```

**当前版本代码**（✅ 最新）：
```typescript
function generateShareLink(
    protocol: string,
    address: string,
    port: number,
    index: number,
    isChain: boolean
): string {
    const {
        globalConfig: { userID, TrPass, hostName },
        dict: { _VL_, _TR_ }
    } = globalThis;

    const isTLS = isHttps(port);
    const { host, sni, allowInsecure } = selectSniHost(address);
    const wsPath = `${generateWsPath(protocol)}?ed=2560`;
    const encodedPath = encodeURIComponent(wsPath);
    const fingerprint = globalThis.settings.fingerprint || 'chrome';
    const remark = generateRemark(index, port, address, protocol, false, isChain);
    const encodedRemark = encodeURIComponent(remark);

    if (protocol === _VL_) {
        // VLESS link format
        const security = isTLS ? 'tls' : 'none';
        const params = new URLSearchParams({
            encryption: 'none',
            security: security,
            type: 'ws',
            host: host,
            path: encodedPath,
            fp: fingerprint
        });

        if (isTLS) {
            params.set('sni', sni);
            params.set('alpn', 'http/1.1');
            if (allowInsecure) {
                params.set('allowInsecure', '1');
            }
        }

        return `vless://${userID}@${address}:${port}?${params.toString()}#${encodedRemark}`;
    } else if (protocol === _TR_) {
        // Trojan link format - ✅ 已修复支持 non-TLS
        const params = new URLSearchParams({
            security: isTLS ? 'tls' : 'none',  // ✅ 动态设置
            type: 'ws',
            host: host,
            path: encodedPath,
            fp: fingerprint
        });

        if (isTLS) {
            params.set('sni', sni);
            params.set('alpn', 'http/1.1');
            if (allowInsecure) {
                params.set('insecure', '1');
                params.set('allowInsecure', '1');
            }
        }

        return `trojan://${TrPass}@${address}:${port}?${params.toString()}#${encodedRemark}`;
    }

    return '';
}

export async function getNodeOnlyLinks(): Promise<Response> {
    const { outProxy, ports } = globalThis.settings;
    const hasChain = !!outProxy;

    const addresses = await getConfigAddresses(false);
    const totalPorts = ports; // ✅ Include all ports (both TLS and non-TLS)
    const protocols = getProtocols();

    const links: string[] = [];
    let index = 1;

    for (const protocol of protocols) {
        let protocolIndex = 1;
        for (const port of totalPorts) {
            for (const addr of addresses) {
                const link = generateShareLink(protocol, addr, port, protocolIndex, false);
                if (link) {
                    links.push(link);
                }

                if (hasChain) {
                    const chainLink = generateShareLink(protocol, addr, port, protocolIndex, true);
                    if (chainLink) {
                        links.push(chainLink);
                    }
                }

                protocolIndex++;
                index++;
            }
        }
    }

    // ✅ Join links with newlines and encode to Base64 for better client compatibility
    const plainText = links.join('\n');
    const base64Content = btoa(plainText);

    return new Response(base64Content, {
        status: 200,
        headers: {
            'Content-Type': 'text/plain;charset=utf-8',
            'Cache-Control': 'no-store',
            'CDN-Cache-Control': 'no-store'
        }
    });
}
```

**目的**：
- `generateShareLink()`: 生成单个 VLESS 或 Trojan 分享链接
- `getNodeOnlyLinks()`: 遍历所有节点配置，生成分享链接列表，返回纯文本响应

---

### 2. `src/common/handlers.ts`

#### 修改位置 1：导入语句（第 6 行）
```typescript
import { getXrCustomConfigs, getXrWarpConfigs, getNodeOnlyLinks } from "@xray/configs";
```

**目的**：导入 NodeOnly 链接生成函数

---

#### 修改位置 2：handleSubscriptions 函数（第 144-152 行）
```typescript
export async function handleSubscriptions(request: Request, env: Env): Promise<Response> {
    await setSettings(request, env);
    const {
        globalConfig: { pathName },
        httpConfig: { client, subPath }
    } = globalThis;

    switch (pathName) {
        case `/sub/nodeonly/${subPath}`:
            return await getNodeOnlyLinks();  // ← 新增路由处理

        case `/sub/normal/${subPath}`:
            // ... 原有代码保持不变
```

**目的**：为 NodeOnly 订阅添加路由处理，当访问 `/sub/nodeonly/{subPath}` 时返回分享链接

---

### 3. `src/assets/panel/index.html`

#### 修改位置：Subscriptions 栏目开头（第 625-736 行之间）
在 Normal 订阅之前插入以下代码：

```html
<details>
    <summary>
        <h3>NodeOnly
            <a href="https://bia-pain-bache.github.io/BPB-Worker-Panel/usage/normal/" target="_blank"
                title="Help">
                <span class="material-symbols-rounded">info</span>
            </a>
        </h3>
    </summary>
    <div class="table-container">
        <table id="nodeonly-configs-table">
            <tr>
                <td>
                    <div>
                        <span class="material-symbols-rounded">verified</span>
                        <span>v2rayNG</span>
                    </div>
                    <div>
                        <span class="material-symbols-rounded">verified</span>
                        <span>MahsaNG</span>
                    </div>
                    <div>
                        <span class="material-symbols-rounded">verified</span>
                        <span>v2rayN</span>
                    </div>
                    <div>
                        <span class="material-symbols-rounded">verified</span>
                        <span>v2rayN-PRO</span>
                    </div>
                    <div>
                        <span class="material-symbols-rounded">verified</span>
                        <span>Streisand</span>
                    </div>
                </td>
                <td>
                    <button title="Display QR code"
                        onclick="openQR('nodeonly', 'xray', 'NodeOnly', 'NodeOnly Subscription')">
                        <span class="material-symbols-rounded">qr_code</span>
                    </button>
                    <button title="Copy subscription URL" onclick="subURL('nodeonly', 'xray', 'NodeOnly')">
                        <span class="material-symbols-rounded">content_copy</span>
                    </button>
                    <button title="Download config" onclick="dlURL('nodeonly', 'xray')">
                        <span class="material-symbols-rounded">download</span>
                    </button>
                </td>
            </tr>
        </table>
    </div>
</details>
```

**目的**：在面板 UI 中添加 NodeOnly 订阅栏目，包含二维码、复制、下载三个按钮

---

### 4. `src/assets/panel/script.js`

#### 修改位置 1：dlURL 函数（第 241-256 行）
```javascript
async function dlURL(path, app) {
    const url = generateSubUrl(path, app);

    try {
        const response = await fetch(url);
        const data = await response.text();

        if (!response.ok) {
            throw new Error(`status ${response.status} at ${response.url} - ${data}`);
        }

        // NodeOnly downloads as .txt file with share links, others as .json
        const fileName = path === 'nodeonly' ? 'nodeonly.txt' : 'config.json';
        downloadText(data, fileName);
    } catch (error) {
        console.error("Download error:", error.message || error);
    }
}
```

**目的**：区分 NodeOnly 和其他订阅的下载文件格式，NodeOnly 下载为 `.txt` 文件

---

#### 修改位置 2：添加 downloadText 函数（第 267 行之后）
```javascript
function downloadText(data, fileName) {
    const blob = new Blob([data], { type: 'text/plain;charset=utf-8' });
    const link = document.createElement('a');
    link.href = URL.createObjectURL(blob);
    link.download = fileName;
    document.body.appendChild(link);
    link.click();
    document.body.removeChild(link);
}
```

**目的**：提供纯文本文件下载功能

---

## 🎯 功能特性

### 用户操作流程

| 操作 | 结果 |
|------|------|
| **点击复制按钮** | 复制订阅 URL 到剪贴板，格式：`https://your-domain/sub/nodeonly/{subPath}#💦 BPB NodeOnly` |
| **扫描二维码** | 显示包含订阅 URL 的二维码，客户端扫描后自动导入 |
| **点击下载按钮** | 下载 `nodeonly.txt` 文件，内容为 BASE64 编码的分享链接（客户端可自动解码） |
| **直接访问订阅 URL** | 浏览器返回 BASE64 编码文本，每行一条分享链接的编码形式 |

### 分享链接格式示例

**VLESS (TLS 端口):**
```
vless://616848a4-bdbe-4263-85cc-5ea4ae9e86a0@188.114.97.3:443?encryption=none&security=tls&sni=example.com&type=ws&host=example.com&path=%2Fxxx%3Fed%3D2560&fp=chrome&alpn=http%2F1.1#%F0%9F%92%A6%201%20-%20VLESS%20-%20IPv4%20%3A%20443
```

**VLESS (non-TLS 端口):**
```
vless://616848a4-bdbe-4263-85cc-5ea4ae9e86a0@188.114.97.3:80?encryption=none&security=none&type=ws&host=example.com&path=%2Fxxx%3Fed%3D2560&fp=chrome#%F0%9F%92%A6%201%20-%20VLESS%20-%20IPv4%20%3A%2080
```

**Trojan (TLS 端口):**
```
trojan://password@host:443?security=tls&sni=example.com&type=ws&host=example.com&path=%2Fxxx%3Fed%3D2560&fp=chrome&alpn=http%2F1.1&insecure=1&allowInsecure=1#TR%20remark
```

**Trojan (non-TLS 端口):**
```
trojan://password@host:80?security=none&type=ws&host=example.com&path=%2Fxxx%3Fed%3D2560&fp=chrome#TR%20remark
```

**BASE64 编码后的订阅内容示例:**
```
dkxFU1M6Ly91dWlkQGhvc3Q6NDQzP3NlY3VyaXR5PXRscyYuLi4jcmVtYXJrCnRyb2phbjovL3Bhc3N3b3JkQGhvc3Q6ODA/c2VjdXJpdHk9bm9uZSYuLi4jcmVtYXJr
```

---

## 🔄 上游同步指南

### 潜在冲突文件

| 文件 | 冲突风险 | 说明 |
|------|----------|------|
| `src/cores/xray/configs.ts` | 🔴 高 | 添加了新函数，原作者可能修改相同区域 |
| `src/common/handlers.ts` | 🔴 高 | 添加了新路由，原作者可能修改 switch 语句 |
| `src/assets/panel/index.html` | 🟡 中 | 添加了新 UI 栏目，原作者可能调整布局 |
| `src/assets/panel/script.js` | 🟡 中 | 修改了下载函数，原作者可能优化相关功能 |

### 同步步骤

```bash
# 1. 添加上游远程仓库（首次操作）
git remote add upstream https://github.com/bia-pain-bache/BPB-Worker-Panel.git

# 2. 获取上游更新
git fetch upstream

# 3. 切换到主分支
git checkout main

# 4. 合并上游更新
git merge upstream/main

# 5. 如果有冲突，Git 会提示
# 使用编辑器解决冲突，查找 <<<<<<< 和 >>>>>>> 标记

# 6. 解决冲突后提交
git add <文件名>
git commit -m "Resolve merge conflicts with upstream"

# 7. 推送到你的 fork
git push origin main
```

### 冲突解决策略

#### 对于 `configs.ts`：
- **保留**：`generateShareLink()` 和 `getNodeOnlyLinks()` 函数
- **检查**：原作者是否修改了 import 语句或其他函数，如有需要手动合并

#### 对于 `handlers.ts`：
- **保留**：`case /sub/nodeonly/${subPath}:` 路由处理
- **检查**：原作者是否在 switch 中添加了新 case，确保语法正确

#### 对于 `index.html`：
- **保留**：NodeOnly 的 `<details>` 块
- **检查**：原作者是否修改了 Normal 或其他栏目的结构，保持 HTML 有效性

#### 对于 `script.js`：
- **保留**：`dlURL()` 中的 NodeOnly 判断逻辑和 `downloadText()` 函数
- **检查**：原作者是否重写了下载相关函数，必要时适配

---

## 📝 测试验证

### 构建命令
```bash
npm install
npm run build
```

### 验证点
1. ✅ 构建成功，无 TypeScript 错误
2. ✅ `dist/worker.js` 生成成功
3. ✅ 面板显示 NodeOnly 栏目
4. ✅ 复制按钮生成正确的订阅 URL
5. ✅ 下载按钮下载 `nodeonly.txt` 文件（BASE64 编码）
6. ✅ 访问订阅 URL 返回 BASE64 编码文本
7. ✅ 客户端可正常解码并导入配置
8. ✅ TLS 和 non-TLS 端口链接都正确生成

---

## 🚨 注意事项

1. **依赖关系**：此功能依赖于项目中已存在的工具函数，不需要额外安装 npm 包
2. **向后兼容**：所有修改都是新增功能，不影响原有的 Normal、Fragment 等订阅
3. **环境变量**：不需要新的环境变量，使用现有的 `userID`、`TrPass` 等配置
4. **协议支持**：当前仅支持 VLESS 和 Trojan，如需添加其他协议需修改 `generateShareLink()` 函数
5. **BASE64 编码**：订阅内容采用 BASE64 编码，提高客户端兼容性，大多数客户端支持自动解码

---

## 📞 维护联系

如遇到同步问题或需要进一步修改，请参考：
- 原始项目：https://github.com/bia-pain-bache/BPB-Worker-Panel
- 本文档最后更新：2026-03-11

## 📦 版本历史

| 版本 | 日期 | 更新内容 |
|------|------|----------|
| v1.0 | 2026-03-11 | 初始版本：实现 NodeOnly 订阅功能 |
| v1.1 | 2026-03-11 | 修复：支持 non-TLS 端口链接生成 |
| v1.2 | 2026-03-11 | 改进：Trojan 链接格式支持 non-TLS |
| v1.3 | 2026-03-11 | 改进：订阅内容改为 BASE64 编码，提高兼容性 |

---

**生成文件**：
- `dist/worker.js` (约 250 KB) - 混淆压缩后的 Worker 脚本
- `dist/worker.zip` (约 86 KB) - ZIP 压缩包版本
