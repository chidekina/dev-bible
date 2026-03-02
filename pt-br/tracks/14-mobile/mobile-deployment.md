# Mobile Deployment

## Visão Geral

Fazer deploy de um app mobile é mais complexo do que fazer deploy de um app web. Binários nativos precisam ser compilados para cada plataforma, assinados com certificados criptográficos e revisados pela Apple e Google antes de chegarem aos usuários. Atualizações over-the-air (OTA) permitem que você contorne esse processo para mudanças em JavaScript. EAS (Expo Application Services) envolve tudo isso em um fluxo coerente de CLI e CI/CD.

Este capítulo cobre o pipeline completo de deploy: configuração de assinatura, EAS Build para binários nativos, EAS Submit para submissão nas lojas, EAS Update para releases OTA e integração com CI/CD.

---

## Pré-requisitos

- [React Native Basics](react-native-basics.md) — configuração de projeto Expo
- Conta de desenvolvedor Apple ($99/ano) para iOS
- Conta no Google Play Console ($25 taxa única) para Android
- `eas-cli` instalado globalmente: `npm install -g eas-cli`

---

## Conceitos Principais

### Build vs Update

```
Build Nativo (EAS Build)               OTA Update (EAS Update)
──────────────────────────             ──────────────────────────
Contém: código nativo + bundle JS      Contém: apenas bundle JS + assets
Requer: revisão Apple/Google           Instantâneo: baixado no lançamento do app
Quando: novas deps nativas, permissões Quando: apenas mudanças em JS/UI/lógica
Loja: App Store / Play Store           Entrega: CDN do EAS Update
Frequência: a cada semanas/meses       Frequência: todos os dias se necessário
Tempo: 15–45 minutos                   Tempo: segundos para publicar
```

**Regra geral:** Se você mudou algo nos diretórios `ios/` ou `android/`, adicionou um módulo nativo ou mudou permissões — você precisa de um build nativo. Mudanças puras em JS/TypeScript → OTA update.

### Profiles

EAS usa profiles de build definidos em `eas.json` para configurar diferentes tipos de build.

```json
// eas.json
{
  "cli": { "version": ">= 7.0.0", "appVersionSource": "remote" },
  "build": {
    "development": {
      "developmentClient": true,
      "distribution": "internal",
      "ios": { "simulator": true },
      "android": { "buildType": "apk" }
    },
    "preview": {
      "distribution": "internal",
      "channel": "preview",
      "android": { "buildType": "apk" }
    },
    "production": {
      "autoIncrement": true,
      "channel": "production"
    }
  },
  "submit": {
    "production": {
      "ios": {
        "appleId": "voce@example.com",
        "ascAppId": "1234567890",
        "appleTeamId": "ABCDE12345"
      },
      "android": {
        "serviceAccountKeyPath": "./google-service-account.json",
        "track": "internal"
      }
    }
  }
}
```

---

## Exemplos Práticos

### Configuração Inicial do EAS

```bash
# 1. Instalar e autenticar
npm install -g eas-cli
eas login

# 2. Inicializar EAS no seu projeto
eas init
# Isso cria um projeto no expo.dev e adiciona projectId ao app.json

# 3. Configurar builds
eas build:configure
# Cria eas.json com profiles padrão

# 4. Configurar credenciais
eas credentials
# Interativo: gera/sobe certificados de assinatura para iOS e Android
```

### Configuração do App

```json
// app.json — configuração completa de produção
{
  "expo": {
    "name": "Meu App",
    "slug": "my-app",
    "version": "1.0.0",
    "runtimeVersion": {
      "policy": "appVersion"
    },
    "orientation": "portrait",
    "icon": "./assets/icon.png",
    "userInterfaceStyle": "automatic",
    "splash": {
      "image": "./assets/splash.png",
      "resizeMode": "contain",
      "backgroundColor": "#ffffff"
    },
    "ios": {
      "bundleIdentifier": "com.example.myapp",
      "buildNumber": "1",
      "supportsTablet": false,
      "infoPlist": {
        "NSCameraUsageDescription": "Este app usa a câmera para escanear códigos de barras.",
        "NSLocationWhenInUseUsageDescription": "Este app usa sua localização para exibir itens próximos."
      }
    },
    "android": {
      "package": "com.example.myapp",
      "versionCode": 1,
      "adaptiveIcon": {
        "foregroundImage": "./assets/adaptive-icon.png",
        "backgroundColor": "#6366f1"
      },
      "permissions": [
        "android.permission.CAMERA",
        "android.permission.ACCESS_FINE_LOCATION"
      ]
    },
    "updates": {
      "url": "https://u.expo.dev/<seu-project-id>",
      "enabled": true,
      "fallbackToCacheTimeout": 0,
      "checkAutomatically": "ON_LOAD"
    },
    "extra": {
      "eas": { "projectId": "seu-project-id" }
    }
  }
}
```

### Executando Builds

```bash
# Build de desenvolvimento (para testar com expo-dev-client)
eas build --profile development --platform ios
eas build --profile development --platform android

# Build para Simulador iOS
eas build --profile development --platform ios --local
# Depois: xcrun simctl install booted path/to/build.app

# Build de preview (APK/IPA compartilhável para testers)
eas build --profile preview --platform all

# Build de produção
eas build --profile production --platform all

# Verificar status do build
eas build:list

# Baixar o artefato
eas build:download --id <build-id>
```

### Assinatura do App

**Assinatura iOS** — EAS gerencia automaticamente provisioning profiles e certificados de assinatura.

```bash
# Gerenciar credenciais automaticamente (recomendado)
eas credentials --platform ios
# Selecione: "Manage all credentials with EAS"

# Manual — suba seu próprio certificado
eas credentials --platform ios
# Selecione: "Upload a distribution certificate"
```

**Assinatura Android** — suba o keystore para o EAS ou deixe o EAS gerar um.

```bash
# Gerar keystore via EAS (recomendado — armazena nas credenciais EAS)
eas credentials --platform android

# Importar keystore existente
eas credentials --platform android
# Selecione: "Upload a keystore"
```

**Crítico:** Guarde seu Android keystore em um lugar seguro. Se você perdê-lo e o app já estiver publicado, não poderá lançar atualizações com o mesmo nome de pacote.

### EAS Submit — Submissão nas Lojas

```bash
# Submeter o último build para a App Store
eas submit --platform ios --latest

# Submeter o último build para o Google Play (trilha de teste interno primeiro)
eas submit --platform android --latest

# Submeter um build específico
eas submit --platform ios --id <build-id>
```

**Pré-requisitos App Store:**
1. App criado no App Store Connect com bundle ID correspondendo ao `app.json`
2. URL da política de privacidade
3. Screenshots para cada tamanho de dispositivo necessário (use `npx expo screenshot`)
4. Metadados do app (descrição, palavras-chave, categoria)

**Pré-requisitos Google Play:**
1. App criado no Play Console
2. JSON da chave de conta de serviço com permissões de "Release manager"
3. Primeiro upload deve ser feito manualmente via UI web do Play Console (subsequentes via API)

### EAS Update — OTA Updates

```bash
# Publicar OTA update no canal de produção
eas update --channel production --message "Corrige bug de login"

# Publicar no canal de preview
eas update --channel preview --message "Nova tela de perfil"

# Reverter para a atualização anterior
eas update:rollback --channel production

# Listar atualizações
eas update:list
```

**Como OTA updates funcionam:**

```
Lançamento do app
  ↓
Verifica CDN do EAS Update para novo bundle correspondendo a:
  - runtimeVersion (compatibilidade com binário nativo)
  - channel (production / preview / etc.)
  ↓
Se atualização disponível:
  Baixa em segundo plano (ou aguarda no primeiro lançamento)
  ↓
Aplica na próxima reinicialização do app
```

**Runtime version** — determina a compatibilidade entre um OTA update e um binário nativo. Quando você muda código nativo, incremente a runtime version (ou deixe `"policy": "appVersion"` fazer automaticamente com base em `version` no app.json).

```tsx
// Verificar e aplicar atualizações programaticamente
import * as Updates from 'expo-updates';

async function checkForUpdates() {
  if (!Updates.isEmbeddedLaunch) return; // pula em dev

  try {
    const update = await Updates.checkForUpdateAsync();
    if (update.isAvailable) {
      await Updates.fetchUpdateAsync();
      await Updates.reloadAsync(); // reinicia o app com o novo bundle
    }
  } catch (error) {
    // Erro de rede — continua com bundle em cache
    console.warn('Verificação de atualização falhou:', error);
  }
}

// Exibir prompt de atualização para o usuário em vez de reload silencioso
async function promptForUpdate() {
  const update = await Updates.checkForUpdateAsync();
  if (!update.isAvailable) return;

  Alert.alert(
    'Atualização Disponível',
    'Uma nova versão está pronta. Reiniciar agora?',
    [
      { text: 'Depois', style: 'cancel' },
      {
        text: 'Reiniciar',
        onPress: async () => {
          await Updates.fetchUpdateAsync();
          await Updates.reloadAsync();
        },
      },
    ]
  );
}
```

---

## Padrões Comuns e Boas Práticas

### Pipeline CI/CD com GitHub Actions

```yaml
# .github/workflows/deploy.yml
name: Deploy

on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

jobs:
  # Em PR: publica OTA update no canal de preview
  preview:
    if: github.event_name == 'pull_request'
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: 20
          cache: npm
      - run: npm ci
      - uses: expo/expo-github-action@v8
        with:
          eas-version: latest
          token: ${{ secrets.EXPO_TOKEN }}
      - run: eas update --channel preview --message "PR ${{ github.event.number }}" --non-interactive

  # Em merge na main: OTA update para produção
  update:
    if: github.event_name == 'push' && github.ref == 'refs/heads/main'
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: 20
          cache: npm
      - run: npm ci
      - uses: expo/expo-github-action@v8
        with:
          eas-version: latest
          token: ${{ secrets.EXPO_TOKEN }}
      - run: eas update --channel production --message "Deploy ${{ github.sha }}" --non-interactive

  # Em tag de versão: build nativo + submissão nas lojas
  native-build:
    if: startsWith(github.ref, 'refs/tags/v')
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: 20
          cache: npm
      - run: npm ci
      - uses: expo/expo-github-action@v8
        with:
          eas-version: latest
          token: ${{ secrets.EXPO_TOKEN }}
      - run: eas build --platform all --profile production --non-interactive
      - run: eas submit --platform all --latest --non-interactive
```

### Variáveis de Ambiente

```bash
# Definir variáveis de ambiente secretas no EAS
eas secret:create --name API_URL --value "https://api.example.com" --scope project
eas secret:create --name SENTRY_DSN --value "https://xxx@sentry.io/123" --scope project

# Listar segredos
eas secret:list
```

```json
// eas.json — referenciar segredos nos builds
{
  "build": {
    "production": {
      "env": {
        "APP_ENV": "production",
        "API_URL": "https://api.example.com"
      }
    },
    "preview": {
      "env": {
        "APP_ENV": "preview",
        "API_URL": "https://staging-api.example.com"
      }
    }
  }
}
```

```tsx
// Acessar no código do app
const API_URL = process.env.EXPO_PUBLIC_API_URL; // prefixo EXPO_PUBLIC_ = incluído no bundle
```

### Gerenciamento de Versão

```bash
# Incrementar versão do app (atualiza app.json + autoIncrement do eas.json)
eas build:version:set --version-code 42 --platform android
eas build:version:set --build-number 42 --platform ios

# Ou use auto-increment no eas.json
{
  "build": {
    "production": {
      "autoIncrement": true  // incrementa versionCode/buildNumber por build
    }
  }
}
```

---

## Anti-Padrões a Evitar

**Usar OTA updates para mudar permissões nativas.** OTA entrega apenas JS. Se você adicionar uma nova permissão (câmera, localização), deve submeter um build nativo pelo processo de revisão da loja.

**Publicar OTA updates sem testar no canal de preview primeiro.** Sempre publique em um canal de preview, teste em dispositivos reais e depois promova para produção. Um único OTA update com falha afeta todos os seus usuários instantaneamente.

**Commitar credenciais de assinatura no git.** Keystores, provisioning profiles e chaves p8 são segredos. Armazene-os nas credenciais EAS (criptografadas) ou em um gerenciador de segredos — nunca no seu repositório.

**Não definir uma política de runtime version.** Sem runtime version, OTA updates serão instalados em binários nativos incompatíveis, causando crashes. Use `"policy": "appVersion"` para vincular automaticamente a compatibilidade OTA à sua versão nativa.

**Pular a trilha de teste interno no Google Play.** Antes de promover para produção, sempre lance para "Internal testing" e teste o binário real da loja. Artefatos de build de CI e binários da loja podem se comportar de forma diferente (ProGuard, diferenças de assinatura).

**Bloquear a thread principal durante atualizações.** `Updates.reloadAsync()` é destrutivo — mata o app e o reinicia. Nunca o chame sem confirmação do usuário, exceto no primeiro lançamento.

---

## Depuração e Resolução de Problemas

### Falhas de Build

```bash
# Ver logs do build
eas build:view --id <build-id>

# Rodar build localmente para iteração mais rápida
eas build --profile development --platform android --local
# Requer: Android Studio + NDK, ou Xcode para iOS

# Validar eas.json
eas build --help
```

**Falhas comuns de build iOS:**
- `No profiles for bundle ID` — execute `eas credentials` para registrar o bundle ID
- `Code signing error` — certificado expirado; regenere via `eas credentials`
- `Provisioning profile doesn't include device` — adicione UDID via `eas device:create`

**Falhas comuns de build Android:**
- `Keystore not found` — re-suba via `eas credentials`
- `Gradle build failed` — geralmente incompatibilidade de módulo nativo; verifique a compatibilidade do módulo com sua versão do Expo SDK

### Resolução de Problemas de OTA Update

```tsx
// Verificar informações da atualização atual
import * as Updates from 'expo-updates';

console.log({
  updateId: Updates.updateId,
  channel: Updates.channel,
  runtimeVersion: Updates.runtimeVersion,
  isEmbeddedLaunch: Updates.isEmbeddedLaunch,
  manifest: Updates.manifest,
});
```

**Atualização não sendo aplicada:** Verifique se o `runtimeVersion` na atualização publicada corresponde ao binário instalado. Versões de runtime incompatíveis ignoram silenciosamente a atualização.

**"Update is too big":** EAS Update tem um limite de 50MB por bundle de atualização. Use imports dinâmicos ou divida funcionalidades grandes em builds nativos.

### Rejeições de Submissão

**App Store:**
- 4.0 Design — funcionalidade insuficiente; apps de demonstração e shells são rejeitados
- 2.1 Performance crashes — teste no dispositivo mais antigo suportado (verifique seu target de deploy iOS)
- 5.1.1 Privacy — descrições de permissão ausentes em `infoPlist`

**Google Play:**
- Violação de política — use o Policy Center para verificar; comum: link de política de privacidade ausente
- Nível de API alvo — deve ter como alvo o nível de API Android mais recente ou anterior

---

## Cenários do Mundo Real

### Rollout Gradual de OTA

```bash
# Publicar para 10% dos usuários no canal de produção
# (EAS Update não suporta percentuais nativamente — use feature flags)
# Padrão: faça deploy OTA para todos, condicione novas funcionalidades a uma flag do seu servidor
```

### Deploy de Hotfix

```bash
# Correção emergencial: pula preview → direto para OTA de produção
git checkout -b hotfix/crash-on-login
# corrija o bug
git commit -m "fix: null check no auth token"
eas update --channel production --message "Hotfix: null check no auth token" --non-interactive
```

### App Multi-Ambiente

```
Branch        → Canal       → Usuários
────────────────────────────────────────
main          → production  → Todos os usuários
staging       → preview     → Time de QA
feature/*     → development → Dispositivos de dev
```

---

## Distribuição Interna

Antes de submeter às lojas, distribua builds para QA e stakeholders para testes.

**TestFlight (iOS):**

```bash
# Compilar e subir para o TestFlight em um único comando
eas build --platform ios --profile preview
eas submit --platform ios --profile preview
# Testers recebem um convite por e-mail; até 10.000 testers externos
# Grupo interno (até 100 contas Apple) recebe acesso imediato
```

**Firebase App Distribution (Android + iOS):**

```bash
npm install --save-dev @react-native-firebase/app-distribution
```

```json
// eas.json — adicionar um profile de submit para Firebase App Distribution
{
  "submit": {
    "preview": {
      "android": {
        "serviceAccountKeyPath": "./firebase-service-account.json",
        "groups": ["qa-team", "stakeholders"],
        "releaseNotes": "Preview build for QA"
      },
      "ios": {
        "appleId": "your@email.com",
        "ascAppId": "1234567890",
        "groups": ["ios-testers"]
      }
    }
  }
}
```

```bash
eas build --platform android --profile preview
eas submit --platform android --profile preview
# Testers instalam o app Firebase App Tester e recebem uma notificação
```

**Instalação direta de APK (Android — mais rápido para devs internos):**

```bash
# Compilar um APK (não AAB) para sideloading
eas build --platform android --profile preview
# Listar builds recentes e pegar a URL de download
eas build:list --platform android --limit 1
# Compartilhe o link de download diretamente — testers habilitam "Instalar apps desconhecidos"
```

Use TestFlight para testes de marcos no iOS, Firebase para QA multiplataforma e APK direto para dispositivos de desenvolvimento.

---

## Rollouts Graduais no Google Play

Libere para uma porcentagem dos usuários, monitore métricas e depois expanda. Isso limita o impacto caso um build tenha um bug crítico.

```bash
# EAS Submit suporta definir percentual inicial de rollout
eas submit --platform android --profile production
# Depois no Google Play Console → Production → Manage release → Edit rollout percentage
```

**Escada típica de rollout:**
1. 1% — smoke test com usuários reais, observe a taxa de ausência de crashes
2. 5% — sinal mais amplo, verifique a taxa de ANR
3. 20% — tráfego significativo, verifique latência p95 e avaliações
4. 50% → 100% — se todas as métricas estiverem estáveis após 24-48h em cada etapa

**Métricas para monitorar entre as etapas:**
- **Usuários sem crash:** deve permanecer acima de 99,5% (limite do Play Store para rebaixamento)
- **Taxa de ANR:** deve ficar abaixo de 0,47% (limite do Play Store)
- **Delta de avaliação:** observe quedas repentinas em avaliações de 1 estrela
- **Firebase Performance:** tempo de inicialização do app, latência de requisições de rede

**Interrompa o rollout imediatamente se as métricas regredirem:**

```
Google Play Console → Production → Manage release → Halt rollout
```

Isso impede novas instalações da versão atual — instalações existentes não são afetadas. Corrija o problema, publique um novo build e reinicie a escada.

**Apple App Store** não suporta rollouts percentuais para releases padrão. Use feature flags (LaunchDarkly, Statsig ou um remote config simples) para condicionar novas funcionalidades independentemente do release.

---

## Rollback de OTA Update

Se um EAS Update causar regressões, reverta sem precisar de submissão na loja.

```bash
# Ver updates recentes no branch de produção
eas update:list --branch production --limit 10

# Rollback: republique um grupo de update anterior no mesmo branch
eas update:republish --group <previous-update-group-id> --branch production

# Alternativa: publique um novo update a partir de um commit git estável
git checkout <stable-sha>
eas update --branch production --message "rollback: revert to stable"
git checkout -  # volte para o branch de trabalho
```

**Detecte lançamentos OTA no app** para monitorar adoção e integridade:

```typescript
import * as Updates from 'expo-updates';
import analytics from '@segment/analytics-react-native';

export async function trackLaunchContext(): Promise<void> {
  if (!Updates.isEmbeddedLaunch) {
    // App lançado a partir de um OTA update (não o binário da loja)
    await analytics.track('app_ota_launch', {
      updateId: Updates.updateId,
      channel: Updates.channel,
    });
  }
}
```

**EAS Secrets — armazene credenciais fora do repositório:**

```bash
# Armazene arquivos e strings sensíveis no EAS (criptografados, por projeto)
eas secret:create --scope project --name GOOGLE_SERVICES_JSON \
  --type file --value ./google-services.json

eas secret:create --scope project --name SENTRY_DSN \
  --type string --value "https://abc@sentry.io/123"

eas secret:create --scope project --name FIREBASE_APP_ID \
  --type string --value "1:1234567890:ios:abcdef"

# Listar todos os segredos
eas secret:list
```

```json
// eas.json — referenciar segredos como variáveis de ambiente nos builds
{
  "build": {
    "production": {
      "env": {
        "SENTRY_DSN": "$SENTRY_DSN",
        "FIREBASE_APP_ID": "$FIREBASE_APP_ID"
      }
    }
  }
}
```

Os segredos são injetados em tempo de build pelos workers do EAS Build — eles nunca aparecem no seu histórico git ou nos logs de build. Use `--scope account` para segredos compartilhados entre todos os seus projetos.

---

## Leitura Adicional

- [Documentação do EAS Build](https://docs.expo.dev/build/introduction/)
- [Documentação do EAS Submit](https://docs.expo.dev/submit/introduction/)
- [Documentação do EAS Update](https://docs.expo.dev/eas-update/introduction/)
- [Diretrizes de Revisão da App Store](https://developer.apple.com/app-store/review/guidelines/)
- [Google Play Policy Center](https://support.google.com/googleplay/android-developer/answer/9876714)
- [expo-updates API](https://docs.expo.dev/versions/latest/sdk/updates/)

---

## Resumo

EAS Build compila binários nativos na nuvem — sem Xcode ou Android Studio local necessário. Três profiles cobrem os principais casos de uso: `development` (dev client com hot reload), `preview` (build de teste compartilhável), `production` (submissão nas lojas). EAS Submit automatiza a submissão na App Store e Play Store. EAS Update publica OTA updates que contornam a revisão da loja — limitado a mudanças em JS e assets. CI/CD: dispare OTA updates no merge para main, builds nativos em tags de versão. Sempre defina uma política de `runtimeVersion` para evitar que OTA updates sejam instalados em binários nativos incompatíveis. Nunca commite credenciais de assinatura no controle de versão — use o armazenamento de credenciais do EAS.
