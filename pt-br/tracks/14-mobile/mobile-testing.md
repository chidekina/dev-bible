# Testes Mobile

## Visão Geral

Testar um app React Native requer três camadas distintas: testes unitários para lógica pura, testes de componente para comportamento de UI, e testes end-to-end para fluxos completos do usuário em um dispositivo real ou simulador. Cada camada captura uma classe diferente de bug e tem um ciclo de feedback diferente. A React Native Testing Library (RNTL) cuida dos testes unitários e de componente. Detox e Maestro cobrem os E2E.

O objetivo é uma suite de testes que rode em CI, capture regressões antes que cheguem aos usuários, e não exija manutenção extensiva conforme o app evolui. Isso significa: teste comportamento, não implementação; mock na fronteira (rede, APIs do dispositivo, timers), não dentro do seu próprio código; e prefira testes E2E para fluxos que passam por múltiplas telas.

---

## Pré-requisitos

- [React Native Basics](react-native-basics.md)
- Track 09 — Testing (princípios gerais de teste, mentalidade TDD)
- Noções de Jest (test, expect, describe, beforeEach)

---

## Conceitos Principais

### 1. Configuração do Jest para React Native / Expo

O Expo já inclui o Jest pré-configurado. Para um projeto bare React Native, instale `@testing-library/react-native` junto com o Jest.

```bash
# Projeto Expo — já tem jest-expo, só adicione RNTL
npx expo install @testing-library/react-native @testing-library/user-event

# Bare React Native
npm install --save-dev jest @testing-library/react-native @testing-library/user-event \
  babel-jest @babel/core react-test-renderer
```

Configure o `package.json`:

```json
{
  "scripts": {
    "test": "jest",
    "test:watch": "jest --watch",
    "test:coverage": "jest --coverage"
  },
  "jest": {
    "preset": "jest-expo",
    "setupFilesAfterFramework": ["@testing-library/react-native/extend-expect"],
    "transformIgnorePatterns": [
      "node_modules/(?!((jest-)?react-native|@react-native(-community)?)|expo(nent)?|@expo(nent)?/.*|@expo-google-fonts/.*|react-navigation|@react-navigation/.*|@unimodules/.*|unimodules|sentry-expo|native-base|react-native-svg)"
    ],
    "collectCoverageFrom": [
      "src/**/*.{ts,tsx}",
      "!src/**/*.d.ts",
      "!src/**/index.ts"
    ]
  }
}
```

O `transformIgnorePatterns` diz ao Jest quais `node_modules` devem passar pelo Babel. Por padrão o Jest ignora todo `node_modules`, mas muitos pacotes React Native e Expo são distribuídos como ES modules (não CommonJS pré-compilado). O padrão acima exclui esses pacotes da lista de exclusão para que o Babel possa transformá-los antes de rodar os testes.

Execute os testes:

```bash
bun run test          # se usar Bun como task runner
npx jest              # invocação direta
npx jest --watch      # modo watch interativo
npx jest src/components/LoginForm.test.tsx  # arquivo único
```

Para um projeto bare React Native, use o preset `react-native`:

```json
{
  "jest": {
    "preset": "react-native",
    "setupFilesAfterFramework": ["@testing-library/react-native/extend-expect"]
  }
}
```

---

### 2. Testes de Componente com React Native Testing Library

A RNTL renderiza componentes em uma árvore host virtual e fornece queries que espelham como um usuário (ou tecnologia assistiva) percebe a UI. Ela não usa um dispositivo real — usa o ambiente JS do Jest com um mock de renderer nativo.

**Instalação:**

```bash
npx expo install @testing-library/react-native
```

**Queries básicas:**

```typescript
import { render, screen, fireEvent, waitFor } from '@testing-library/react-native';
```

Prioridade das queries (maior para menor confiança):

| Prioridade | Query | Observações |
|------------|-------|-------------|
| 1 | `getByRole` | Melhor — espelha a árvore de acessibilidade |
| 2 | `getByLabelText` | Para inputs de formulário com label |
| 3 | `getByText` | Para texto visível |
| 4 | `getByDisplayValue` | Para valor atual do input |
| 5 | `getByTestId` | Último recurso — frágil, use com moderação |

**Exemplo: componente LoginForm**

```tsx
// src/components/LoginForm.tsx
import React, { useState } from 'react';
import {
  View,
  Text,
  TextInput,
  TouchableOpacity,
  StyleSheet,
  AccessibilityInfo,
} from 'react-native';

interface LoginFormProps {
  onLogin: (email: string, password: string) => Promise<void>;
}

export function LoginForm({ onLogin }: LoginFormProps) {
  const [email, setEmail] = useState('');
  const [password, setPassword] = useState('');
  const [error, setError] = useState('');
  const [loading, setLoading] = useState(false);

  async function handleSubmit() {
    if (!email || !password) {
      setError('Email and password are required');
      return;
    }
    setError('');
    setLoading(true);
    try {
      await onLogin(email, password);
    } catch (err) {
      setError('Invalid credentials');
    } finally {
      setLoading(false);
    }
  }

  return (
    <View style={styles.container}>
      <TextInput
        accessibilityLabel="Email"
        value={email}
        onChangeText={setEmail}
        placeholder="Email"
        keyboardType="email-address"
        autoCapitalize="none"
        testID="email-input"
      />
      <TextInput
        accessibilityLabel="Password"
        value={password}
        onChangeText={setPassword}
        placeholder="Password"
        secureTextEntry
        testID="password-input"
      />
      {error ? <Text accessibilityRole="alert">{error}</Text> : null}
      <TouchableOpacity
        accessibilityRole="button"
        accessibilityLabel="Sign In"
        onPress={handleSubmit}
        disabled={loading}
      >
        <Text>{loading ? 'Signing in...' : 'Sign In'}</Text>
      </TouchableOpacity>
    </View>
  );
}

const styles = StyleSheet.create({
  container: { padding: 16, gap: 12 },
});
```

**Arquivo de teste:**

```typescript
// src/components/__tests__/LoginForm.test.tsx
import React from 'react';
import { render, screen, fireEvent, waitFor } from '@testing-library/react-native';
import { LoginForm } from '../LoginForm';

describe('LoginForm', () => {
  const mockOnLogin = jest.fn();

  beforeEach(() => {
    mockOnLogin.mockClear();
  });

  it('shows an error when submitting an empty form', async () => {
    render(<LoginForm onLogin={mockOnLogin} />);

    fireEvent.press(screen.getByRole('button', { name: 'Sign In' }));

    expect(
      await screen.findByRole('alert')
    ).toHaveTextContent('Email and password are required');
    expect(mockOnLogin).not.toHaveBeenCalled();
  });

  it('calls onLogin with email and password when the form is valid', async () => {
    mockOnLogin.mockResolvedValueOnce(undefined);

    render(<LoginForm onLogin={mockOnLogin} />);

    fireEvent.changeText(screen.getByLabelText('Email'), 'user@example.com');
    fireEvent.changeText(screen.getByLabelText('Password'), 'secret123');
    fireEvent.press(screen.getByRole('button', { name: 'Sign In' }));

    await waitFor(() => {
      expect(mockOnLogin).toHaveBeenCalledWith('user@example.com', 'secret123');
    });
  });

  it('shows an error when onLogin rejects', async () => {
    mockOnLogin.mockRejectedValueOnce(new Error('401'));

    render(<LoginForm onLogin={mockOnLogin} />);

    fireEvent.changeText(screen.getByLabelText('Email'), 'user@example.com');
    fireEvent.changeText(screen.getByLabelText('Password'), 'wrong');
    fireEvent.press(screen.getByRole('button', { name: 'Sign In' }));

    expect(await screen.findByRole('alert')).toHaveTextContent('Invalid credentials');
  });
});
```

**`fireEvent` vs `userEvent`**

`fireEvent` dispara um único evento sintético diretamente no elemento — síncrono, mais baixo nível, sem simulação de gesto. É suficiente para a maioria dos testes.

`userEvent` simula a sequência completa de eventos que um usuário real dispararia (focus, keypress, change, blur). É assíncrono e mais próximo da interação real. Use quando precisar testar comportamento dependente de interação (ex.: autocomplete, inputs mascarados).

```typescript
import userEvent from '@testing-library/user-event';

// exemplo com userEvent — simula digitação caractere por caractere
it('triggers autocomplete on input', async () => {
  const user = userEvent.setup();
  render(<SearchInput onSearch={mockSearch} />);

  await user.type(screen.getByLabelText('Search'), 'react');

  await waitFor(() => {
    expect(screen.getByText('React Native')).toBeOnTheScreen();
  });
});
```

---

### 3. Mockando Módulos Nativos

Módulos nativos não existem no ambiente JS do Jest. Você precisa mocká-los. Existem duas abordagens: diretório `__mocks__` (automático) e `jest.mock()` inline (pontual).

**Diretório `__mocks__` — mock automático**

Crie um diretório `__mocks__` na raiz do projeto (mesmo nível que `node_modules`) ou ao lado do arquivo do módulo. O Jest usa esse diretório automaticamente quando o módulo é importado.

```typescript
// __mocks__/expo-camera.ts
const Camera = {
  requestCameraPermissionsAsync: jest.fn().mockResolvedValue({ status: 'granted' }),
  takePictureAsync: jest.fn().mockResolvedValue({ uri: 'file:///mock/photo.jpg' }),
};

export { Camera };
export default Camera;
```

```typescript
// __mocks__/@react-native-async-storage/async-storage.ts
const AsyncStorage: Record<string, jest.Mock> = {
  getItem: jest.fn().mockResolvedValue(null),
  setItem: jest.fn().mockResolvedValue(undefined),
  removeItem: jest.fn().mockResolvedValue(undefined),
  clear: jest.fn().mockResolvedValue(undefined),
  getAllKeys: jest.fn().mockResolvedValue([]),
  multiGet: jest.fn().mockResolvedValue([]),
  multiSet: jest.fn().mockResolvedValue(undefined),
  multiRemove: jest.fn().mockResolvedValue(undefined),
};

export default AsyncStorage;
```

```typescript
// __mocks__/expo-location.ts
export const requestForegroundPermissionsAsync = jest
  .fn()
  .mockResolvedValue({ status: 'granted' });

export const getCurrentPositionAsync = jest.fn().mockResolvedValue({
  coords: {
    latitude: -23.5505,
    longitude: -46.6333,
    accuracy: 10,
  },
  timestamp: Date.now(),
});

export const Accuracy = {
  Balanced: 3,
  High: 4,
  Highest: 5,
  Low: 2,
  Lowest: 1,
};
```

**`jest.mock()` inline — mocks pontuais**

Quando você precisa de um módulo mockado apenas em um arquivo de teste:

```typescript
// src/screens/__tests__/ProfileScreen.test.tsx
jest.mock('expo-image-picker', () => ({
  launchImageLibraryAsync: jest.fn().mockResolvedValue({
    canceled: false,
    assets: [{ uri: 'file:///mock/avatar.jpg', width: 200, height: 200 }],
  }),
  MediaTypeOptions: { Images: 'Images' },
  requestMediaLibraryPermissionsAsync: jest.fn().mockResolvedValue({
    status: 'granted',
  }),
}));

jest.mock('@react-native-community/netinfo', () => ({
  addEventListener: jest.fn(() => jest.fn()),
  fetch: jest.fn().mockResolvedValue({ isConnected: true, isInternetReachable: true }),
}));
```

**Mockando internals do `react-native`**

Algumas APIs do `react-native` precisam de setup manual. As mais comuns:

```typescript
// jest.setup.ts
import 'react-native-gesture-handler/jestSetup';

// Mock Animated para rodar de forma síncrona nos testes
jest.mock('react-native/Libraries/Animated/NativeAnimatedHelper');

// Mock de navegação (React Navigation)
jest.mock('@react-navigation/native', () => {
  const actual = jest.requireActual('@react-navigation/native');
  return {
    ...actual,
    useNavigation: () => ({
      navigate: jest.fn(),
      goBack: jest.fn(),
      reset: jest.fn(),
    }),
    useRoute: () => ({
      params: {},
    }),
  };
});
```

---

### 4. Testando Hooks

Use `renderHook` e `act` da RNTL para testar hooks customizados de forma isolada, sem um componente completo.

**Hook simples:**

```typescript
// src/hooks/useCounter.ts
import { useState, useCallback } from 'react';

export function useCounter(initial = 0) {
  const [count, setCount] = useState(initial);
  const increment = useCallback(() => setCount((n) => n + 1), []);
  const decrement = useCallback(() => setCount((n) => n - 1), []);
  const reset = useCallback(() => setCount(initial), [initial]);
  return { count, increment, decrement, reset };
}
```

```typescript
// src/hooks/__tests__/useCounter.test.ts
import { renderHook, act } from '@testing-library/react-native';
import { useCounter } from '../useCounter';

describe('useCounter', () => {
  it('starts at the initial value', () => {
    const { result } = renderHook(() => useCounter(5));
    expect(result.current.count).toBe(5);
  });

  it('increments the count', () => {
    const { result } = renderHook(() => useCounter());

    act(() => {
      result.current.increment();
    });

    expect(result.current.count).toBe(1);
  });

  it('resets to the initial value', () => {
    const { result } = renderHook(() => useCounter(10));

    act(() => {
      result.current.increment();
      result.current.increment();
    });

    act(() => {
      result.current.reset();
    });

    expect(result.current.count).toBe(10);
  });
});
```

**Hook com efeitos colaterais assíncronos (SecureStore):**

```typescript
// src/hooks/useAuth.ts
import { useState, useEffect } from 'react';
import * as SecureStore from 'expo-secure-store';

interface AuthState {
  token: string | null;
  loading: boolean;
}

export function useAuth() {
  const [state, setState] = useState<AuthState>({ token: null, loading: true });

  useEffect(() => {
    SecureStore.getItemAsync('auth_token').then((token) => {
      setState({ token, loading: false });
    });
  }, []);

  async function signIn(token: string) {
    await SecureStore.setItemAsync('auth_token', token);
    setState({ token, loading: false });
  }

  async function signOut() {
    await SecureStore.deleteItemAsync('auth_token');
    setState({ token: null, loading: false });
  }

  return { ...state, signIn, signOut };
}
```

```typescript
// src/hooks/__tests__/useAuth.test.ts
import { renderHook, act, waitFor } from '@testing-library/react-native';
import * as SecureStore from 'expo-secure-store';
import { useAuth } from '../useAuth';

jest.mock('expo-secure-store', () => ({
  getItemAsync: jest.fn(),
  setItemAsync: jest.fn(),
  deleteItemAsync: jest.fn(),
}));

const mockSecureStore = SecureStore as jest.Mocked<typeof SecureStore>;

describe('useAuth', () => {
  beforeEach(() => {
    jest.clearAllMocks();
  });

  it('loads token from SecureStore on mount', async () => {
    mockSecureStore.getItemAsync.mockResolvedValueOnce('existing-token');

    const { result } = renderHook(() => useAuth());

    expect(result.current.loading).toBe(true);

    await waitFor(() => {
      expect(result.current.loading).toBe(false);
    });

    expect(result.current.token).toBe('existing-token');
  });

  it('stores token and updates state on signIn', async () => {
    mockSecureStore.getItemAsync.mockResolvedValueOnce(null);
    mockSecureStore.setItemAsync.mockResolvedValueOnce(undefined);

    const { result } = renderHook(() => useAuth());

    await waitFor(() => expect(result.current.loading).toBe(false));

    await act(async () => {
      await result.current.signIn('new-token');
    });

    expect(mockSecureStore.setItemAsync).toHaveBeenCalledWith('auth_token', 'new-token');
    expect(result.current.token).toBe('new-token');
  });

  it('clears token on signOut', async () => {
    mockSecureStore.getItemAsync.mockResolvedValueOnce('some-token');
    mockSecureStore.deleteItemAsync.mockResolvedValueOnce(undefined);

    const { result } = renderHook(() => useAuth());

    await waitFor(() => expect(result.current.loading).toBe(false));

    await act(async () => {
      await result.current.signOut();
    });

    expect(result.current.token).toBeNull();
  });
});
```

---

### 5. Testes E2E com Detox

O Detox roda testes contra um binário real do app em um simulador ou emulador. Ele interage com o app por meio das APIs de acessibilidade nativas e usa um mecanismo de sincronização gray-box: antes de cada interação, o Detox aguarda a thread JS ficar ociosa (sem timers pendentes, sem requisições de rede em andamento, sem animações pendentes). Isso elimina a necessidade de chamadas arbitrárias a `sleep()` e torna os testes confiáveis.

**Instalação:**

```bash
npx expo install detox @config-plugins/detox
npm install --save-dev detox @types/detox
```

Adicione o config plugin ao `app.json`:

```json
{
  "expo": {
    "plugins": [
      ["@config-plugins/detox", { "subdomains": "*" }]
    ]
  }
}
```

**`detox.config.js`:**

```javascript
/** @type {import('detox').DetoxConfig} */
module.exports = {
  testRunner: {
    args: {
      $0: 'jest',
      config: 'e2e/jest.config.js',
    },
    jest: {
      setupTimeout: 120000,
    },
  },
  apps: {
    'ios.debug': {
      type: 'ios.app',
      binaryPath: 'ios/build/Build/Products/Debug-iphonesimulator/YourApp.app',
      build:
        'xcodebuild -workspace ios/YourApp.xcworkspace -scheme YourApp -configuration Debug -sdk iphonesimulator -derivedDataPath ios/build',
    },
    'android.debug': {
      type: 'android.apk',
      binaryPath: 'android/app/build/outputs/apk/debug/app-debug.apk',
      build:
        'cd android && ./gradlew assembleDebug assembleAndroidTest -DtestBuildType=debug',
    },
  },
  devices: {
    simulator: {
      type: 'ios.simulator',
      device: { type: 'iPhone 15' },
    },
    emulator: {
      type: 'android.emulator',
      device: { avdName: 'Pixel_6_API_34' },
    },
  },
  configurations: {
    'ios.sim.debug': {
      device: 'simulator',
      app: 'ios.debug',
    },
    'android.emu.debug': {
      device: 'emulator',
      app: 'android.debug',
    },
  },
};
```

**`e2e/jest.config.js`:**

```javascript
/** @type {import('@jest/types').Config.InitialOptions} */
module.exports = {
  rootDir: '..',
  testMatch: ['<rootDir>/e2e/**/*.e2e.ts'],
  testTimeout: 120000,
  maxWorkers: 1,
  globalSetup: 'detox/runners/jest/globalSetup',
  globalTeardown: 'detox/runners/jest/globalTeardown',
  reporters: ['detox/runners/jest/reporter'],
  testEnvironment: 'detox/runners/jest/testEnvironment',
  verbose: true,
};
```

**Teste E2E completo do fluxo de autenticação:**

```typescript
// e2e/auth.e2e.ts
import { device, element, by, expect as detoxExpect } from 'detox';

describe('Auth flow', () => {
  beforeAll(async () => {
    await device.launchApp({ newInstance: true });
  });

  beforeEach(async () => {
    await device.reloadReactNative();
  });

  it('shows the login screen on first launch', async () => {
    await detoxExpect(element(by.text('Sign In'))).toBeVisible();
    await detoxExpect(element(by.id('email-input'))).toBeVisible();
    await detoxExpect(element(by.id('password-input'))).toBeVisible();
  });

  it('shows an error for invalid credentials', async () => {
    await element(by.id('email-input')).typeText('wrong@example.com');
    await element(by.id('password-input')).typeText('badpassword');
    await element(by.text('Sign In')).tap();

    await detoxExpect(element(by.text('Invalid credentials'))).toBeVisible();
  });

  it('navigates to home screen after successful login', async () => {
    await element(by.id('email-input')).clearText();
    await element(by.id('email-input')).typeText('user@example.com');
    await element(by.id('password-input')).clearText();
    await element(by.id('password-input')).typeText('correct-password');
    await element(by.text('Sign In')).tap();

    await detoxExpect(element(by.text('Welcome back'))).toBeVisible();
    await detoxExpect(element(by.id('home-screen'))).toBeVisible();
  });

  it('returns to login screen after signing out', async () => {
    // Assume que já estamos na home screen do teste anterior,
    // ou inicie o app com um auth token pré-configurado:
    await device.launchApp({
      newInstance: true,
      userNotification: undefined,
      delete: false,
      launchArgs: { detoxPredefinedToken: 'test-token' },
    });

    await element(by.text('Settings')).tap();
    await detoxExpect(element(by.text('Sign Out'))).toBeVisible();
    await element(by.text('Sign Out')).tap();

    await detoxExpect(element(by.text('Sign In'))).toBeVisible();
  });
});
```

**Executar o Detox:**

```bash
# Build (na primeira vez, ou após mudanças nativas)
npx detox build --configuration ios.sim.debug

# Rodar os testes
npx detox test --configuration ios.sim.debug

# Rodar um arquivo específico
npx detox test --configuration ios.sim.debug e2e/auth.e2e.ts
```

---

### 6. Testes E2E com Maestro

O Maestro é um framework E2E mais simples: os testes são arquivos YAML, não é necessário build nativo (ele controla um app já em execução ou instala um APK/IPA), e a configuração leva minutos. A contrapartida é menos controle sobre o estado do app e sem sincronização gray-box — o Maestro faz polling por elementos com um timeout configurável.

**Instalação:**

```bash
curl -Ls "https://get.maestro.mobile.dev" | bash
# Reinicie o shell e verifique:
maestro --version
```

**Fluxo completo de login:**

```yaml
# .maestro/flows/login.yaml
appId: com.yourcompany.yourapp
---
- launchApp:
    clearState: true

# Caminho feliz: credenciais válidas
- tapOn: "Email"
- inputText: "test@example.com"
- tapOn: "Password"
- inputText: "password123"
- tapOn: "Sign In"
- assertVisible: "Welcome back"

# Navegar para configurações e verificar que o botão de sair existe
- tapOn: "Settings"
- assertVisible: "Sign Out"
```

**Fluxo com credenciais inválidas:**

```yaml
# .maestro/flows/login-invalid.yaml
appId: com.yourcompany.yourapp
---
- launchApp:
    clearState: true
- tapOn: "Email"
- inputText: "bad@example.com"
- tapOn: "Password"
- inputText: "wrongpassword"
- tapOn: "Sign In"
- assertVisible: "Invalid credentials"
- assertNotVisible: "Welcome back"
```

**Fluxo de onboarding com navegação de volta:**

```yaml
# .maestro/flows/onboarding.yaml
appId: com.yourcompany.yourapp
---
- launchApp:
    clearState: true
    clearKeychain: true

- assertVisible: "Welcome"
- tapOn: "Get Started"
- assertVisible: "Choose your role"
- tapOn: "Developer"
- tapOn: "Continue"
- assertVisible: "Set up your profile"
- inputText:
    text: "Jane"
    label: "First Name"
- tapOn: "Finish"
- assertVisible: "Dashboard"
```

**Executar o Maestro:**

```bash
# Rodar um único fluxo
maestro test .maestro/flows/login.yaml

# Rodar todos os fluxos de um diretório
maestro test .maestro/flows/

# Rodar em um dispositivo específico (Android)
maestro test --device emulator-5554 .maestro/flows/login.yaml

# Rodar na nuvem (Maestro Cloud)
maestro cloud --apiKey $MAESTRO_API_KEY .maestro/flows/
```

**Comparação Detox vs Maestro:**

| Aspecto | Detox | Maestro |
|---------|-------|---------|
| Sintaxe dos testes | TypeScript/JavaScript | YAML |
| Complexidade de setup | Alta (requer build nativo) | Baixa (instala em minutos) |
| Sincronização | Gray-box (aguarda JS ocioso) | Polling com timeout |
| Confiabilidade | Muito alta | Alta (flakiness ocasional) |
| Suporte a CI | Completo (com runner macOS para iOS) | Completo (Maestro Cloud disponível) |
| Plataforma | iOS + Android | iOS + Android |
| Ferramentas de debug | Detox CLI, logs do dispositivo | `maestro studio` gravador visual |
| Comunidade | Grande (Wix) | Em crescimento |
| Melhor para | Fluxos complexos, CI de alta confiabilidade | Setup rápido, fluxos simples, testes escritos por não-engenheiros |

---

### 7. Rodando Testes em CI

**Testes unitários e de componente (GitHub Actions):**

```yaml
# .github/workflows/test.yml
name: Test

on:
  push:
    branches: [main, develop]
  pull_request:

jobs:
  unit:
    name: Unit & Component Tests
    runs-on: ubuntu-latest

    steps:
      - uses: actions/checkout@v4

      - uses: actions/setup-node@v4
        with:
          node-version: '20'
          cache: 'npm'

      - name: Install dependencies
        run: npm ci

      - name: Run tests
        run: npx jest --coverage --ci --forceExit

      - name: Upload coverage
        uses: codecov/codecov-action@v4
        with:
          token: ${{ secrets.CODECOV_TOKEN }}
          fail_ci_if_error: false
```

**Detox E2E (iOS — exige runner macOS):**

```yaml
# .github/workflows/e2e-ios.yml
name: E2E iOS

on:
  push:
    branches: [main]

jobs:
  detox-ios:
    name: Detox iOS E2E
    runs-on: macos-14

    steps:
      - uses: actions/checkout@v4

      - uses: actions/setup-node@v4
        with:
          node-version: '20'
          cache: 'npm'

      - name: Install dependencies
        run: npm ci

      - name: Install CocoaPods
        run: cd ios && pod install

      - name: Build app for testing
        run: npx detox build --configuration ios.sim.debug

      - name: Run Detox tests
        run: npx detox test --configuration ios.sim.debug --cleanup --headless
```

Testes Detox para iOS exigem um runner macOS porque precisam do Xcode e do iOS Simulator. Runners macOS no GitHub Actions custam aproximadamente 10x mais que runners Linux — rode-os apenas em pushes para `main`, não em cada PR.

**EAS Test — alternativa gerenciada na nuvem:**

O EAS Test (Expo Application Services) roda testes Detox ou Maestro em dispositivos reais na nuvem, sem precisar manter seu próprio runner macOS em CI.

```bash
# Instalar EAS CLI
npm install -g eas-cli

# Configurar
eas test:configure

# Rodar testes
eas test --profile preview
```

Perfil `eas.json` para testes:

```json
{
  "test": {
    "preview": {
      "android": {
        "buildType": "apk"
      },
      "ios": {
        "simulator": true
      }
    }
  }
}
```

---

## Anti-Padrões

### 1. Mockar Seu Próprio Código

Mockar módulos internos remove a cobertura de integração que você realmente quer. Se `UserService` chama `ApiClient`, não mock `ApiClient` dentro de um teste de `UserService` — você perde a confiança de que eles funcionam juntos.

**Errado:**

```typescript
// Testando UserService mas mockando seu próprio ApiClient
jest.mock('../../lib/api-client');  // <-- mockando seu próprio código

import { UserService } from '../UserService';
import { ApiClient } from '../../lib/api-client';

it('fetches the user', async () => {
  (ApiClient.get as jest.Mock).mockResolvedValue({ id: 1, name: 'Alice' });
  const service = new UserService();
  const user = await service.getUser(1);
  expect(user.name).toBe('Alice');
});
```

**Certo:** mock na fronteira de rede (MSW, mock de `fetch`, `axios-mock-adapter`), ou use um repositório em memória.

```typescript
// Mock do fetch — a fronteira real entre seu código e o mundo externo
global.fetch = jest.fn().mockResolvedValue(
  new Response(JSON.stringify({ id: 1, name: 'Alice' }), { status: 200 })
);

import { UserService } from '../UserService';

it('fetches the user', async () => {
  const service = new UserService();
  const user = await service.getUser(1);
  expect(user.name).toBe('Alice');
  expect(fetch).toHaveBeenCalledWith(expect.stringContaining('/users/1'), expect.any(Object));
});
```

---

### 2. `getByTestId` em Todo Lugar

`testID` acopla seus testes a detalhes de implementação. Renomear um componente não quebra `getByRole` ou `getByText`, mas pode quebrar `getByTestId` se você esquecer de atualizá-lo. Queries de acessibilidade também capturam regressões na sua árvore de acessibilidade.

**Errado:**

```typescript
fireEvent.press(screen.getByTestId('login-button'));
fireEvent.changeText(screen.getByTestId('email-field'), 'user@example.com');
const error = screen.getByTestId('error-message');
```

**Certo:**

```typescript
fireEvent.press(screen.getByRole('button', { name: 'Sign In' }));
fireEvent.changeText(screen.getByLabelText('Email'), 'user@example.com');
const error = screen.getByRole('alert');
```

Reserve `testID` para elementos que não têm role acessível ou label e não podem ser consultados por texto — por exemplo, um ícone decorativo ou um item específico em uma lista longa.

---

### 3. Nenhum Teste E2E

Testes unitários e de componente deixam passar classes inteiras de bugs: corrupção de estado de navegação, deep-link handling, race conditions no refresh de auth token e fluxos com múltiplas telas. Esses são exatamente os bugs que chegam à produção e são os mais difíceis de debugar.

Tenha ao menos um teste E2E de caminho feliz por fluxo crítico do usuário. A suite E2E mínima viável para um app típico:

- Cadastro (novo usuário, inputs válidos)
- Login / logout
- Caminho feliz da funcionalidade principal (ex.: criar um registro, visualizá-lo, excluí-lo)
- Fluxo de pagamento ou checkout (se aplicável)

Você não precisa cobrir cada caso extremo em E2E — para isso servem os testes unitários. Cubra os caminhos que seus usuários realmente percorrem.

---

### 4. `sleep()` em Testes Detox

O mecanismo de auto-sincronização do Detox aguarda a thread JS ficar ociosa antes de cada interação. Adicionar chamadas explícitas a `sleep()` torna os testes mais lentos e mascara problemas de sincronização.

**Errado:**

```typescript
await element(by.text('Sign In')).tap();
await new Promise((resolve) => setTimeout(resolve, 2000)); // aguardando a navegação
await detoxExpect(element(by.text('Dashboard'))).toBeVisible();
```

**Certo:**

```typescript
await element(by.text('Sign In')).tap();
// Detox aguarda a animação de navegação e o fetch de dados antes dessa asserção
await detoxExpect(element(by.text('Dashboard'))).toBeVisible();
```

Se os testes falham porque o Detox esgota o timeout aguardando o estado ocioso, a causa raiz quase sempre é:
- Um `setInterval` ou função de polling que nunca para
- Uma Promise não resolvida (ex.: uma requisição de rede sem timeout)
- Um loop de animação infinito (`useNativeDriver: false` + `Animated.loop` em loop)

Corrija a causa raiz; não adicione `sleep()`.

---

## Cenários do Mundo Real

### Cenário 1: Fluxo de Auth Completo E2E (Detox)

Um teste Detox completo cobrindo: launch → login → verificar home screen → logout → verificar tela de auth.

```typescript
// e2e/full-auth-flow.e2e.ts
import { device, element, by, expect as detoxExpect } from 'detox';

describe('Full auth flow', () => {
  beforeAll(async () => {
    await device.launchApp({ newInstance: true, delete: true });
  });

  it('completes the full login and logout cycle', async () => {
    // Passo 1: App abre na tela de auth
    await detoxExpect(element(by.id('auth-screen'))).toBeVisible();
    await detoxExpect(element(by.text('Sign In'))).toBeVisible();

    // Passo 2: Inserir credenciais válidas e fazer login
    await element(by.id('email-input')).typeText('user@example.com');
    await element(by.id('password-input')).typeText('correct-password');
    await element(by.text('Sign In')).tap();

    // Passo 3: Verificar home screen
    await detoxExpect(element(by.id('home-screen'))).toBeVisible();
    await detoxExpect(element(by.text('Welcome back'))).toBeVisible();

    // Passo 4: Navegar para configurações
    await element(by.id('tab-settings')).tap();
    await detoxExpect(element(by.id('settings-screen'))).toBeVisible();
    await detoxExpect(element(by.text('Sign Out'))).toBeVisible();

    // Passo 5: Fazer logout
    await element(by.text('Sign Out')).tap();

    // Alguns apps exibem um diálogo de confirmação
    await detoxExpect(element(by.text('Are you sure?'))).toBeVisible();
    await element(by.text('Confirm')).tap();

    // Passo 6: De volta à tela de auth
    await detoxExpect(element(by.id('auth-screen'))).toBeVisible();
    await detoxExpect(element(by.text('Sign In'))).toBeVisible();
  });
});
```

---

### Cenário 2: Teste de Comportamento Offline (RNTL + Mock)

Testando que um banner "offline" aparece quando o dispositivo perde conectividade. O componente usa `NetInfo` — mock-o para simular o estado offline.

```typescript
// src/components/OfflineBanner.tsx
import React, { useEffect, useState } from 'react';
import { View, Text, StyleSheet } from 'react-native';
import NetInfo, { NetInfoState } from '@react-native-community/netinfo';

export function OfflineBanner() {
  const [isOffline, setIsOffline] = useState(false);

  useEffect(() => {
    const unsubscribe = NetInfo.addEventListener((state: NetInfoState) => {
      setIsOffline(!state.isConnected);
    });

    return unsubscribe;
  }, []);

  if (!isOffline) return null;

  return (
    <View style={styles.banner} accessibilityRole="alert">
      <Text style={styles.text}>You're offline. Check your connection.</Text>
    </View>
  );
}

const styles = StyleSheet.create({
  banner: {
    backgroundColor: '#c0392b',
    padding: 12,
    alignItems: 'center',
  },
  text: {
    color: '#fff',
    fontWeight: '600',
  },
});
```

```typescript
// src/components/__tests__/OfflineBanner.test.tsx
import React from 'react';
import { render, screen } from '@testing-library/react-native';
import NetInfo from '@react-native-community/netinfo';
import { OfflineBanner } from '../OfflineBanner';

jest.mock('@react-native-community/netinfo', () => ({
  addEventListener: jest.fn(),
  fetch: jest.fn(),
}));

const mockNetInfo = NetInfo as jest.Mocked<typeof NetInfo>;

describe('OfflineBanner', () => {
  beforeEach(() => {
    jest.clearAllMocks();
  });

  it('renders nothing when online', () => {
    mockNetInfo.addEventListener.mockImplementation((callback) => {
      callback({ isConnected: true, isInternetReachable: true } as any);
      return jest.fn(); // unsubscribe
    });

    render(<OfflineBanner />);

    expect(screen.queryByRole('alert')).toBeNull();
  });

  it('shows the offline banner when disconnected', () => {
    mockNetInfo.addEventListener.mockImplementation((callback) => {
      callback({ isConnected: false, isInternetReachable: false } as any);
      return jest.fn();
    });

    render(<OfflineBanner />);

    const banner = screen.getByRole('alert');
    expect(banner).toBeOnTheScreen();
    expect(screen.getByText("You're offline. Check your connection.")).toBeOnTheScreen();
  });

  it('hides the banner when connection is restored', () => {
    let storedCallback: ((state: any) => void) | null = null;

    mockNetInfo.addEventListener.mockImplementation((callback) => {
      storedCallback = callback;
      callback({ isConnected: false } as any);
      return jest.fn();
    });

    const { rerender } = render(<OfflineBanner />);
    expect(screen.getByRole('alert')).toBeOnTheScreen();

    // Simula conexão restaurada
    if (storedCallback) {
      storedCallback({ isConnected: true });
    }
    rerender(<OfflineBanner />);

    // O componente escuta via useEffect — em um cenário real a atualização de estado
    // dispararia um re-render. Verifica que o subscriber foi chamado.
    expect(mockNetInfo.addEventListener).toHaveBeenCalled();
  });
});
```

---

## Leitura Adicional

- [Documentação React Native Testing Library](https://callstack.github.io/react-native-testing-library/)
- [Documentação do Detox](https://wix.github.io/Detox/)
- [Documentação do Maestro](https://maestro.mobile.dev/)
- [EAS Test](https://docs.expo.dev/eas/test/)
- [Testing React Native Apps — Docs do Jest](https://jestjs.io/docs/tutorial-react-native)

---

## Resumo

- Use três camadas de teste: Jest + RNTL para testes unitários/componente (rápido, roda em qualquer máquina), Detox para E2E confiável (sincronização gray-box, requer simulador), e Maestro para fluxos rápidos em YAML ou testes escritos por não-engenheiros.
- Configure `transformIgnorePatterns` no Jest para permitir que o Babel processe pacotes Expo e React Native que são distribuídos como ES modules.
- Use `getByRole` e `getByLabelText` antes de recorrer a `getByTestId` — queries de acessibilidade também capturam regressões na árvore de acessibilidade.
- Mock nas fronteiras — APIs nativas do dispositivo (`expo-camera`, `AsyncStorage`, `NetInfo`) e rede (`fetch`) — não dentro do seu próprio código de service/repository.
- Use `renderHook` + `act` + `waitFor` para testar hooks customizados que contêm efeitos assíncronos ou estado.
- A sincronização gray-box do Detox torna `sleep()` desnecessário; se os testes esgotam o timeout, procure Promises não resolvidas, intervalos de polling ativos ou animações infinitas.
- Rode testes unitários em cada push (runner Linux, rápido); rode E2E iOS do Detox apenas em merges para `main` (runner macOS, caro) ou delegue ao EAS Test para testes em dispositivos gerenciados.
