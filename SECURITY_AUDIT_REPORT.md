# CKB Explorer Frontend — Security Audit Report

**Date:** 2026-02-28  
**Scope:** Full codebase security review  
**Auditor:** Automated Security Audit Agent  

---

## Executive Summary

This report documents the findings of a security review of the CKB Explorer Frontend (Next.js) codebase. The review focused on common web security vulnerabilities including XSS, SSRF, open redirect, CSP weaknesses, sensitive data exposure, dependency risks, and insecure coding patterns.

**Overall Risk Level: Medium**

No critical remote code execution or authentication bypass vulnerabilities were found. The application is a read-only blockchain explorer with no user authentication, which limits the attack surface. However, several medium and low severity issues were identified that should be addressed.

---

## Findings

### 🔴 HIGH Severity

#### H-1: Weak Content Security Policy (CSP)

**File:** `next.config.ts` (lines 29-38)

```typescript
"script-src 'self' 'unsafe-inline' 'unsafe-eval' https://vercel.live",
"connect-src *",
"default-src *",
```

**Issue:**
- `'unsafe-inline'` and `'unsafe-eval'` in `script-src` significantly weaken XSS protection. An attacker who can inject content could execute arbitrary scripts.
- `connect-src *` allows the frontend to make requests to any domain, which expands the attack surface for data exfiltration if XSS is achieved.
- `default-src *` is overly permissive.
- There is a commented-out, more restrictive `connect-src` that limits connections to known domains — this restrictive version should be used instead.

**Recommendation:**
- Remove `'unsafe-eval'` unless absolutely necessary.
- Replace `connect-src *` with the commented-out restrictive version.
- Replace `default-src *` with `default-src 'self'`.
- Consider using nonces for inline scripts.

---

#### H-2: Potential Open Redirect in `handleRedirectFromAggron`

**File:** `src/utils/util.ts` (lines 192-205)

```typescript
export const handleRedirectFromAggron = () => {
  const redirect = `${window.location.protocol}//${CURRENT_TESTNET}.${window.location.host}${window.location.pathname.replace(testnetNameRegexp, "")}`;
  window.location.href = redirect;
};
```

**Issue:**
- The redirect URL is partially constructed from `window.location.host`, which can be manipulated by an attacker (e.g., via a crafted link with `Host` header manipulation on some setups). While the regex only matches specific pathname prefixes (`/aggron` or `/pudge`), the construct `CURRENT_TESTNET.${host}` could result in unexpected redirects if `host` contains unexpected values.

**Recommendation:**
- Validate the redirect URL against a whitelist of allowed domains before performing the redirect.

---

### 🟡 MEDIUM Severity

#### M-1: No Input Validation on URL Path Parameters

**File:** `src/server/explorer/defineAPI.ts` (line 99)

```typescript
queryUrl = queryUrl.replace(`{${key}}`, value as string);
```

**Issue:**
- Path parameters (e.g., `typeScriptHash`, `txHash`, `address`, `id`) are inserted directly into URL strings without sanitization or encoding. While these are sent to the backend API, a maliciously crafted path parameter could inject path traversal or special characters.
- No URL encoding with `encodeURIComponent()` is applied to path parameters.

**Recommendation:**
- Apply `encodeURIComponent()` to all path parameter values before URL substitution.

---

#### M-2: Missing Error Response Validation in `requestAPI`

**File:** `src/utils/request.ts` (lines 62-90)

```typescript
export async function requestAPI(url: string, config: RequestConfig) {
  let response = null
  try {
    response = await axios(`${url}`, { ... });
  } catch (e: any) {
    response = { data: { code: e.status, message: e.message, data: null } }
  }
  const bizDataOnly = config.getWholeBizData !== true
  if (bizDataOnly) response.data = response.data.data  // potential null access
  ...
}
```

**Issues:**
1. **Null dereference:** If `response` is `null` (catch block failed to set it, or an unexpected error shape), `response.data` will throw.
2. **Unvalidated response structure:** `response.data.data` assumes a specific API response shape. If the backend returns an unexpected format, this crashes silently.
3. **Error type casting:** `(e: any)` — `e.status` and `e.message` may not exist on all error types.

**Recommendation:**
- Add null/undefined checks before accessing nested properties.
- Validate the response structure before accessing `.data.data`.

---

#### M-3: `document` Access in Server-Side Context

**File:** `src/utils/request.ts` (line 68)

```typescript
"Acccept-Language": document.documentElement.getAttribute("lang") === "zh" ? "zh_CN" : "en_US"
```

**Issues:**
1. **SSR crash:** `document` is not available in Node.js/server-side context. If `requestAPI` is ever called during SSR, this will throw `ReferenceError: document is not defined`.
2. **Typo:** `"Acccept-Language"` has three 'c's — should be `"Accept-Language"`.

**Recommendation:**
- Add `typeof document !== 'undefined'` guard or use a request interceptor that only runs client-side.
- Fix the header name typo.

---

#### M-4: Hardcoded External Service URLs

**File:** `src/services/UtilityService/index.ts` (line 4)

```typescript
const UTILITY_ENDPOINT = "https://ckb-utilities.random-walk.co.jp";
```

**File:** `src/utils/spore.ts` (lines 21-29)

```typescript
config.setDobDecodeServerURL(
  isMainnet() ? "https://dob-decoder.rgbpp.io" : "https://dob0-decoder-dev.omiga.io"
);
```

**Issue:**
- Multiple external service URLs are hardcoded rather than configured via environment variables. If any of these third-party domains are compromised or change, the application cannot be reconfigured without a code change and redeployment.
- These endpoints receive user-initiated data (IP addresses, addresses, token IDs).

**Recommendation:**
- Move all external service URLs to environment variables with validation in `env.ts`.

---

#### M-5: Missing Response Validation for External API Calls

**Files:** `src/services/UtilityService/index.ts`, `src/services/DidService/index.ts`, `src/services/NodeProbService/index.ts`, `src/utils/spore.ts`

**Issue:**
- Multiple `fetch()` calls parse the response with `.json()` without validating the response structure. If an external service returns malformed JSON or an unexpected shape, the application could crash or process incorrect data.
- Example from UtilityService:
  ```typescript
  const response = await fetch(`${UTILITY_ENDPOINT}/api/price`);
  const data = await response.json();
  return data;
  ```

**Recommendation:**
- Add runtime validation (e.g., Zod schemas) for all external API responses.
- Wrap `.json()` calls in try-catch blocks.

---

#### M-6: Cookie Set Without `Secure` or `SameSite` Flags

**File:** `src/utils/i18n.ts` (line 68)

```typescript
document.cookie = `NEXT_LOCALE=${newLocale};expires=${date.toUTCString()};path=/`;
```

**Issue:**
- The `NEXT_LOCALE` cookie is set without `Secure`, `SameSite`, or `HttpOnly` flags.
- While this is a non-sensitive locale cookie, it's best practice to set `Secure` and `SameSite=Lax` to prevent CSRF and leakage over HTTP.

**Recommendation:**
- Add `Secure` and `SameSite=Lax` flags:
  ```typescript
  document.cookie = `NEXT_LOCALE=${newLocale};expires=${date.toUTCString()};path=/;Secure;SameSite=Lax`;
  ```

---

#### M-7: User-Controlled Input in `decodeURIComponent(window.location.hash)`

**File:** `src/app/(pages)/[locale]/scripts/page.client.tsx` (line 23)

```typescript
const getHash = () => typeof window !== "undefined"
  ? decodeURIComponent(window.location.hash).slice(1) : "";
```

**Issue:**
- `window.location.hash` is user-controlled. While `decodeURIComponent` can throw on malformed URIs (e.g., `%ZZ`), this is not wrapped in try-catch.
- The decoded value is used to look up database entries and manipulate DOM (scrolling to elements), which could lead to unexpected behavior.

**Recommendation:**
- Wrap `decodeURIComponent` in try-catch.
- Validate/sanitize the hash value before using it as a lookup key.

---

### 🟢 LOW Severity

#### L-1: TypeScript Build Errors Ignored

**File:** `next.config.ts` (line 18)

```typescript
typescript: { ignoreBuildErrors: true },
```

**Issue:**
- TypeScript type-checking errors are completely ignored during builds. This means type errors that could indicate logic bugs, null safety issues, or incorrect data handling will not block deployment.

**Recommendation:**
- Remove `ignoreBuildErrors: true` and fix all TypeScript errors before deploying.

---

#### L-2: `Math.random()` Used for Cryptographic-Adjacent Purpose

**File:** `src/utils/util.ts` (line 459)

```typescript
export function randomInt(min: number, max: number) {
  return min + Math.floor(Math.random() * (max - min + 1));
}
```

**Issue:**
- `Math.random()` is not cryptographically secure. While this function appears to be used for non-security-critical purposes (UI), it should not be used if any caller needs unpredictable values.

**Recommendation:**
- If used for UI randomness only, document this. If any security-sensitive use exists, switch to `crypto.getRandomValues()`.

---

#### L-3: PersistenceService Uses `localStorage` Without Size Management

**File:** `src/services/PersistenceService/index.ts`

**Issue:**
- The PersistenceService stores data in `localStorage` via `JSON.stringify` without size limits. Large responses could fill `localStorage`, causing `QuotaExceededError` and potentially affecting other applications on the same domain.
- The CacheService TODO at line 88 acknowledges this: `// TODO: When there is not enough space, automatically remove old cache according to some rules.`

**Recommendation:**
- Implement LRU or TTL-based cache eviction.
- Add error handling for `QuotaExceededError`.

---

#### L-4: `window.open` Without Full `noopener noreferrer`

**File:** `src/app/(pages)/[locale]/transaction/[txHash]/components/RawTransactionView/index.tsx` (line 58)

```typescript
window.open(`/transaction/${select.value}`, "_blank");
```

**Issue:**
- `window.open` without `noopener` allows the opened page to access `window.opener`, which can be exploited for reverse tabnapping.
- Other instances in the codebase correctly use `"noopener noreferrer"` but this one does not.

**Recommendation:**
- Always pass `"noopener noreferrer"` as the third argument.

---

#### L-5: `NodeService` RPC Without Input Validation

**File:** `src/services/NodeService/index.ts`

```typescript
async getBlockEconomicState(blockHash: string) {
  return this.callRpc<CKBComponents.BlockEconomicState>('get_block_economic_state', [blockHash])
}
```

**Issue:**
- The `blockHash` parameter is passed directly to the CKB RPC node without validating it's a valid hex hash. While the RPC node will reject invalid inputs, validating on the client side prevents unnecessary network requests and provides better error messages.

**Recommendation:**
- Validate hash format (e.g., `0x[0-9a-f]{64}`) before making RPC calls.

---

#### L-6: `UtilityService.fetchIpsInfo` — IP Addresses Sent to Third-Party

**File:** `src/services/UtilityService/index.ts` (lines 21-33)

```typescript
export const fetchIpsInfo = async (ips: string[]) => {
  const data = await fetch(`${UTILITY_ENDPOINT}/api/ips`, {
    method: "POST",
    body: JSON.stringify({ ips }),
  }).then((res) => res.json());
  return data;
};
```

**Issue:**
- User IP addresses (likely CKB node peer IPs) are sent to a third-party service without encryption notice or user consent.
- No validation that the IPs array doesn't contain malicious data.

**Recommendation:**
- Validate IP format before sending.
- Document/disclose the data sharing to users.

---

#### L-7: `Spore.getSporeImg` — Potential Data URI Injection

**File:** `src/utils/spore.ts` (lines 138-141)

```typescript
if (contentType.startsWith("image")) {
  const base64Data = hexToBase64(content);
  return `data:${contentType};base64,${base64Data}`;
}
```

**Issue:**
- The `contentType` is extracted from blockchain cell data (user-controlled on-chain data). A malicious contentType could potentially be crafted to include unexpected characters, though the `startsWith("image")` check limits this.

**Recommendation:**
- Validate `contentType` against a whitelist of allowed image MIME types (e.g., `image/png`, `image/jpeg`, `image/svg+xml`, `image/gif`).

---

## Informational Notes

### I-1: No Authentication or Authorization
The application has no user authentication system (commented out in `request.ts`). This is expected for a read-only blockchain explorer, but should be noted if any write operations are planned.

### I-2: Environment Variable Validation
The `env.ts` uses T3 env with Zod validation, which is a good practice. However, `SKIP_ENV_VALIDATION` can bypass all validation during builds.

### I-3: No Rate Limiting
The frontend makes direct API calls without rate limiting. While rate limiting is typically a backend concern, the frontend could implement request throttling to prevent abuse.

### I-4: Security Headers Present
Good security headers are configured: `X-Frame-Options: DENY`, `X-Content-Type-Options: nosniff`, `Strict-Transport-Security`, `X-Powered-By: ""`. These are well-configured.

### I-5: No `dangerouslySetInnerHTML` Usage
The codebase does not use React's `dangerouslySetInnerHTML`, which eliminates a common XSS vector. This is good practice.

### I-6: No Direct `eval()` or `Function()` Usage  
No direct `eval()` or `new Function()` calls were found in the application code (only in dependencies).

---

## TODO — Areas Requiring Further Review

### Critical Priority
- [ ] **CSP hardening**: Replace `default-src *`, remove `unsafe-eval`, restrict `connect-src` to known domains (next.config.ts)
- [ ] **Path parameter encoding**: Add `encodeURIComponent()` to all URL path parameter substitutions (src/server/explorer/defineAPI.ts)
- [ ] **Open redirect validation**: Add domain whitelist check in `handleRedirectFromAggron` (src/utils/util.ts)

### High Priority
- [ ] **SSR safety in `requestAPI`**: Guard `document` access for server-side rendering (src/utils/request.ts)
- [ ] **Fix `Accept-Language` header typo**: `"Acccept-Language"` → `"Accept-Language"` (src/utils/request.ts)
- [ ] **Null safety in `requestAPI`**: Add null checks for response chain (src/utils/request.ts)
- [ ] **TypeScript build errors**: Remove `ignoreBuildErrors: true` and fix type errors (next.config.ts)
- [ ] **External URL configuration**: Move hardcoded URLs to environment variables (UtilityService, spore.ts)

### Medium Priority
- [ ] **Cookie security flags**: Add `Secure; SameSite=Lax` to locale cookie (src/utils/i18n.ts)
- [ ] **Hash decoding safety**: Add try-catch around `decodeURIComponent(window.location.hash)` (scripts/page.client.tsx)
- [ ] **External API response validation**: Add Zod or similar runtime validation for all external API responses
- [ ] **`window.open` security**: Ensure all `window.open` calls include `"noopener noreferrer"` (RawTransactionView)
- [ ] **RPC input validation**: Validate hash/address format before making CKB RPC calls (NodeService)
- [ ] **Spore contentType validation**: Whitelist allowed MIME types for data URI construction (spore.ts)
- [ ] **IP data validation**: Validate IP format in `fetchIpsInfo` before sending to third-party (UtilityService)

### Low Priority
- [ ] **localStorage cache eviction**: Implement LRU or size-based eviction in CacheService/PersistenceService
- [ ] **`Math.random()` audit**: Verify no security-sensitive callers of `randomInt` function
- [ ] **Dependency audit**: Run `npm audit` to check for known vulnerabilities in dependencies
- [ ] **Error boundary completeness**: Review all page-level error boundaries for graceful degradation
- [ ] **Sentry re-enablement**: Error tracking is completely commented out (src/utils/error.ts) — consider re-enabling for production monitoring

### Further Investigation Needed
- [ ] Review all dynamic route parameters (`[locale]`, `[txHash]`, `[blockIndex]`, `[udtTypeHash]`, `[collectionId]`) for injection risks
- [ ] Audit the `ckb-labels` git submodule for supply chain risks
- [ ] Review the `plugins/i18n-resource` webpack plugin for path traversal issues
- [ ] Analyze the `@ckb-ccc/ccc` and `@nervosnetwork/ckb-sdk-*` dependencies for known vulnerabilities
- [ ] Review all `useQuery` and `useMutation` hooks for proper error handling and data validation
- [ ] Audit the `database/` directory for any hardcoded sensitive data in known scripts/UDT data files

---

## Dependency Quick Check

Key dependencies to monitor for security advisories:

| Package | Version | Risk Notes |
|---------|---------|------------|
| `next` | ^15.5.7 | Keep updated — frequent security patches |
| `axios` | ^1.11.0 | SSRF risks in server-side usage |
| `lodash` | ^4.17.21 | Prototype pollution (fixed in this version) |
| `react` | ^19.0.0 | Keep updated |
| `webpack` | ^5.101.0 | Supply chain risk — keep updated |

---

*End of Report*
