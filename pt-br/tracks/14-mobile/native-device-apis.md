# APIs Nativas do Dispositivo

## Visão Geral

O Expo oferece um conjunto curado de wrappers de APIs nativas que funcionam tanto no iOS quanto no Android sem precisar escrever nenhum código nativo. Câmera, localização, notificações push, sensores, haptics e biblioteca de mídia estão todos disponíveis por meio de pacotes `expo-*` instalados via `npx expo install` — que fixa automaticamente a versão compatível com o SDK. O modelo de permissões é consistente em todos eles: solicitar permissão → verificar o resultado → usar a API ou exibir UI de fallback.

Este capítulo cobre as APIs nativas mais utilizadas com padrões prontos para produção. Cada exemplo trata as permissões corretamente e inclui limpeza para operações contínuas (watchers de localização, assinaturas de sensores, listeners de notificações) para evitar consumo excessivo de bateria e vazamentos de memória. A leitura deste capítulo pressupõe que você tem um projeto Expo SDK 50+ com Expo Router ou React Navigation já configurado.

---

## Pré-requisitos

- [React Native Basics](react-native-basics.md) — componentes principais, StyleSheet, navegação
- [React Native Advanced](react-native-advanced.md) — hooks, cleanup do useEffect, TypeScript
- Familiaridade básica com `async/await` e ciclo de vida do React

---

## Conceitos Principais

### O Modelo de Permissões

Toda API com suporte de hardware exige permissão explícita do usuário. O Expo padroniza isso com um padrão consistente:

```typescript
// Toda API de permissão expo-* segue esse formato
const { status } = await SomeModule.requestPermissionsAsync();
// status: 'granted' | 'denied' | 'undetermined'

// Ou use o hook declarativo (preferido para fluxos orientados por UI)
const [permission, requestPermission] = useSomePermissions();
// permission.granted: boolean
// permission.canAskAgain: boolean — false após o usuário selecionar "Não perguntar novamente"
```

**Regras principais:**
- Solicite permissões no momento do primeiro uso, não na inicialização do app.
- Se `canAskAgain` for `false`, o diálogo de permissão nunca mais será exibido — o único caminho a seguir é enviar o usuário para a tela de Configurações do SO via `Linking.openSettings()`.
- O iOS exige strings de descrição de uso em `app.json`; sem elas o app crasha. O Android exige entradas `<uses-permission>` no manifest — o Expo cuida disso automaticamente com base nos pacotes SDK instalados.

```json
// app.json — obrigatório para iOS
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

## Exemplos Práticos

### 1. Câmera (expo-camera)

```bash
npx expo install expo-camera
```

O hook `useCameraPermissions()` gerencia tanto o estado da permissão quanto a chamada de solicitação. Envolva-o cedo na árvore de renderização para que o estado de carregamento (`permission === null`) renderize uma view vazia em vez de piscar a UI errada.

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

  // Estado de permissão ainda carregando
  if (!permission) {
    return <View style={styles.container} />;
  }

  // Permissão negada — explique o motivo e ofereça um caminho alternativo
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
      // Enviar para media library, fazer upload para o servidor, etc.
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

#### Scanner de Código de Barras

`CameraView` tem suporte nativo a leitura de código de barras. Use debounce em `onBarcodeScanned` para evitar que o callback dispare múltiplas vezes por leitura — a câmera atualiza a 30fps e emitirá o mesmo código em frames consecutivos.

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

### 2. Localização (expo-location)

```bash
npx expo install expo-location
```

Existem dois níveis de permissão: **foreground** (app visível) e **background** (executa quando o app está em segundo plano ou fechado). Sempre comece pelo foreground — o background requer uma aprovação adicional no nível do SO e precisa ser justificado.

```typescript
import * as Location from 'expo-location';

// One-shot: obtém a posição atual uma única vez
export async function getCurrentLocation(): Promise<Location.LocationObject | null> {
  const { status } = await Location.requestForegroundPermissionsAsync();
  if (status !== 'granted') return null;

  return Location.getCurrentPositionAsync({
    accuracy: Location.Accuracy.Balanced,
  });
}
```

#### Rastreamento Contínuo de Localização

`watchPositionAsync` retorna um objeto de assinatura. Chamar `subscriber.remove()` para o watcher. Esquecer essa limpeza faz o GPS rodar indefinidamente, drenando a bateria mesmo após o componente ser desmontado.

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
          timeInterval: 5000,    // mínimo de ms entre atualizações
          distanceInterval: 10,  // mínimo de metros entre atualizações
        },
        (loc) => setLocation(loc),
      );
    })();

    return () => {
      // SEMPRE faça o cleanup — ou o watcher roda para sempre e drena a bateria
      subscriber?.remove();
    };
  }, []);

  return { location, error };
}
```

**Níveis de precisão** (escolha o mais baixo que atenda ao seu requisito — cada nível acima aumenta o consumo de energia):

| Nível | Descrição | Caso de uso típico |
|---|---|---|
| `Accuracy.Lowest` | ~3 km | Exibição em nível de cidade |
| `Accuracy.Low` | ~1 km | Mapas regionais |
| `Accuracy.Balanced` | ~100 m | Mapas gerais, delivery |
| `Accuracy.High` | ~10 m | Navegação turn-by-turn |
| `Accuracy.BestForNavigation` | ~1 m | Navegação para pedestres, rastreamento esportivo |

#### Localização em Background

Localização em background requer uma solicitação de permissão separada e uma task ativa no `TaskManager`. Solicite isso apenas se o seu recurso realmente precisar — a Apple rejeita apps que solicitam localização em background sem justificativa clara.

```typescript
import * as Location from 'expo-location';
import * as TaskManager from 'expo-task-manager';

const LOCATION_TASK_NAME = 'background-location-task';

// Defina a task no nível do módulo (topo do arquivo, fora de qualquer componente)
TaskManager.defineTask(LOCATION_TASK_NAME, ({ data, error }) => {
  if (error) {
    console.error('Background location error:', error.message);
    return;
  }
  const { locations } = data as { locations: Location.LocationObject[] };
  // Enviar para o servidor, atualizar DB local, etc.
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
    timeInterval: 10000,   // 10 segundos
    distanceInterval: 20,  // 20 metros
    showsBackgroundLocationIndicator: true, // iOS: barra azul na status bar
  });

  return true;
}

export async function stopBackgroundTracking(): Promise<void> {
  await Location.stopLocationUpdatesAsync(LOCATION_TASK_NAME);
}
```

---

### 3. Notificações Push (expo-notifications)

```bash
npx expo install expo-notifications expo-device
```

Notificações push requerem um EAS project ID (do `app.json` via `eas build`). O serviço de push do Expo roteia para Apple APNs (iOS) e Firebase FCM (Android) de forma transparente.

#### Configuração Inicial

Chame `setNotificationHandler` no nível do módulo — antes de qualquer componente renderizar — para controlar como as notificações são exibidas quando o app está em primeiro plano.

```typescript
import * as Notifications from 'expo-notifications';

// Nível de módulo: configure o comportamento de exibição em foreground uma única vez
Notifications.setNotificationHandler({
  handleNotification: async () => ({
    shouldShowAlert: true,
    shouldPlaySound: false,
    shouldSetBadge: true,
  }),
});
```

#### Registrando para Tokens Push

```typescript
import * as Notifications from 'expo-notifications';
import * as Device from 'expo-device';
import Constants from 'expo-constants';

export async function registerForPushNotifications(): Promise<string | null> {
  // Tokens push não funcionam em simuladores — sempre verifique primeiro
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
  return tokenData.data; // "ExponentPushToken[xxxx]" — armazene no seu backend
}
```

#### Ouvindo Notificações em um Componente

```tsx
import { useEffect, useRef } from 'react';
import * as Notifications from 'expo-notifications';
import { useRouter } from 'expo-router';

export function usePushNotifications() {
  const router = useRouter();
  const notificationListener = useRef<Notifications.Subscription | null>(null);
  const responseListener = useRef<Notifications.Subscription | null>(null);

  useEffect(() => {
    // Dispara quando uma notificação chega enquanto o app está em foreground
    notificationListener.current =
      Notifications.addNotificationReceivedListener((notification) => {
        console.log('Notification received:', notification.request.content);
      });

    // Dispara quando o usuário toca em uma notificação (foreground ou background)
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

#### Enviando do Seu Servidor

A API de Push do Expo aceita um POST com um array de mensagens (até 100 por requisição):

```typescript
// lado do servidor: src/services/push.service.ts
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
  // result.data é um array de receipts — verifique se há erros
  for (const receipt of result.data ?? []) {
    if (receipt.status === 'error') {
      console.error('Push send error:', receipt.message);
    }
  }
}

// Uso
await sendPushNotifications([
  {
    to: 'ExponentPushToken[xxxxxxxxxxxxxxxxxxxxxx]',
    title: 'Order shipped',
    body: 'Your order #1234 is on its way.',
    data: { screen: 'orders/1234' }, // passado para o response listener
    badge: 1,
  },
]);
```

---

### 4. Sensores (expo-sensors)

```bash
npx expo install expo-sensors
```

Todas as APIs de sensores seguem o mesmo padrão de assinatura: `Sensor.addListener(callback)` retorna uma assinatura com um método `.remove()`. Chame `.remove()` no cleanup do `useEffect`.

#### Acelerômetro

Mede a aceleração (em g-force) nos eixos x, y, z. Útil para detecção de shake, heurísticas de orientação do dispositivo ou apps de fitness.

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
      subscription?.remove(); // cleanup obrigatório
    };
  }, [updateIntervalMs]);

  return { data, available };
}
```

#### Detecção de Shake com Acelerômetro

```typescript
import { useEffect, useRef } from 'react';
import { Accelerometer } from 'expo-sensors';

const SHAKE_THRESHOLD = 1.8; // g-force — ajuste conforme necessário

export function useShakeDetector(onShake: () => void) {
  const lastShakeRef = useRef(0);

  useEffect(() => {
    Accelerometer.setUpdateInterval(100);

    const subscription = Accelerometer.addListener(({ x, y, z }) => {
      const totalForce = Math.sqrt(x * x + y * y + z * z);

      if (
        totalForce > SHAKE_THRESHOLD &&
        Date.now() - lastShakeRef.current > 1000 // debounce de 1s
      ) {
        lastShakeRef.current = Date.now();
        onShake();
      }
    });

    return () => subscription.remove();
  }, [onShake]);
}
```

#### Comparação de Sensores

| Sensor | Mede | Casos de uso comuns |
|---|---|---|
| `Accelerometer` | Aceleração linear (g) | Detecção de shake, contagem de passos, inclinação |
| `Gyroscope` | Taxa de rotação (rad/s) | Auto-rotação de tela, orientação em AR |
| `Magnetometer` | Campo magnético (μT) | Bússola, detecção de metais |
| `Barometer` | Pressão atmosférica (hPa) | Altitude, tendências climáticas |
| `DeviceMotion` | Orientação + gravidade (combinado) | Apps de AR, pitch/roll/yaw do dispositivo |
| `Pedometer` | Contagem de passos | Fitness, estatísticas de caminhada |

`DeviceMotion` é a API de mais alto nível — ela funde acelerômetro, giroscópio e magnetômetro em `rotation`, `rotationRate`, `acceleration` e `orientation`. Use quando precisar de orientação absoluta do dispositivo em vez de dados brutos do sensor.

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

  // orientation.rotation: { alpha, beta, gamma } em radianos
  // orientation.rotationRate: { alpha, beta, gamma } em rad/s
  return orientation;
}
```

---

### 5. Haptics (expo-haptics)

```bash
npx expo install expo-haptics
```

Haptics reforçam interações de UI ao combinar o peso físico de uma ação. Use o feedback mais leve que ainda comunica a ação claramente — haptics excessivamente pesados parecem agressivos.

```typescript
import * as Haptics from 'expo-haptics';

// Toque leve — pressão de botão padrão, seleção de item de lista
await Haptics.impactAsync(Haptics.ImpactFeedbackStyle.Light);

// Toque médio — seleção de card, toggle switch
await Haptics.impactAsync(Haptics.ImpactFeedbackStyle.Medium);

// Toque forte — confirmação de ação destrutiva, ativação por long-press
await Haptics.impactAsync(Haptics.ImpactFeedbackStyle.Heavy);

// Seleção alterada — tick de picker, mudança de controle segmentado
await Haptics.selectionAsync();

// Feedback estilo notificação — eventos do sistema
await Haptics.notificationAsync(Haptics.NotificationFeedbackType.Success);
await Haptics.notificationAsync(Haptics.NotificationFeedbackType.Warning);
await Haptics.notificationAsync(Haptics.NotificationFeedbackType.Error);
```

**Uso prático — adicione haptics a componentes interativos:**

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

**Observação sobre plataformas:** O iOS suporta todas as variantes de `ImpactFeedbackStyle` e `NotificationFeedbackType` via Taptic Engine. O Android suporta `impactAsync` e `selectionAsync` por meio de vibração básica — a distinção entre Light, Medium e Heavy pode não ser perceptível em todos os dispositivos Android.

---

### 6. Biblioteca de Mídia e Seletor de Imagens

```bash
npx expo install expo-media-library expo-image-picker
```

`expo-media-library` gerencia os assets salvos (fotos, vídeos) no dispositivo. `expo-image-picker` fornece a UI do seletor de arquivos para escolher da biblioteca ou acionar a câmera.

#### Salvando uma Foto no Rolo da Câmera

Após capturar uma foto com `CameraView`, salve-a na biblioteca de fotos do dispositivo:

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

#### Selecionando uma Foto da Biblioteca

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
    quality: 0.8, // 0–1; reduza para menor tamanho de upload
  });

  if (result.canceled) return null;
  return result.assets[0].uri;
}
```

#### Selecionando com a Câmera

```typescript
export async function takePhotoWithPicker(): Promise<string | null> {
  const { status } = await ImagePicker.requestCameraPermissionsAsync();
  if (status !== 'granted') return null;

  const result = await ImagePicker.launchCameraAsync({
    allowsEditing: true,
    aspect: [1, 1], // recorte quadrado para avatares
    quality: 0.9,
  });

  if (result.canceled) return null;
  return result.assets[0].uri;
}
```

#### Componente de Seletor de Avatar

Combinando picker + media library em um componente reutilizável:

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

## Padrões Comuns

### Centralizando Permissões em um Hook

Quando múltiplos recursos precisam de localização ou câmera, solicite e armazene o status em um hook compartilhado para evitar diálogos duplicados:

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

### Fallback "Abrir Configurações"

Quando `canAskAgain` é false, o diálogo de permissão está permanentemente suprimido. Exiba uma mensagem clara e um botão que abre o painel de configurações do SO diretamente:

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

## Anti-Padrões

**Chamar a API antes de solicitar permissão.** Todas as APIs de hardware lançam erro ou retornam null sem uma concessão. Sempre verifique `permission.granted` antes de chamar.

**Não limpar as assinaturas.** Watchers de localização, assinaturas de sensores e listeners de notificações que continuam rodando após um componente ser desmontado drenam a bateria e causam vazamentos de memória. Toda chamada `addListener` ou `watchPositionAsync` deve ter um `.remove()` correspondente no cleanup do `useEffect`.

```tsx
// RUIM: assinatura vaza quando o componente é desmontado
useEffect(() => {
  Accelerometer.addListener(setData);
}, []);

// BOM: sempre retorne o cleanup
useEffect(() => {
  const sub = Accelerometer.addListener(setData);
  return () => sub.remove();
}, []);
```

**Solicitar todas as permissões na inicialização.** Pedir câmera + localização + notificações assim que o app abre leva os usuários a negar tudo por reflexo. Solicite cada permissão no momento em que o recurso é usado pela primeira vez.

**Não tratar `canAskAgain: false`.** Uma permissão negada com `canAskAgain: false` significa que o diálogo do SO sumiu permanentemente. Se você só exibe `requestPermission()` nesse estado, tocar nele não faz nada — o usuário fica preso. Sempre verifique `canAskAgain` e mostre `Linking.openSettings()` como fallback.

**Usar `expo-image-picker` para a câmera em uma tela de Câmera.** `launchCameraAsync` abre um modal de câmera do sistema — o usuário sai brevemente do seu app. Quando você precisa de uma visualização de câmera ao vivo com UI personalizada (overlays, processamento em tempo real, leitura de código de barras), use `expo-camera` diretamente.

**Não usar debounce em leituras de código de barras.** `onBarcodeScanned` dispara em cada frame da câmera onde um código de barras é visível — tipicamente 10 a 30 vezes por segundo. Sem debounce, você fará 30 chamadas de API por leitura. Use uma verificação de timestamp com `useRef` e um threshold de 300ms.

---

## Cenários do Mundo Real

### Cenário 1: Delivery de Comida — Rastreamento de Localização do Entregador

Um app de entregador que envia localização em tempo real para a central. Requer localização em background pois os entregadores bloqueiam o celular.

```typescript
import * as Location from 'expo-location';
import * as TaskManager from 'expo-task-manager';

const DRIVER_LOCATION_TASK = 'driver-location-task';

// Defina a task no nível do módulo
TaskManager.defineTask(DRIVER_LOCATION_TASK, async ({ data, error }) => {
  if (error || !data) return;

  const { locations } = data as { locations: Location.LocationObject[] };
  const latest = locations[locations.length - 1];

  // Envia para a API da central (fire-and-forget em background)
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
    // falha silenciosa — a próxima atualização compensará
  });
});

export async function startDriverTracking(): Promise<boolean> {
  const { status: fg } = await Location.requestForegroundPermissionsAsync();
  if (fg !== 'granted') return false;

  const { status: bg } = await Location.requestBackgroundPermissionsAsync();
  if (bg !== 'granted') return false;

  await Location.startLocationUpdatesAsync(DRIVER_LOCATION_TASK, {
    accuracy: Location.Accuracy.High,
    timeInterval: 5000,   // 5 segundos
    distanceInterval: 15, // 15 metros
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

### Cenário 2: E-commerce — Scanner de Código de Barras de Produto

Escanear o código de barras de um produto → buscar no catálogo → navegar para o detalhe do produto.

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

    // Confirma a leitura com feedback háptico
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

## Leitura Adicional

- [Documentação do expo-camera](https://docs.expo.dev/versions/latest/sdk/camera/)
- [Documentação do expo-location](https://docs.expo.dev/versions/latest/sdk/location/)
- [Documentação do expo-notifications](https://docs.expo.dev/versions/latest/sdk/notifications/)
- [Documentação do expo-sensors](https://docs.expo.dev/versions/latest/sdk/sensors/)
- [Documentação do expo-haptics](https://docs.expo.dev/versions/latest/sdk/haptics/)
- [Documentação do expo-media-library](https://docs.expo.dev/versions/latest/sdk/media-library/)
- [Guia de Push Notifications do Expo](https://docs.expo.dev/push-notifications/overview/)
- [What PWA Can Do Today — visão geral de APIs de dispositivo](https://whatpwacando.today/)

---

## Resumo

- O Expo encapsula APIs de hardware nativo em pacotes `expo-*` consistentes; instale com `npx expo install` para obter a versão compatível com o SDK.
- O modelo de permissões é uniforme: solicitar → verificar status → usar a API ou exibir fallback. Sempre trate `canAskAgain: false` oferecendo `Linking.openSettings()`.
- `expo-camera` fornece preview de câmera ao vivo, captura de fotos e leitura de código de barras via `CameraView`; use o hook `useCameraPermissions()` para gerenciamento declarativo de permissões.
- `expo-location` suporta rastreamento one-shot (`getCurrentPositionAsync`) e contínuo (`watchPositionAsync`); sempre chame `subscriber.remove()` no cleanup do `useEffect` para parar o GPS e evitar consumo de bateria.
- Notificações push requerem `expo-notifications` + `expo-device`, um EAS project ID e `setNotificationHandler` no nível do módulo; o serviço de push do Expo roteia para APNs e FCM de forma transparente.
- Cada sensor (Accelerometer, Gyroscope, Magnetometer, Barometer, DeviceMotion) segue o mesmo padrão de assinatura `addListener` / `remove` — esquecer o cleanup vaza o sensor.
- `expo-haptics` reforça interações de UI; use `impactAsync(Light)` para pressões normais, `Heavy` para ações destrutivas e `notificationAsync(Success/Warning/Error)` para resultados estilo sistema.
