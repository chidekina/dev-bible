# Mobile Security

## Overview

Mobile apps face a distinct threat surface that web apps do not. Sensitive data stored as plaintext on-device, network traffic intercepted by a local proxy, binaries reverse-engineered to extract API keys, and stolen devices granting access to authenticated sessions — each of these is a real attack vector, not a theoretical one. A single misconfiguration can expose thousands of users: a token left in AsyncStorage, an HTTP endpoint in production, or a hardcoded secret in the JS bundle that any attacker can extract in minutes.

This chapter covers the OWASP Mobile Top 10 (2024 edition), hardware-backed secure storage via iOS Keychain and Android Keystore, certificate pinning to block MITM attacks, biometric authentication gating, jailbreak/root detection, and build-time code hardening. These are not optional hardening measures — they are the minimum baseline for any app that handles authentication or personal data.

The security model for mobile differs from web in one critical way: the client is untrusted by default. Users run your app on rooted devices, intercept your API calls, and patch your binary. Design your server-side authorization as if the client can be fully compromised. Everything in this chapter slows attackers down — your server-side controls stop them.

---

## Prerequisites

- [React Native Basics](./react-native-basics.md)
- Track 07 — Security (OWASP Top 10 for web — mobile shares many concepts)
- Familiarity with async/await and TypeScript generics

---

## Core Concepts

### 1. Secure Storage — expo-secure-store

`AsyncStorage` is a key-value store that writes to plaintext files on disk. On a rooted Android device, `adb pull /data/data/com.yourapp/databases/` retrieves those files with no authentication required. On iOS, a non-encrypted backup exposes them. Never store secrets, tokens, or PII in AsyncStorage.

`expo-secure-store` wraps the platform's hardware-backed secure storage:

- **iOS**: Keychain Services — encrypted by the device passcode. On devices with a Secure Enclave (A7 chip and later, including all modern iPhones), the encryption key never leaves the secure hardware.
- **Android 6+ (API 23+)**: Android Keystore — keys are generated inside and never extracted from the TEE (Trusted Execution Environment) or Secure Element.

```bash
npx expo install expo-secure-store
```

```typescript
// lib/token-store.ts
import * as SecureStore from 'expo-secure-store';

const TOKEN_KEY = 'auth_token';
const REFRESH_KEY = 'refresh_token';

// keychainAccessible controls WHEN the item can be read.
// WHEN_UNLOCKED_THIS_DEVICE_ONLY:
//   - readable only while the device is unlocked
//   - not migrated to new devices via backups
//   - correct default for auth tokens
const ACCESS_OPTION: SecureStore.SecureStoreOptions = {
  keychainAccessible: SecureStore.WHEN_UNLOCKED_THIS_DEVICE_ONLY,
};

export async function saveTokens(access: string, refresh: string): Promise<void> {
  await Promise.all([
    SecureStore.setItemAsync(TOKEN_KEY, access, ACCESS_OPTION),
    SecureStore.setItemAsync(REFRESH_KEY, refresh, ACCESS_OPTION),
  ]);
}

export async function getAccessToken(): Promise<string | null> {
  return SecureStore.getItemAsync(TOKEN_KEY, ACCESS_OPTION);
}

export async function getRefreshToken(): Promise<string | null> {
  return SecureStore.getItemAsync(REFRESH_KEY, ACCESS_OPTION);
}

export async function clearTokens(): Promise<void> {
  await Promise.all([
    SecureStore.deleteItemAsync(TOKEN_KEY),
    SecureStore.deleteItemAsync(REFRESH_KEY),
  ]);
}
```

**The four accessibility levels** and when to use each:

| Option | Readable when | Migrated to new device | Use case |
|---|---|---|---|
| `AFTER_FIRST_UNLOCK` | After first unlock post-reboot | Yes (iOS iCloud backup) | Background refresh tokens |
| `AFTER_FIRST_UNLOCK_THIS_DEVICE_ONLY` | After first unlock post-reboot | No | Background tokens, no cloud backup |
| `WHEN_UNLOCKED` | Device is unlocked | Yes | Active-use credentials |
| `WHEN_UNLOCKED_THIS_DEVICE_ONLY` | Device is unlocked | No | Auth tokens (**recommended default**) |

```typescript
// ❌ Never store secrets here — plaintext on disk
import AsyncStorage from '@react-native-async-storage/async-storage';
await AsyncStorage.setItem('auth_token', token);
// Attacker with adb root: adb pull /data/data/com.app/files/RCTAsyncLocalStorage_V1/

// ✅ Hardware-backed encryption
import * as SecureStore from 'expo-secure-store';
await SecureStore.setItemAsync('auth_token', token, {
  keychainAccessible: SecureStore.WHEN_UNLOCKED_THIS_DEVICE_ONLY,
});
```

**Size limit**: SecureStore values are capped at 2 KB. For larger encrypted payloads (e.g., a user profile object), encrypt the data with a symmetric key, store the key in SecureStore, and write the ciphertext to the filesystem or AsyncStorage.

---

### 2. Certificate Pinning

TLS protects traffic from passive eavesdropping but not from an attacker who can install a trusted CA on the device — which is trivially done on any device the attacker controls. Tools like Proxyman, Charles Proxy, and Burp Suite work exactly this way: they install their own CA certificate and perform a MITM attack against your app.

Certificate pinning makes the app reject connections unless the server's certificate (or its public key) matches a value bundled at build time. Even with a trusted MITM CA installed, the attacker cannot forge your server's pinned key.

```bash
npx expo install react-native-ssl-pinning
```

First, extract your server's public key hash:

```bash
# Extract the public key hash (SPKI SHA-256) for your API host
openssl s_client -connect api.yourapp.com:443 < /dev/null 2>/dev/null \
  | openssl x509 -pubkey -noout \
  | openssl pkey -pubin -outform DER \
  | openssl dgst -sha256 -binary \
  | base64
```

Bundle the certificate file in your project (`assets/certs/api_cert.cer`) and configure pinning:

```typescript
// lib/pinned-fetch.ts
import { fetch as pinnedFetch } from 'react-native-ssl-pinning';

const API_BASE = 'https://api.yourapp.com';

interface PinnedRequestOptions {
  method?: 'GET' | 'POST' | 'PUT' | 'PATCH' | 'DELETE';
  headers?: Record<string, string>;
  body?: string;
}

export async function secureFetch(
  path: string,
  options: PinnedRequestOptions = {}
): Promise<Response> {
  // Disable pinning in dev to allow Proxyman/Charles during development
  if (__DEV__) {
    return fetch(`${API_BASE}${path}`, options);
  }

  return pinnedFetch(`${API_BASE}${path}`, {
    method: options.method ?? 'GET',
    sslPinning: {
      certs: ['api_cert'], // filename without extension, from assets/certs/
    },
    headers: {
      'Content-Type': 'application/json',
      ...options.headers,
    },
    body: options.body,
  });
}
```

**Certificate rotation**: When your TLS certificate expires or is replaced, your pinned app will reject connections. Mitigate this by:

1. Pinning to the intermediate CA certificate (longer-lived than leaf certs)
2. Bundling a backup pin (the upcoming certificate's public key hash)
3. Implementing a remote config endpoint (over a non-pinned channel) that can update pins via OTA

```typescript
// Always pin to at least two certificates: current + upcoming
sslPinning: {
  certs: ['api_cert_current', 'api_cert_backup'],
}
```

**Important**: `react-native-ssl-pinning` requires a bare workflow or `expo-dev-client`. It does not work in Expo Go because Expo Go bundles its own networking stack.

---

### 3. Biometric Authentication

Biometric authentication (Face ID, Touch ID, fingerprint) adds a second factor without password friction. The critical pattern is using biometrics as a gate to unlock a SecureStore-held token — not as the primary authentication mechanism.

```bash
npx expo install expo-local-authentication
```

```typescript
// lib/biometrics.ts
import * as LocalAuthentication from 'expo-local-authentication';

export type BiometricResult =
  | { success: true }
  | { success: false; reason: 'not_available' | 'not_enrolled' | 'cancelled' | 'failed' | 'lockout' };

export async function checkBiometricSupport(): Promise<{
  available: boolean;
  enrolled: boolean;
  types: LocalAuthentication.AuthenticationType[];
}> {
  const [hasHardware, isEnrolled, types] = await Promise.all([
    LocalAuthentication.hasHardwareAsync(),
    LocalAuthentication.isEnrolledAsync(),
    LocalAuthentication.supportedAuthenticationTypesAsync(),
  ]);

  return { available: hasHardware, enrolled: isEnrolled, types };
}

export async function authenticateWithBiometrics(
  promptMessage = 'Authenticate to continue'
): Promise<BiometricResult> {
  const { available, enrolled } = await checkBiometricSupport();

  if (!available) return { success: false, reason: 'not_available' };
  if (!enrolled) return { success: false, reason: 'not_enrolled' };

  const result = await LocalAuthentication.authenticateAsync({
    promptMessage,
    fallbackLabel: 'Use Passcode', // shown on iOS when biometrics fail
    disableDeviceFallback: false,  // allow passcode as fallback
    cancelLabel: 'Cancel',
  });

  if (result.success) return { success: true };

  if (result.error === 'user_cancel') return { success: false, reason: 'cancelled' };
  if (result.error === 'lockout' || result.error === 'lockout_permanent') {
    return { success: false, reason: 'lockout' };
  }

  return { success: false, reason: 'failed' };
}
```

Gate token retrieval behind biometric auth:

```typescript
// hooks/useSecureToken.ts
import { useState, useCallback } from 'react';
import * as SecureStore from 'expo-secure-store';
import { authenticateWithBiometrics } from '../lib/biometrics';

export function useSecureToken() {
  const [token, setToken] = useState<string | null>(null);
  const [authState, setAuthState] = useState<'idle' | 'authenticating' | 'authenticated' | 'failed'>('idle');

  const unlockToken = useCallback(async () => {
    setAuthState('authenticating');

    const biometricResult = await authenticateWithBiometrics('Authenticate to access your account');

    if (!biometricResult.success) {
      setAuthState('failed');
      return null;
    }

    // Only read the token AFTER biometric success
    const storedToken = await SecureStore.getItemAsync('auth_token');
    setToken(storedToken);
    setAuthState('authenticated');
    return storedToken;
  }, []);

  const clearToken = useCallback(() => {
    setToken(null);
    setAuthState('idle');
  }, []);

  return { token, authState, unlockToken, clearToken };
}
```

**Biometric type detection** — show the correct icon in your UI:

```typescript
import * as LocalAuthentication from 'expo-local-authentication';

export async function getBiometricLabel(): Promise<string> {
  const types = await LocalAuthentication.supportedAuthenticationTypesAsync();

  if (types.includes(LocalAuthentication.AuthenticationType.FACIAL_RECOGNITION)) {
    return 'Face ID'; // iOS Face ID or Android face unlock
  }
  if (types.includes(LocalAuthentication.AuthenticationType.FINGERPRINT)) {
    return 'Touch ID'; // iOS Touch ID or Android fingerprint
  }
  if (types.includes(LocalAuthentication.AuthenticationType.IRIS)) {
    return 'Iris scan';
  }
  return 'Biometrics';
}
```

---

### 4. Jailbreak and Root Detection

A jailbroken iOS device or rooted Android device bypasses the OS security model: any app can read any other app's files, debuggers can attach to processes, and system call hooks can intercept crypto operations. These capabilities are used by attackers to extract tokens from memory, dump SecureStore contents, and bypass certificate pinning.

```bash
npx expo install react-native-jail-monkey
```

```typescript
// lib/device-integrity.ts
import JailMonkey from 'react-native-jail-monkey';
import { Platform } from 'react-native';

export type IntegrityReason =
  | 'jailbreak'
  | 'root'
  | 'hook_detected'
  | 'external_storage'
  | 'debugger_attached'
  | 'emulator';

export interface IntegrityResult {
  safe: boolean;
  reasons: IntegrityReason[];
}

export function checkDeviceIntegrity(): IntegrityResult {
  const reasons: IntegrityReason[] = [];

  if (JailMonkey.isJailBroken()) {
    reasons.push(Platform.OS === 'ios' ? 'jailbreak' : 'root');
  }

  if (JailMonkey.hookDetected()) {
    // Detects Cydia Substrate, Frida, and similar hooking frameworks
    reasons.push('hook_detected');
  }

  if (Platform.OS === 'android' && JailMonkey.isOnExternalStorage()) {
    // App installed on external storage — weaker isolation guarantees
    reasons.push('external_storage');
  }

  if (!__DEV__ && JailMonkey.isDebuggedMode()) {
    // Debugger attached in production — suspicious
    reasons.push('debugger_attached');
  }

  // Optionally detect emulators (will trigger false positives in CI)
  // if (JailMonkey.isEmulator()) reasons.push('emulator');

  return { safe: reasons.length === 0, reasons };
}
```

What to do when integrity checks fail:

```typescript
// In your app's root initialization
import { checkDeviceIntegrity } from './lib/device-integrity';
import { reportIntegrityViolation } from './lib/api';

export async function initializeSecurity(): Promise<void> {
  const integrity = checkDeviceIntegrity();

  if (!integrity.safe) {
    // 1. Report to backend for fraud/anomaly detection
    await reportIntegrityViolation(integrity.reasons).catch(() => {
      // Fire-and-forget — don't block the user flow on network failure
    });

    // 2. Decide response based on your risk model:
    // Option A (high-security apps): hard block
    // Alert.alert('Device Not Supported', 'This app cannot run on modified devices.');
    // return;

    // Option B (most apps): disable sensitive features, warn user
    console.warn('Device integrity check failed:', integrity.reasons);
    // Disable: payment flows, biometric unlock, document upload, etc.
  }
}
```

**False positives**: Some Android OEM devices (Samsung Knox, some Xiaomi) trigger `isJailBroken()` incorrectly. Emulators used in CI will also trigger detections. Strategy: use `isJailBroken()` as a signal to increase backend scrutiny (risk scoring), not as a hard block, unless you are building a banking or fintech app where the business requirement mandates it.

---

### 5. Code Obfuscation

Hermes compiles JavaScript to bytecode at build time, which is not human-readable. However, tools like `hermes-dec` can decompile Hermes bytecode back to readable JavaScript. Obfuscation adds another layer of friction.

```bash
# Add JavaScript-level obfuscation
npm install --save-dev react-native-obfuscating-transformer
```

```javascript
// babel.config.js
module.exports = {
  presets: ['babel-preset-expo'],
  plugins: [
    ...(process.env.NODE_ENV === 'production'
      ? [
          [
            'react-native-obfuscating-transformer',
            {
              compact: true,
              controlFlowFlattening: true,        // flattens control flow graph
              controlFlowFlatteningThreshold: 0.75,
              deadCodeInjection: true,            // inserts fake code paths
              deadCodeInjectionThreshold: 0.4,
              stringEncryption: true,             // encrypts string literals
              stringEncryptionThreshold: 0.75,
              identifierNamesGenerator: 'hexadecimal',
              transformObjectKeys: true,
            },
          ],
        ]
      : []),
  ],
};
```

Android additionally runs ProGuard/R8 on the Java/Kotlin wrapper layer. Expo production builds enable this by default. For bare workflow:

```groovy
// android/app/build.gradle
buildTypes {
  release {
    minifyEnabled true      // R8 minification + dead code elimination
    shrinkResources true    // removes unused resources
    proguardFiles getDefaultProguardFile('proguard-android-optimize.txt'), 'proguard-rules.pro'
  }
}
```

**The critical caveat**: obfuscation slows down reverse engineering — it does not prevent it. A determined attacker with Frida and enough time will reconstruct your logic. The only secure solution for secrets is to keep them on the server. If a secret must exist on the client, accept that it will eventually be extracted.

```typescript
// ❌ No amount of obfuscation makes this safe
const API_KEY = 'sk-live-abc123def456'; // Will be extracted

// ✅ The correct architecture: proxy sensitive calls through your backend
// Client calls: POST /api/ai/complete
// Server holds the OpenAI key, makes the call, returns the result
```

---

### 6. Secure Networking

**iOS — App Transport Security (ATS)**

ATS is enforced by default on iOS. All HTTP connections are blocked at the OS level unless explicitly whitelisted in `Info.plist`. Expo managed workflow configures ATS automatically in production builds.

```json
// app.json — explicit ATS enforcement (also the default)
{
  "expo": {
    "ios": {
      "infoPlist": {
        "NSAppTransportSecurity": {
          "NSAllowsArbitraryLoads": false
        }
      }
    }
  }
}
```

**Android — cleartext traffic**

Android allows HTTP by default. Block it explicitly:

```json
// app.json
{
  "expo": {
    "android": {
      "usesCleartextTraffic": false
    }
  }
}
```

**Attaching auth headers from SecureStore**

```typescript
// lib/api.ts
import * as SecureStore from 'expo-secure-store';
import { getRefreshToken, saveTokens, clearTokens } from './token-store';

const API_BASE = process.env.EXPO_PUBLIC_API_URL ?? 'https://api.yourapp.com';

interface ApiResponse<T> {
  data: T | null;
  error: string | null;
  status: number;
}

async function apiFetch<T>(
  path: string,
  options: RequestInit = {}
): Promise<ApiResponse<T>> {
  const token = await SecureStore.getItemAsync('auth_token');

  const response = await fetch(`${API_BASE}${path}`, {
    ...options,
    headers: {
      'Content-Type': 'application/json',
      ...options.headers,
      ...(token ? { Authorization: `Bearer ${token}` } : {}),
    },
  });

  // Handle token refresh on 401
  if (response.status === 401) {
    const refreshed = await attemptTokenRefresh();
    if (refreshed) {
      // Retry once with the new token
      return apiFetch<T>(path, options);
    }
    await clearTokens();
    return { data: null, error: 'Session expired', status: 401 };
  }

  if (!response.ok) {
    const body = await response.text();
    return { data: null, error: body, status: response.status };
  }

  const data = (await response.json()) as T;
  return { data, error: null, status: response.status };
}

async function attemptTokenRefresh(): Promise<boolean> {
  const refreshToken = await getRefreshToken();
  if (!refreshToken) return false;

  try {
    const response = await fetch(`${API_BASE}/auth/refresh`, {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify({ refreshToken }),
    });

    if (!response.ok) return false;

    const { accessToken, refreshToken: newRefreshToken } = await response.json();
    await saveTokens(accessToken, newRefreshToken);
    return true;
  } catch {
    return false;
  }
}

export const api = {
  get: <T>(path: string) => apiFetch<T>(path, { method: 'GET' }),
  post: <T>(path: string, body: unknown) =>
    apiFetch<T>(path, { method: 'POST', body: JSON.stringify(body) }),
  put: <T>(path: string, body: unknown) =>
    apiFetch<T>(path, { method: 'PUT', body: JSON.stringify(body) }),
  delete: <T>(path: string) => apiFetch<T>(path, { method: 'DELETE' }),
};
```

**Environment-specific base URLs**

Never hardcode production URLs — use Expo's public env vars:

```bash
# .env.production
EXPO_PUBLIC_API_URL=https://api.yourapp.com

# .env.development
EXPO_PUBLIC_API_URL=https://dev-api.yourapp.com
```

Variables prefixed with `EXPO_PUBLIC_` are inlined at build time and safe to reference in client code. Never prefix secrets (API keys, signing keys) with `EXPO_PUBLIC_` — they will be embedded in the bundle.

---

### 7. OWASP Mobile Top 10

The OWASP Mobile Top 10 (2024) covers the most common vulnerabilities in mobile apps. The table below maps each risk to its React Native / Expo mitigation.

| # | Vulnerability | React Native / Expo Mitigation |
|---|---|---|
| M1 | Improper Credential Usage | `expo-secure-store` for all tokens; never `AsyncStorage` for secrets |
| M2 | Inadequate Supply Chain Security | Lock dependency versions with lockfiles; run `npm audit` in CI |
| M3 | Insecure Authentication / Authorization | Biometric gate + short-lived JWTs (15 min); validate identity on every server request |
| M4 | Insufficient Input / Output Validation | Zod schema validation; sanitize before rendering user content in `WebView` |
| M5 | Insecure Communication | HTTPS only; certificate pinning for sensitive APIs; disable ATS exceptions |
| M6 | Inadequate Privacy Controls | Minimize on-device data; encrypt PII at rest; respect `ATTrackingManager` on iOS |
| M7 | Insufficient Binary Protections | Hermes bytecode + ProGuard/R8 (Android) + JS obfuscation |
| M8 | Security Misconfiguration | Disable debug mode in production; remove dev backdoors and mock auth flows |
| M9 | Insecure Data Storage | `SecureStore` for secrets; SQLCipher for encrypted SQLite if storing PII locally |
| M10 | Insufficient Cryptography | Use platform Keychain/Keystore APIs; never implement custom crypto primitives |

**M4 — WebView input sanitization** deserves a code example. If you render user-generated content in a `WebView`, sanitize it first:

```typescript
// ❌ Dangerous: user content rendered directly in WebView
<WebView source={{ html: userContent }} />

// ✅ Sanitize HTML before rendering
import sanitizeHtml from 'sanitize-html';

const safeHtml = sanitizeHtml(userContent, {
  allowedTags: ['b', 'i', 'em', 'strong', 'p', 'br', 'ul', 'ol', 'li'],
  allowedAttributes: {},
  disallowedTagsMode: 'discard',
});

<WebView
  source={{ html: safeHtml }}
  originWhitelist={['about:blank']} // prevent navigation
  sandbox="allow-same-origin"
/>
```

**M9 — SQLite encryption with SQLCipher**

For apps that must store PII locally (offline health records, financial data):

```bash
npx expo install @op-engineering/op-sqlite
# op-sqlite supports SQLCipher encryption
```

```typescript
// lib/db.ts
import { open } from '@op-engineering/op-sqlite';
import * as SecureStore from 'expo-secure-store';
import * as Crypto from 'expo-crypto';

async function getOrCreateDbKey(): Promise<string> {
  const existing = await SecureStore.getItemAsync('db_encryption_key');
  if (existing) return existing;

  // Generate a cryptographically random key
  const bytes = await Crypto.getRandomBytesAsync(32);
  const key = Buffer.from(bytes).toString('hex');
  await SecureStore.setItemAsync('db_encryption_key', key, {
    keychainAccessible: SecureStore.WHEN_UNLOCKED_THIS_DEVICE_ONLY,
  });
  return key;
}

export async function openEncryptedDb() {
  const key = await getOrCreateDbKey();
  return open({
    name: 'app.db',
    encryptionKey: key, // SQLCipher AES-256 encryption
  });
}
```

---

## Anti-Patterns

**AsyncStorage for tokens — the exact attack**

On a rooted Android device:

```bash
# Pull the AsyncStorage database
adb shell
su
cat /data/data/com.yourapp/files/RCTAsyncLocalStorage_V1/catalystLocalStorage

# Output: {"auth_token":"eyJhbGciOiJSUzI1NiIsInR5..."}
# The token is in plaintext. The attacker now has full API access.
```

The `expo-secure-store` file for the same token is an encrypted binary blob — unreadable without the hardware-backed decryption key.

**API keys in the JS bundle**

With Hermes bytecode:

```bash
# Decompile the Hermes bundle
npm install -g hermes-dec
hermes-dec index.android.bundle -o decompiled.js

# Then search for secrets
grep -i "api_key\|secret\|token\|key" decompiled.js | head -20
# Found: const STRIPE_SECRET_KEY = "sk_live_abc123..."
```

With plaintext JS (non-Hermes builds), it is even simpler:

```bash
strings assets/index.jsbundle | grep -i "api_key\|sk_live\|secret"
```

**The fix is architectural, not a hiding technique**: move the secret to your backend. The client calls `POST /api/payments/charge` — your server holds the Stripe secret key and makes the Stripe API call.

**Trusting client-sent user IDs**

```typescript
// ❌ Server trusting the userId from the request body
app.post('/api/orders', async (req) => {
  const { userId, items } = req.body; // attacker sends any userId they want
  await createOrder({ userId, items });
});

// ✅ Always derive identity from the verified JWT
app.post('/api/orders', requireAuth, async (req) => {
  const userId = req.user.sub; // extracted from verified JWT, not from body
  const { items } = req.body;
  await createOrder({ userId, items });
});
```

**HTTP endpoints in production**

```typescript
// ❌ HTTP in production — traffic is visible to anyone on the same network
const API_URL = 'http://api.yourapp.com';

// ❌ Whitelisting HTTP via ATS exception (breaks MITM protection)
// app.json: "NSAllowsArbitraryLoads": true  — DO NOT DO THIS in production

// ✅ HTTPS everywhere, including local dev tunnels
const API_URL = 'https://api.yourapp.com';
// For local dev: use 'npx expo start --tunnel' which provides an HTTPS URL
```

**Storing sensitive data in navigation params**

```typescript
// ❌ Sensitive data in navigation params — serialized to navigation state,
// potentially persisted to disk by React Navigation's state persistence feature
navigation.navigate('Profile', { ssn: '123-45-6789', creditCard: '4111...' });

// ✅ Pass only IDs; fetch sensitive data in the target screen from SecureStore or API
navigation.navigate('Profile', { userId: '123' });
// ProfileScreen fetches the sensitive fields from the API using the stored token
```

---

## Real-World Scenarios

### Scenario 1: Banking App Security Stack

A banking app requires the highest practical security level on a React Native stack. The layers work in combination — each mitigates a different attack vector:

```
Layer 1: HTTPS + Certificate Pinning
  → Blocks network-level MITM (Proxyman, Burp Suite, malicious WiFi)

Layer 2: SecureStore with WHEN_UNLOCKED_THIS_DEVICE_ONLY
  → Tokens inaccessible to other apps and not exported in backups

Layer 3: Biometric gate on every sensitive action
  → Stolen unlocked phone cannot perform transfers without biometrics

Layer 4: Jailbreak / root detection
  → Reduces attack surface; triggers backend risk scoring

Layer 5: Short-lived JWTs (15-minute expiry) + refresh token rotation
  → Limits window of exploitation if a token is extracted

Layer 6: Server-side device fingerprinting
  → Detects anomalous request patterns (new device, unusual location)
```

```typescript
// hooks/useBankingAuth.ts
import { useCallback, useEffect } from 'react';
import { checkDeviceIntegrity } from '../lib/device-integrity';
import { authenticateWithBiometrics } from '../lib/biometrics';
import { getAccessToken } from '../lib/token-store';
import { api } from '../lib/api';

export function useBankingAuth() {
  useEffect(() => {
    // Check integrity on mount — report violations but do not hard-block
    // (hard-block logic lives in the backend's risk engine)
    const integrity = checkDeviceIntegrity();
    if (!integrity.safe) {
      api.post('/security/integrity-report', {
        reasons: integrity.reasons,
        timestamp: new Date().toISOString(),
      });
    }
  }, []);

  const authorizeTransfer = useCallback(async (transferPayload: unknown) => {
    // Every transfer requires fresh biometric confirmation
    const auth = await authenticateWithBiometrics('Confirm transfer with biometrics');
    if (!auth.success) throw new Error('Authentication required');

    const token = await getAccessToken();
    if (!token) throw new Error('Not authenticated');

    return api.post('/transfers', transferPayload);
  }, []);

  return { authorizeTransfer };
}
```

### Scenario 2: Health App with Encrypted Local Storage

A health app stores clinical notes offline. These qualify as PHI (Protected Health Information) and require encryption at rest in addition to the standard controls.

```typescript
// lib/health-db.ts
import * as SecureStore from 'expo-secure-store';
import * as Crypto from 'expo-crypto';
import { open, type DB } from '@op-engineering/op-sqlite';

let _db: DB | null = null;

async function getEncryptionKey(): Promise<string> {
  const stored = await SecureStore.getItemAsync('health_db_key', {
    keychainAccessible: SecureStore.WHEN_UNLOCKED_THIS_DEVICE_ONLY,
  });
  if (stored) return stored;

  const bytes = await Crypto.getRandomBytesAsync(32);
  const key = Buffer.from(bytes).toString('hex');

  await SecureStore.setItemAsync('health_db_key', key, {
    keychainAccessible: SecureStore.WHEN_UNLOCKED_THIS_DEVICE_ONLY,
  });
  return key;
}

export async function getHealthDb(): Promise<DB> {
  if (_db) return _db;

  const key = await getEncryptionKey();
  _db = open({ name: 'health.db', encryptionKey: key });

  await _db.execute(`
    CREATE TABLE IF NOT EXISTS clinical_notes (
      id TEXT PRIMARY KEY,
      patient_id TEXT NOT NULL,
      content TEXT NOT NULL,     -- encrypted at rest by SQLCipher
      created_at TEXT NOT NULL,
      synced INTEGER DEFAULT 0
    )
  `);

  return _db;
}

// Audit log: every PHI access is recorded
export async function logDataAccess(
  userId: string,
  resourceType: string,
  resourceId: string
): Promise<void> {
  await api.post('/audit/access', {
    userId,
    resourceType,
    resourceId,
    deviceTime: new Date().toISOString(),
  });
}
```

For file storage (images, PDFs), encrypt before writing to disk:

```typescript
// lib/encrypted-file.ts
import * as FileSystem from 'expo-file-system';
import * as Crypto from 'expo-crypto';

// AES-256-GCM via SubtleCrypto (available in React Native via the polyfill)
export async function encryptFile(plaintext: Uint8Array): Promise<{
  ciphertext: string;
  iv: string;
  tag: string;
}> {
  const ivBytes = await Crypto.getRandomBytesAsync(12); // 96-bit IV for GCM
  const keyBytes = await Crypto.getRandomBytesAsync(32);

  const key = await crypto.subtle.importKey(
    'raw',
    keyBytes,
    { name: 'AES-GCM' },
    false,
    ['encrypt']
  );

  const ciphertextBuffer = await crypto.subtle.encrypt(
    { name: 'AES-GCM', iv: ivBytes },
    key,
    plaintext
  );

  return {
    ciphertext: Buffer.from(ciphertextBuffer).toString('base64'),
    iv: Buffer.from(ivBytes).toString('base64'),
    tag: '', // AES-GCM appends the tag to the ciphertext — included above
  };
}
```

---

## Debugging & Testing Security Controls

**Testing certificate pinning**

1. Set up Proxyman or Charles Proxy with their CA installed on your test device
2. Run the app with pinning enabled — API calls should fail with a network error
3. Confirm that `__DEV__` bypasses pinning (calls succeed in development)
4. Run with `--configuration Release` equivalent to test production pinning

**Testing SecureStore**

```typescript
// In __tests__/token-store.test.ts
import * as SecureStore from 'expo-secure-store';
import { saveTokens, getAccessToken, clearTokens } from '../lib/token-store';

jest.mock('expo-secure-store');

describe('token-store', () => {
  beforeEach(() => jest.clearAllMocks());

  it('saves tokens with correct accessibility option', async () => {
    const mockSetItem = SecureStore.setItemAsync as jest.Mock;
    mockSetItem.mockResolvedValue(undefined);

    await saveTokens('access-123', 'refresh-456');

    expect(mockSetItem).toHaveBeenCalledWith(
      'auth_token',
      'access-123',
      expect.objectContaining({
        keychainAccessible: SecureStore.WHEN_UNLOCKED_THIS_DEVICE_ONLY,
      })
    );
  });

  it('returns null when no token is stored', async () => {
    (SecureStore.getItemAsync as jest.Mock).mockResolvedValue(null);
    expect(await getAccessToken()).toBeNull();
  });

  it('clears both tokens on sign-out', async () => {
    const mockDelete = SecureStore.deleteItemAsync as jest.Mock;
    mockDelete.mockResolvedValue(undefined);

    await clearTokens();

    expect(mockDelete).toHaveBeenCalledWith('auth_token');
    expect(mockDelete).toHaveBeenCalledWith('refresh_token');
  });
});
```

**Testing biometric flow**

```typescript
// In __tests__/biometrics.test.ts
import * as LocalAuthentication from 'expo-local-authentication';
import { authenticateWithBiometrics } from '../lib/biometrics';

jest.mock('expo-local-authentication');

describe('authenticateWithBiometrics', () => {
  it('returns not_available when hardware is absent', async () => {
    (LocalAuthentication.hasHardwareAsync as jest.Mock).mockResolvedValue(false);
    (LocalAuthentication.isEnrolledAsync as jest.Mock).mockResolvedValue(false);
    (LocalAuthentication.supportedAuthenticationTypesAsync as jest.Mock).mockResolvedValue([]);

    const result = await authenticateWithBiometrics();
    expect(result).toEqual({ success: false, reason: 'not_available' });
  });

  it('returns success on successful authentication', async () => {
    (LocalAuthentication.hasHardwareAsync as jest.Mock).mockResolvedValue(true);
    (LocalAuthentication.isEnrolledAsync as jest.Mock).mockResolvedValue(true);
    (LocalAuthentication.supportedAuthenticationTypesAsync as jest.Mock).mockResolvedValue([1]);
    (LocalAuthentication.authenticateAsync as jest.Mock).mockResolvedValue({ success: true });

    const result = await authenticateWithBiometrics('Test prompt');
    expect(result).toEqual({ success: true });
  });

  it('returns cancelled when user dismisses', async () => {
    (LocalAuthentication.hasHardwareAsync as jest.Mock).mockResolvedValue(true);
    (LocalAuthentication.isEnrolledAsync as jest.Mock).mockResolvedValue(true);
    (LocalAuthentication.supportedAuthenticationTypesAsync as jest.Mock).mockResolvedValue([1]);
    (LocalAuthentication.authenticateAsync as jest.Mock).mockResolvedValue({
      success: false,
      error: 'user_cancel',
    });

    const result = await authenticateWithBiometrics();
    expect(result).toEqual({ success: false, reason: 'cancelled' });
  });
});
```

---

## Further Reading

- [OWASP Mobile Security Testing Guide](https://owasp.org/www-project-mobile-security-testing-guide/)
- [OWASP Mobile Top 10 (2024)](https://owasp.org/www-project-mobile-top-10/)
- [expo-secure-store docs](https://docs.expo.dev/versions/latest/sdk/securestore/)
- [expo-local-authentication docs](https://docs.expo.dev/versions/latest/sdk/local-authentication/)
- [react-native-ssl-pinning](https://github.com/MaxToyberman/react-native-ssl-pinning)
- [Android Keystore System](https://developer.android.com/training/articles/keystore)
- [iOS Keychain Services](https://developer.apple.com/documentation/security/keychain_services)
- [NowSecure Mobile Security Checklist](https://www.nowsecure.com/resources/checklists/)

---

## Summary

- Use `expo-secure-store` with `WHEN_UNLOCKED_THIS_DEVICE_ONLY` for all tokens and secrets — `AsyncStorage` is plaintext on disk and readable with `adb` on rooted devices.
- Certificate pinning prevents MITM attacks from proxy tools and malicious CAs; bundle at least two pins (current + backup) and disable pinning via `__DEV__` for development.
- Gate sensitive operations behind `expo-local-authentication` — biometrics unlock the SecureStore-held token, they do not replace server-side authentication.
- Use `react-native-jail-monkey` to detect jailbreak/root; treat detections as a risk signal for server-side fraud scoring rather than a hard client-side block (false positives exist on some OEM devices).
- JavaScript obfuscation and Hermes bytecode slow down reverse engineering but do not prevent it — never rely on them to protect secrets; move secrets to the server.
- Block HTTP in production: set `usesCleartextTraffic: false` on Android and keep `NSAllowsArbitraryLoads: false` on iOS.
- Encrypt SQLite databases with SQLCipher (via `@op-engineering/op-sqlite`) when storing PII locally; store the database key in SecureStore.
