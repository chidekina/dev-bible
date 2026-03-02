# Segurança Mobile

## Visão Geral

Apps mobile enfrentam uma superfície de ataque distinta, diferente das aplicações web. Dados sensíveis armazenados em texto puro no dispositivo, tráfego de rede interceptado por um proxy local, binários revertidos para extrair API keys, e dispositivos roubados concedendo acesso a sessões autenticadas — cada um desses é um vetor de ataque real, não teórico. Uma única configuração errada pode expor milhares de usuários: um token deixado no AsyncStorage, um endpoint HTTP em produção, ou um segredo hardcoded no bundle JS que qualquer atacante pode extrair em minutos.

Este capítulo cobre o OWASP Mobile Top 10 (edição 2024), armazenamento seguro com suporte de hardware via iOS Keychain e Android Keystore, certificate pinning para bloquear ataques MITM, autenticação biométrica como portão de acesso, detecção de jailbreak/root e hardening de código em tempo de build. Essas não são medidas opcionais de proteção — são o baseline mínimo para qualquer app que lida com autenticação ou dados pessoais.

O modelo de segurança mobile difere do web em um aspecto crítico: o cliente é não confiável por padrão. Usuários executam seu app em dispositivos com root, interceptam suas chamadas de API e alteram o seu binário. Projete a autorização no lado do servidor como se o cliente pudesse estar completamente comprometido. Tudo neste capítulo desacelera os atacantes — os controles server-side os barram de verdade.

---

## Pré-requisitos

- [React Native Básico](./react-native-basics.md)
- Track 07 — Segurança (OWASP Top 10 para web — mobile compartilha muitos conceitos)
- Familiaridade com async/await e generics em TypeScript

---

## Conceitos Principais

### 1. Armazenamento Seguro — expo-secure-store

`AsyncStorage` é um armazenamento chave-valor que escreve em arquivos de texto puro no disco. Em um dispositivo Android com root, `adb pull /data/data/com.yourapp/databases/` recupera esses arquivos sem nenhuma autenticação necessária. No iOS, um backup não criptografado os expõe. Nunca armazene segredos, tokens ou PII no AsyncStorage.

`expo-secure-store` encapsula o armazenamento seguro com suporte de hardware da plataforma:

- **iOS**: Keychain Services — criptografado pela senha do dispositivo. Em dispositivos com Secure Enclave (chip A7 em diante, incluindo todos os iPhones modernos), a chave de criptografia nunca sai do hardware seguro.
- **Android 6+ (API 23+)**: Android Keystore — as chaves são geradas internamente e nunca extraídas do TEE (Trusted Execution Environment) ou Secure Element.

```bash
npx expo install expo-secure-store
```

```typescript
// lib/token-store.ts
import * as SecureStore from 'expo-secure-store';

const TOKEN_KEY = 'auth_token';
const REFRESH_KEY = 'refresh_token';

// keychainAccessible controla QUANDO o item pode ser lido.
// WHEN_UNLOCKED_THIS_DEVICE_ONLY:
//   - legível apenas enquanto o dispositivo está desbloqueado
//   - não é migrado para novos dispositivos via backups
//   - padrão correto para tokens de autenticação
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

**Os quatro níveis de acessibilidade** e quando usar cada um:

| Opção | Legível quando | Migrado para novo dispositivo | Caso de uso |
|---|---|---|---|
| `AFTER_FIRST_UNLOCK` | Após primeiro desbloqueio pós-reboot | Sim (backup iCloud no iOS) | Refresh tokens em background |
| `AFTER_FIRST_UNLOCK_THIS_DEVICE_ONLY` | Após primeiro desbloqueio pós-reboot | Não | Tokens em background, sem backup na nuvem |
| `WHEN_UNLOCKED` | Dispositivo está desbloqueado | Sim | Credenciais de uso ativo |
| `WHEN_UNLOCKED_THIS_DEVICE_ONLY` | Dispositivo está desbloqueado | Não | Tokens de autenticação (**padrão recomendado**) |

```typescript
// ❌ Nunca armazene segredos aqui — texto puro no disco
import AsyncStorage from '@react-native-async-storage/async-storage';
await AsyncStorage.setItem('auth_token', token);
// Atacante com adb root: adb pull /data/data/com.app/files/RCTAsyncLocalStorage_V1/

// ✅ Criptografia com suporte de hardware
import * as SecureStore from 'expo-secure-store';
await SecureStore.setItemAsync('auth_token', token, {
  keychainAccessible: SecureStore.WHEN_UNLOCKED_THIS_DEVICE_ONLY,
});
```

**Limite de tamanho**: valores do SecureStore são limitados a 2 KB. Para payloads criptografados maiores (ex: um objeto de perfil do usuário), criptografe os dados com uma chave simétrica, armazene a chave no SecureStore e grave o ciphertext no sistema de arquivos ou AsyncStorage.

---

### 2. Certificate Pinning

TLS protege o tráfego de escutas passivas, mas não de um atacante que pode instalar uma CA confiável no dispositivo — o que é trivialmente feito em qualquer dispositivo que o atacante controla. Ferramentas como Proxyman, Charles Proxy e Burp Suite funcionam exatamente assim: instalam seu próprio certificado CA e realizam um ataque MITM contra o seu app.

Certificate pinning faz o app rejeitar conexões a menos que o certificado do servidor (ou sua chave pública) corresponda a um valor embutido em tempo de build. Mesmo com uma CA MITM confiável instalada, o atacante não consegue forjar a chave pinada do seu servidor.

```bash
npx expo install react-native-ssl-pinning
```

Primeiro, extraia o hash da chave pública do seu servidor:

```bash
# Extraia o hash da chave pública (SPKI SHA-256) para o seu host de API
openssl s_client -connect api.yourapp.com:443 < /dev/null 2>/dev/null \
  | openssl x509 -pubkey -noout \
  | openssl pkey -pubin -outform DER \
  | openssl dgst -sha256 -binary \
  | base64
```

Inclua o arquivo do certificado no projeto (`assets/certs/api_cert.cer`) e configure o pinning:

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
  // Desativa o pinning em dev para permitir Proxyman/Charles durante o desenvolvimento
  if (__DEV__) {
    return fetch(`${API_BASE}${path}`, options);
  }

  return pinnedFetch(`${API_BASE}${path}`, {
    method: options.method ?? 'GET',
    sslPinning: {
      certs: ['api_cert'], // nome do arquivo sem extensão, em assets/certs/
    },
    headers: {
      'Content-Type': 'application/json',
      ...options.headers,
    },
    body: options.body,
  });
}
```

**Rotação de certificados**: quando o seu certificado TLS expirar ou for substituído, o app com pinning vai rejeitar conexões. Para mitigar isso:

1. Faça o pin no certificado da CA intermediária (mais duradouro que certs folha)
2. Inclua um pin de backup (hash da chave pública do próximo certificado)
3. Implemente um endpoint de configuração remota (por um canal sem pinning) que possa atualizar os pins via OTA

```typescript
// Sempre faça pin em pelo menos dois certificados: atual + próximo
sslPinning: {
  certs: ['api_cert_current', 'api_cert_backup'],
}
```

**Importante**: `react-native-ssl-pinning` requer bare workflow ou `expo-dev-client`. Não funciona no Expo Go porque ele usa seu próprio stack de rede.

---

### 3. Autenticação Biométrica

Autenticação biométrica (Face ID, Touch ID, impressão digital) adiciona um segundo fator sem o atrito de senha. O padrão crítico é usar biometria como portão para desbloquear um token armazenado no SecureStore — não como o mecanismo principal de autenticação.

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
    fallbackLabel: 'Use Passcode', // exibido no iOS quando a biometria falha
    disableDeviceFallback: false,  // permite senha como fallback
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

Proteja a recuperação do token com autenticação biométrica:

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

    // Lê o token SOMENTE APÓS o sucesso biométrico
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

**Detecção do tipo biométrico** — exiba o ícone correto na UI:

```typescript
import * as LocalAuthentication from 'expo-local-authentication';

export async function getBiometricLabel(): Promise<string> {
  const types = await LocalAuthentication.supportedAuthenticationTypesAsync();

  if (types.includes(LocalAuthentication.AuthenticationType.FACIAL_RECOGNITION)) {
    return 'Face ID'; // Face ID do iOS ou desbloqueio facial do Android
  }
  if (types.includes(LocalAuthentication.AuthenticationType.FINGERPRINT)) {
    return 'Touch ID'; // Touch ID do iOS ou impressão digital do Android
  }
  if (types.includes(LocalAuthentication.AuthenticationType.IRIS)) {
    return 'Iris scan';
  }
  return 'Biometrics';
}
```

---

### 4. Detecção de Jailbreak e Root

Um dispositivo iOS com jailbreak ou Android com root contorna o modelo de segurança do SO: qualquer app pode ler os arquivos de outros apps, depuradores podem se conectar a processos, e hooks em chamadas de sistema podem interceptar operações criptográficas. Essas capacidades são usadas por atacantes para extrair tokens da memória, despejar o conteúdo do SecureStore e contornar o certificate pinning.

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
    // Detecta Cydia Substrate, Frida e frameworks de hooking similares
    reasons.push('hook_detected');
  }

  if (Platform.OS === 'android' && JailMonkey.isOnExternalStorage()) {
    // App instalado em armazenamento externo — garantias de isolamento mais fracas
    reasons.push('external_storage');
  }

  if (!__DEV__ && JailMonkey.isDebuggedMode()) {
    // Depurador conectado em produção — suspeito
    reasons.push('debugger_attached');
  }

  // Opcionalmente detecta emuladores (vai gerar falsos positivos em CI)
  // if (JailMonkey.isEmulator()) reasons.push('emulator');

  return { safe: reasons.length === 0, reasons };
}
```

O que fazer quando as verificações de integridade falham:

```typescript
// Na inicialização raiz do app
import { checkDeviceIntegrity } from './lib/device-integrity';
import { reportIntegrityViolation } from './lib/api';

export async function initializeSecurity(): Promise<void> {
  const integrity = checkDeviceIntegrity();

  if (!integrity.safe) {
    // 1. Reporta ao backend para detecção de fraude/anomalia
    await reportIntegrityViolation(integrity.reasons).catch(() => {
      // Fire-and-forget — não bloqueie o fluxo do usuário em caso de falha de rede
    });

    // 2. Decida a resposta com base no seu modelo de risco:
    // Opção A (apps de alta segurança): bloqueio total
    // Alert.alert('Dispositivo Não Suportado', 'Este app não pode rodar em dispositivos modificados.');
    // return;

    // Opção B (maioria dos apps): desabilita funcionalidades sensíveis, avisa o usuário
    console.warn('Verificação de integridade do dispositivo falhou:', integrity.reasons);
    // Desabilita: fluxos de pagamento, desbloqueio biométrico, upload de documentos, etc.
  }
}
```

**Falsos positivos**: alguns dispositivos OEM Android (Samsung Knox, alguns Xiaomi) acionam `isJailBroken()` incorretamente. Emuladores usados em CI também vão disparar detecções. Estratégia: use `isJailBroken()` como um sinal para aumentar o escrutínio no backend (pontuação de risco), não como um bloqueio definitivo — a menos que você esteja construindo um app bancário ou fintech onde o requisito de negócio exige isso.

---

### 5. Ofuscação de Código

O Hermes compila o JavaScript para bytecode em tempo de build, que não é legível por humanos. Porém, ferramentas como `hermes-dec` conseguem descompilar o bytecode Hermes de volta para JavaScript legível. A ofuscação adiciona mais uma camada de atrito.

```bash
# Adiciona ofuscação em nível JavaScript
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
              controlFlowFlattening: true,        // achata o grafo de fluxo de controle
              controlFlowFlatteningThreshold: 0.75,
              deadCodeInjection: true,            // insere caminhos de código falsos
              deadCodeInjectionThreshold: 0.4,
              stringEncryption: true,             // criptografa literais de string
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

O Android também executa ProGuard/R8 na camada Java/Kotlin. As builds de produção do Expo habilitam isso por padrão. Para bare workflow:

```groovy
// android/app/build.gradle
buildTypes {
  release {
    minifyEnabled true      // minificação R8 + eliminação de código morto
    shrinkResources true    // remove recursos não utilizados
    proguardFiles getDefaultProguardFile('proguard-android-optimize.txt'), 'proguard-rules.pro'
  }
}
```

**A ressalva crítica**: a ofuscação desacelera a engenharia reversa — ela não a impede. Um atacante determinado com Frida e tempo suficiente vai reconstruir a sua lógica. A única solução segura para segredos é mantê-los no servidor. Se um segredo precisar existir no cliente, aceite que ele eventualmente será extraído.

```typescript
// ❌ Nenhuma quantidade de ofuscação torna isso seguro
const API_KEY = 'sk-live-abc123def456'; // Vai ser extraído

// ✅ A arquitetura correta: faça proxy de chamadas sensíveis pelo seu backend
// Cliente chama: POST /api/ai/complete
// Servidor guarda a chave OpenAI, faz a chamada e retorna o resultado
```

---

### 6. Rede Segura

**iOS — App Transport Security (ATS)**

O ATS é aplicado por padrão no iOS. Todas as conexões HTTP são bloqueadas no nível do SO, a menos que sejam explicitamente autorizadas no `Info.plist`. O workflow gerenciado do Expo configura o ATS automaticamente em builds de produção.

```json
// app.json — aplicação explícita do ATS (também é o padrão)
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

**Android — tráfego em texto puro**

O Android permite HTTP por padrão. Bloqueie-o explicitamente:

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

**Anexando headers de autenticação vindos do SecureStore**

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

  // Trata refresh do token em caso de 401
  if (response.status === 401) {
    const refreshed = await attemptTokenRefresh();
    if (refreshed) {
      // Tenta novamente com o novo token
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

**URLs base por ambiente**

Nunca hardcode URLs de produção — use as variáveis de ambiente públicas do Expo:

```bash
# .env.production
EXPO_PUBLIC_API_URL=https://api.yourapp.com

# .env.development
EXPO_PUBLIC_API_URL=https://dev-api.yourapp.com
```

Variáveis prefixadas com `EXPO_PUBLIC_` são injetadas em tempo de build e podem ser referenciadas com segurança no código do cliente. Nunca prefixe segredos (API keys, chaves de assinatura) com `EXPO_PUBLIC_` — eles serão embutidos no bundle.

---

### 7. OWASP Mobile Top 10

O OWASP Mobile Top 10 (2024) cobre as vulnerabilidades mais comuns em apps mobile. A tabela abaixo mapeia cada risco para sua mitigação em React Native / Expo.

| # | Vulnerabilidade | Mitigação em React Native / Expo |
|---|---|---|
| M1 | Uso Inadequado de Credenciais | `expo-secure-store` para todos os tokens; nunca `AsyncStorage` para segredos |
| M2 | Segurança Inadequada da Cadeia de Suprimentos | Trave versões de dependências com lockfiles; execute `npm audit` no CI |
| M3 | Autenticação / Autorização Insegura | Portão biométrico + JWTs de curta duração (15 min); valide identidade em toda requisição ao servidor |
| M4 | Validação Insuficiente de Entrada / Saída | Validação com schema Zod; sanitize antes de renderizar conteúdo do usuário em `WebView` |
| M5 | Comunicação Insegura | Somente HTTPS; certificate pinning para APIs sensíveis; desabilite exceções no ATS |
| M6 | Controles de Privacidade Inadequados | Minimize dados no dispositivo; criptografe PII em repouso; respeite `ATTrackingManager` no iOS |
| M7 | Proteções Binárias Insuficientes | Bytecode Hermes + ProGuard/R8 (Android) + ofuscação JS |
| M8 | Configuração Incorreta de Segurança | Desabilite o modo debug em produção; remova backdoors de dev e fluxos de autenticação mock |
| M9 | Armazenamento de Dados Inseguro | `SecureStore` para segredos; SQLCipher para SQLite criptografado ao armazenar PII localmente |
| M10 | Criptografia Insuficiente | Use as APIs Keychain/Keystore da plataforma; nunca implemente primitivos criptográficos customizados |

**M4 — sanitização de entrada em WebView** merece um exemplo de código. Se você renderizar conteúdo gerado pelo usuário em uma `WebView`, sanitize-o primeiro:

```typescript
// ❌ Perigoso: conteúdo do usuário renderizado diretamente no WebView
<WebView source={{ html: userContent }} />

// ✅ Sanitize o HTML antes de renderizar
import sanitizeHtml from 'sanitize-html';

const safeHtml = sanitizeHtml(userContent, {
  allowedTags: ['b', 'i', 'em', 'strong', 'p', 'br', 'ul', 'ol', 'li'],
  allowedAttributes: {},
  disallowedTagsMode: 'discard',
});

<WebView
  source={{ html: safeHtml }}
  originWhitelist={['about:blank']} // impede navegação
  sandbox="allow-same-origin"
/>
```

**M9 — Criptografia SQLite com SQLCipher**

Para apps que precisam armazenar PII localmente (prontuários offline, dados financeiros):

```bash
npx expo install @op-engineering/op-sqlite
# op-sqlite suporta criptografia SQLCipher
```

```typescript
// lib/db.ts
import { open } from '@op-engineering/op-sqlite';
import * as SecureStore from 'expo-secure-store';
import * as Crypto from 'expo-crypto';

async function getOrCreateDbKey(): Promise<string> {
  const existing = await SecureStore.getItemAsync('db_encryption_key');
  if (existing) return existing;

  // Gera uma chave criptograficamente aleatória
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
    encryptionKey: key, // criptografia AES-256 do SQLCipher
  });
}
```

---

## Anti-Padrões

**AsyncStorage para tokens — o ataque exato**

Em um dispositivo Android com root:

```bash
# Puxa o banco de dados do AsyncStorage
adb shell
su
cat /data/data/com.yourapp/files/RCTAsyncLocalStorage_V1/catalystLocalStorage

# Saída: {"auth_token":"eyJhbGciOiJSUzI1NiIsInR5..."}
# O token está em texto puro. O atacante agora tem acesso total à API.
```

O arquivo do `expo-secure-store` para o mesmo token é um blob binário criptografado — ilegível sem a chave de descriptografia com suporte de hardware.

**API keys no bundle JS**

Com bytecode Hermes:

```bash
# Descompila o bundle Hermes
npm install -g hermes-dec
hermes-dec index.android.bundle -o decompiled.js

# Depois busca por segredos
grep -i "api_key\|secret\|token\|key" decompiled.js | head -20
# Encontrado: const STRIPE_SECRET_KEY = "sk_live_abc123..."
```

Com JS em texto puro (builds sem Hermes), é ainda mais simples:

```bash
strings assets/index.jsbundle | grep -i "api_key\|sk_live\|secret"
```

**A correção é arquitetural, não uma técnica de esconder**: mova o segredo para o seu backend. O cliente chama `POST /api/payments/charge` — o seu servidor guarda a chave secreta do Stripe e faz a chamada para a API do Stripe.

**Confiar em IDs de usuário enviados pelo cliente**

```typescript
// ❌ Servidor confiando no userId vindo do body da requisição
app.post('/api/orders', async (req) => {
  const { userId, items } = req.body; // atacante envia qualquer userId que quiser
  await createOrder({ userId, items });
});

// ✅ Sempre derive a identidade do JWT verificado
app.post('/api/orders', requireAuth, async (req) => {
  const userId = req.user.sub; // extraído do JWT verificado, não do body
  const { items } = req.body;
  await createOrder({ userId, items });
});
```

**Endpoints HTTP em produção**

```typescript
// ❌ HTTP em produção — tráfego visível para qualquer pessoa na mesma rede
const API_URL = 'http://api.yourapp.com';

// ❌ Autorizando HTTP via exceção no ATS (quebra a proteção MITM)
// app.json: "NSAllowsArbitraryLoads": true  — NÃO FAÇA ISSO em produção

// ✅ HTTPS em todo lugar, incluindo tunnels de dev local
const API_URL = 'https://api.yourapp.com';
// Para dev local: use 'npx expo start --tunnel' que fornece uma URL HTTPS
```

**Armazenando dados sensíveis em parâmetros de navegação**

```typescript
// ❌ Dados sensíveis em parâmetros de navegação — serializados no estado de navegação,
// potencialmente persistidos no disco pelo recurso de persistência de estado do React Navigation
navigation.navigate('Profile', { ssn: '123-45-6789', creditCard: '4111...' });

// ✅ Passe apenas IDs; busque dados sensíveis na tela de destino via SecureStore ou API
navigation.navigate('Profile', { userId: '123' });
// ProfileScreen busca os campos sensíveis da API usando o token armazenado
```

---

## Cenários do Mundo Real

### Cenário 1: Stack de Segurança para App Bancário

Um app bancário requer o maior nível prático de segurança em uma stack React Native. As camadas funcionam em combinação — cada uma mitiga um vetor de ataque diferente:

```
Camada 1: HTTPS + Certificate Pinning
  → Bloqueia MITM em nível de rede (Proxyman, Burp Suite, WiFi malicioso)

Camada 2: SecureStore com WHEN_UNLOCKED_THIS_DEVICE_ONLY
  → Tokens inacessíveis a outros apps e não exportados em backups

Camada 3: Portão biométrico em cada ação sensível
  → Telefone desbloqueado roubado não consegue realizar transferências sem biometria

Camada 4: Detecção de jailbreak / root
  → Reduz a superfície de ataque; aciona pontuação de risco no backend

Camada 5: JWTs de curta duração (expiração em 15 minutos) + rotação de refresh token
  → Limita a janela de exploração caso um token seja extraído

Camada 6: Fingerprinting de dispositivo no servidor
  → Detecta padrões anômalos de requisição (novo dispositivo, localização incomum)
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
    // Verifica integridade ao montar — reporta violações mas não bloqueia definitivamente
    // (a lógica de bloqueio definitivo fica no motor de risco do backend)
    const integrity = checkDeviceIntegrity();
    if (!integrity.safe) {
      api.post('/security/integrity-report', {
        reasons: integrity.reasons,
        timestamp: new Date().toISOString(),
      });
    }
  }, []);

  const authorizeTransfer = useCallback(async (transferPayload: unknown) => {
    // Cada transferência requer confirmação biométrica fresca
    const auth = await authenticateWithBiometrics('Confirm transfer with biometrics');
    if (!auth.success) throw new Error('Authentication required');

    const token = await getAccessToken();
    if (!token) throw new Error('Not authenticated');

    return api.post('/transfers', transferPayload);
  }, []);

  return { authorizeTransfer };
}
```

### Cenário 2: App de Saúde com Armazenamento Local Criptografado

Um app de saúde armazena notas clínicas offline. Esses dados se enquadram como PHI (Informação de Saúde Protegida) e exigem criptografia em repouso além dos controles padrão.

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
      content TEXT NOT NULL,     -- criptografado em repouso pelo SQLCipher
      created_at TEXT NOT NULL,
      synced INTEGER DEFAULT 0
    )
  `);

  return _db;
}

// Log de auditoria: todo acesso a PHI é registrado
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

Para armazenamento de arquivos (imagens, PDFs), criptografe antes de gravar no disco:

```typescript
// lib/encrypted-file.ts
import * as FileSystem from 'expo-file-system';
import * as Crypto from 'expo-crypto';

// AES-256-GCM via SubtleCrypto (disponível no React Native via polyfill)
export async function encryptFile(plaintext: Uint8Array): Promise<{
  ciphertext: string;
  iv: string;
  tag: string;
}> {
  const ivBytes = await Crypto.getRandomBytesAsync(12); // IV de 96 bits para GCM
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
    tag: '', // AES-GCM anexa a tag ao ciphertext — já incluída acima
  };
}
```

---

## Depuração e Testes de Controles de Segurança

**Testando certificate pinning**

1. Configure o Proxyman ou Charles Proxy com a CA instalada no seu dispositivo de teste
2. Execute o app com o pinning habilitado — as chamadas de API devem falhar com erro de rede
3. Confirme que `__DEV__` bypassa o pinning (chamadas funcionam em desenvolvimento)
4. Execute com `--configuration Release` equivalente para testar o pinning em produção

**Testando o SecureStore**

```typescript
// Em __tests__/token-store.test.ts
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

**Testando o fluxo biométrico**

```typescript
// Em __tests__/biometrics.test.ts
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

## Leitura Adicional

- [OWASP Mobile Security Testing Guide](https://owasp.org/www-project-mobile-security-testing-guide/)
- [OWASP Mobile Top 10 (2024)](https://owasp.org/www-project-mobile-top-10/)
- [Documentação do expo-secure-store](https://docs.expo.dev/versions/latest/sdk/securestore/)
- [Documentação do expo-local-authentication](https://docs.expo.dev/versions/latest/sdk/local-authentication/)
- [react-native-ssl-pinning](https://github.com/MaxToyberman/react-native-ssl-pinning)
- [Android Keystore System](https://developer.android.com/training/articles/keystore)
- [iOS Keychain Services](https://developer.apple.com/documentation/security/keychain_services)
- [NowSecure Mobile Security Checklist](https://www.nowsecure.com/resources/checklists/)

---

## Resumo

- Use `expo-secure-store` com `WHEN_UNLOCKED_THIS_DEVICE_ONLY` para todos os tokens e segredos — `AsyncStorage` é texto puro no disco e legível via `adb` em dispositivos com root.
- Certificate pinning previne ataques MITM de ferramentas proxy e CAs maliciosas; inclua pelo menos dois pins (atual + backup) e desative o pinning via `__DEV__` para desenvolvimento.
- Proteja operações sensíveis com `expo-local-authentication` — biometria desbloqueia o token armazenado no SecureStore, ela não substitui a autenticação server-side.
- Use `react-native-jail-monkey` para detectar jailbreak/root; trate as detecções como um sinal de risco para pontuação de fraude no servidor, não como um bloqueio definitivo no cliente (existem falsos positivos em alguns dispositivos OEM).
- Ofuscação JavaScript e bytecode Hermes desaceleram a engenharia reversa, mas não a impedem — nunca confie neles para proteger segredos; mova os segredos para o servidor.
- Bloqueie HTTP em produção: defina `usesCleartextTraffic: false` no Android e mantenha `NSAllowsArbitraryLoads: false` no iOS.
- Criptografe bancos de dados SQLite com SQLCipher (via `@op-engineering/op-sqlite`) ao armazenar PII localmente; guarde a chave do banco no SecureStore.
