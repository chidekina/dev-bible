# React Native Advanced

## Visão Geral

Quando seu app React Native já é funcional, os próximos desafios são performance, animações complexas e integração nativa. Este capítulo cobre as ferramentas e padrões que distinguem apps de qualidade em produção de protótipos: medir e eliminar quedas de frames, fazer bridge com código nativo, criar animações fluidas com Reanimated e Gesture Handler, e implementar deep linking.

A arquitetura do React Native é central para entender performance. A thread JS roda seu código React. A thread de UI renderiza views nativas. A comunicação entre elas (a "bridge") era historicamente o gargalo. A nova arquitetura do React Native (JSI + Fabric) elimina a bridge para acesso nativo síncrono — Reanimated 2+ e a New Architecture aproveitam isso diretamente.

---

## Pré-requisitos

- [React Native Basics](react-native-basics.md) — componentes principais, StyleSheet, navegação
- Entendimento de hooks React (useMemo, useCallback, useRef)
- Familiaridade básica com conceitos de engine JavaScript (event loop, microtasks)

---

## Conceitos Principais

### O Modelo de Threads

```
Thread JS          Thread de UI        Thread de Módulos Nativos
─────────          ────────────        ──────────────────────────
Render React       Atualizações de     Câmera, Localização, etc.
Lógica de negócios  views              I/O de arquivo
Atualizações state  Gestos             Rede (nível mais baixo)
Tratamento eventos  Animações

Arquitetura antiga: JS ↔ Bridge (async, JSON serializado) ↔ Nativo
Nova Arquitetura:   JS ↔ JSI (bindings síncronos em C++) ↔ Nativo
```

Um frame é descartado quando a thread de UI fica bloqueada por mais de 16ms (60fps). Causas comuns:
- Computação JS pesada na thread JS durante a rolagem
- Muitas chamadas a `setState` em rápida sucessão
- Árvores de componentes grandes re-renderizando a cada mudança de offset de scroll
- Animações conduzidas por JavaScript (use Reanimated)

### Hermes

Hermes é um engine JavaScript otimizado para React Native. Ative-o para inicialização significativamente mais rápida e menor uso de memória.

```json
// app.json (Expo)
{
  "expo": {
    "jsEngine": "hermes"
  }
}
```

Hermes pré-compila JS para bytecode em tempo de build, eliminando o tempo de parse na inicialização. Também vem com um profiler de memória embutido acessível via Chrome DevTools.

### Profiling de Performance

```
React DevTools Profiler → identificar renders lentos
Flipper Performance → timeline de frames das threads JS/UI
Android Studio CPU Profiler → análise de thread nativa
Instruments (Xcode) → profiling de memória e CPU no iOS
```

**Encontrando renders lentos:**
```tsx
// Envolva com Profiler para medir o custo de render
import { Profiler } from 'react';

function onRenderCallback(id, phase, actualDuration) {
  if (actualDuration > 16) {
    console.warn(`Render lento: ${id} levou ${actualDuration.toFixed(2)}ms`);
  }
}

<Profiler id="ItemList" onRender={onRenderCallback}>
  <ItemList data={data} />
</Profiler>
```

---

## Exemplos Práticos

### Otimizando a Performance da FlatList

```tsx
import { FlatList, View } from 'react-native';

// RUIM: cria novas funções e objetos a cada render
function BadList({ items }) {
  return (
    <FlatList
      data={items}
      renderItem={({ item }) => <ItemCard item={item} />} // nova função a cada render
      keyExtractor={(item) => item.id}                    // nova função a cada render
      contentContainerStyle={{ padding: 16 }}            // novo objeto a cada render
    />
  );
}

// BOM: referências estáveis
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
      getItemLayout={getItemLayout} // pula medição se a altura for fixa
      removeClippedSubviews          // desmonta itens fora da tela
      maxToRenderPerBatch={8}        // itens renderizados por ciclo JS
      updateCellsBatchingPeriod={50} // ms entre renders em lote
      windowSize={11}                // renderiza 5 viewports acima e abaixo
      initialNumToRender={10}        // itens no primeiro render
    />
  );
}

// ItemCard deve ser memoizado
const ItemCard = React.memo(function ItemCard({ item }: { item: Item }) {
  return (
    <View>
      <Text>{item.title}</Text>
    </View>
  );
});
```

### Reanimated 2 — Animações Nativas

```bash
npx expo install react-native-reanimated
# Adicione ao babel.config.js: plugins: ['react-native-reanimated/plugin']
```

Reanimated executa a lógica de animação na thread de UI, contornando a thread JS completamente.

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

// Animação de spring simples
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
        <Text>Me pressione</Text>
      </Animated.View>
    </Pressable>
  );
}

// Animação de header baseada em scroll
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

// Indicador de carregamento pulsante
function PulsingDot() {
  const opacity = useSharedValue(1);

  useEffect(() => {
    opacity.value = withRepeat(
      withSequence(
        withTiming(0.2, { duration: 800, easing: Easing.inOut(Easing.ease) }),
        withTiming(1, { duration: 800, easing: Easing.inOut(Easing.ease) })
      ),
      -1, // infinito
      true // reverso
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

// Card arrastável com retorno ao ponto de origem
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
      // Retorna à origem
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

// Deslizar para dispensar
function SwipeToDismiss({ onDismiss, children }) {
  const translateX = useSharedValue(0);
  const DISMISS_THRESHOLD = 150;

  const gesture = Gesture.Pan()
    .activeOffsetX([-10, 10]) // ativa apenas em swipe horizontal
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

### Módulos Nativos (Expo Modules API)

Quando você precisa acessar APIs de plataforma nativas não cobertas pelo SDK do Expo:

```typescript
// modules/my-native-module/index.ts
import { requireNativeModule } from 'expo-modules-core';

const MyNativeModule = requireNativeModule('MyNativeModule');

export async function getBiometricToken(): Promise<string> {
  return MyNativeModule.getBiometricToken();
}
```

Para módulos nativos complexos, use `expo-module-scripts` para scaffoldar:

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

// Configuração de linking para React Navigation
const linking = {
  prefixes: ['myapp://', 'https://myapp.example.com'],
  config: {
    screens: {
      Home: '',
      Profile: 'user/:userId',
      Settings: 'settings',
      // Aninhado
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

## Padrões Comuns e Boas Práticas

### Estratégia de Memoização

```tsx
// Memorize apenas componentes que:
// 1. Renderizam com frequência (em listas, animações)
// 2. Recebem props estáveis (primitivos ou objetos memoizados)
// 3. São caros de renderizar

const ExpensiveItem = React.memo(
  function ExpensiveItem({ item, onPress }: Props) {
    return <View>...</View>;
  },
  // Igualdade personalizada — re-renderiza apenas se id ou title mudou
  (prev, next) => prev.item.id === next.item.id && prev.item.title === next.item.title
);

// Memorize callbacks passados para itens de lista
const handlePress = useCallback((id: string) => {
  navigation.navigate('Detail', { id });
}, [navigation]);

// Memorize derivações custosas
const sortedItems = useMemo(
  () => [...items].sort((a, b) => b.createdAt - a.createdAt),
  [items]
);
```

### InteractionManager — Adie Trabalho Pesado

```tsx
import { InteractionManager } from 'react-native';

function HeavyScreen() {
  const [ready, setReady] = useState(false);

  useEffect(() => {
    // Aguarda a animação de navegação concluir antes de renderizar conteúdo pesado
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

## Anti-Padrões a Evitar

**Animar com `setState`.** Atualizações de state causam re-renders no React; animações a 60fps não podem ser conduzidas por state JS. Use shared values do Reanimated na thread de UI.

**Usar a API `Animated` (core) para animações complexas.** A API `Animated` embutida do React Native roda na thread JS por padrão (a menos que você use `useNativeDriver: true`). Prefira sempre Reanimated 2+ para novas animações.

**Definir `useNativeDriver: true` para propriedades de layout.** O native driver suporta apenas transform e opacity. Usá-lo em `width`, `height` ou `margin` causa crash em produção.

**Aninhar VirtualizedList dentro de ScrollView.** FlatList dentro de ScrollView desativa a virtualização — todas as linhas renderizam de uma vez. Use `SectionList` ou uma abordagem de header personalizado.

**Usar `console.log` em produção.** Cada chamada de log é síncrona na thread JS. Use uma biblioteca de logging com remoção de logs em produção (`babel-plugin-transform-remove-console`).

---

## Depuração e Resolução de Problemas

### Diagnóstico: Thread JS vs Thread de UI

- **Animações travadas** — a thread de UI está descartando frames; frequentemente causado por recalculação de estilo pesada ou um gesto que dispara atualizações de state
- **Respostas de botão com atraso** — a thread JS está ocupada; faça profiling com o React DevTools Profiler
- **Gesto suave, atualização pós-gesto travada** — latência da bridge; considere manter o state da UI em um shared value até o gesto terminar e então commitar uma vez

### Depurador Hermes

```bash
# Conectar Chrome DevTools ao Hermes
# Expo: agite o dispositivo > Open JS Debugger
# Bare: adb forward tcp:8081 tcp:8081 depois chrome://inspect
```

### Erros de Worklet no Reanimated

Worklets (funções que rodam na thread de UI) não podem acessar variáveis do closure React sem `runOnJS` ou captura explícita. O erro `Trying to access property 'X' of an undefined worklet` significa que você referenciou uma função não-worklet ou uma variável de closure não capturada.

```tsx
// RUIM: someFunction não é um worklet
const gesture = Gesture.Tap().onEnd(() => {
  someFunction(); // Erro: não é um worklet
});

// BOM: rodar explicitamente na thread JS
const gesture = Gesture.Tap().onEnd(() => {
  runOnJS(someFunction)();
});
```

---

## Cenários do Mundo Real

### Feed com Scroll Infinito e Paginação

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

## Leitura Adicional

- [Documentação do Reanimated](https://docs.swmansion.com/react-native-reanimated/)
- [Documentação do Gesture Handler](https://docs.swmansion.com/react-native-gesture-handler/)
- [Nova Arquitetura do React Native](https://reactnative.dev/docs/new-architecture-intro)
- [Expo Modules API](https://docs.expo.dev/modules/overview/)
- [Flashlist — alternativa ao FlatList](https://shopify.github.io/flash-list/)

---

## Resumo

Performance no React Native significa manter a thread JS e a thread de UI a 60fps. Faça profiling antes de otimizar. Memorize `renderItem`, `keyExtractor` e componentes de itens de lista. Use Reanimated 2+ para todas as animações — ele roda na thread de UI e contorna a bridge. Gesture Handler oferece reconhecedores de gesto que também rodam nativamente. Evite conduzir animações com `setState`. Deep linking requer um URL scheme no `app.json` e uma configuração de `linking` no `NavigationContainer`. Módulos nativos podem ser escritos com a Expo Modules API para bridges TypeScript-first limpas.
