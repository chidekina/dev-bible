# React Native Advanced

## Overview

Once your React Native app is functional, the next challenges are performance, complex animations, and native integration. This chapter covers the tools and patterns that distinguish production-quality apps from prototypes: measuring and eliminating frame drops, bridging native code, creating fluid animations with Reanimated and Gesture Handler, and implementing deep linking.

React Native's architecture is central to understanding performance. The JS thread runs your React code. The UI thread renders native views. Communication between them (the "bridge") was historically the bottleneck. React Native's new architecture (JSI + Fabric) eliminates the bridge for synchronous native access — Reanimated 2+ and the New Architecture leverage this directly.

---

## Prerequisites

- [React Native Basics](react-native-basics.md) — core components, StyleSheet, navigation
- Understanding of React hooks (useMemo, useCallback, useRef)
- Basic familiarity with JavaScript engine concepts (event loop, microtasks)

---

## Core Concepts

### The Threading Model

```
JS Thread          UI Thread           Native Modules Thread
─────────          ─────────           ──────────────────────
React render       View updates        Camera, Location, etc.
Business logic     Gestures            File I/O
State updates      Animations          Network (lower level)
Event handling

Old Architecture: JS ↔ Bridge (async, serialized JSON) ↔ Native
New Architecture: JS ↔ JSI (synchronous C++ bindings) ↔ Native
```

A dropped frame occurs when the UI thread is blocked for more than 16ms (60fps). Common causes:
- Heavy JS computation on the JS thread during scrolling
- Too many `setState` calls in rapid succession
- Large component trees re-rendering on every scroll offset change
- JavaScript-driven animations (use Reanimated instead)

### Hermes

Hermes is a JavaScript engine optimized for React Native. Enable it for significantly faster startup and lower memory usage.

```json
// app.json (Expo)
{
  "expo": {
    "jsEngine": "hermes"
  }
}
```

Hermes pre-compiles JS to bytecode at build time, eliminating parse time at startup. It also ships with a built-in memory profiler accessible via Chrome DevTools.

### Performance Profiling

```
React DevTools Profiler → identify slow renders
Flipper Performance → JS/UI thread frame timeline
Android Studio CPU Profiler → native thread analysis
Instruments (Xcode) → iOS memory and CPU profiling
```

**Finding slow renders:**
```tsx
// Wrap with Profiler to measure render cost
import { Profiler } from 'react';

function onRenderCallback(id, phase, actualDuration) {
  if (actualDuration > 16) {
    console.warn(`Slow render: ${id} took ${actualDuration.toFixed(2)}ms`);
  }
}

<Profiler id="ItemList" onRender={onRenderCallback}>
  <ItemList data={data} />
</Profiler>
```

---

## Hands-On Examples

### Optimizing FlatList Performance

```tsx
import { FlatList, View } from 'react-native';

// BAD: creates new functions and objects every render
function BadList({ items }) {
  return (
    <FlatList
      data={items}
      renderItem={({ item }) => <ItemCard item={item} />} // new function every render
      keyExtractor={(item) => item.id}                    // new function every render
      contentContainerStyle={{ padding: 16 }}            // new object every render
    />
  );
}

// GOOD: stable references
const CONTENT_STYLE = { padding: 16 };

function GoodList({ items }: { items: Item[] }) {
  const renderItem = useCallback(
    ({ item }: { item: Item }) => <ItemCard item={item} />,
    []
  );

  const keyExtractor = useCallback((item: Item) => item.id, []);

  const getItemLayout = useCallback(
    (_: unknown, index: number) => ({
      length: ITEM_HEIGHT,
      offset: ITEM_HEIGHT * index,
      index,
    }),
    []
  );

  return (
    <FlatList
      data={items}
      renderItem={renderItem}
      keyExtractor={keyExtractor}
      contentContainerStyle={CONTENT_STYLE}
      getItemLayout={getItemLayout} // skip measurement if fixed height
      removeClippedSubviews          // unmount offscreen items
      maxToRenderPerBatch={8}        // items rendered per JS cycle
      updateCellsBatchingPeriod={50} // ms between batch renders
      windowSize={11}                // render 5 viewports up and down
      initialNumToRender={10}        // items on first render
    />
  );
}

// ItemCard must be memoized
const ItemCard = React.memo(function ItemCard({ item }: { item: Item }) {
  return (
    <View>
      <Text>{item.title}</Text>
    </View>
  );
});
```

### Reanimated 2 — Native Animations

```bash
npx expo install react-native-reanimated
# Add to babel.config.js: plugins: ['react-native-reanimated/plugin']
```

Reanimated runs animation logic on the UI thread, bypassing the JS thread entirely.

```tsx
import Animated, {
  useSharedValue,
  useAnimatedStyle,
  withSpring,
  withTiming,
  withSequence,
  withRepeat,
  Easing,
  interpolate,
  Extrapolation,
  runOnJS,
} from 'react-native-reanimated';

// Simple spring animation
function SpringButton() {
  const scale = useSharedValue(1);

  const animatedStyle = useAnimatedStyle(() => ({
    transform: [{ scale: scale.value }],
  }));

  return (
    <Pressable
      onPressIn={() => { scale.value = withSpring(0.95); }}
      onPressOut={() => { scale.value = withSpring(1); }}
    >
      <Animated.View style={[styles.button, animatedStyle]}>
        <Text>Press me</Text>
      </Animated.View>
    </Pressable>
  );
}

// Scroll-based header animation
function AnimatedHeader({ scrollY }: { scrollY: Animated.SharedValue<number> }) {
  const animatedStyle = useAnimatedStyle(() => {
    const opacity = interpolate(
      scrollY.value,
      [0, 100],
      [0, 1],
      Extrapolation.CLAMP
    );
    const translateY = interpolate(
      scrollY.value,
      [0, 100],
      [-50, 0],
      Extrapolation.CLAMP
    );
    return { opacity, transform: [{ translateY }] };
  });

  return <Animated.View style={[styles.header, animatedStyle]} />;
}

// Pulsing loading indicator
function PulsingDot() {
  const opacity = useSharedValue(1);

  useEffect(() => {
    opacity.value = withRepeat(
      withSequence(
        withTiming(0.2, { duration: 800, easing: Easing.inOut(Easing.ease) }),
        withTiming(1, { duration: 800, easing: Easing.inOut(Easing.ease) })
      ),
      -1, // infinite
      true // reverse
    );
  }, []);

  const animatedStyle = useAnimatedStyle(() => ({ opacity: opacity.value }));

  return <Animated.View style={[styles.dot, animatedStyle]} />;
}
```

### Gesture Handler

```bash
npx expo install react-native-gesture-handler
```

```tsx
import { GestureDetector, Gesture } from 'react-native-gesture-handler';
import Animated, {
  useSharedValue,
  useAnimatedStyle,
  withSpring,
} from 'react-native-reanimated';

// Draggable card with snap-back
function DraggableCard() {
  const offsetX = useSharedValue(0);
  const offsetY = useSharedValue(0);
  const savedOffsetX = useSharedValue(0);
  const savedOffsetY = useSharedValue(0);

  const gesture = Gesture.Pan()
    .onStart(() => {
      savedOffsetX.value = offsetX.value;
      savedOffsetY.value = offsetY.value;
    })
    .onUpdate((e) => {
      offsetX.value = savedOffsetX.value + e.translationX;
      offsetY.value = savedOffsetY.value + e.translationY;
    })
    .onEnd(() => {
      // Snap back to origin
      offsetX.value = withSpring(0);
      offsetY.value = withSpring(0);
    });

  const animatedStyle = useAnimatedStyle(() => ({
    transform: [
      { translateX: offsetX.value },
      { translateY: offsetY.value },
    ],
  }));

  return (
    <GestureDetector gesture={gesture}>
      <Animated.View style={[styles.card, animatedStyle]} />
    </GestureDetector>
  );
}

// Swipe to dismiss
function SwipeToDismiss({ onDismiss, children }) {
  const translateX = useSharedValue(0);
  const DISMISS_THRESHOLD = 150;

  const gesture = Gesture.Pan()
    .activeOffsetX([-10, 10]) // only activate on horizontal swipe
    .onUpdate((e) => {
      translateX.value = e.translationX;
    })
    .onEnd((e) => {
      if (Math.abs(e.translationX) > DISMISS_THRESHOLD) {
        const direction = e.translationX > 0 ? 400 : -400;
        translateX.value = withSpring(direction);
        runOnJS(onDismiss)();
      } else {
        translateX.value = withSpring(0);
      }
    });

  const animatedStyle = useAnimatedStyle(() => ({
    transform: [{ translateX: translateX.value }],
  }));

  return (
    <GestureDetector gesture={gesture}>
      <Animated.View style={animatedStyle}>{children}</Animated.View>
    </GestureDetector>
  );
}
```

### Native Modules (Expo Modules API)

When you need to access native platform APIs not covered by Expo's SDK:

```typescript
// modules/my-native-module/index.ts
import { requireNativeModule } from 'expo-modules-core';

const MyNativeModule = requireNativeModule('MyNativeModule');

export async function getBiometricToken(): Promise<string> {
  return MyNativeModule.getBiometricToken();
}
```

For complex native modules, use `expo-module-scripts` to scaffold:

```bash
npx create-expo-module my-module
```

### Deep Linking

```tsx
// app.json
{
  "expo": {
    "scheme": "myapp",
    "ios": { "bundleIdentifier": "com.example.myapp" },
    "android": { "package": "com.example.myapp" }
  }
}

// Linking configuration for React Navigation
const linking = {
  prefixes: ['myapp://', 'https://myapp.example.com'],
  config: {
    screens: {
      Home: '',
      Profile: 'user/:userId',
      Settings: 'settings',
      // Nested
      Feed: {
        screens: {
          Article: 'article/:articleId',
        },
      },
    },
  },
};

function App() {
  return (
    <NavigationContainer linking={linking} fallback={<ActivityIndicator />}>
      <RootNavigator />
    </NavigationContainer>
  );
}
```

---

## Common Patterns & Best Practices

### Memoization Strategy

```tsx
// Only memo components that:
// 1. Render frequently (in lists, animations)
// 2. Receive stable props (primitives or memoized objects)
// 3. Are expensive to render

const ExpensiveItem = React.memo(
  function ExpensiveItem({ item, onPress }: Props) {
    return <View>...</View>;
  },
  // Custom equality — only re-render if id or title changed
  (prev, next) => prev.item.id === next.item.id && prev.item.title === next.item.title
);

// Memoize callbacks passed to list items
const handlePress = useCallback((id: string) => {
  navigation.navigate('Detail', { id });
}, [navigation]);

// Memoize expensive derivations
const sortedItems = useMemo(
  () => [...items].sort((a, b) => b.createdAt - a.createdAt),
  [items]
);
```

### InteractionManager — Defer Heavy Work

```tsx
import { InteractionManager } from 'react-native';

function HeavyScreen() {
  const [ready, setReady] = useState(false);

  useEffect(() => {
    // Wait for navigation animation to complete before rendering heavy content
    const task = InteractionManager.runAfterInteractions(() => {
      setReady(true);
    });
    return () => task.cancel();
  }, []);

  if (!ready) return <LoadingPlaceholder />;
  return <HeavyContent />;
}
```

---

## Anti-Patterns to Avoid

**Animating with `setState`.** State updates cause React re-renders; 60fps animations cannot be driven by JS state. Use Reanimated shared values on the UI thread.

**Using `Animated` (core) API for complex animations.** React Native's built-in `Animated` API runs on the JS thread by default (unless you use `useNativeDriver: true`). Always prefer Reanimated 2+ for new animations.

**Setting `useNativeDriver: true` for layout properties.** Native driver only supports transform and opacity. Using it on `width`, `height`, or `margin` crashes in production.

**Nesting VirtualizedList inside ScrollView.** FlatList inside ScrollView disables virtualization — all rows render at once. Use `SectionList` or a custom header approach instead.

**Using `console.log` in production.** Every log call is synchronous on the JS thread. Use a logging library with production log stripping (`babel-plugin-transform-remove-console`).

---

## Debugging & Troubleshooting

### JS Thread vs UI Thread Diagnosis

- **Jerky animations** — the UI thread is dropping frames; often caused by heavy style recalculation or a gesture that triggers state updates
- **Delayed button responses** — the JS thread is busy; profile with the React DevTools Profiler
- **Smooth gesture, janky post-gesture update** — bridge latency; consider keeping the UI state in a shared value until the gesture ends, then commit once

### Hermes Debugger

```bash
# Connect Chrome DevTools to Hermes
# Expo: shake device > Open JS Debugger
# Bare: adb forward tcp:8081 tcp:8081 then chrome://inspect
```

### Reanimated Worklet Errors

Worklets (functions that run on the UI thread) cannot access variables from the React closure without `runOnJS` or explicit capture. The error `Trying to access property 'X' of an undefined worklet` means you referenced a non-worklet function or a closure variable not captured.

```tsx
// BAD: someFunction is not a worklet
const gesture = Gesture.Tap().onEnd(() => {
  someFunction(); // Error: not a worklet
});

// GOOD: run on JS thread explicitly
const gesture = Gesture.Tap().onEnd(() => {
  runOnJS(someFunction)();
});
```

---

## Real-World Scenarios

### Infinite Scroll Feed with Pagination

```tsx
function InfiniteList() {
  const { data, fetchNextPage, hasNextPage, isFetchingNextPage } =
    useInfiniteQuery({
      queryKey: ['items'],
      queryFn: ({ pageParam = 0 }) => fetchItems({ cursor: pageParam }),
      getNextPageParam: (lastPage) => lastPage.nextCursor,
    });

  const items = data?.pages.flatMap((p) => p.items) ?? [];

  const onEndReached = useCallback(() => {
    if (hasNextPage && !isFetchingNextPage) fetchNextPage();
  }, [hasNextPage, isFetchingNextPage, fetchNextPage]);

  return (
    <FlatList
      data={items}
      renderItem={renderItem}
      keyExtractor={keyExtractor}
      onEndReached={onEndReached}
      onEndReachedThreshold={0.5}
      ListFooterComponent={
        isFetchingNextPage ? <ActivityIndicator /> : null
      }
    />
  );
}
```

---

## Further Reading

- [Reanimated documentation](https://docs.swmansion.com/react-native-reanimated/)
- [Gesture Handler documentation](https://docs.swmansion.com/react-native-gesture-handler/)
- [React Native New Architecture](https://reactnative.dev/docs/new-architecture-intro)
- [Expo Modules API](https://docs.expo.dev/modules/overview/)
- [Flashlist — alternative to FlatList](https://shopify.github.io/flash-list/)

---

## Summary

Performance in React Native means keeping the JS thread and UI thread at 60fps. Profile before optimizing. Memoize `renderItem`, `keyExtractor`, and list item components. Use Reanimated 2+ for all animations — it runs on the UI thread and sidesteps the bridge. Gesture Handler provides gesture recognizers that also run natively. Avoid driving animations with `setState`. Deep linking requires a URL scheme in `app.json` and a `linking` config on the `NavigationContainer`. Native modules can be written with the Expo Modules API for clean TypeScript-first bridges.
