# Mobile Navigation

## Overview

Navigation in mobile apps is fundamentally different from web routing. On the web, the browser manages a flat history stack. On mobile, navigation is a hierarchy: screens stack on top of each other, tabs switch between sibling sections, and drawers slide in from the side. Users expect platform-native transitions — iOS slide-in-from-right, Android shared element transitions, bottom sheets.

React Navigation is the de facto standard library for React Native navigation. It provides stack, tab, drawer, and bottom-sheet navigators that compose into complex navigation trees.

---

## Prerequisites

- [React Native Basics](react-native-basics.md)
- Understanding of TypeScript generics (for typed navigation)
- `@react-navigation/native` installed

---

## Core Concepts

### Navigator Types

**Stack Navigator** — screens stack on top of each other, back button pops

```
[ Screen A ] → push → [ Screen A | Screen B ] → push → [ Screen A | Screen B | Screen C ]
                                                         ← back pops Screen C
```

**Tab Navigator** — parallel screens, tab bar at bottom (or top)

```
[ Home | Search | Profile ]
  ↑ active               ↑ tap switches, no stacking
```

**Drawer Navigator** — slide-in menu from left edge

**Bottom Sheet** (via `@gorhom/bottom-sheet`) — partial-screen overlay

Navigators compose: each tab can contain its own stack, which can launch modals.

### Type-Safe Navigation

Full type safety requires declaring a param list for every navigator.

```tsx
// navigation/types.ts
import type {
  NativeStackScreenProps,
  NativeStackNavigationProp,
} from '@react-navigation/native-stack';
import type { BottomTabScreenProps } from '@react-navigation/bottom-tabs';
import type { CompositeScreenProps } from '@react-navigation/native';

// Root stack (modals + tab container)
export type RootStackParamList = {
  Tabs: undefined;
  Modal: { message: string };
  ImageViewer: { imageUrl: string; title: string };
};

// Tab navigator
export type TabParamList = {
  HomeStack: undefined;
  SearchStack: undefined;
  ProfileStack: undefined;
};

// Home stack (nested inside HomeTab)
export type HomeStackParamList = {
  Feed: undefined;
  ArticleDetail: { articleId: string; title: string };
  AuthorProfile: { authorId: string };
};

// Composed props for screens inside nested navigators
export type ArticleDetailScreenProps = CompositeScreenProps<
  NativeStackScreenProps<HomeStackParamList, 'ArticleDetail'>,
  BottomTabScreenProps<TabParamList>
>;

// Navigation prop only (for components, not screens)
export type HomeNavigationProp = NativeStackNavigationProp<HomeStackParamList>;
```

### useNavigation and useRoute

```tsx
import { useNavigation, useRoute } from '@react-navigation/native';
import type { HomeNavigationProp } from '../navigation/types';

// Inside a component (not the screen itself)
function BackButton() {
  const navigation = useNavigation<HomeNavigationProp>();
  return <Pressable onPress={() => navigation.goBack()} />;
}

// Typed route params
function ArticleDetail() {
  const route = useRoute<ArticleDetailScreenProps['route']>();
  const { articleId, title } = route.params; // fully typed
  // ...
}
```

---

## Hands-On Examples

### Complete Navigation Setup

```tsx
// App.tsx
import { NavigationContainer } from '@react-navigation/native';
import { createNativeStackNavigator } from '@react-navigation/native-stack';
import { createBottomTabNavigator } from '@react-navigation/bottom-tabs';
import { linking } from './navigation/linking';

const RootStack = createNativeStackNavigator<RootStackParamList>();
const Tab = createBottomTabNavigator<TabParamList>();

function TabNavigator() {
  return (
    <Tab.Navigator
      screenOptions={({ route }) => ({
        headerShown: false,
        tabBarIcon: ({ focused, color, size }) => (
          <TabIcon name={route.name} focused={focused} color={color} size={size} />
        ),
        tabBarActiveTintColor: '#6366f1',
        tabBarInactiveTintColor: '#9ca3af',
      })}
    >
      <Tab.Screen name="HomeStack" component={HomeStackNavigator} options={{ title: 'Home' }} />
      <Tab.Screen name="SearchStack" component={SearchStackNavigator} options={{ title: 'Search' }} />
      <Tab.Screen name="ProfileStack" component={ProfileStackNavigator} options={{ title: 'Profile' }} />
    </Tab.Navigator>
  );
}

const HomeStack = createNativeStackNavigator<HomeStackParamList>();

function HomeStackNavigator() {
  return (
    <HomeStack.Navigator>
      <HomeStack.Screen name="Feed" component={FeedScreen} />
      <HomeStack.Screen
        name="ArticleDetail"
        component={ArticleDetailScreen}
        options={({ route }) => ({ title: route.params.title })}
      />
    </HomeStack.Navigator>
  );
}

export default function App() {
  return (
    <NavigationContainer linking={linking}>
      <RootStack.Navigator screenOptions={{ headerShown: false }}>
        <RootStack.Screen name="Tabs" component={TabNavigator} />
        <RootStack.Screen
          name="Modal"
          component={ModalScreen}
          options={{ presentation: 'modal' }}
        />
        <RootStack.Screen
          name="ImageViewer"
          component={ImageViewerScreen}
          options={{ presentation: 'fullScreenModal', animation: 'fade' }}
        />
      </RootStack.Navigator>
    </NavigationContainer>
  );
}
```

### Auth Flow Pattern

The key insight: navigators conditionally render based on auth state. There is no "redirect" — you swap entire navigator trees.

```tsx
// navigation/RootNavigator.tsx
import { useAuth } from '../context/AuthContext';

export function RootNavigator() {
  const { user, isLoading } = useAuth();

  if (isLoading) {
    return <SplashScreen />;
  }

  return (
    <NavigationContainer>
      {user ? <AuthenticatedNavigator /> : <UnauthenticatedNavigator />}
    </NavigationContainer>
  );
}

// Unauthenticated: onboarding, login, signup
const AuthStack = createNativeStackNavigator<AuthStackParamList>();

function UnauthenticatedNavigator() {
  return (
    <AuthStack.Navigator screenOptions={{ headerShown: false }}>
      <AuthStack.Screen name="Welcome" component={WelcomeScreen} />
      <AuthStack.Screen name="Login" component={LoginScreen} />
      <AuthStack.Screen name="Signup" component={SignupScreen} />
      <AuthStack.Screen name="ForgotPassword" component={ForgotPasswordScreen} />
    </AuthStack.Navigator>
  );
}

// Authenticated: main app
function AuthenticatedNavigator() {
  return (
    <RootStack.Navigator>
      <RootStack.Screen name="Tabs" component={TabNavigator} />
      {/* ... */}
    </RootStack.Navigator>
  );
}

// When user signs in, just update the auth context
// React Navigation automatically transitions to AuthenticatedNavigator
function LoginScreen() {
  const { signIn } = useAuth();

  const handleLogin = async () => {
    const token = await api.login(email, password);
    await signIn(token); // triggers navigator swap
  };
}
```

### Header Customization

```tsx
<Stack.Screen
  name="ArticleDetail"
  component={ArticleDetailScreen}
  options={({ route, navigation }) => ({
    // Dynamic title from params
    title: route.params.title,

    // Custom header right button
    headerRight: () => (
      <Pressable onPress={() => navigation.navigate('Share', { articleId: route.params.articleId })}>
        <ShareIcon />
      </Pressable>
    ),

    // Back button label (iOS only)
    headerBackTitle: 'Feed',

    // Transparent header with absolute positioning
    headerTransparent: true,
    headerBlurEffect: 'regular', // iOS blur effect

    // Custom header
    header: ({ navigation, route, options, back }) => (
      <CustomHeader title={options.title} onBack={back ? navigation.goBack : undefined} />
    ),
  })}
/>
```

### Deep Links

```tsx
// navigation/linking.ts
import type { LinkingOptions } from '@react-navigation/native';
import type { RootStackParamList } from './types';

export const linking: LinkingOptions<RootStackParamList> = {
  prefixes: [
    'myapp://',
    'https://myapp.example.com',
    'https://www.myapp.example.com',
  ],
  config: {
    screens: {
      Tabs: {
        screens: {
          HomeStack: {
            screens: {
              Feed: 'feed',
              ArticleDetail: 'article/:articleId',
            },
          },
          ProfileStack: {
            screens: {
              Profile: 'user/:userId',
            },
          },
        },
      },
      Modal: 'modal',
    },
  },

  // Custom URL parsing (optional)
  getStateFromPath: (path, options) => {
    // Handle legacy URLs, query params, etc.
    if (path.startsWith('/legacy/')) {
      const id = path.replace('/legacy/', '');
      return { routes: [{ name: 'ArticleDetail', params: { articleId: id } }] };
    }
    // Fall back to default parsing
    return getStateFromPath(path, options);
  },
};
```

**Testing deep links:**

```bash
# iOS Simulator
xcrun simctl openurl booted "myapp://article/123"

# Android Emulator
adb shell am start -a android.intent.action.VIEW -d "myapp://article/123"

# Expo
npx uri-scheme open "myapp://article/123" --ios
```

### Universal Links (HTTPS deep links)

Universal links require server-side configuration files and eliminate the custom scheme.

**iOS — Apple App Site Association (AASA)**
```json
// Serve at: https://myapp.example.com/.well-known/apple-app-site-association
{
  "applinks": {
    "apps": [],
    "details": [
      {
        "appID": "TEAMID.com.example.myapp",
        "paths": ["/article/*", "/user/*"]
      }
    ]
  }
}
```

**Android — Asset Links**
```json
// Serve at: https://myapp.example.com/.well-known/assetlinks.json
[
  {
    "relation": ["delegate_permission/common.handle_all_urls"],
    "target": {
      "namespace": "android_app",
      "package_name": "com.example.myapp",
      "sha256_cert_fingerprints": ["<YOUR_CERT_FINGERPRINT>"]
    }
  }
]
```

---

## Common Patterns & Best Practices

### Navigation Event Listeners

```tsx
// Run code when screen comes into focus (e.g., refresh data)
useFocusEffect(
  useCallback(() => {
    fetchLatestData();

    return () => {
      // Optional cleanup when screen loses focus
    };
  }, [])
);

// Listen to navigation events
useEffect(() => {
  const unsubscribe = navigation.addListener('beforeRemove', (e) => {
    if (!hasUnsavedChanges) return;

    e.preventDefault(); // Prevent the default action (going back)

    Alert.alert(
      'Discard changes?',
      'You have unsaved changes. Discard them and leave the screen?',
      [
        { text: "Don't leave", style: 'cancel' },
        {
          text: 'Discard',
          style: 'destructive',
          onPress: () => navigation.dispatch(e.data.action),
        },
      ]
    );
  });

  return unsubscribe;
}, [navigation, hasUnsavedChanges]);
```

### Passing Callbacks via Navigation

Do not pass functions as navigation params — they are not serializable and break deep linking. Use context or a state manager instead.

```tsx
// BAD
navigation.navigate('Picker', { onSelect: handleSelect }); // not serializable

// GOOD: use an event emitter or shared state
const pickerStore = usePickerStore();
navigation.navigate('Picker', { pickerId: 'destination' });

// In Picker screen:
const handleConfirm = (value) => {
  pickerStore.confirm('destination', value);
  navigation.goBack();
};
```

---

## Anti-Patterns to Avoid

**Passing non-serializable params.** Functions, class instances, and complex objects as navigation params break deep linking, state persistence, and React Navigation's own serialization. Use primitive types: strings, numbers, booleans, and plain objects.

**Calling `navigation.navigate` during render.** Navigation must be triggered by user actions or effects, never during the render phase.

**Using `navigation.reset` excessively.** `reset` destroys the entire navigation state. Use it only for logout or onboarding completion. For redirecting after an action, use `replace` or conditional rendering.

**Deep nesting without purpose.** More than 3 levels of nested navigators become hard to reason about and type. Flatten your structure where possible.

**Forgetting `headerShown: false` on nested navigators.** Each nested navigator adds its own header by default, resulting in double headers. Explicitly set `headerShown: false` on nested navigators and let the parent manage the header.

---

## Debugging & Troubleshooting

**"Cannot navigate to screen X" in deep link**
The screen is not registered in the linking config or the navigator order is wrong. Use `getStateFromPath` to debug: log what state it produces for your URL.

**Double header on nested tab screens**
Add `headerShown: false` to the Tab.Navigator `screenOptions`, and let each inner stack manage its own header.

**`useFocusEffect` firing on initial mount AND on focus**
This is expected behavior. Use a ref to track initial mount if you need to skip it.

**Back button on Android not working**
Ensure the root navigator receives focus. Physical back button fires `navigation.goBack()` — if your navigator has no screens to go back to, it exits the app (correct behavior).

---

## Real-World Scenarios

### Tab with Badge Count

```tsx
function TabNavigator() {
  const { unreadCount } = useNotifications();

  return (
    <Tab.Navigator>
      <Tab.Screen
        name="Inbox"
        component={InboxScreen}
        options={{
          tabBarBadge: unreadCount > 0 ? (unreadCount > 99 ? '99+' : unreadCount) : undefined,
        }}
      />
    </Tab.Navigator>
  );
}
```

### Onboarding Flow with Persistence

```tsx
function RootNavigator() {
  const { user } = useAuth();
  const [hasSeenOnboarding] = usePersistedState('onboarding_complete', false);

  if (!user) return <AuthNavigator />;
  if (!hasSeenOnboarding) return <OnboardingNavigator />;
  return <MainNavigator />;
}
```

---

## Typed Navigation with TypeScript

Type every screen and its params to catch navigation errors at compile time.

```typescript
// types/navigation.ts — define every screen and its param contract
import type { NativeStackNavigationProp } from '@react-navigation/native-stack';
import type { BottomTabNavigationProp } from '@react-navigation/bottom-tabs';
import type { RouteProp } from '@react-navigation/native';

export type RootStackParamList = {
  Home: undefined;                           // no params
  Profile: { userId: string };               // required param
  Post: { postId: string; title?: string };  // optional param
  EditPost: { postId: string; draft?: boolean };
};

export type TabParamList = {
  Feed: undefined;
  Search: { query?: string };
  Notifications: undefined;
  Settings: undefined;
};

// Per-screen typed navigation prop
export type HomeScreenNavigationProp = NativeStackNavigationProp<
  RootStackParamList,
  'Home'
>;

// Per-screen typed route prop
export type ProfileScreenRouteProp = RouteProp<RootStackParamList, 'Profile'>;
```

```typescript
// ProfileScreen.tsx — typed params, no runtime guessing
import { useRoute } from '@react-navigation/native';
import type { ProfileScreenRouteProp } from '../types/navigation';

export function ProfileScreen() {
  const route = useRoute<ProfileScreenRouteProp>();
  // route.params.userId is string — guaranteed, no undefined check
  const { userId } = route.params;

  return <UserProfile userId={userId} />;
}
```

```typescript
// Global type declaration — makes useNavigation() inferred everywhere
// Add this once, e.g. in types/navigation.ts

declare global {
  namespace ReactNavigation {
    interface RootParamList extends RootStackParamList {}
  }
}

// Now useNavigation() is fully typed without the generic everywhere:
import { useNavigation } from '@react-navigation/native';

function SomeButton() {
  const navigation = useNavigation(); // inferred type
  // navigation.navigate('Post', { postId: '123' });  ✓
  // navigation.navigate('Post', {});                 ✗ TS error: postId required
  // navigation.navigate('Nonexistent');              ✗ TS error: not in param list
}
```

---

## Universal Links (iOS) and App Links (Android)

Universal Links and App Links open your app when a user taps a URL on the web — no custom scheme needed, no "Open in app?" prompt.

**Step 1 — Host the verification file:**

```json
// https://yourdomain.com/.well-known/apple-app-site-association (iOS)
{
  "applinks": {
    "apps": [],
    "details": [
      {
        "appID": "TEAMID.com.yourcompany.yourapp",
        "paths": ["/post/*", "/profile/*", "/invite/*", "/NOT /admin/*"]
      }
    ]
  }
}
```

```json
// https://yourdomain.com/.well-known/assetlinks.json (Android)
[{
  "relation": ["delegate_permission/common.handle_all_urls"],
  "target": {
    "namespace": "android_app",
    "package_name": "com.yourcompany.yourapp",
    "sha256_cert_fingerprints": ["AB:CD:EF:..."]
  }
}]
```

The file must be served with `Content-Type: application/json` and no redirects on the verification path.

**Step 2 — Configure in app.json:**

```json
{
  "expo": {
    "ios": {
      "associatedDomains": ["applinks:yourdomain.com"]
    },
    "android": {
      "intentFilters": [
        {
          "action": "VIEW",
          "autoVerify": true,
          "data": [
            {
              "scheme": "https",
              "host": "yourdomain.com",
              "pathPrefix": "/post"
            },
            {
              "scheme": "https",
              "host": "yourdomain.com",
              "pathPrefix": "/profile"
            }
          ],
          "category": ["BROWSABLE", "DEFAULT"]
        }
      ]
    }
  }
}
```

**Step 3 — Wire into React Navigation linking config:**

```typescript
// navigation/linking.ts
import type { LinkingOptions } from '@react-navigation/native';
import type { RootStackParamList } from '../types/navigation';

export const linking: LinkingOptions<RootStackParamList> = {
  prefixes: [
    'https://yourdomain.com',
    'myapp://', // custom scheme fallback for dev
  ],
  config: {
    screens: {
      Home: '',
      Post: 'post/:postId',
      Profile: 'profile/:userId',
    },
  },
};

// In NavigationContainer:
// <NavigationContainer linking={linking} fallback={<LoadingScreen />}>
```

**Test on iOS simulator:**

```bash
xcrun simctl openurl booted "https://yourdomain.com/post/123"
```

**Test on Android emulator:**

```bash
adb shell am start -W -a android.intent.action.VIEW -d "https://yourdomain.com/post/123"
```

---

## Bottom Sheet with Navigation

`@gorhom/bottom-sheet` is the standard for dismissible bottom drawers in React Native. It integrates with Gesture Handler and Reanimated for smooth 60fps animations.

```bash
npx expo install @gorhom/bottom-sheet react-native-gesture-handler react-native-reanimated
```

Wrap the app root with `GestureHandlerRootView`:

```tsx
// App.tsx / _layout.tsx
import { GestureHandlerRootView } from 'react-native-gesture-handler';

export default function RootLayout() {
  return (
    <GestureHandlerRootView style={{ flex: 1 }}>
      <NavigationContainer>
        <RootNavigator />
      </NavigationContainer>
    </GestureHandlerRootView>
  );
}
```

```tsx
// components/FilterSheet.tsx
import BottomSheet, {
  BottomSheetView,
  BottomSheetBackdrop,
} from '@gorhom/bottom-sheet';
import { useRef, useCallback, useMemo } from 'react';
import type { BottomSheetBackdropProps } from '@gorhom/bottom-sheet';

interface Props {
  onClose: () => void;
}

export function FilterSheet({ onClose }: Props) {
  const ref = useRef<BottomSheet>(null);
  const snapPoints = useMemo(() => ['40%', '75%'], []);

  const renderBackdrop = useCallback(
    (props: BottomSheetBackdropProps) => (
      <BottomSheetBackdrop
        {...props}
        disappearsOnIndex={-1}
        appearsOnIndex={0}
        pressBehavior="close"
      />
    ),
    [],
  );

  const handleChange = useCallback(
    (index: number) => {
      if (index === -1) onClose();
    },
    [onClose],
  );

  return (
    <BottomSheet
      ref={ref}
      index={0}           // 0 = first snap point on mount
      snapPoints={snapPoints}
      enablePanDownToClose
      backdropComponent={renderBackdrop}
      onChange={handleChange}
    >
      <BottomSheetView style={styles.content}>
        <FilterOptions />
      </BottomSheetView>
    </BottomSheet>
  );
}
```

**Pattern: sheet controlled by parent state:**

```tsx
function SearchScreen() {
  const [showFilters, setShowFilters] = useState(false);

  return (
    <View style={{ flex: 1 }}>
      <SearchBar onFilterPress={() => setShowFilters(true)} />
      <ResultsList />
      {showFilters && (
        <FilterSheet onClose={() => setShowFilters(false)} />
      )}
    </View>
  );
}
```

---

## Further Reading

- [React Navigation docs](https://reactnavigation.org/docs/getting-started)
- [React Navigation — Type checking](https://reactnavigation.org/docs/typescript)
- [React Navigation — Deep linking](https://reactnavigation.org/docs/deep-linking)
- [Universal links on iOS](https://developer.apple.com/documentation/xcode/supporting-associated-domains)
- [Android App Links](https://developer.android.com/training/app-links)

---

## Summary

React Navigation composes stack, tab, and drawer navigators into a tree. Type safety requires declaring a `ParamList` type for every navigator and using `CompositeScreenProps` for nested screens. The auth flow pattern swaps navigator trees based on auth state — no redirects. Deep links map URL paths to screen params via the `linking` config on `NavigationContainer`. Universal links require server-side JSON files (`apple-app-site-association` and `assetlinks.json`). Avoid passing non-serializable values as navigation params — they break deep linking.
