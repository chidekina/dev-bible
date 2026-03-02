# React Native Basics

## Visão Geral

React Native permite que você construa apps mobile para iOS e Android usando React e JavaScript, compartilhando lógica de negócios enquanto renderiza componentes de UI nativos (não uma WebView). O modelo mental é idêntico ao do React: componentes, props, state, hooks — mas em vez de `<div>` e `<span>` você usa `<View>` e `<Text>`, e o CSS é substituído por objetos StyleSheet.

Expo é o ponto de partida recomendado para 95% dos projetos. Ele oferece um serviço de build gerenciado, atualizações over-the-air e um conjunto curado de módulos nativos sem exigir que você mantenha configurações de build do Xcode ou Android Studio.

---

## Pré-requisitos

- Track 02 — Frontend (componentes React, hooks, TypeScript)
- Entendimento básico de convenções de UI mobile (tabs, stacks, gestos)
- Node.js 18+ instalado

---

## Conceitos Principais

### Expo vs Bare Workflow

**Expo Managed Workflow**
- Expo gerencia o processo de build nativo via EAS Build (nuvem)
- Você nunca toca nos diretórios do Xcode ou `android/`
- Acesso a APIs nativas através de pacotes `expo-*` (`expo-camera`, `expo-location`, etc.)
- Limitado às APIs que o Expo encapsulou — suficiente para 90%+ dos apps

```bash
npx create-expo-app@latest my-app --template blank-typescript
cd my-app
npx expo start
```

**Bare Workflow / React Native CLI**
- Acesso completo ao código nativo — modifique os diretórios `ios/` e `android/`
- Necessário para: SDKs nativos de terceiros não presentes no ecossistema Expo, módulos nativos personalizados, configurações de build específicas
- Você gerencia upgrades do Xcode e Android Studio

```bash
npx @react-native-community/cli@latest init MyApp --template react-native-template-typescript
```

**Expo com bare** (o melhor dos dois mundos):
```bash
npx create-expo-app@latest my-app
# Mais tarde, ejetar se necessário:
npx expo prebuild
```

### Componentes Principais

React Native vem com um pequeno conjunto de componentes primitivos que mapeiam para elementos de UI nativos.

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

// View — equivalente ao <div>, o contêiner fundamental de layout
<View style={styles.container}>
  {/* Text — TODO texto deve estar dentro de <Text> */}
  <Text style={styles.title}>Olá React Native</Text>

  {/* TextInput — input controlado */}
  <TextInput
    value={value}
    onChangeText={setValue}
    placeholder="Digite aqui"
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

  {/* Pressable — recomendado em vez de TouchableOpacity para código novo */}
  <Pressable
    onPress={handlePress}
    style={({ pressed }) => [styles.button, pressed && styles.buttonPressed]}
  >
    <Text>Me pressione</Text>
  </Pressable>
</View>
```

### StyleSheet

Estilos são objetos JavaScript, não arquivos CSS. `StyleSheet.create()` valida estilos em desenvolvimento e os otimiza em tempo de execução.

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
    // Sombra (iOS)
    shadowColor: '#000',
    shadowOffset: { width: 0, height: 2 },
    shadowOpacity: 0.1,
    shadowRadius: 4,
    // Sombra (Android)
    elevation: 3,
  },
  button: {
    backgroundColor: '#6366f1',
    borderRadius: 8,
    paddingVertical: 12,
    paddingHorizontal: 24,
    alignItems: 'center',
  },
  // Largura responsiva
  fullWidth: {
    width: SCREEN_WIDTH - 32,
  },
});
```

**Diferenças principais em relação ao CSS:**
- Todos os valores são números (pixels independentes de densidade), não strings com unidades
- Sem `px`, `em`, `rem` — `fontSize: 16` significa 16dp (density-independent pixels)
- Sem cascata — cada elemento se estiliza explicitamente
- A direção do `flex` padrão é `column` (vertical), não `row`
- Sem propriedades de atalho (`padding: '8px 16px'` é inválido; use `paddingVertical` / `paddingHorizontal`)

### Flexbox no React Native

React Native usa Yoga, a implementação de flexbox do Facebook. Os padrões diferem do CSS:

| Propriedade    | Padrão CSS     | Padrão React Native |
|----------------|----------------|----------------------|
| `flexDirection`| `row`          | `column`             |
| `alignContent` | `stretch`      | `flex-start`         |
| `flexShrink`   | `1`            | `0`                  |

```tsx
// Padrões de layout comuns
const styles = StyleSheet.create({
  // Contêiner de tela cheia
  screen: { flex: 1 },

  // Linha horizontal com espaço entre os itens
  row: { flexDirection: 'row', justifyContent: 'space-between', alignItems: 'center' },

  // Conteúdo centralizado
  centered: { flex: 1, justifyContent: 'center', alignItems: 'center' },

  // Botão fixo no rodapé
  footer: { position: 'absolute', bottom: 0, left: 0, right: 0, padding: 16 },

  // Ocupa o espaço restante
  grow: { flex: 1 },
});
```

### SafeAreaView e Teclado

```tsx
import { SafeAreaView, KeyboardAvoidingView, Platform } from 'react-native';
import { SafeAreaProvider } from 'react-native-safe-area-context'; // expo install

// Envolve o app raiz
function App() {
  return (
    <SafeAreaProvider>
      <Navigation />
    </SafeAreaProvider>
  );
}

// Tela individual
function LoginScreen() {
  return (
    <SafeAreaView style={{ flex: 1 }}>
      <KeyboardAvoidingView
        style={{ flex: 1 }}
        behavior={Platform.OS === 'ios' ? 'padding' : 'height'}
      >
        {/* Conteúdo que se ajusta ao teclado */}
      </KeyboardAvoidingView>
    </SafeAreaView>
  );
}
```

---

## Exemplos Práticos

### FlatList — Listas Longas Eficientes

Nunca use `ScrollView` para listas longas. `FlatList` renderiza apenas os itens visíveis.

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
      ListEmptyComponent={<Text>Nenhum item encontrado</Text>}
      refreshControl={
        <RefreshControl refreshing={refreshing} onRefresh={handleRefresh} />
      }
      // Props de performance
      removeClippedSubviews={true}
      maxToRenderPerBatch={10}
      windowSize={5}
    />
  );
}
```

### Código Específico por Plataforma

```tsx
import { Platform } from 'react-native';

// Verificação de plataforma inline
const paddingTop = Platform.OS === 'ios' ? 44 : 24;

// Platform.select — mais limpo para múltiplos valores
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

// Arquivos específicos por plataforma
// Button.ios.tsx  — carregado automaticamente no iOS
// Button.android.tsx — carregado automaticamente no Android
// Button.tsx — fallback para web ou lógica compartilhada
```

### Hook Personalizado com AsyncStorage

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

## Padrões Comuns e Boas Práticas

### Navegação Tipada com React Navigation

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
      <Text>Ir para o Perfil</Text>
    </Pressable>
  );
}
```

### Gerenciamento de Permissões

```tsx
import * as Location from 'expo-location';
import * as Notifications from 'expo-notifications';

async function requestLocation() {
  const { status } = await Location.requestForegroundPermissionsAsync();
  if (status !== 'granted') {
    Alert.alert(
      'Permissão Necessária',
      'O acesso à localização é necessário para exibir itens próximos.',
      [
        { text: 'Cancelar', style: 'cancel' },
        { text: 'Abrir Configurações', onPress: () => Linking.openSettings() },
      ]
    );
    return null;
  }
  return Location.getCurrentPositionAsync({});
}
```

---

## Anti-Padrões a Evitar

**Colocar texto fora do `<Text>`.** React Native lança um erro se uma string for filha direta de `<View>`. Todo texto deve ser envolvido em `<Text>`.

**Usar `ScrollView` para listas longas.** Ele renderiza todos os filhos de uma vez. Para listas com mais de ~20 itens, use `FlatList` ou `SectionList`.

**Objetos de estilo inline no `render`.** `{ flex: 1 }` cria um novo objeto a cada render, quebrando as otimizações de `PureComponent` e `React.memo`. Mova todos os estilos para `StyleSheet.create()`.

**Ignorar o teclado.** Em mobile, o teclado virtual cobre metade da tela. Envolva formulários em `KeyboardAvoidingView` e teste com o teclado aberto.

**Não testar em dispositivos reais cedo.** O Simulador do iOS e o Emulador do Android não replicam a performance de um dispositivo real, a latência de toque ou o comportamento de câmera/sensores. Teste em hardware antes de qualquer release.

**Bloquear a thread JS.** React Native tem uma única thread JS. Computações síncronas pesadas (parsing de JSON, ordenação de arrays grandes) causam queda de frames. Mova para `useEffect` ou um worker.

---

## Depuração e Resolução de Problemas

### Ferramentas

- **Expo Go** — escaneie o QR code para rodar seu app em um dispositivo real sem fazer build
- **React Native Debugger** — app standalone, conecta ao Metro bundler
- **Flipper** — plataforma de depuração mobile do Facebook (inspetor de rede, inspetor de layout, browser de banco de dados)
- **Reactotron** — alternativa ao Flipper para inspeção de state/rede
- **Metro bundler** — pressione `i` (iOS), `a` (Android), `r` (reload) no terminal

### Problemas Comuns

**Red box "Text strings must be rendered within a `<Text>` component"**
Você tem uma string diretamente dentro de uma `<View>`. Verifique padrões como `{condition && 'alguma string'}`.

**"Invariant Violation: requireNativeComponent" em uma nova instalação**
Um módulo nativo não está vinculado. Execute `npx expo install` ou `pod install` (bare workflow).

**App funciona no Expo Go mas crasha na build de produção**
Geralmente um módulo nativo não incluído no SDK padrão do Expo. Use `expo-dev-client` para dependências nativas personalizadas.

**`useEffect` rodando duas vezes em desenvolvimento**
Comportamento intencional do React 18 StrictMode. Acontece apenas em dev. Não lute contra isso — escreva effects que são idempotentes ou retorne uma função de limpeza.

---

## Cenários do Mundo Real

### Fluxo de Autenticação

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
    // Carrega token persistido na inicialização
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

// No navegador: exibe a stack de auth ou a stack do app com base no usuário
function RootNavigator() {
  const { user } = useAuth();
  return user ? <AppStack /> : <AuthStack />;
}
```

---

## Leitura Adicional

- [Documentação do React Native](https://reactnative.dev/docs/getting-started)
- [Documentação do Expo](https://docs.expo.dev/)
- [Documentação do React Navigation](https://reactnavigation.org/docs/getting-started)
- [React Native Directory](https://reactnative.directory/) — encontre bibliotecas nativas
- [Referência do Expo SDK](https://docs.expo.dev/versions/latest/)

---

## Resumo

React Native renderiza UI nativa usando componentes React. `View`, `Text`, `TextInput`, `FlatList` e `Pressable` são os primitivos principais. Estilos são objetos JavaScript usando unidades dp, flex padrão é coluna e sombras de caixa requerem propriedades específicas por plataforma. O Expo managed workflow é o padrão correto — ele cuida dos builds e módulos nativos sem exigir conhecimento de Xcode/Android Studio. Use `FlatList` para todas as listas longas, `KeyboardAvoidingView` para formulários e `Platform.OS` para comportamento condicional. Teste em dispositivos reais cedo.
