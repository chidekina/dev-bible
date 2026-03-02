# Native Device APIs

## Overview

Expo provides a curated set of native API wrappers that work across iOS and Android without writing any native code. Camera, location, push notifications, sensors, haptics, and media library are all available through `expo-*` packages installed via `npx expo install` — which pins the correct SDK-compatible version automatically. The permission model is consistent across all of them: request permission → check the result → use the API or show fallback UI.

This chapter covers the most commonly needed native APIs with production-ready patterns. Every example handles permissions correctly and includes cleanup for continuous operations (location watchers, sensor subscriptions, notification listeners) to prevent battery drain and memory leaks. Reading this chapter assumes you have an Expo SDK 50+ project with Expo Router or React Navigation already configured.

---

## Prerequisites

- [React Native Basics](react-native-basics.md) — core components, StyleSheet, navigation
- [React Native Advanced](react-native-advanced.md) — hooks, useEffect cleanup, TypeScript
- Basic familiarity with `async/await` and React lifecycle

---

## Core Concepts

### The Permission Model

Every hardware-backed API requires explicit user permission. Expo standardizes this with a consistent pattern:

```typescript
// Every expo-* permission API follows this shape
const { status } = await SomeModule.requestPermissionsAsync();
// status: 'granted' | 'denied' | 'undetermined'

// Or use the declarative hook (preferred for UI-driven flows)
const [permission, requestPermission] = useSomePermissions();
// permission.granted: boolean
// permission.canAskAgain: boolean — false after user selects "Don't ask again"
```

**Key rules:**
- Request permissions at the moment of first use, not on app launch.
- If `canAskAgain` is `false`, the permission dialog will never show again — the only path forward is sending the user to the OS Settings screen via `Linking.openSettings()`.
- iOS requires usage description strings in `app.json`; without them the app crashes. Android requires `<uses-permission>` entries in the manifest — Expo handles this automatically based on which SDK packages you install.

```json
// app.json — required for iOS
{
  "expo": {
    "ios": {
      "infoPlist": {
        "NSCameraUsageDescription": "Used to scan product barcodes.",
        "NSLocationWhenInUseUsageDescription": "Used to show nearby stores.",
        "NSMicrophoneUsageDescription": "Used to record video."
      }
    }
  }
}
```

---

## Hands-On Examples

### 1. Camera (expo-camera)

```bash
npx expo install expo-camera
```

The `useCameraPermissions()` hook drives both the permission state and the request call. Wrap it early in the render tree so the loading state (`permission === null`) renders an empty view rather than flashing the wrong UI.

```tsx
import { CameraView, CameraType, useCameraPermissions } from 'expo-camera';
import { useRef, useState } from 'react';
import {
  Button,
  Linking,
  StyleSheet,
  Text,
  TouchableOpacity,
  View,
} from 'react-native';

export function CameraScreen() {
  const [facing, setFacing] = useState<CameraType>('back');
  const [permission, requestPermission] = useCameraPermissions();
  const cameraRef = useRef<CameraView>(null);

  // Permission state is loading
  if (!permission) {
    return <View style={styles.container} />;
  }

  // Permission denied — explain why and offer a path forward
  if (!permission.granted) {
    return (
      <View style={styles.container}>
        <Text style={styles.message}>
          Camera access is needed to take photos and scan barcodes.
        </Text>
        {permission.canAskAgain ? (
          <Button onPress={requestPermission} title="Grant Permission" />
        ) : (
          <Button
            onPress={() => Linking.openSettings()}
            title="Open Settings"
          />
        )}
      </View>
    );
  }

  const takePicture = async () => {
    const photo = await cameraRef.current?.takePictureAsync({ quality: 0.8 });
    if (photo) {
      console.log('Photo URI:', photo.uri);
      // Pass to media library, upload to server, etc.
    }
  };

  const toggleFacing = () =>
    setFacing((f) => (f === 'back' ? 'front' : 'back'));

  return (
    <View style={styles.container}>
      <CameraView style={styles.camera} facing={facing} ref={cameraRef}>
        <View style={styles.buttonRow}>
          <TouchableOpacity style={styles.button} onPress={toggleFacing}>
            <Text style={styles.buttonText}>Flip</Text>
          </TouchableOpacity>
          <TouchableOpacity style={styles.button} onPress={takePicture}>
            <Text style={styles.buttonText}>Snap</Text>
          </TouchableOpacity>
        </View>
      </CameraView>
    </View>
  );
}

const styles = StyleSheet.create({
  container: { flex: 1, justifyContent: 'center' },
  camera: { flex: 1 },
  message: { textAlign: 'center', paddingBottom: 16, paddingHorizontal: 24 },
  buttonRow: {
    flex: 1,
    flexDirection: 'row',
    backgroundColor: 'transparent',
    justifyContent: 'space-around',
    alignItems: 'flex-end',
    marginBottom: 32,
  },
  button: {
    padding: 16,
    backgroundColor: 'rgba(0,0,0,0.4)',
    borderRadius: 8,
  },
  buttonText: { fontSize: 18, fontWeight: '600', color: 'white' },
});
```

#### Barcode Scanner

`CameraView` has first-class barcode scanning support. Debounce `onBarcodeScanned` to prevent the callback firing multiple times per scan — the camera updates at 30fps and will emit the same barcode on consecutive frames.

```tsx
import { CameraView, useCameraPermissions, type BarcodeScanningResult } from 'expo-camera';
import { useRef, useState } from 'react';
import { StyleSheet, View } from 'react-native';

export function BarcodeScanner({
  onScan,
}: {
  onScan: (data: string) => void;
}) {
  const [permission] = useCameraPermissions();
  const lastScanRef = useRef<number>(0);
  const DEBOUNCE_MS = 300;

  if (!permission?.granted) return null;

  const handleBarcodeScanned = (result: BarcodeScanningResult) => {
    const now = Date.now();
    if (now - lastScanRef.current < DEBOUNCE_MS) return;
    lastScanRef.current = now;
    onScan(result.data);
  };

  return (
    <View style={styles.container}>
      <CameraView
        style={StyleSheet.absoluteFill}
        facing="back"
        onBarcodeScanned={handleBarcodeScanned}
        barcodeScannerSettings={{
          barcodeTypes: ['ean13', 'ean8', 'qr', 'code128'],
        }}
      />
    </View>
  );
}

const styles = StyleSheet.create({
  container: { flex: 1 },
});
```

---

### 2. Location (expo-location)

```bash
npx expo install expo-location
```

There are two permission tiers: **foreground** (app is visible) and **background** (runs when app is backgrounded or closed). Always start with foreground — background requires an additional OS-level approval and must be justified.

```typescript
import * as Location from 'expo-location';

// One-shot: get the current position once
export async function getCurrentLocation(): Promise<Location.LocationObject | null> {
  const { status } = await Location.requestForegroundPermissionsAsync();
  if (status !== 'granted') return null;

  return Location.getCurrentPositionAsync({
    accuracy: Location.Accuracy.Balanced,
  });
}
```

#### Continuous Location Tracking

`watchPositionAsync` returns a subscription object. Calling `subscriber.remove()` stops the watcher. Forgetting this cleanup means the GPS runs indefinitely, draining the battery even after the component unmounts.

```typescript
import { useEffect, useState } from 'react';
import * as Location from 'expo-location';

export function useCurrentLocation() {
  const [location, setLocation] = useState<Location.LocationObject | null>(null);
  const [error, setError] = useState<string | null>(null);

  useEffect(() => {
    let subscriber: Location.LocationSubscription | null = null;

    (async () => {
      const { status } = await Location.requestForegroundPermissionsAsync();
      if (status !== 'granted') {
        setError('Location permission denied');
        return;
      }

      subscriber = await Location.watchPositionAsync(
        {
          accuracy: Location.Accuracy.Balanced,
          timeInterval: 5000,    // minimum ms between updates
          distanceInterval: 10,  // minimum meters between updates
        },
        (loc) => setLocation(loc),
      );
    })();

    return () => {
      // ALWAYS clean up — or the watcher runs forever and drains battery
      subscriber?.remove();
    };
  }, []);

  return { location, error };
}
```

**Accuracy levels** (choose the lowest that meets your requirement — each step up increases power draw):

| Level | Description | Typical use case |
|---|---|---|
| `Accuracy.Lowest` | ~3 km | City-level display |
| `Accuracy.Low` | ~1 km | Regional maps |
| `Accuracy.Balanced` | ~100 m | General maps, delivery |
| `Accuracy.High` | ~10 m | Turn-by-turn navigation |
| `Accuracy.BestForNavigation` | ~1 m | Pedestrian navigation, sports tracking |

#### Background Location

Background location requires a separate permission request and an active `TaskManager` task. Only request this if your feature genuinely needs it — Apple rejects apps that request background location without clear justification.

```typescript
import * as Location from 'expo-location';
import * as TaskManager from 'expo-task-manager';

const LOCATION_TASK_NAME = 'background-location-task';

// Define the task at module level (top of the file, outside any component)
TaskManager.defineTask(LOCATION_TASK_NAME, ({ data, error }) => {
  if (error) {
    console.error('Background location error:', error.message);
    return;
  }
  const { locations } = data as { locations: Location.LocationObject[] };
  // Send to server, update local DB, etc.
  console.log('Background locations:', locations);
});

export async function startBackgroundTracking(): Promise<boolean> {
  const { status: fgStatus } =
    await Location.requestForegroundPermissionsAsync();
  if (fgStatus !== 'granted') return false;

  const { status: bgStatus } =
    await Location.requestBackgroundPermissionsAsync();
  if (bgStatus !== 'granted') return false;

  await Location.startLocationUpdatesAsync(LOCATION_TASK_NAME, {
    accuracy: Location.Accuracy.Balanced,
    timeInterval: 10000,   // 10 seconds
    distanceInterval: 20,  // 20 meters
    showsBackgroundLocationIndicator: true, // iOS: blue bar in status bar
  });

  return true;
}

export async function stopBackgroundTracking(): Promise<void> {
  await Location.stopLocationUpdatesAsync(LOCATION_TASK_NAME);
}
```

---

### 3. Push Notifications (expo-notifications)

```bash
npx expo install expo-notifications expo-device
```

Push notifications require an EAS project ID (from `app.json` via `eas build`). The Expo push service routes to Apple APNs (iOS) and Firebase FCM (Android) transparently.

#### Initial Setup

Call `setNotificationHandler` at module level — before any component renders — to control how notifications are displayed when the app is in the foreground.

```typescript
import * as Notifications from 'expo-notifications';

// Module-level: configure foreground display behavior once
Notifications.setNotificationHandler({
  handleNotification: async () => ({
    shouldShowAlert: true,
    shouldPlaySound: false,
    shouldSetBadge: true,
  }),
});
```

#### Registering for Push Tokens

```typescript
import * as Notifications from 'expo-notifications';
import * as Device from 'expo-device';
import Constants from 'expo-constants';

export async function registerForPushNotifications(): Promise<string | null> {
  // Push tokens do not work on simulators — always check first
  if (!Device.isDevice) {
    console.warn('Push notifications require a physical device');
    return null;
  }

  const { status: existingStatus } = await Notifications.getPermissionsAsync();
  let finalStatus = existingStatus;

  if (existingStatus !== 'granted') {
    const { status } = await Notifications.requestPermissionsAsync();
    finalStatus = status;
  }

  if (finalStatus !== 'granted') {
    return null;
  }

  const projectId = Constants.expoConfig?.extra?.eas?.projectId;
  if (!projectId) {
    throw new Error('EAS project ID missing from app.json > extra.eas.projectId');
  }

  const tokenData = await Notifications.getExpoPushTokenAsync({ projectId });
  return tokenData.data; // "ExponentPushToken[xxxx]" — store this on your backend
}
```

#### Listening for Notifications in a Component

```tsx
import { useEffect, useRef } from 'react';
import * as Notifications from 'expo-notifications';
import { useRouter } from 'expo-router';

export function usePushNotifications() {
  const router = useRouter();
  const notificationListener = useRef<Notifications.Subscription | null>(null);
  const responseListener = useRef<Notifications.Subscription | null>(null);

  useEffect(() => {
    // Fires when a notification arrives while app is in foreground
    notificationListener.current =
      Notifications.addNotificationReceivedListener((notification) => {
        console.log('Notification received:', notification.request.content);
      });

    // Fires when the user taps a notification (foreground or background)
    responseListener.current =
      Notifications.addNotificationResponseReceivedListener((response) => {
        const data = response.notification.request.content.data as {
          screen?: string;
        };
        if (data.screen) {
          router.push(`/${data.screen}`);
        }
      });

    return () => {
      notificationListener.current?.remove();
      responseListener.current?.remove();
    };
  }, [router]);
}
```

#### Sending from Your Server

The Expo Push API accepts a POST with an array of messages (up to 100 per request):

```typescript
// server-side: src/services/push.service.ts
interface PushMessage {
  to: string;       // "ExponentPushToken[xxxx]"
  title: string;
  body: string;
  data?: Record<string, unknown>;
  badge?: number;
}

export async function sendPushNotifications(
  messages: PushMessage[],
): Promise<void> {
  const response = await fetch('https://exp.host/--/api/v2/push/send', {
    method: 'POST',
    headers: {
      Accept: 'application/json',
      'Accept-Encoding': 'gzip, deflate',
      'Content-Type': 'application/json',
    },
    body: JSON.stringify(messages),
  });

  const result = await response.json();
  // result.data is an array of receipts — check for errors
  for (const receipt of result.data ?? []) {
    if (receipt.status === 'error') {
      console.error('Push send error:', receipt.message);
    }
  }
}

// Usage
await sendPushNotifications([
  {
    to: 'ExponentPushToken[xxxxxxxxxxxxxxxxxxxxxx]',
    title: 'Order shipped',
    body: 'Your order #1234 is on its way.',
    data: { screen: 'orders/1234' }, // passed to the response listener
    badge: 1,
  },
]);
```

---

### 4. Sensors (expo-sensors)

```bash
npx expo install expo-sensors
```

All sensor APIs follow the same subscription pattern: `Sensor.addListener(callback)` returns a subscription with a `.remove()` method. Call `.remove()` in the `useEffect` cleanup.

#### Accelerometer

Measures acceleration (in g-force) along x, y, z axes. Useful for shake detection, device orientation heuristics, or fitness apps.

```typescript
import { useEffect, useState } from 'react';
import { Accelerometer } from 'expo-sensors';

interface AccelerometerData {
  x: number;
  y: number;
  z: number;
}

export function useAccelerometer(updateIntervalMs = 100) {
  const [data, setData] = useState<AccelerometerData>({ x: 0, y: 0, z: 0 });
  const [available, setAvailable] = useState(true);

  useEffect(() => {
    let subscription: ReturnType<typeof Accelerometer.addListener> | null = null;

    Accelerometer.isAvailableAsync().then((isAvailable) => {
      if (!isAvailable) {
        setAvailable(false);
        return;
      }

      Accelerometer.setUpdateInterval(updateIntervalMs);
      subscription = Accelerometer.addListener(setData);
    });

    return () => {
      subscription?.remove(); // mandatory cleanup
    };
  }, [updateIntervalMs]);

  return { data, available };
}
```

#### Shake Detection with Accelerometer

```typescript
import { useEffect, useRef } from 'react';
import { Accelerometer } from 'expo-sensors';

const SHAKE_THRESHOLD = 1.8; // g-force — tune to taste

export function useShakeDetector(onShake: () => void) {
  const lastShakeRef = useRef(0);

  useEffect(() => {
    Accelerometer.setUpdateInterval(100);

    const subscription = Accelerometer.addListener(({ x, y, z }) => {
      const totalForce = Math.sqrt(x * x + y * y + z * z);

      if (
        totalForce > SHAKE_THRESHOLD &&
        Date.now() - lastShakeRef.current > 1000 // debounce 1s
      ) {
        lastShakeRef.current = Date.now();
        onShake();
      }
    });

    return () => subscription.remove();
  }, [onShake]);
}
```

#### Sensor Comparison

| Sensor | Measures | Common use cases |
|---|---|---|
| `Accelerometer` | Linear acceleration (g) | Shake detection, step counting, tilt |
| `Gyroscope` | Rotation rate (rad/s) | Screen auto-rotation, AR orientation |
| `Magnetometer` | Magnetic field (μT) | Compass, metal detection |
| `Barometer` | Atmospheric pressure (hPa) | Altitude, weather trends |
| `DeviceMotion` | Orientation + gravity (combined) | AR apps, device pitch/roll/yaw |
| `Pedometer` | Step count | Fitness, walking stats |

`DeviceMotion` is the highest-level API — it fuses accelerometer, gyroscope, and magnetometer into `rotation`, `rotationRate`, `acceleration`, and `orientation`. Use it when you need absolute device orientation rather than raw sensor data.

```typescript
import { DeviceMotion } from 'expo-sensors';

export function useDeviceOrientation() {
  const [orientation, setOrientation] =
    useState<DeviceMotion.DeviceMotionMeasurement | null>(null);

  useEffect(() => {
    DeviceMotion.setUpdateInterval(200);
    const subscription = DeviceMotion.addListener(setOrientation);
    return () => subscription.remove();
  }, []);

  // orientation.rotation: { alpha, beta, gamma } in radians
  // orientation.rotationRate: { alpha, beta, gamma } in rad/s
  return orientation;
}
```

---

### 5. Haptics (expo-haptics)

```bash
npx expo install expo-haptics
```

Haptics reinforce UI interactions by matching the physical weight of an action. Use the lightest feedback that still communicates the action clearly — overly heavy haptics feel aggressive.

```typescript
import * as Haptics from 'expo-haptics';

// Light tap — standard button press, list item selection
await Haptics.impactAsync(Haptics.ImpactFeedbackStyle.Light);

// Medium tap — card selection, toggle switch
await Haptics.impactAsync(Haptics.ImpactFeedbackStyle.Medium);

// Heavy tap — destructive action confirmation, long-press activation
await Haptics.impactAsync(Haptics.ImpactFeedbackStyle.Heavy);

// Selection changed — picker wheel tick, segmented control change
await Haptics.selectionAsync();

// Notification-style feedback — system events
await Haptics.notificationAsync(Haptics.NotificationFeedbackType.Success);
await Haptics.notificationAsync(Haptics.NotificationFeedbackType.Warning);
await Haptics.notificationAsync(Haptics.NotificationFeedbackType.Error);
```

**Practical usage — add haptics to interactive components:**

```tsx
import * as Haptics from 'expo-haptics';
import { Pressable, StyleSheet, Text } from 'react-native';

interface HapticButtonProps {
  onPress: () => void;
  label: string;
  destructive?: boolean;
}

export function HapticButton({ onPress, label, destructive }: HapticButtonProps) {
  const handlePress = async () => {
    await Haptics.impactAsync(
      destructive
        ? Haptics.ImpactFeedbackStyle.Heavy
        : Haptics.ImpactFeedbackStyle.Light,
    );
    onPress();
  };

  return (
    <Pressable
      style={[styles.button, destructive && styles.destructive]}
      onPress={handlePress}
    >
      <Text style={styles.label}>{label}</Text>
    </Pressable>
  );
}

const styles = StyleSheet.create({
  button: {
    paddingVertical: 12,
    paddingHorizontal: 24,
    backgroundColor: '#6366f1',
    borderRadius: 8,
    alignItems: 'center',
  },
  destructive: { backgroundColor: '#ef4444' },
  label: { color: 'white', fontWeight: '600', fontSize: 16 },
});
```

**Platform note:** iOS supports all `ImpactFeedbackStyle` and `NotificationFeedbackType` variants via the Taptic Engine. Android supports `impactAsync` and `selectionAsync` through basic vibration — the distinction between Light, Medium, and Heavy may not be perceptible on all Android devices.

---

### 6. Media Library and Image Picker

```bash
npx expo install expo-media-library expo-image-picker
```

`expo-media-library` manages saved assets (photos, videos) on the device. `expo-image-picker` provides the file picker UI for selecting from the library or triggering the camera.

#### Saving a Photo to Camera Roll

After capturing a photo with `CameraView`, save it to the device's photo library:

```typescript
import * as MediaLibrary from 'expo-media-library';

export async function savePhotoToCameraRoll(uri: string): Promise<boolean> {
  const { status } = await MediaLibrary.requestPermissionsAsync();
  if (status !== 'granted') return false;

  try {
    await MediaLibrary.saveToLibraryAsync(uri);
    return true;
  } catch (err) {
    console.error('Failed to save photo:', err);
    return false;
  }
}
```

#### Picking a Photo from the Library

```typescript
import * as ImagePicker from 'expo-image-picker';

export async function pickImageFromLibrary(): Promise<string | null> {
  const { status } =
    await ImagePicker.requestMediaLibraryPermissionsAsync();
  if (status !== 'granted') return null;

  const result = await ImagePicker.launchImageLibraryAsync({
    mediaTypes: ImagePicker.MediaTypeOptions.Images,
    allowsEditing: true,
    aspect: [4, 3],
    quality: 0.8, // 0–1; reduce for smaller upload size
  });

  if (result.canceled) return null;
  return result.assets[0].uri;
}
```

#### Picking with Camera

```typescript
export async function takePhotoWithPicker(): Promise<string | null> {
  const { status } = await ImagePicker.requestCameraPermissionsAsync();
  if (status !== 'granted') return null;

  const result = await ImagePicker.launchCameraAsync({
    allowsEditing: true,
    aspect: [1, 1], // square crop for avatars
    quality: 0.9,
  });

  if (result.canceled) return null;
  return result.assets[0].uri;
}
```

#### Avatar Picker Component

Combining picker + media library into a reusable component:

```tsx
import * as ImagePicker from 'expo-image-picker';
import { useState } from 'react';
import { Image, Pressable, StyleSheet, Text, View } from 'react-native';

export function AvatarPicker({
  onSelect,
}: {
  onSelect: (uri: string) => void;
}) {
  const [preview, setPreview] = useState<string | null>(null);

  const handlePress = async () => {
    const { status } =
      await ImagePicker.requestMediaLibraryPermissionsAsync();
    if (status !== 'granted') return;

    const result = await ImagePicker.launchImageLibraryAsync({
      mediaTypes: ImagePicker.MediaTypeOptions.Images,
      allowsEditing: true,
      aspect: [1, 1],
      quality: 0.8,
    });

    if (!result.canceled) {
      const uri = result.assets[0].uri;
      setPreview(uri);
      onSelect(uri);
    }
  };

  return (
    <Pressable style={styles.container} onPress={handlePress}>
      {preview ? (
        <Image source={{ uri: preview }} style={styles.avatar} />
      ) : (
        <View style={styles.placeholder}>
          <Text style={styles.placeholderText}>Tap to select photo</Text>
        </View>
      )}
    </Pressable>
  );
}

const styles = StyleSheet.create({
  container: { alignSelf: 'center' },
  avatar: { width: 100, height: 100, borderRadius: 50 },
  placeholder: {
    width: 100,
    height: 100,
    borderRadius: 50,
    backgroundColor: '#e5e7eb',
    justifyContent: 'center',
    alignItems: 'center',
  },
  placeholderText: { fontSize: 12, color: '#6b7280', textAlign: 'center' },
});
```

---

## Common Patterns

### Centralizing Permissions in a Hook

When multiple features need location or camera, request and cache the status in a shared hook to avoid duplicate dialogs:

```typescript
import { useCallback, useState } from 'react';
import * as Location from 'expo-location';
import * as Notifications from 'expo-notifications';
import { useCameraPermissions } from 'expo-camera';

export function useAppPermissions() {
  const [locationGranted, setLocationGranted] = useState(false);
  const [notificationsGranted, setNotificationsGranted] = useState(false);
  const [cameraPermission, requestCameraPermission] = useCameraPermissions();

  const requestLocation = useCallback(async () => {
    const { status } = await Location.requestForegroundPermissionsAsync();
    setLocationGranted(status === 'granted');
    return status === 'granted';
  }, []);

  const requestNotifications = useCallback(async () => {
    const { status } = await Notifications.requestPermissionsAsync();
    setNotificationsGranted(status === 'granted');
    return status === 'granted';
  }, []);

  return {
    camera: {
      granted: cameraPermission?.granted ?? false,
      request: requestCameraPermission,
    },
    location: {
      granted: locationGranted,
      request: requestLocation,
    },
    notifications: {
      granted: notificationsGranted,
      request: requestNotifications,
    },
  };
}
```

### "Open Settings" Fallback

When `canAskAgain` is false, the permission dialog is permanently suppressed. Show a clear message and a button that opens the OS settings pane directly:

```tsx
import { Alert, Linking } from 'react-native';

export function openPermissionSettings(permissionName: string) {
  Alert.alert(
    `${permissionName} Required`,
    `Please enable ${permissionName} in Settings to use this feature.`,
    [
      { text: 'Cancel', style: 'cancel' },
      {
        text: 'Open Settings',
        onPress: () => Linking.openSettings(),
      },
    ],
  );
}
```

---

## Anti-Patterns to Avoid

**Calling the API before requesting permission.** All hardware APIs throw or return null without a grant. Always check `permission.granted` before calling.

**Not cleaning up subscriptions.** Location watchers, sensor subscriptions, and notification listeners left running after a component unmounts drain battery and cause memory leaks. Every `addListener` or `watchPositionAsync` call must have a matching `.remove()` in the `useEffect` cleanup.

```tsx
// BAD: subscription leaks when the component unmounts
useEffect(() => {
  Accelerometer.addListener(setData);
}, []);

// GOOD: always return the cleanup
useEffect(() => {
  const sub = Accelerometer.addListener(setData);
  return () => sub.remove();
}, []);
```

**Requesting all permissions on launch.** Asking for camera + location + notifications the moment the app opens leads to users denying everything reflexively. Request each permission at the point where the feature is first used.

**Not handling `canAskAgain: false`.** A denied permission with `canAskAgain: false` means the OS dialog is gone permanently. If you only show `requestPermission()` in this state, tapping it does nothing — the user is stuck. Always check `canAskAgain` and show `Linking.openSettings()` as the fallback.

**Using `expo-image-picker` for the camera in a Camera screen.** `launchCameraAsync` opens a system camera modal — the user leaves your app briefly. When you need a live camera view with custom UI (overlays, real-time processing, barcode scanning), use `expo-camera` directly.

**Not debouncing barcode scans.** `onBarcodeScanned` fires on every camera frame where a barcode is visible — typically 10–30 times per second. Without a debounce, you will make 30 API calls per scan. Use a `useRef` timestamp check with a 300ms threshold.

---

## Real-World Scenarios

### Scenario 1: Food Delivery — Driver Location Tracking

A driver app that sends real-time location to dispatch. Requires background location since drivers lock their phones.

```typescript
import * as Location from 'expo-location';
import * as TaskManager from 'expo-task-manager';

const DRIVER_LOCATION_TASK = 'driver-location-task';

// Define task at module level
TaskManager.defineTask(DRIVER_LOCATION_TASK, async ({ data, error }) => {
  if (error || !data) return;

  const { locations } = data as { locations: Location.LocationObject[] };
  const latest = locations[locations.length - 1];

  // Send to dispatch API (fire-and-forget in background)
  fetch('/api/driver/location', {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({
      lat: latest.coords.latitude,
      lng: latest.coords.longitude,
      accuracy: latest.coords.accuracy,
      timestamp: latest.timestamp,
    }),
  }).catch(() => {
    // silently fail — next update will catch up
  });
});

export async function startDriverTracking(): Promise<boolean> {
  const { status: fg } = await Location.requestForegroundPermissionsAsync();
  if (fg !== 'granted') return false;

  const { status: bg } = await Location.requestBackgroundPermissionsAsync();
  if (bg !== 'granted') return false;

  await Location.startLocationUpdatesAsync(DRIVER_LOCATION_TASK, {
    accuracy: Location.Accuracy.High,
    timeInterval: 5000,   // 5 seconds
    distanceInterval: 15, // 15 meters
    showsBackgroundLocationIndicator: true,
    foregroundService: {
      notificationTitle: 'Location Active',
      notificationBody: 'Your location is being tracked for delivery.',
    },
  });

  return true;
}

export async function stopDriverTracking(): Promise<void> {
  const isRunning = await Location.hasStartedLocationUpdatesAsync(
    DRIVER_LOCATION_TASK,
  );
  if (isRunning) {
    await Location.stopLocationUpdatesAsync(DRIVER_LOCATION_TASK);
  }
}
```

### Scenario 2: E-commerce — Product Barcode Scanner

Scan a product barcode → look up in catalog → navigate to product detail.

```tsx
import { CameraView, useCameraPermissions, type BarcodeScanningResult } from 'expo-camera';
import * as Haptics from 'expo-haptics';
import { useRef, useState } from 'react';
import { ActivityIndicator, StyleSheet, Text, View } from 'react-native';
import { useRouter } from 'expo-router';

export function ProductScanner() {
  const [permission, requestPermission] = useCameraPermissions();
  const [scanning, setScanning] = useState(true);
  const [loading, setLoading] = useState(false);
  const lastScanRef = useRef(0);
  const router = useRouter();

  if (!permission) return <View style={styles.container} />;

  if (!permission.granted) {
    return (
      <View style={styles.center}>
        <Text>Camera permission is required to scan products.</Text>
      </View>
    );
  }

  const handleScan = async (result: BarcodeScanningResult) => {
    const now = Date.now();
    if (!scanning || now - lastScanRef.current < 300) return;
    lastScanRef.current = now;
    setScanning(false);
    setLoading(true);

    // Confirm the scan with haptic feedback
    await Haptics.notificationAsync(Haptics.NotificationFeedbackType.Success);

    try {
      const response = await fetch(`/api/products/barcode/${result.data}`);
      if (!response.ok) {
        setLoading(false);
        setScanning(true);
        return;
      }
      const product = await response.json();
      router.push(`/products/${product.id}`);
    } catch {
      setLoading(false);
      setScanning(true);
    }
  };

  return (
    <View style={styles.container}>
      <CameraView
        style={StyleSheet.absoluteFill}
        facing="back"
        onBarcodeScanned={scanning ? handleScan : undefined}
        barcodeScannerSettings={{
          barcodeTypes: ['ean13', 'ean8', 'qr', 'code128'],
        }}
      />
      {loading && (
        <View style={styles.overlay}>
          <ActivityIndicator size="large" color="white" />
          <Text style={styles.overlayText}>Looking up product…</Text>
        </View>
      )}
    </View>
  );
}

const styles = StyleSheet.create({
  container: { flex: 1 },
  center: { flex: 1, justifyContent: 'center', alignItems: 'center' },
  overlay: {
    ...StyleSheet.absoluteFillObject,
    backgroundColor: 'rgba(0,0,0,0.6)',
    justifyContent: 'center',
    alignItems: 'center',
    gap: 12,
  },
  overlayText: { color: 'white', fontSize: 16 },
});
```

---

## Further Reading

- [expo-camera docs](https://docs.expo.dev/versions/latest/sdk/camera/)
- [expo-location docs](https://docs.expo.dev/versions/latest/sdk/location/)
- [expo-notifications docs](https://docs.expo.dev/versions/latest/sdk/notifications/)
- [expo-sensors docs](https://docs.expo.dev/versions/latest/sdk/sensors/)
- [expo-haptics docs](https://docs.expo.dev/versions/latest/sdk/haptics/)
- [expo-media-library docs](https://docs.expo.dev/versions/latest/sdk/media-library/)
- [Expo Push Notifications guide](https://docs.expo.dev/push-notifications/overview/)
- [What PWA Can Do Today — device APIs overview](https://whatpwacando.today/)

---

## Summary

- Expo wraps native hardware APIs behind consistent `expo-*` packages; install with `npx expo install` to get the SDK-compatible version.
- The permission model is uniform: request → check status → use API or show fallback. Always handle `canAskAgain: false` by offering `Linking.openSettings()`.
- `expo-camera` provides live camera preview, photo capture, and barcode scanning via `CameraView`; use the `useCameraPermissions()` hook for declarative permission management.
- `expo-location` supports one-shot (`getCurrentPositionAsync`) and continuous (`watchPositionAsync`) tracking; always call `subscriber.remove()` in `useEffect` cleanup to stop GPS and prevent battery drain.
- Push notifications require `expo-notifications` + `expo-device`, an EAS project ID, and `setNotificationHandler` at module level; the Expo push service routes to APNs and FCM transparently.
- Every sensor (Accelerometer, Gyroscope, Magnetometer, Barometer, DeviceMotion) follows the same `addListener` / `remove` subscription pattern — missing the cleanup leaks the sensor.
- `expo-haptics` reinforces UI interactions; use `impactAsync(Light)` for normal presses, `Heavy` for destructive actions, and `notificationAsync(Success/Warning/Error)` for system-style outcomes.
