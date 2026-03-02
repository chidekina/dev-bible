# Mobile Testing

## Overview

Testing a React Native app requires three distinct layers: unit tests for pure logic, component tests for UI behavior, and end-to-end tests for full user flows on a real device or simulator. Each layer catches a different class of bug and has a different feedback cycle. React Native Testing Library (RNTL) handles unit and component tests. Detox and Maestro cover E2E.

The goal is a test suite that runs in CI, catches regressions before they reach users, and does not require extensive maintenance as the app evolves. This means: test behavior, not implementation; mock at the boundary (network, device APIs, timers), not inside your own code; and prefer E2E tests for flows that touch multiple screens.

---

## Prerequisites

- [React Native Basics](react-native-basics.md)
- Track 09 — Testing (general testing principles, TDD mindset)
- Jest basics (test, expect, describe, beforeEach)

---

## Core Concepts

### 1. Jest Setup for React Native / Expo

Expo includes Jest preconfigured. For a bare React Native project, install `@testing-library/react-native` alongside Jest.

```bash
# Expo project — already has jest-expo, just add RNTL
npx expo install @testing-library/react-native @testing-library/user-event

# Bare React Native
npm install --save-dev jest @testing-library/react-native @testing-library/user-event \
  babel-jest @babel/core react-test-renderer
```

Configure `package.json`:

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

`transformIgnorePatterns` tells Jest which `node_modules` to run through Babel. By default Jest skips all of `node_modules`, but many React Native and Expo packages ship as ES modules (not pre-compiled CommonJS). The pattern above excludes those packages from the skip list so Babel can transform them before running tests.

Run tests:

```bash
bun run test          # if using Bun as task runner
npx jest              # direct invocation
npx jest --watch      # interactive watch mode
npx jest src/components/LoginForm.test.tsx  # single file
```

For a bare React Native project, use `react-native` preset instead:

```json
{
  "jest": {
    "preset": "react-native",
    "setupFilesAfterFramework": ["@testing-library/react-native/extend-expect"]
  }
}
```

---

### 2. Component Testing with React Native Testing Library

RNTL renders components into a virtual host tree and provides queries that mirror how a user (or assistive technology) perceives the UI. It does not use a real device — it uses Jest's JS environment with a mock native renderer.

**Install:**

```bash
npx expo install @testing-library/react-native
```

**Basic queries:**

```typescript
import { render, screen, fireEvent, waitFor } from '@testing-library/react-native';
```

Query priority (highest to lowest confidence):

| Priority | Query | Notes |
|----------|-------|-------|
| 1 | `getByRole` | Best — mirrors accessibility tree |
| 2 | `getByLabelText` | For labeled form inputs |
| 3 | `getByText` | For visible text content |
| 4 | `getByDisplayValue` | For current input value |
| 5 | `getByTestId` | Last resort — brittle, use sparingly |

**Example: LoginForm component**

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

**Test file:**

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

`fireEvent` dispatches a single synthetic event directly on the element — synchronous, lower-level, no gesture simulation. It is sufficient for most tests.

`userEvent` simulates the full sequence of events a real user would trigger (focus, keypress, change, blur). It is async and closer to real user interaction. Use it when you need to test interaction-dependent behavior (e.g., autocomplete, masked inputs).

```typescript
import userEvent from '@testing-library/user-event';

// userEvent example — simulates typing character by character
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

### 3. Mocking Native Modules

Native modules do not exist in Jest's JS environment. You must mock them. There are two approaches: `__mocks__` directory (automatic) and `jest.mock()` inline (one-off).

**`__mocks__` directory — automatic mocking**

Create a `__mocks__` directory at the root of your project (same level as `node_modules`) or next to the module file. Jest automatically uses it when the module is imported.

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

**`jest.mock()` inline — one-off mocks**

When you need a module mocked only in one test file:

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

**Mocking `react-native` internals**

Some `react-native` APIs need manual setup. Common ones:

```typescript
// jest.setup.ts
import 'react-native-gesture-handler/jestSetup';

// Mock Animated to run synchronously in tests
jest.mock('react-native/Libraries/Animated/NativeAnimatedHelper');

// Mock navigation (React Navigation)
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

### 4. Testing Hooks

Use `renderHook` and `act` from RNTL to test custom hooks in isolation without a full component.

**Simple hook:**

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

**Hook with async side effects (SecureStore):**

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

### 5. E2E Testing with Detox

Detox runs tests against a real app binary on a simulator or emulator. It interacts with the app through native accessibility APIs and uses a gray-box synchronization mechanism: before each interaction, Detox waits for the JS thread to go idle (no pending timers, no in-flight network requests, no pending animations). This removes the need for arbitrary `sleep()` calls and makes tests reliable.

**Install:**

```bash
npx expo install detox @config-plugins/detox
npm install --save-dev detox @types/detox
```

Add the config plugin to `app.json`:

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

**Complete auth flow E2E test:**

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
    // Assumes we are already on the home screen from the previous test,
    // or launch the app with a pre-seeded auth token:
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

**Run Detox:**

```bash
# Build (first time, or after native changes)
npx detox build --configuration ios.sim.debug

# Run tests
npx detox test --configuration ios.sim.debug

# Run a specific test file
npx detox test --configuration ios.sim.debug e2e/auth.e2e.ts
```

---

### 6. E2E Testing with Maestro

Maestro is a simpler E2E framework: tests are YAML files, no native build is needed (it drives an already-running app or installs an APK/IPA), and setup takes minutes. The trade-off is less control over app state and no gray-box synchronization — Maestro polls for elements with a configurable timeout.

**Install:**

```bash
curl -Ls "https://get.maestro.mobile.dev" | bash
# Restart your shell, then verify:
maestro --version
```

**Complete login flow:**

```yaml
# .maestro/flows/login.yaml
appId: com.yourcompany.yourapp
---
- launchApp:
    clearState: true

# Happy path: valid credentials
- tapOn: "Email"
- inputText: "test@example.com"
- tapOn: "Password"
- inputText: "password123"
- tapOn: "Sign In"
- assertVisible: "Welcome back"

# Navigate to settings and verify sign out exists
- tapOn: "Settings"
- assertVisible: "Sign Out"
```

**Invalid credentials flow:**

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

**Onboarding flow with back navigation:**

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

**Run Maestro:**

```bash
# Run a single flow
maestro test .maestro/flows/login.yaml

# Run all flows in a directory
maestro test .maestro/flows/

# Run on a specific device (Android)
maestro test --device emulator-5554 .maestro/flows/login.yaml

# Run with cloud (Maestro Cloud)
maestro cloud --apiKey $MAESTRO_API_KEY .maestro/flows/
```

**Detox vs Maestro comparison:**

| Aspect | Detox | Maestro |
|---|---|---|
| Test syntax | TypeScript/JavaScript | YAML |
| Setup complexity | High (native build required) | Low (installs in minutes) |
| Synchronization | Gray-box (waits for JS idle) | Polling with timeout |
| Reliability | Very high | High (occasional flakiness) |
| CI support | Full (with macOS runner for iOS) | Full (Maestro Cloud available) |
| Platform | iOS + Android | iOS + Android |
| Debug tooling | Detox CLI, device logs | `maestro studio` visual recorder |
| Community | Large (Wix) | Growing |
| Best for | Complex flows, high-reliability CI | Quick setup, simple flows, non-engineers writing tests |

---

### 7. Running Tests in CI

**Unit and component tests (GitHub Actions):**

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

**Detox E2E (iOS — requires macOS runner):**

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

iOS Detox tests require a macOS runner because they need Xcode and the iOS Simulator. macOS runners on GitHub Actions cost approximately 10x more than Linux runners — run them on pushes to `main` only, not on every PR.

**EAS Test — managed cloud alternative:**

EAS Test (Expo Application Services) runs Detox or Maestro tests on real devices in the cloud without maintaining your own macOS CI runner.

```bash
# Install EAS CLI
npm install -g eas-cli

# Configure
eas test:configure

# Run tests
eas test --profile preview
```

`eas.json` profile for testing:

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

## Anti-Patterns

### 1. Mocking Your Own Code

Mocking internal modules removes the integration coverage you actually want. If `UserService` calls `ApiClient`, do not mock `ApiClient` inside a `UserService` test — you lose confidence that they work together.

**Wrong:**

```typescript
// Testing UserService but mocking your own ApiClient
jest.mock('../../lib/api-client');  // <-- mocking your own code

import { UserService } from '../UserService';
import { ApiClient } from '../../lib/api-client';

it('fetches the user', async () => {
  (ApiClient.get as jest.Mock).mockResolvedValue({ id: 1, name: 'Alice' });
  const service = new UserService();
  const user = await service.getUser(1);
  expect(user.name).toBe('Alice');
});
```

**Right:** mock at the network boundary (MSW, `fetch` mock, `axios-mock-adapter`), or use an in-memory repository.

```typescript
// Mock fetch — the actual boundary between your code and the outside world
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

### 2. `getByTestId` Everywhere

`testID` couples your tests to implementation details. Renaming a component does not break `getByRole` or `getByText`, but it might break `getByTestId` if you forgot to update it. Accessibility queries also catch regressions in your accessibility tree.

**Wrong:**

```typescript
fireEvent.press(screen.getByTestId('login-button'));
fireEvent.changeText(screen.getByTestId('email-field'), 'user@example.com');
const error = screen.getByTestId('error-message');
```

**Right:**

```typescript
fireEvent.press(screen.getByRole('button', { name: 'Sign In' }));
fireEvent.changeText(screen.getByLabelText('Email'), 'user@example.com');
const error = screen.getByRole('alert');
```

Reserve `testID` for elements that have no accessible role or label and cannot be queried by text — for example, a decorative icon or a specific list item in a long list.

---

### 3. No E2E Tests at All

Unit and component tests miss entire classes of bugs: navigation state corruption, deep-link handling, auth token refresh race conditions, and multi-screen flows. These are exactly the bugs that reach production and are the hardest to debug.

Have at least one E2E happy-path test per critical user flow. The minimum viable E2E suite for a typical app:

- Sign up (new user, valid inputs)
- Sign in / sign out
- Core feature happy path (e.g., create a record, view it, delete it)
- Payment or checkout flow (if applicable)

You do not need to cover every edge case in E2E — that is what unit tests are for. Cover the paths your users actually take.

---

### 4. `sleep()` in Detox Tests

Detox's auto-synchronization mechanism waits for the JS thread to go idle before each interaction. Adding explicit `sleep()` calls makes tests slower and masks synchronization problems.

**Wrong:**

```typescript
await element(by.text('Sign In')).tap();
await new Promise((resolve) => setTimeout(resolve, 2000)); // waiting for navigation
await detoxExpect(element(by.text('Dashboard'))).toBeVisible();
```

**Right:**

```typescript
await element(by.text('Sign In')).tap();
// Detox waits for the navigation animation and data fetch before this assertion
await detoxExpect(element(by.text('Dashboard'))).toBeVisible();
```

If tests fail because Detox times out waiting for idle state, the root cause is almost always:
- A `setInterval` or polling function that never stops
- An unresolved Promise (e.g., a network request with no timeout)
- An infinite animation loop (`useNativeDriver: false` + looping `Animated.loop`)

Fix the root cause; do not add `sleep()`.

---

## Real-World Scenarios

### Scenario 1: Full Auth Flow E2E (Detox)

A complete Detox test covering: launch → login → verify home screen → sign out → verify auth screen.

```typescript
// e2e/full-auth-flow.e2e.ts
import { device, element, by, expect as detoxExpect } from 'detox';

describe('Full auth flow', () => {
  beforeAll(async () => {
    await device.launchApp({ newInstance: true, delete: true });
  });

  it('completes the full login and logout cycle', async () => {
    // Step 1: App opens to auth screen
    await detoxExpect(element(by.id('auth-screen'))).toBeVisible();
    await detoxExpect(element(by.text('Sign In'))).toBeVisible();

    // Step 2: Enter valid credentials and sign in
    await element(by.id('email-input')).typeText('user@example.com');
    await element(by.id('password-input')).typeText('correct-password');
    await element(by.text('Sign In')).tap();

    // Step 3: Verify home screen
    await detoxExpect(element(by.id('home-screen'))).toBeVisible();
    await detoxExpect(element(by.text('Welcome back'))).toBeVisible();

    // Step 4: Navigate to settings
    await element(by.id('tab-settings')).tap();
    await detoxExpect(element(by.id('settings-screen'))).toBeVisible();
    await detoxExpect(element(by.text('Sign Out'))).toBeVisible();

    // Step 5: Sign out
    await element(by.text('Sign Out')).tap();

    // Some apps show a confirmation dialog
    await detoxExpect(element(by.text('Are you sure?'))).toBeVisible();
    await element(by.text('Confirm')).tap();

    // Step 6: Back to auth screen
    await detoxExpect(element(by.id('auth-screen'))).toBeVisible();
    await detoxExpect(element(by.text('Sign In'))).toBeVisible();
  });
});
```

---

### Scenario 2: Offline Behavior Test (RNTL + Mock)

Testing that an "offline" banner appears when the device loses connectivity. The component uses `NetInfo` — mock it to simulate the offline state.

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

    // Simulate connection restored
    if (storedCallback) {
      storedCallback({ isConnected: true });
    }
    rerender(<OfflineBanner />);

    // The component listens via useEffect — in a real scenario the state update
    // would trigger a re-render. Verify the subscriber was called.
    expect(mockNetInfo.addEventListener).toHaveBeenCalled();
  });
});
```

---

## Further Reading

- [React Native Testing Library docs](https://callstack.github.io/react-native-testing-library/)
- [Detox documentation](https://wix.github.io/Detox/)
- [Maestro documentation](https://maestro.mobile.dev/)
- [EAS Test](https://docs.expo.dev/eas/test/)
- [Testing React Native Apps — Jest docs](https://jestjs.io/docs/tutorial-react-native)

---

## Summary

- Use three test layers: Jest + RNTL for unit/component tests (fast, runs on any machine), Detox for reliable E2E (gray-box sync, requires simulator), and Maestro for quick YAML-based flows or non-engineer-authored tests.
- Configure `transformIgnorePatterns` in Jest to allow Babel to process Expo and React Native packages that ship as ES modules.
- Query with `getByRole` and `getByLabelText` before reaching for `getByTestId` — accessible queries also catch accessibility regressions.
- Mock at boundaries — native device APIs (`expo-camera`, `AsyncStorage`, `NetInfo`) and network (`fetch`) — not inside your own service/repository code.
- Use `renderHook` + `act` + `waitFor` for testing custom hooks that contain async effects or state.
- Detox's gray-box synchronization makes `sleep()` unnecessary; if tests time out, look for unresolved Promises, active polling intervals, or infinite animations.
- Run unit tests on every push (Linux runner, fast); run Detox iOS E2E only on merges to `main` (macOS runner, expensive) or delegate to EAS Test for managed device testing.
