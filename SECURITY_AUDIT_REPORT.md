# CKB Explorer Frontend — 安全审计报告

**日期：** 2026-03-12  
**范围：** 全代码库安全审查  
**审计依据：** [AI-Driven Security Audit Skill](https://github.com/gpBlockchain/ckb-test-skills/blob/main/.claude/skills/security-audit/SKILL.md)  
**参考文档：** [先前安全审计报告](https://github.com/sunchengzhu/ckb-explorer-frontend-next/blob/copilot/start-pr-review-process/SECURITY_AUDIT_REPORT.md)（2026-02-28）

---

## 1. 执行摘要

本报告是对 CKB Explorer Frontend（Next.js）代码库的安全审计结果。审计基于标准化的安全审计流程，涵盖输入验证、CSP 策略、认证授权、业务逻辑、依赖安全、序列化/反序列化、错误处理等多个维度。

**整体风险评级：中等**

该应用是一个只读的区块链浏览器，没有用户认证系统，攻击面相对有限。未发现关键的远程代码执行或认证绕过漏洞。但存在若干中高严重级别的问题需要关注。

### 项目概况
| 项目       | 详情                                |
|------------|-------------------------------------|
| 语言       | TypeScript (Next.js 15 / React 19)  |
| 项目类型   | Web 前端服务（区块链浏览器）         |
| 依赖数量   | ~70+ 生产依赖                        |
| 源文件数   | ~371 个 TypeScript/TSX 文件          |
| 现有测试数 | **0**（无任何测试文件）              |

---

## 2. develop 分支修复状态分析

> develop 分支提交 `84674c03`（2026-03-02，"chore: security issue fix"）及后续提交对先前审计报告中多项问题进行了修复。以下逐项对照分析：

### 2.1 已修复的问题

| 编号 | 严重级别 | 问题标题 | 修复状态 | 修复详情 |
|------|---------|---------|---------|---------|
| H-1 | 🔴 高危 | CSP 策略过于宽松 | ✅ **已修复** | `default-src` 改为 `'self'`；`unsafe-eval` 仅在开发模式启用；`connect-src` 限定为已知域名列表；外部服务 URL 配置化并动态拼接 CSP |
| H-2 | 🔴 高危 | `handleRedirectFromAggron` 开放重定向 | ✅ **已修复** | 添加了域名白名单验证（`allowedHosts` + `.nervos.org` 后缀检查） |
| M-1 | 🟡 中危 | URL 路径参数无输入编码 | ✅ **已修复** | `src/server/utils.ts` 中路径参数替换改用 `encodeURIComponent()` |
| M-2 | 🟡 中危 | `requestAPI` 缺少错误响应验证 | ✅ **已修复** | 添加了安全的错误处理（fallback 值）、响应结构校验、null 安全访问（`response.data?.data ?? null`） |
| M-3 | 🟡 中危 | `requestAPI` 中 `document` 在 SSR 上下文访问 + 拼写错误 | ✅ **已修复** | 添加了 `typeof document !== 'undefined'` 守卫；修正 `"Acccept-Language"` → `"Accept-Language"` |
| M-4 | 🟡 中危 | 外部服务 URL 硬编码 | ✅ **已修复** | `UtilityService` 改用 `env.NEXT_PUBLIC_UTILITY_ENDPOINT`；`spore.ts` 改用环境变量；`env.ts` 新增多个外部 URL 环境变量定义（含 Zod 验证） |
| M-6 | 🟡 中危 | Cookie 缺少 `Secure`/`SameSite` 标志 | ✅ **已修复** | Cookie 设置添加了 `;Secure;SameSite=Lax` |
| M-7 | 🟡 中危 | `decodeURIComponent(window.location.hash)` 可能抛异常 | ✅ **已修复** | 用 try-catch IIFE 包裹，解码失败回退到原始 hash |
| L-1 | 🟢 低危 | TypeScript 构建错误被忽略 | ✅ **已修复** | `ignoreBuildErrors: true` 被注释掉并加 TODO 标注 |
| L-4 | 🟢 低危 | `window.open` 缺少 `noopener noreferrer` | ✅ **已修复** | `RawTransactionView` 中改为 `"noopener,noreferrer"` |
| L-7 | 🟢 低危 | Spore `contentType` 未做白名单验证 | ✅ **已修复** | 添加了 `allowedImageTypes` 白名单（`image/png`, `image/jpeg` 等），双重验证 `startsWith("image")` + 白名单 |

### 2.2 未修复的问题

| 编号 | 严重级别 | 问题标题 | 状态 | 说明 |
|------|---------|---------|------|------|
| M-5 | 🟡 中危 | 外部 API 调用缺少响应结构验证 | ❌ **未修复** | `UtilityService`、`DidService` 等的 `fetch().json()` 调用仍未对响应结构进行运行时验证 |
| L-2 | 🟢 低危 | `Math.random()` 用于非加密场景 | ⚠️ **未处理** | 低优先级，用于 UI 随机化，风险可接受 |
| L-3 | 🟢 低危 | `PersistenceService` 无 localStorage 容量管理 | ⚠️ **未处理** | 低优先级 |
| L-5 | 🟢 低危 | `NodeService` RPC 无输入格式验证 | ⚠️ **未处理** | 低优先级 |
| L-6 | 🟢 低危 | IP 地址发送至第三方无验证 | ⚠️ **未处理** | 低优先级 |

### 2.3 关于 `img-src` CSP 策略的说明

develop 分支的 CSP 中 `img-src` 在生产环境设置为 `https: data:`，在开发环境设置为 `* data:`。这是**有意为之**的设计决定——因为区块链浏览器需要展示来自多个第三方域名的 NFT/DOB 图片（如 Spore NFT 图片、DOB 渲染图片等），如果进一步收紧 `img-src` 限制，页面上的第三方图片将无法正常显示。**此问题可以忽略**。

---

## 3. 当前代码库独立审计发现

> 以下为对当前分支（基于合并提交 `4107e40`）的独立安全审计发现。

### 3.1 🔴 高危发现

#### AUDIT-CSP-001: CSP 策略严重薄弱

**状态：** ❌ 发现漏洞  
**文件：** `next.config.ts:29-38`

**问题描述：**
```typescript
"default-src *",
"script-src 'self' 'unsafe-inline' 'unsafe-eval' https://vercel.live",
"connect-src *",
```

- `default-src *` 允许任意来源的资源加载
- `script-src` 中 `'unsafe-inline'` 和 `'unsafe-eval'` 严重削弱 XSS 防护
- `connect-src *` 允许前端向任意域名发起请求，若被 XSS 攻击利用可导致数据外泄
- 已注释掉的更严格的 `connect-src` 配置未被启用

**严重级别：** 高  
**影响范围：** 全站 XSS 防护  
**develop 分支修复状态：** ✅ 已修复

---

#### AUDIT-REDIRECT-001: `handleRedirectFromAggron` 开放重定向

**状态：** ❌ 发现漏洞  
**文件：** `src/utils/util.ts:192-205`

**问题描述：**
```typescript
const redirect = `${window.location.protocol}//${CURRENT_TESTNET}.${window.location.host}${...}`;
window.location.href = redirect;
```

重定向 URL 部分来自 `window.location.host`，攻击者可通过构造恶意链接（如配合 Host 头操纵）使用户跳转到恶意站点。

**严重级别：** 高  
**影响范围：** 可被利用进行钓鱼攻击  
**develop 分支修复状态：** ✅ 已修复

---

### 3.2 🟡 中危发现

#### AUDIT-INPUT-001: URL 路径参数未编码

**状态：** ⚠️ 建议改进  
**文件：** `src/server/utils.ts:99`

```typescript
queryUrl = queryUrl.replace(`{${key}}`, value as string);
```

路径参数直接插入 URL 字符串，未使用 `encodeURIComponent()` 编码。恶意构造的参数可能注入路径遍历或特殊字符。

**严重级别：** 中  
**develop 分支修复状态：** ✅ 已修复

---

#### AUDIT-REQUEST-001: `requestAPI` 多项安全问题

**状态：** ❌ 发现漏洞  
**文件：** `src/utils/request.ts:62-90`

**问题 1 — SSR 崩溃风险：**
```typescript
"Acccept-Language": document.documentElement.getAttribute("lang") === "zh" ? "zh_CN" : "en_US"
```
- `document` 在服务端渲染时不存在，将抛出 `ReferenceError`
- 请求头名称拼写错误：`"Acccept-Language"` 应为 `"Accept-Language"`

**问题 2 — Null 引用风险：**
```typescript
response.data = response.data.data  // response 可能为 null
```
- `response` 为 `null` 时（catch 块未正确设置或非预期错误形状）将抛出异常
- 未验证响应数据结构，假定特定的 API 响应格式

**问题 3 — 错误处理不健壮：**
```typescript
catch (e: any) {
  response = { data: { code: e.status, message: e.message, data: null } }
}
```
- `e.status` 和 `e.message` 可能不存在于所有错误类型

**严重级别：** 中  
**develop 分支修复状态：** ✅ 已修复（三个问题均已修复）

---

#### AUDIT-COOKIE-001: Cookie 缺少安全标志

**状态：** ⚠️ 建议改进  
**文件：** `src/utils/i18n.ts:68`

```typescript
document.cookie = `NEXT_LOCALE=${newLocale};expires=${date.toUTCString()};path=/`;
```

虽然是非敏感的语言偏好 Cookie，但缺少 `Secure` 和 `SameSite` 标志，可能在 HTTP 连接上泄露或被 CSRF 利用。

**严重级别：** 中  
**develop 分支修复状态：** ✅ 已修复

---

#### AUDIT-DECODE-001: `decodeURIComponent` 异常未捕获

**状态：** ⚠️ 建议改进  
**文件：** `src/app/(pages)/[locale]/scripts/page.client.tsx:22-24`

```typescript
const getHash = () => typeof window !== "undefined"
  ? decodeURIComponent(window.location.hash).slice(1) : "";
```

`window.location.hash` 是用户可控的。`decodeURIComponent` 对畸形 URI（如 `%ZZ`）会抛出 `URIError`，此处未用 try-catch 包裹。

**严重级别：** 中  
**develop 分支修复状态：** ✅ 已修复

---

#### AUDIT-EXTERNAL-001: 外部服务 URL 硬编码

**状态：** ⚠️ 建议改进

**涉及文件：**
- `src/services/UtilityService/index.ts:3` — `"https://ckb-utilities.random-walk.co.jp"` 硬编码
- `src/utils/spore.ts:21-24` — DOB 解码器 URL 硬编码
- `src/utils/spore.ts:28-29` — Omiga API URL 硬编码

若第三方域名被劫持或变更，需修改代码并重新部署才能应对。

**严重级别：** 中  
**develop 分支修复状态：** ✅ 已修复

---

#### AUDIT-RESPONSE-001: 外部 API 响应缺少结构验证

**状态：** ⚠️ 建议改进  
**涉及文件：** `src/services/UtilityService/index.ts`、`src/services/DidService/index.ts`、`src/utils/spore.ts`

多处 `fetch().json()` 调用未验证响应结构：
```typescript
const response = await fetch(`${UTILITY_ENDPOINT}/api/price`);
const data = await response.json();  // 未验证响应是否合法
return data;
```

若外部服务返回畸形 JSON 或非预期结构，应用可能崩溃或处理错误数据。

**严重级别：** 中  
**develop 分支修复状态：** ❌ 未修复

**修复建议：**
- 使用 Zod schema 对所有外部 API 响应进行运行时验证
- 将 `.json()` 调用包裹在 try-catch 中

---

### 3.3 🟢 低危发现

#### AUDIT-WINDOW-001: `window.open` 缺少安全参数

**状态：** ⚠️ 建议改进  
**文件：** `src/app/(pages)/[locale]/transaction/[txHash]/components/RawTransactionView/index.tsx:58`

```typescript
window.open(`/transaction/${select.value}`, "_blank");
```

缺少 `noopener noreferrer` 参数，打开的页面可通过 `window.opener` 访问原始页面，存在反向标签劫持（reverse tabnapping）风险。

**严重级别：** 低  
**develop 分支修复状态：** ✅ 已修复

---

#### AUDIT-BUILD-001: TypeScript 构建错误被忽略

**状态：** ⚠️ 建议改进  
**文件：** `next.config.ts:17`

```typescript
typescript: { ignoreBuildErrors: true },
```

TypeScript 类型检查错误在构建时被完全忽略，可能掩盖逻辑错误、null 安全问题。

**严重级别：** 低  
**develop 分支修复状态：** ✅ 已修复（已注释并加 TODO）

---

#### AUDIT-SPORE-001: Spore `contentType` 未做白名单验证

**状态：** ⚠️ 建议改进  
**文件：** `src/utils/spore.ts:138-141`

```typescript
if (contentType.startsWith("image")) {
  const base64Data = hexToBase64(content);
  return `data:${contentType};base64,${base64Data}`;
}
```

`contentType` 来自区块链上的链上数据（用户可控），虽有 `startsWith("image")` 检查，但恶意 contentType 可能包含非预期字符。

**严重级别：** 低  
**develop 分支修复状态：** ✅ 已修复（添加了 MIME 类型白名单）

---

#### AUDIT-RANDOM-001: `Math.random()` 非加密安全

**状态：** ℹ️ 信息  
**文件：** `src/utils/util.ts:458-460`

```typescript
export function randomInt(min: number, max: number) {
  return min + Math.floor(Math.random() * (max - min + 1));
}
```

`Math.random()` 不是密码学安全的随机数生成器。当前仅用于 UI 随机化，风险可接受。

**严重级别：** 低  
**develop 分支修复状态：** ⚠️ 未处理（可接受）

---

#### AUDIT-RPC-001: NodeService RPC 无输入格式验证

**状态：** ⚠️ 建议改进  
**文件：** `src/services/NodeService/index.ts:45-46`

```typescript
async getBlockEconomicState(blockHash: string) {
  return this.callRpc<...>('get_block_economic_state', [blockHash])
}
```

`blockHash` 参数未验证是否为合法的十六进制哈希格式。虽然 RPC 节点会拒绝无效输入，但客户端验证可减少无效网络请求。

**严重级别：** 低  
**develop 分支修复状态：** ⚠️ 未处理

---

#### AUDIT-STORAGE-001: localStorage 无容量管理

**状态：** ⚠️ 建议改进  
**文件：** `src/services/PersistenceService/index.ts`

`PersistenceService` 使用 `localStorage` 存储数据时无大小限制。大量响应可能填满 `localStorage` 导致 `QuotaExceededError`。

**严重级别：** 低  
**develop 分支修复状态：** ⚠️ 未处理

---

#### AUDIT-IP-001: IP 地址发送至第三方未验证

**状态：** ⚠️ 建议改进  
**文件：** `src/services/UtilityService/index.ts:21-33`

IP 地址数组直接发送至第三方服务，未验证格式且无用户告知。

**严重级别：** 低  
**develop 分支修复状态：** ⚠️ 未处理

---

### 3.4 ℹ️ 信息性说明

| 编号 | 说明 |
|------|------|
| I-1 | **无用户认证系统**：应用为只读区块链浏览器，认证代码已注释。若计划添加写操作需重新评估。 |
| I-2 | **环境变量验证良好**：使用 T3 env + Zod 验证，但 `SKIP_ENV_VALIDATION` 可绕过所有验证。 |
| I-3 | **无速率限制**：前端直接调用 API 无限流。虽然通常是后端关注点，但前端可实现请求节流防止滥用。 |
| I-4 | **安全头配置良好**：`X-Frame-Options: DENY`、`X-Content-Type-Options: nosniff`、`Strict-Transport-Security` 等均已正确配置。 |
| I-5 | **无 `dangerouslySetInnerHTML`**：代码库未使用 React 的 `dangerouslySetInnerHTML`，消除了常见的 XSS 向量。 |
| I-6 | **无 `eval()`/`Function()` 调用**：应用代码中未发现直接的 `eval()` 或 `new Function()` 调用。 |
| I-7 | **无 `innerHTML` 使用**：未发现直接的 `innerHTML` 操作。 |
| I-8 | **零测试覆盖**：整个项目无任何测试文件（unit/integration/e2e），这是一个显著的质量风险。 |
| I-9 | **ESLint 安全规则宽松**：多个 TypeScript 安全规则被关闭（`no-unsafe-member-access`、`no-explicit-any` 等），降低了静态分析的安全检测能力。 |
| I-10 | **Sentry 错误监控已禁用**：`src/utils/error.ts` 中 Sentry 集成被完全注释，生产环境缺少错误监控。 |

---

## 4. 风险评级总览

| 级别 | 数量 | 说明 |
|------|------|------|
| 🔴 Critical | 0 | 无关键漏洞 |
| 🔴 High | 2 | CSP 策略薄弱、开放重定向（develop 分支均已修复） |
| 🟡 Medium | 7 | URL 编码、请求安全、Cookie、外部 API 验证等 |
| 🟢 Low | 6 | window.open、构建配置、随机数、输入验证等 |
| ℹ️ Info | 10 | 信息性说明 |

---

## 5. develop 分支修复总结

develop 分支的安全修复提交 `84674c03`（2026-03-02）**有效修复了先前审计报告中绝大多数问题**：

### 修复统计
- **已修复：11 项**（H-1, H-2, M-1, M-2, M-3, M-4, M-6, M-7, L-1, L-4, L-7）
- **未修复：1 项**（M-5 外部 API 响应验证）
- **低优先级未处理：4 项**（L-2, L-3, L-5, L-6）
- **有意保留：1 项**（`img-src https: data:` —— 因第三方图片显示需求忽略）

### 修复质量评估
修复代码整体质量良好：
- CSP 策略修复彻底，动态构建 CSP 字符串避免硬编码
- 开放重定向修复采用白名单 + 域名后缀双重验证
- `requestAPI` 修复全面覆盖了 SSR 安全、null 安全、错误处理
- 环境变量配置化改造干净，保留了合理的默认值

### 建议后续修复
1. **M-5 外部 API 响应验证**：建议使用 Zod 为外部 API 响应添加运行时 schema 验证
2. **缺少 `Referrer-Policy` 头**：建议添加 `Referrer-Policy: strict-origin-when-cross-origin`
3. **缺少 `Permissions-Policy` 头**：建议添加以限制浏览器功能 API 的使用

---

## 6. 改进建议（非漏洞类）

| 优先级 | 建议 | 说明 |
|--------|------|------|
| 高 | 建立测试基础设施 | 当前零测试覆盖，建议添加关键路径的单元测试和集成测试 |
| 高 | 启用 TypeScript 严格构建 | 解决类型错误后移除 `ignoreBuildErrors` |
| 中 | 收紧 ESLint 规则 | 逐步启用 `@typescript-eslint/no-unsafe-*` 规则 |
| 中 | 添加请求超时机制 | 使用 `AbortController` 为所有 fetch 调用设置超时 |
| 中 | 恢复 Sentry 错误监控 | 重新启用生产环境错误追踪 |
| 低 | 实现 localStorage 缓存淘汰 | LRU 或 TTL 策略 |
| 低 | 添加安全响应头 | `Referrer-Policy`、`Permissions-Policy` |
| 低 | 依赖安全扫描自动化 | 集成 `npm audit` 到 CI/CD 流程 |

---

## 7. 依赖安全快查

| 包名 | 版本 | 风险说明 |
|------|------|---------|
| `next` | ^15.5.7 | 保持更新 —— 频繁发布安全补丁 |
| `axios` | ^1.11.0 | 服务端使用时注意 SSRF 风险 |
| `lodash` | ^4.17.21 | 原型污染（此版本已修复） |
| `react` | ^19.0.0 | 保持更新 |
| `zod` | ^3.24.2 | 安全，推荐用于运行时验证 |
| `i18next` | ^25.3.2 | 注意 XSS 防护配置 |

---

## 附录 A: 审计覆盖矩阵

| 审计维度 | 覆盖状态 | 发现数 |
|---------|---------|--------|
| DIM-INPUT: 输入验证 | ✅ 已审计 | 3 |
| DIM-AUTH: 认证与授权 | ✅ 已审计 | 0（无认证系统） |
| DIM-LOGIC: 业务逻辑 | ✅ 已审计 | 2 |
| DIM-DEPS: 依赖安全 | ✅ 已审计 | 0（无已知 CVE） |
| DIM-SERDE: 序列化/反序列化 | ✅ 已审计 | 1 |
| DIM-ERRINFO: 错误处理与信息泄露 | ✅ 已审计 | 2 |
| DIM-CRYPTO: 密码学操作 | N/A | 不涉及 |
| DIM-MEMORY: 内存安全 | N/A | 不涉及（JavaScript） |
| DIM-CONTRACT: 智能合约 | N/A | 不涉及（前端项目） |

---

## 附录 B: 修复建议优先级清单

### 关键优先级（当前分支）
- [ ] **CSP 加固**：替换 `default-src *`，移除 `unsafe-eval`，限制 `connect-src`（`next.config.ts`）
- [ ] **路径参数编码**：添加 `encodeURIComponent()` 到所有 URL 路径参数替换（`src/server/utils.ts`）
- [ ] **开放重定向防护**：添加域名白名单验证（`src/utils/util.ts`）

### 高优先级
- [ ] **`requestAPI` SSR 安全**：添加 `document` 访问守卫（`src/utils/request.ts`）
- [ ] **修正 `Accept-Language` 拼写**：`"Acccept-Language"` → `"Accept-Language"`（`src/utils/request.ts`）
- [ ] **`requestAPI` Null 安全**：添加响应链 null 检查（`src/utils/request.ts`）
- [ ] **外部 URL 配置化**：将硬编码 URL 迁移到环境变量（`UtilityService`、`spore.ts`）

### 中优先级
- [ ] **Cookie 安全标志**：添加 `Secure; SameSite=Lax`（`src/utils/i18n.ts`）
- [ ] **Hash 解码安全**：添加 try-catch 包裹 `decodeURIComponent`（`scripts/page.client.tsx`）
- [ ] **外部 API 响应验证**：使用 Zod 添加运行时验证
- [ ] **`window.open` 安全**：确保所有调用包含 `"noopener noreferrer"`
- [ ] **Spore contentType 验证**：白名单限制允许的 MIME 类型

### 低优先级
- [ ] **localStorage 缓存淘汰**：实现 LRU 或基于容量的淘汰策略
- [ ] **`Math.random()` 审查**：确认无安全敏感的调用方
- [ ] **依赖审计**：运行 `npm audit` 检查已知漏洞
- [ ] **RPC 输入验证**：验证 hash/address 格式

> **注意：** 以上大部分关键和高优先级问题已在 develop 分支中修复。建议尽快将 develop 分支合并到生产环境。

---

*报告结束*
