# React Native Basics

## Overview

React Native lets you build mobile apps for iOS and Android using React and JavaScript, sharing business logic while rendering native UI components (not a WebView). The mental model is identical to React: components, props, state, hooks — but instead of `<div>` and `<span>` you use `<View>` and `<Text>`, and CSS is replaced by StyleSheet objects.

Expo is the recommended starting point for 95% of projects. It provides a managed build service, over-the-air updates, and a curated set of native modules without requiring you to maintain Xcode or Android Studio build configurations.

---

## Prerequisites

- Track 02 — Frontend (React components, hooks, TypeScript)
- Basic understanding of mobile UI conventions (tabs, stacks, gestures)
- Node.js 18+ installed

---

## Core Concepts

### Expo vs Bare Workflow

**Expo Managed Workflow**
- Expo manages the native build process via EAS Build (cloud)
- You never touch Xcode or `android/` directories
- Access to native APIs through `expo-*` packages (`expo-camera`, `expo-location`, etc.)
- Limited to APIs that Expo has wrapped — sufficient for 90%+ of apps

```bash
npx create-expo-app@latest my-app --template blank-typescript
cd my-app
npx expo start
```

**Bare Workflow / React Native CLI**
- Full access to native code — modify `ios/` and `android/` directories
- Required for: third-party native SDKs not in Expo ecosystem, custom native modules, specific build configurations
- You manage Xcode and Android Studio upgrades

```bash
npx @react-native-community/cli@latest init MyApp --template react-native-template-typescript
```

**Expo with bare** (best of both worlds):
```bash
npx create-expo-app@latest my-app
# Later, eject if needed:
npx expo prebuild
```

### Core Components

React Native ships with a small set of primitive components that map to native UI elements.

```tsx
import {
  View,
  Text,
  TextInput,
  Image,
  ScrollView,
  TouchableOpacity,
  Pressable,
  FlatList,
  SafeAreaView,
  StatusBar,
  ActivityIndicator,
} from 'react-native';

// View — equivalent to <div>, the fundamental layout container
<View style={styles.container}>
  {/* Text — ALL text must be inside <Text> */}
  <Text style={styles.title}>Hello React Native</Text>

  {/* TextInput — controlled input */}
  <TextInput
    value={value}
    onChangeText={setValue}
    placeholder="Type here"
    keyboardType="email-address"
    autoCapitalize="none"
    secureTextEntry={false}
  />

  {/* Image */}
  <Image
    source={{ uri: 'https://example.com/photo.jpg' }}
    style={{ width: 200, height: 200 }}
    resizeMode="cover"
  />

  {/* Pressable — recommended over TouchableOpacity for new code */}
  <Pressable
    onPress={handlePress}
    style={({ pressed }) => [styles.button, pressed && styles.buttonPressed]}
  >
    <Text>Press me</Text>
  </Pressable>
</View>
```

### StyleSheet

Styles are JavaScript objects, not CSS files. `StyleSheet.create()` validates styles in development and optimizes them at runtime.

```tsx
import { StyleSheet, Dimensions } from 'react-native';

const { width: SCREEN_WIDTH } = Dimensions.get('window');

const styles = StyleSheet.create({
  container: {
    flex: 1,
    backgroundColor: '#fff',
    paddingHorizontal: 16,
    paddingTop: 24,
  },
  title: {
    fontSize: 24,
    fontWeight: '700',
    color: '#111827',
    marginBottom: 8,
  },
  card: {
    backgroundColor: '#f9fafb',
    borderRadius: 12,
    padding: 16,
    marginBottom: 12,
    // Shadow (iOS)
    shadowColor: '#000',
    shadowOffset: { width: 0, height: 2 },
    shadowOpacity: 0.1,
    shadowRadius: 4,
    // Shadow (Android)
    elevation: 3,
  },
  button: {
    backgroundColor: '#6366f1',
    borderRadius: 8,
    paddingVertical: 12,
    paddingHorizontal: 24,
    alignItems: 'center',
  },
  // Responsive width
  fullWidth: {
    width: SCREEN_WIDTH - 32,
  },
});
```

**Key differences from CSS:**
- All values are numbers (device-independent pixels), not strings with units
- No `px`, `em`, `rem` — `fontSize: 16` means 16dp (density-independent pixels)
- No cascading — each element styles itself explicitly
- `flex` direction defaults to `column` (vertical), not `row`
- No shorthand properties (`padding: '8px 16px'` is invalid; use `paddingVertical` / `paddingHorizontal`)

### Flexbox in React Native

React Native uses Yoga, Facebook's implementation of flexbox. Defaults differ from CSS:

| Property       | CSS default    | React Native default |
|----------------|----------------|----------------------|
| `flexDirection`| `row`          | `column`             |
| `alignContent` | `stretch`      | `flex-start`         |
| `flexShrink`   | `1`            | `0`                  |

```tsx
// Common layout patterns
const styles = StyleSheet.create({
  // Full screen container
  screen: { flex: 1 },

  // Horizontal row with space between
  row: { flexDirection: 'row', justifyContent: 'space-between', alignItems: 'center' },

  // Centered content
  centered: { flex: 1, justifyContent: 'center', alignItems: 'center' },

  // Fixed bottom button
  footer: { position: 'absolute', bottom: 0, left: 0, right: 0, padding: 16 },

  // Take remaining space
  grow: { flex: 1 },
});
```

### SafeAreaView and Keyboard

```tsx
import { SafeAreaView, KeyboardAvoidingView, Platform } from 'react-native';
import { SafeAreaProvider } from 'react-native-safe-area-context'; // expo install

// Wrap root app
function App() {
  return (
    <SafeAreaProvider>
      <Navigation />
    </SafeAreaProvider>
  );
}

// Individual screen
function LoginScreen() {
  return (
    <SafeAreaView style={{ flex: 1 }}>
      <KeyboardAvoidingView
        style={{ flex: 1 }}
        behavior={Platform.OS === 'ios' ? 'padding' : 'height'}
      >
        {/* Content that adjusts for keyboard */}
      </KeyboardAvoidingView>
    </SafeAreaView>
  );
}
```

---

## Hands-On Examples

### FlatList — Efficient Long Lists

Never use `ScrollView` for long lists. `FlatList` renders only visible items.

```tsx
import { FlatList, View, Text, StyleSheet, RefreshControl } from 'react-native';

type Item = { id: string; title: string; subtitle: string };

function ItemList({ data }: { data: Item[] }) {
  const [refreshing, setRefreshing] = useState(false);

  const renderItem = useCallback(({ item }: { item: Item }) => (
    <View style={styles.item}>
      <Text style={styles.title}>{item.title}</Text>
      <Text style={styles.subtitle}>{item.subtitle}</Text>
    </View>
  ), []);

  const keyExtractor = useCallback((item: Item) => item.id, []);

  return (
    <FlatList
      data={data}
      renderItem={renderItem}
      keyExtractor={keyExtractor}
      contentContainerStyle={styles.list}
      ItemSeparatorComponent={() => <View style={styles.separator} />}
      ListEmptyComponent={<Text>No items found</Text>}
      refreshControl={
        <RefreshControl refreshing={refreshing} onRefresh={handleRefresh} />
      }
      // Performance props
      removeClippedSubviews={true}
      maxToRenderPerBatch={10}
      windowSize={5}
    />
  );
}
```

### Platform-Specific Code

```tsx
import { Platform } from 'react-native';

// Inline platform check
const paddingTop = Platform.OS === 'ios' ? 44 : 24;

// Platform.select — cleaner for multiple values
const styles = StyleSheet.create({
  header: {
    ...Platform.select({
      ios: {
        shadowColor: '#000',
        shadowOffset: { width: 0, height: 1 },
        shadowOpacity: 0.2,
        shadowRadius: 2,
      },
      android: {
        elevation: 4,
      },
    }),
  },
});

// Platform-specific files
// Button.ios.tsx  — picked up automatically on iOS
// Button.android.tsx — picked up automatically on Android
// Button.tsx — fallback for web or shared logic
```

### Custom Hook with AsyncStorage

```tsx
import AsyncStorage from '@react-native-async-storage/async-storage';
// expo install @react-native-async-storage/async-storage

function usePersistedState<T>(key: string, defaultValue: T) {
  const [value, setValue] = useState<T>(defaultValue);
  const [loaded, setLoaded] = useState(false);

  useEffect(() => {
    AsyncStorage.getItem(key).then((stored) => {
      if (stored !== null) setValue(JSON.parse(stored) as T);
      setLoaded(true);
    });
  }, [key]);

  const set = useCallback(
    async (newValue: T) => {
      setValue(newValue);
      await AsyncStorage.setItem(key, JSON.stringify(newValue));
    },
    [key]
  );

  return [value, set, loaded] as const;
}
```

---

## Common Patterns & Best Practices

### Typed Navigation with React Navigation

```bash
npx expo install @react-navigation/native @react-navigation/native-stack react-native-screens react-native-safe-area-context
```

```tsx
// types/navigation.ts
import type { NativeStackScreenProps } from '@react-navigation/native-stack';

export type RootStackParamList = {
  Home: undefined;
  Profile: { userId: string };
  Settings: undefined;
};

export type HomeScreenProps = NativeStackScreenProps<RootStackParamList, 'Home'>;
export type ProfileScreenProps = NativeStackScreenProps<RootStackParamList, 'Profile'>;

// App.tsx
import { NavigationContainer } from '@react-navigation/native';
import { createNativeStackNavigator } from '@react-navigation/native-stack';

const Stack = createNativeStackNavigator<RootStackParamList>();

export default function App() {
  return (
    <NavigationContainer>
      <Stack.Navigator initialRouteName="Home">
        <Stack.Screen name="Home" component={HomeScreen} />
        <Stack.Screen name="Profile" component={ProfileScreen} />
        <Stack.Screen name="Settings" component={SettingsScreen} />
      </Stack.Navigator>
    </NavigationContainer>
  );
}

// HomeScreen.tsx
function HomeScreen({ navigation }: HomeScreenProps) {
  return (
    <Pressable onPress={() => navigation.navigate('Profile', { userId: '123' })}>
      <Text>Go to Profile</Text>
    </Pressable>
  );
}
```

### Handling Permissions

```tsx
import * as Location from 'expo-location';
import * as Notifications from 'expo-notifications';

async function requestLocation() {
  const { status } = await Location.requestForegroundPermissionsAsync();
  if (status !== 'granted') {
    Alert.alert(
      'Permission Required',
      'Location access is needed to show nearby items.',
      [
        { text: 'Cancel', style: 'cancel' },
        { text: 'Open Settings', onPress: () => Linking.openSettings() },
      ]
    );
    return null;
  }
  return Location.getCurrentPositionAsync({});
}
```

---

## Anti-Patterns to Avoid

**Putting text outside `<Text>`.** React Native throws an error if a string is a direct child of `<View>`. All text must be wrapped in `<Text>`.

**Using `ScrollView` for long lists.** It renders all children at once. For lists of more than ~20 items, use `FlatList` or `SectionList`.

**Inline style objects in `render`.** `{ flex: 1 }` creates a new object on every render, breaking `PureComponent` and `React.memo` optimizations. Move all styles to `StyleSheet.create()`.

**Ignoring the keyboard.** On mobile the soft keyboard covers half the screen. Wrap forms in `KeyboardAvoidingView` and test with the keyboard open.

**Not testing on real devices early.** The iOS Simulator and Android Emulator do not replicate real device performance, touch latency, or camera/sensor behavior. Test on hardware before any release.

**Blocking the JS thread.** React Native has a single JS thread. Heavy synchronous computation (JSON parsing, sorting large arrays) causes frame drops. Move to `useEffect` or a worker.

---

## Debugging & Troubleshooting

### Tools

- **Expo Go** — scan QR code to run your app on a real device without building
- **React Native Debugger** — standalone app, connects to Metro bundler
- **Flipper** — Facebook's mobile debugging platform (network inspector, layout inspector, database browser)
- **Reactotron** — alternative to Flipper for state/network inspection
- **Metro bundler** — press `i` (iOS), `a` (Android), `r` (reload) in the terminal

### Common Issues

**Red box "Text strings must be rendered within a `<Text>` component"**
You have a string directly inside a `<View>`. Check for `{condition && 'some string'}` patterns.

**"Invariant Violation: requireNativeComponent" on a new install**
A native module is not linked. Run `npx expo install` or `pod install` (bare workflow).

**App works in Expo Go but crashes in production build**
Usually a native module not included in the default Expo SDK. Use `expo-dev-client` for custom native dependencies.

**`useEffect` running twice in development**
React 18 StrictMode intentional behavior. Only happens in dev. Do not fight it — write effects that are idempotent or return a cleanup function.

---

## Real-World Scenarios

### Authentication Flow

```tsx
// AuthContext.tsx
type AuthContextValue = {
  user: User | null;
  signIn: (token: string) => Promise<void>;
  signOut: () => Promise<void>;
};

const AuthContext = createContext<AuthContextValue | null>(null);

export function AuthProvider({ children }: { children: React.ReactNode }) {
  const [user, setUser] = useState<User | null>(null);
  const [loading, setLoading] = useState(true);

  useEffect(() => {
    // Load persisted token on startup
    SecureStore.getItemAsync('auth_token').then(async (token) => {
      if (token) {
        const user = await fetchCurrentUser(token);
        setUser(user);
      }
      setLoading(false);
    });
  }, []);

  if (loading) return <SplashScreen />;

  return (
    <AuthContext.Provider value={{ user, signIn, signOut }}>
      {children}
    </AuthContext.Provider>
  );
}

// In navigator: show auth stack or app stack based on user
function RootNavigator() {
  const { user } = useAuth();
  return user ? <AppStack /> : <AuthStack />;
}
```

---

## Hermes Engine

Hermes is a JavaScript engine optimized for React Native, enabled by default since Expo SDK 48. It pre-compiles JavaScript to bytecode at build time, improving startup time and reducing memory usage compared to JavaScriptCore.

Benefits:
- Faster TTI (time-to-interactive) — JS is pre-compiled, no JIT warmup
- Lower memory footprint — better GC tuned for mobile constraints
- Earlier error detection — syntax errors surface at build time, not runtime

```typescript
// Verify Hermes is running at runtime
const isHermes = () => !!global.HermesInternal;

if (__DEV__) {
  console.log('Running on Hermes:', isHermes());
  console.log('Runtime properties:', global.HermesInternal?.getRuntimeProperties());
}
```

```json
// app.json — Hermes is on by default in Expo SDK 48+
{
  "expo": {
    "jsEngine": "hermes"
  }
}
```

To revert to JavaScriptCore (for debugging edge cases or incompatible libraries), set `"jsEngine": "jsc"`. Always test on Hermes before shipping — a small number of older libraries have Hermes incompatibilities.

---

## Metro Bundler Configuration

Metro is React Native's JavaScript bundler. The default config works for most apps, but you'll need to customize it for monorepos, SVG support, or path aliases.

```javascript
// metro.config.js
const { getDefaultConfig } = require('expo/metro-config');
const path = require('path');

const config = getDefaultConfig(__dirname);

// 1. SVG support — transform SVGs as React components
config.transformer.babelTransformerPath = require.resolve('react-native-svg-transformer');
config.resolver.assetExts = config.resolver.assetExts.filter((ext) => ext !== 'svg');
config.resolver.sourceExts = [...config.resolver.sourceExts, 'svg'];

// 2. Path aliases — import from '@components/Button' instead of '../../components/Button'
config.resolver.alias = {
  '@components': path.resolve(__dirname, 'src/components'),
  '@hooks': path.resolve(__dirname, 'src/hooks'),
  '@lib': path.resolve(__dirname, 'src/lib'),
  '@types': path.resolve(__dirname, 'src/types'),
};

// 3. Monorepo — watch packages outside the app directory
config.watchFolders = [path.resolve(__dirname, '../../packages')];

module.exports = config;
```

Also update `tsconfig.json` so TypeScript resolves the aliases:

```json
{
  "compilerOptions": {
    "baseUrl": ".",
    "paths": {
      "@components/*": ["src/components/*"],
      "@hooks/*": ["src/hooks/*"],
      "@lib/*": ["src/lib/*"],
      "@types/*": ["src/types/*"]
    }
  }
}
```

---

## Platform-Specific File Extensions

Metro resolves `.ios.ts` and `.android.ts` before `.ts`, enabling clean per-platform implementations without `Platform.OS` conditionals scattered through your code.

```
src/components/
  DatePicker.tsx          # fallback (web / unknown platforms)
  DatePicker.ios.tsx      # auto-used on iOS
  DatePicker.android.tsx  # auto-used on Android
```

```tsx
// DatePicker.ios.tsx — native iOS date picker
import DateTimePicker from '@react-native-community/datetimepicker';

interface Props {
  value: Date;
  onChange: (date: Date) => void;
}

export function DatePicker({ value, onChange }: Props) {
  return (
    <DateTimePicker
      value={value}
      mode="date"
      display="spinner"
      onChange={(_, selectedDate) => selectedDate && onChange(selectedDate)}
    />
  );
}
```

```tsx
// DatePicker.android.tsx — Android uses a dialog (imperative API)
import { Pressable, Text } from 'react-native';
import { DateTimePickerAndroid } from '@react-native-community/datetimepicker';

interface Props {
  value: Date;
  onChange: (date: Date) => void;
}

export function DatePicker({ value, onChange }: Props) {
  const openPicker = () =>
    DateTimePickerAndroid.open({
      value,
      mode: 'date',
      onChange: (_, selectedDate) => selectedDate && onChange(selectedDate),
    });

  return (
    <Pressable onPress={openPicker}>
      <Text>{value.toLocaleDateString()}</Text>
    </Pressable>
  );
}
```

Consumer code imports `DatePicker` from the shared path — Metro picks the right implementation automatically. No `if (Platform.OS === 'android')` needed.

The pattern works for any platform-divergent component: camera interfaces, share sheets, permissions dialogs, native pickers.

---

## Debugging

**React Native DevTools** (recommended, Expo SDK 50+):

```bash
npx expo start
# Press 'j' in the terminal to open the debugger
# Opens Chrome DevTools connected to your app's JS runtime
```

Features: breakpoints, step-through, network inspector (fetch/XHR), React component tree, performance profiler, source maps.

**Shake gesture / dev menu:**
- iOS simulator: Cmd+D
- Android simulator: Cmd+M (macOS) or Ctrl+M (Windows/Linux)
- Physical device: shake

**Flipper** (legacy — avoid for new projects):
- Requires native build and heavy native dependencies
- Frequently breaks across RN upgrades
- React Native DevTools replaces it for nearly all use cases
- Remove from existing projects: `npx react-native-clean-project`

**LogBox** — customizing the error overlay:

```typescript
import { LogBox } from 'react-native';

// Suppress specific warnings — use sparingly, prefer fixing root cause
LogBox.ignoreLogs([
  'Require cycle:',            // Common in large codebases, often benign
  'Non-serializable values',   // Fix: don't pass functions through navigation params
  'VirtualizedLists should',   // Fix: don't nest FlatList inside ScrollView
]);

// Disable entirely during Detox / Maestro E2E tests
if (process.env.E2E === 'true') {
  LogBox.ignoreAllLogs();
}
```

**Console.log in production:** Remove with Babel:

```javascript
// babel.config.js
module.exports = {
  plugins: process.env.NODE_ENV === 'production'
    ? [['transform-remove-console', { exclude: ['error', 'warn'] }]]
    : [],
};
```

---

## Further Reading

- [React Native docs](https://reactnative.dev/docs/getting-started)
- [Expo docs](https://docs.expo.dev/)
- [React Navigation docs](https://reactnavigation.org/docs/getting-started)
- [React Native Directory](https://reactnative.directory/) — find native libraries
- [Expo SDK reference](https://docs.expo.dev/versions/latest/)

---

## Summary

React Native renders native UI using React components. `View`, `Text`, `TextInput`, `FlatList`, and `Pressable` are the core primitives. Styles are JavaScript objects using dp units, flex defaults to column direction, and box shadows require platform-specific properties. Expo managed workflow is the right default — it handles builds and native modules without requiring Xcode/Android Studio knowledge. Use `FlatList` for all long lists, `KeyboardAvoidingView` for forms, and `Platform.OS` for conditional behavior. Test on real devices early.
