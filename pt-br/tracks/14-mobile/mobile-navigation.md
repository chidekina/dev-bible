# Mobile Navigation

## Visão Geral

Navegação em apps mobile é fundamentalmente diferente do roteamento web. Na web, o browser gerencia uma pilha de histórico plana. No mobile, a navegação é uma hierarquia: telas se empilham umas sobre as outras, tabs alternam entre seções irmãs, e drawers deslizam pela lateral. Usuários esperam transições nativas da plataforma — deslizar da direita no iOS, transições de elemento compartilhado no Android, bottom sheets.

React Navigation é a biblioteca padrão de facto para navegação em React Native. Ela oferece navegadores de stack, tab, drawer e bottom-sheet que se compõem em árvores de navegação complexas.

---

## Pré-requisitos

- [React Native Basics](react-native-basics.md)
- Entendimento de generics TypeScript (para navegação tipada)
- `@react-navigation/native` instalado

---

## Conceitos Principais

### Tipos de Navegadores

**Stack Navigator** — telas se empilham umas sobre as outras, botão voltar desfaz o empilhamento

```
[ Tela A ] → push → [ Tela A | Tela B ] → push → [ Tela A | Tela B | Tela C ]
                                                   ← voltar remove a Tela C
```

**Tab Navigator** — telas paralelas, barra de tabs na parte inferior (ou superior)

```
[ Início | Busca | Perfil ]
  ↑ ativa               ↑ toque alterna, sem empilhamento
```

**Drawer Navigator** — menu deslizante pela borda esquerda

**Bottom Sheet** (via `@gorhom/bottom-sheet`) — overlay de tela parcial

Navegadores se compõem: cada tab pode ter sua própria stack, que pode abrir modais.

### Navegação Type-Safe

Segurança completa de tipos requer declarar uma lista de parâmetros para cada navegador.

```tsx
// navigation/types.ts
import type {
  NativeStackScreenProps,
  NativeStackNavigationProp,
} from '@react-navigation/native-stack';
import type { BottomTabScreenProps } from '@react-navigation/bottom-tabs';
import type { CompositeScreenProps } from '@react-navigation/native';

// Stack raiz (modais + contêiner de tabs)
export type RootStackParamList = {
  Tabs: undefined;
  Modal: { message: string };
  ImageViewer: { imageUrl: string; title: string };
};

// Navegador de tabs
export type TabParamList = {
  HomeStack: undefined;
  SearchStack: undefined;
  ProfileStack: undefined;
};

// Stack de Home (aninhada dentro da HomeTab)
export type HomeStackParamList = {
  Feed: undefined;
  ArticleDetail: { articleId: string; title: string };
  AuthorProfile: { authorId: string };
};

// Props compostas para telas dentro de navegadores aninhados
export type ArticleDetailScreenProps = CompositeScreenProps<
  NativeStackScreenProps<HomeStackParamList, 'ArticleDetail'>,
  BottomTabScreenProps<TabParamList>
>;

// Apenas a prop de navigation (para componentes, não telas)
export type HomeNavigationProp = NativeStackNavigationProp<HomeStackParamList>;
```

### useNavigation e useRoute

```tsx
import { useNavigation, useRoute } from '@react-navigation/native';
import type { HomeNavigationProp } from '../navigation/types';

// Dentro de um componente (não a tela em si)
function BackButton() {
  const navigation = useNavigation<HomeNavigationProp>();
  return <Pressable onPress={() => navigation.goBack()} />;
}

// Params de rota tipados
function ArticleDetail() {
  const route = useRoute<ArticleDetailScreenProps['route']>();
  const { articleId, title } = route.params; // totalmente tipado
  // ...
}
```

---

## Exemplos Práticos

### Configuração Completa de Navegação

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
      <Tab.Screen name="HomeStack" component={HomeStackNavigator} options={{ title: 'Início' }} />
      <Tab.Screen name="SearchStack" component={SearchStackNavigator} options={{ title: 'Busca' }} />
      <Tab.Screen name="ProfileStack" component={ProfileStackNavigator} options={{ title: 'Perfil' }} />
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

### Padrão de Fluxo de Autenticação

O insight principal: navegadores renderizam condicionalmente com base no estado de auth. Não existe "redirecionamento" — você troca árvores inteiras de navegadores.

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

// Não autenticado: onboarding, login, cadastro
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

// Autenticado: app principal
function AuthenticatedNavigator() {
  return (
    <RootStack.Navigator>
      <RootStack.Screen name="Tabs" component={TabNavigator} />
      {/* ... */}
    </RootStack.Navigator>
  );
}

// Quando o usuário faz login, apenas atualize o contexto de auth
// React Navigation faz a transição automaticamente para AuthenticatedNavigator
function LoginScreen() {
  const { signIn } = useAuth();

  const handleLogin = async () => {
    const token = await api.login(email, password);
    await signIn(token); // dispara a troca de navegador
  };
}
```

### Customização de Header

```tsx
<Stack.Screen
  name="ArticleDetail"
  component={ArticleDetailScreen}
  options={({ route, navigation }) => ({
    // Título dinâmico a partir dos params
    title: route.params.title,

    // Botão personalizado à direita do header
    headerRight: () => (
      <Pressable onPress={() => navigation.navigate('Share', { articleId: route.params.articleId })}>
        <ShareIcon />
      </Pressable>
    ),

    // Label do botão voltar (somente iOS)
    headerBackTitle: 'Feed',

    // Header transparente com posicionamento absoluto
    headerTransparent: true,
    headerBlurEffect: 'regular', // efeito de blur do iOS

    // Header personalizado
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

  // Parsing de URL personalizado (opcional)
  getStateFromPath: (path, options) => {
    // Trata URLs legadas, query params, etc.
    if (path.startsWith('/legacy/')) {
      const id = path.replace('/legacy/', '');
      return { routes: [{ name: 'ArticleDetail', params: { articleId: id } }] };
    }
    // Cai para o parsing padrão
    return getStateFromPath(path, options);
  },
};
```

**Testando deep links:**

```bash
# Simulador iOS
xcrun simctl openurl booted "myapp://article/123"

# Emulador Android
adb shell am start -a android.intent.action.VIEW -d "myapp://article/123"

# Expo
npx uri-scheme open "myapp://article/123" --ios
```

### Universal Links (deep links HTTPS)

Universal links requerem arquivos de configuração no servidor e eliminam o scheme personalizado.

**iOS — Apple App Site Association (AASA)**
```json
// Servir em: https://myapp.example.com/.well-known/apple-app-site-association
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
// Servir em: https://myapp.example.com/.well-known/assetlinks.json
[
  {
    "relation": ["delegate_permission/common.handle_all_urls"],
    "target": {
      "namespace": "android_app",
      "package_name": "com.example.myapp",
      "sha256_cert_fingerprints": ["<SEU_CERT_FINGERPRINT>"]
    }
  }
]
```

---

## Padrões Comuns e Boas Práticas

### Event Listeners de Navegação

```tsx
// Rodar código quando a tela fica em foco (ex.: atualizar dados)
useFocusEffect(
  useCallback(() => {
    fetchLatestData();

    return () => {
      // Limpeza opcional quando a tela perde o foco
    };
  }, [])
);

// Escutar eventos de navegação
useEffect(() => {
  const unsubscribe = navigation.addListener('beforeRemove', (e) => {
    if (!hasUnsavedChanges) return;

    e.preventDefault(); // Impede a ação padrão (voltar)

    Alert.alert(
      'Descartar alterações?',
      'Você tem alterações não salvas. Descartá-las e sair da tela?',
      [
        { text: 'Não sair', style: 'cancel' },
        {
          text: 'Descartar',
          style: 'destructive',
          onPress: () => navigation.dispatch(e.data.action),
        },
      ]
    );
  });

  return unsubscribe;
}, [navigation, hasUnsavedChanges]);
```

### Passar Callbacks via Navegação

Não passe funções como params de navegação — elas não são serializáveis e quebram deep linking. Use context ou um gerenciador de state.

```tsx
// RUIM
navigation.navigate('Picker', { onSelect: handleSelect }); // não serializável

// BOM: use um event emitter ou state compartilhado
const pickerStore = usePickerStore();
navigation.navigate('Picker', { pickerId: 'destination' });

// Na tela Picker:
const handleConfirm = (value) => {
  pickerStore.confirm('destination', value);
  navigation.goBack();
};
```

---

## Anti-Padrões a Evitar

**Passar params não serializáveis.** Funções, instâncias de classe e objetos complexos como params de navegação quebram deep linking, persistência de state e a própria serialização do React Navigation. Use tipos primitivos: strings, números, booleanos e objetos simples.

**Chamar `navigation.navigate` durante o render.** A navegação deve ser disparada por ações do usuário ou effects, nunca durante a fase de render.

**Usar `navigation.reset` em excesso.** `reset` destrói todo o estado de navegação. Use apenas para logout ou conclusão de onboarding. Para redirecionar após uma ação, use `replace` ou renderização condicional.

**Aninhamento profundo sem propósito.** Mais de 3 níveis de navegadores aninhados ficam difíceis de raciocinar e tipar. Achate sua estrutura onde possível.

**Esquecer `headerShown: false` em navegadores aninhados.** Cada navegador aninhado adiciona seu próprio header por padrão, resultando em headers duplos. Defina explicitamente `headerShown: false` em navegadores aninhados e deixe o pai gerenciar o header.

---

## Depuração e Resolução de Problemas

**"Cannot navigate to screen X" em deep link**
A tela não está registrada na configuração de linking ou a ordem do navegador está errada. Use `getStateFromPath` para depurar: logue o que ele produz para sua URL.

**Header duplo em telas de tabs aninhadas**
Adicione `headerShown: false` ao `screenOptions` do Tab.Navigator e deixe cada stack interna gerenciar seu próprio header.

**`useFocusEffect` disparando na montagem inicial E no foco**
Este é o comportamento esperado. Use um ref para rastrear a montagem inicial se precisar pular.

**Botão voltar no Android não funcionando**
Certifique-se de que o navegador raiz recebe foco. O botão voltar físico dispara `navigation.goBack()` — se o seu navegador não tem telas para voltar, ele sai do app (comportamento correto).

---

## Cenários do Mundo Real

### Tab com Contador de Badge

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

### Fluxo de Onboarding com Persistência

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

## Navegação Tipada com TypeScript

Type every screen and its params to catch navigation errors at compile time.

```typescript
// types/navigation.ts — define every screen and its param contract
import type { NativeStackNavigationProp } from '@react-navigation/native-stack';
import type { BottomTabNavigationProp } from '@react-navigation/bottom-tabs';
import type { RouteProp } from '@react-navigation/native';

export type RootStackParamList = {
  Home: undefined;                           // sem params
  Profile: { userId: string };               // param obrigatório
  Post: { postId: string; title?: string };  // param opcional
  EditPost: { postId: string; draft?: boolean };
};

export type TabParamList = {
  Feed: undefined;
  Search: { query?: string };
  Notifications: undefined;
  Settings: undefined;
};

// Prop de navegação tipada por tela
export type HomeScreenNavigationProp = NativeStackNavigationProp<
  RootStackParamList,
  'Home'
>;

// Prop de rota tipada por tela
export type ProfileScreenRouteProp = RouteProp<RootStackParamList, 'Profile'>;
```

```typescript
// ProfileScreen.tsx — params tipados, sem adivinhação em runtime
import { useRoute } from '@react-navigation/native';
import type { ProfileScreenRouteProp } from '../types/navigation';

export function ProfileScreen() {
  const route = useRoute<ProfileScreenRouteProp>();
  // route.params.userId é string — garantido, sem necessidade de verificar undefined
  const { userId } = route.params;

  return <UserProfile userId={userId} />;
}
```

```typescript
// Declaração de tipo global — faz useNavigation() ser inferido em todo lugar
// Adicione isso uma vez, ex.: em types/navigation.ts

declare global {
  namespace ReactNavigation {
    interface RootParamList extends RootStackParamList {}
  }
}

// Agora useNavigation() é totalmente tipado sem o genérico em todo lugar:
import { useNavigation } from '@react-navigation/native';

function SomeButton() {
  const navigation = useNavigation(); // tipo inferido
  // navigation.navigate('Post', { postId: '123' });  ✓
  // navigation.navigate('Post', {});                 ✗ erro TS: postId obrigatório
  // navigation.navigate('Nonexistent');              ✗ erro TS: não está na lista de params
}
```

---

## Universal Links (iOS) e App Links (Android)

Universal Links e App Links abrem seu app quando o usuário toca em uma URL na web — sem scheme personalizado, sem prompt "Abrir no app?".

**Passo 1 — Hospede o arquivo de verificação:**

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

O arquivo deve ser servido com `Content-Type: application/json` e sem redirecionamentos no caminho de verificação.

**Passo 2 — Configure no app.json:**

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

**Passo 3 — Conecte à configuração de linking do React Navigation:**

```typescript
// navigation/linking.ts
import type { LinkingOptions } from '@react-navigation/native';
import type { RootStackParamList } from '../types/navigation';

export const linking: LinkingOptions<RootStackParamList> = {
  prefixes: [
    'https://yourdomain.com',
    'myapp://', // scheme personalizado como fallback para dev
  ],
  config: {
    screens: {
      Home: '',
      Post: 'post/:postId',
      Profile: 'profile/:userId',
    },
  },
};

// No NavigationContainer:
// <NavigationContainer linking={linking} fallback={<LoadingScreen />}>
```

**Teste no simulador iOS:**

```bash
xcrun simctl openurl booted "https://yourdomain.com/post/123"
```

**Teste no emulador Android:**

```bash
adb shell am start -W -a android.intent.action.VIEW -d "https://yourdomain.com/post/123"
```

---

## Bottom Sheet com Navegação

`@gorhom/bottom-sheet` é o padrão para drawers inferiores dispensáveis no React Native. Integra-se com Gesture Handler e Reanimated para animações suaves a 60fps.

```bash
npx expo install @gorhom/bottom-sheet react-native-gesture-handler react-native-reanimated
```

Envolva a raiz do app com `GestureHandlerRootView`:

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
      index={0}           // 0 = primeiro snap point ao montar
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

**Padrão: sheet controlada pelo state do pai:**

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

## Leitura Adicional

- [Documentação do React Navigation](https://reactnavigation.org/docs/getting-started)
- [React Navigation — Verificação de tipos](https://reactnavigation.org/docs/typescript)
- [React Navigation — Deep linking](https://reactnavigation.org/docs/deep-linking)
- [Universal links no iOS](https://developer.apple.com/documentation/xcode/supporting-associated-domains)
- [Android App Links](https://developer.android.com/training/app-links)

---

## Resumo

React Navigation compõe navegadores de stack, tab e drawer em uma árvore. Segurança de tipos requer declarar um tipo `ParamList` para cada navegador e usar `CompositeScreenProps` para telas aninhadas. O padrão de fluxo de auth troca árvores de navegadores com base no estado de autenticação — sem redirecionamentos. Deep links mapeiam caminhos de URL para params de tela via configuração de `linking` no `NavigationContainer`. Universal links requerem arquivos JSON no servidor (`apple-app-site-association` e `assetlinks.json`). Evite passar valores não serializáveis como params de navegação — eles quebram o deep linking.
