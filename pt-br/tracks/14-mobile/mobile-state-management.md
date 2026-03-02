# Mobile State Management

## Visão Geral

Gerenciamento de estado no mobile tem restrições únicas que apps web raramente enfrentam: o armazenamento é síncrono em algumas plataformas e assíncrono em outras, apps vão para segundo plano e retornam, e usuários esperam funcionalidade offline. Uma camada de state mobile bem arquitetada separa o estado do servidor (React Query) do estado do cliente (Zustand), persiste o que precisa ser persistido (MMKV) e lida com transições de rede graciosamente.

Este capítulo cobre o trio que torna o state mobile robusto: Zustand para estado do cliente, MMKV para armazenamento síncrono rápido e React Query para estado do servidor com suporte offline.

---

## Pré-requisitos

- [React Native Basics](react-native-basics.md) — componentes e hooks principais
- [Mobile Navigation](mobile-navigation.md) — entendendo o ciclo de vida do app
- Familiaridade com React Query do Track 02 ou 03

---

## Conceitos Principais

### Comparação de Opções de Armazenamento

| Armazenamento    | Síncrono | Criptografado | Limite de tamanho | Caso de uso                   |
|------------------|----------|---------------|-------------------|-------------------------------|
| MMKV             | Sim      | Opcional      | Grande            | State do app, prefs, cache    |
| AsyncStorage     | Não      | Não           | 6MB               | Legado; evite em código novo  |
| SecureStore      | Não      | Sim           | 2KB               | Tokens, segredos              |
| SQLite (Expo)    | Não      | Opcional      | Grande            | Dados relacionais, queries    |
| RNFS (filesystem)| Não      | Não           | Disco             | Arquivos, imagens, documentos |

**MMKV** é o padrão correto para a maioria dos states persistentes: é síncrono (baseado em mmap), 10x mais rápido que AsyncStorage, suporta criptografia e tem uma API simples de chave-valor.

**SecureStore** (Expo) usa o iOS Keychain e Android Keystore para credenciais. Máximo de 2KB por valor.

```tsx
// Padrão de armazenamento de token
import * as SecureStore from 'expo-secure-store';

export const tokenStorage = {
  get: (key: string) => SecureStore.getItemAsync(key),
  set: (key: string, value: string) => SecureStore.setItemAsync(key, value),
  delete: (key: string) => SecureStore.deleteItemAsync(key),
};
```

### Estado do Cliente vs Estado do Servidor

```
Estado do Cliente               Estado do Servidor
─────────────────────           ─────────────────────────────────
State de UI (modal aberto)      Dados da sua API
Preferências do usuário         Listas, feeds paginados
Rascunho de formulário          Perfil do usuário
Carrinho / seleção              Notificações
Histórico de navegação          Resultados de busca

Gerenciado por: Zustand         Gerenciado por: React Query
Persistido em: MMKV             Em cache em: cache do React Query
                                Persistido em: MMKV (opcional)
```

---

## Exemplos Práticos

### Zustand Store com Persistência MMKV

```bash
npx expo install react-native-mmkv zustand
```

```tsx
// lib/storage.ts
import { MMKV } from 'react-native-mmkv';

export const storage = new MMKV({
  id: 'app-storage',
  // encryptionKey: 'sua-chave-de-criptografia', // para dados sensíveis
});

// Adaptador de storage para Zustand
export const zustandMMKVStorage = {
  getItem: (name: string) => storage.getString(name) ?? null,
  setItem: (name: string, value: string) => storage.set(name, value),
  removeItem: (name: string) => storage.delete(name),
};
```

```tsx
// stores/userPreferences.ts
import { create } from 'zustand';
import { persist, createJSONStorage } from 'zustand/middleware';
import { zustandMMKVStorage } from '../lib/storage';

type Theme = 'light' | 'dark' | 'system';

interface UserPreferencesState {
  theme: Theme;
  fontSize: 'small' | 'medium' | 'large';
  notificationsEnabled: boolean;
  setTheme: (theme: Theme) => void;
  setFontSize: (size: UserPreferencesState['fontSize']) => void;
  toggleNotifications: () => void;
}

export const useUserPreferences = create<UserPreferencesState>()(
  persist(
    (set) => ({
      theme: 'system',
      fontSize: 'medium',
      notificationsEnabled: true,
      setTheme: (theme) => set({ theme }),
      setFontSize: (fontSize) => set({ fontSize }),
      toggleNotifications: () =>
        set((state) => ({ notificationsEnabled: !state.notificationsEnabled })),
    }),
    {
      name: 'user-preferences',
      storage: createJSONStorage(() => zustandMMKVStorage),
    }
  )
);
```

```tsx
// stores/cart.ts — estado efêmero (sem persistência)
interface CartItem {
  productId: string;
  quantity: number;
  price: number;
}

interface CartState {
  items: CartItem[];
  addItem: (item: CartItem) => void;
  removeItem: (productId: string) => void;
  updateQuantity: (productId: string, quantity: number) => void;
  clear: () => void;
  total: () => number;
}

export const useCart = create<CartState>()((set, get) => ({
  items: [],
  addItem: (item) =>
    set((state) => {
      const existing = state.items.find((i) => i.productId === item.productId);
      if (existing) {
        return {
          items: state.items.map((i) =>
            i.productId === item.productId
              ? { ...i, quantity: i.quantity + item.quantity }
              : i
          ),
        };
      }
      return { items: [...state.items, item] };
    }),
  removeItem: (productId) =>
    set((state) => ({ items: state.items.filter((i) => i.productId !== productId) })),
  updateQuantity: (productId, quantity) =>
    set((state) => ({
      items:
        quantity <= 0
          ? state.items.filter((i) => i.productId !== productId)
          : state.items.map((i) =>
              i.productId === productId ? { ...i, quantity } : i
            ),
    })),
  clear: () => set({ items: [] }),
  total: () => get().items.reduce((sum, item) => sum + item.price * item.quantity, 0),
}));
```

### React Query no Mobile

```bash
npx expo install @tanstack/react-query
```

```tsx
// lib/queryClient.ts
import { QueryClient } from '@tanstack/react-query';
import { createAsyncStoragePersister } from '@tanstack/query-async-storage-persister';
import { persistQueryClient } from '@tanstack/react-query-persist-client';
import { zustandMMKVStorage } from './storage';

export const queryClient = new QueryClient({
  defaultOptions: {
    queries: {
      staleTime: 1000 * 60 * 5,    // 5 minutos
      gcTime: 1000 * 60 * 60 * 24, // 24 horas (persiste cache)
      retry: (failureCount, error) => {
        // Não tenta novamente em erros 4xx
        if (error instanceof ApiError && error.status < 500) return false;
        return failureCount < 3;
      },
      networkMode: 'offlineFirst', // serve do cache quando offline
    },
    mutations: {
      networkMode: 'offlineFirst',
    },
  },
});

// Persiste o cache de queries no MMKV
const persister = createAsyncStoragePersister({
  storage: {
    getItem: async (key) => zustandMMKVStorage.getItem(key),
    setItem: async (key, value) => zustandMMKVStorage.setItem(key, value),
    removeItem: async (key) => zustandMMKVStorage.removeItem(key),
  },
  throttleTime: 1000,
});

persistQueryClient({
  queryClient,
  persister,
  maxAge: 1000 * 60 * 60 * 24, // 24 horas
});
```

```tsx
// App.tsx
import { QueryClientProvider } from '@tanstack/react-query';
import { queryClient } from './lib/queryClient';

export default function App() {
  return (
    <QueryClientProvider client={queryClient}>
      <NavigationContainer>
        <RootNavigator />
      </NavigationContainer>
    </QueryClientProvider>
  );
}
```

```tsx
// hooks/useArticles.ts
import { useQuery, useMutation, useQueryClient } from '@tanstack/react-query';

export function useArticles() {
  return useQuery({
    queryKey: ['articles'],
    queryFn: () => api.getArticles(),
    placeholderData: (previousData) => previousData, // mantém dados antigos enquanto busca
  });
}

export function useArticle(id: string) {
  return useQuery({
    queryKey: ['articles', id],
    queryFn: () => api.getArticle(id),
    enabled: !!id,
  });
}

export function useToggleFavorite() {
  const queryClient = useQueryClient();

  return useMutation({
    mutationFn: ({ articleId, favorited }: { articleId: string; favorited: boolean }) =>
      api.toggleFavorite(articleId, favorited),

    // Atualização otimista
    onMutate: async ({ articleId, favorited }) => {
      await queryClient.cancelQueries({ queryKey: ['articles', articleId] });

      const previousArticle = queryClient.getQueryData<Article>(['articles', articleId]);

      queryClient.setQueryData<Article>(['articles', articleId], (old) =>
        old ? { ...old, favorited } : old
      );

      return { previousArticle };
    },

    onError: (_err, { articleId }, context) => {
      // Reverte em caso de erro
      if (context?.previousArticle) {
        queryClient.setQueryData(['articles', articleId], context.previousArticle);
      }
    },

    onSettled: (_data, _err, { articleId }) => {
      // Sempre rebusca para sincronizar com o servidor
      queryClient.invalidateQueries({ queryKey: ['articles', articleId] });
    },
  });
}
```

### Padrão de Sincronização Offline

Para apps que precisam de escrita offline completa (não apenas leitura):

```tsx
// stores/syncQueue.ts — persiste mutations pendentes
interface PendingMutation {
  id: string;
  type: 'CREATE_COMMENT' | 'UPDATE_PROFILE' | 'TOGGLE_FAVORITE';
  payload: unknown;
  timestamp: number;
  retries: number;
}

interface SyncQueueState {
  queue: PendingMutation[];
  enqueue: (mutation: Omit<PendingMutation, 'id' | 'timestamp' | 'retries'>) => void;
  remove: (id: string) => void;
  incrementRetries: (id: string) => void;
}

export const useSyncQueue = create<SyncQueueState>()(
  persist(
    (set) => ({
      queue: [],
      enqueue: (mutation) =>
        set((state) => ({
          queue: [
            ...state.queue,
            { ...mutation, id: crypto.randomUUID(), timestamp: Date.now(), retries: 0 },
          ],
        })),
      remove: (id) =>
        set((state) => ({ queue: state.queue.filter((m) => m.id !== id) })),
      incrementRetries: (id) =>
        set((state) => ({
          queue: state.queue.map((m) =>
            m.id === id ? { ...m, retries: m.retries + 1 } : m
          ),
        })),
    }),
    {
      name: 'sync-queue',
      storage: createJSONStorage(() => zustandMMKVStorage),
    }
  )
);

// hooks/useSyncProcessor.ts — processa fila quando online
import NetInfo from '@react-native-community/netinfo';

export function useSyncProcessor() {
  const { queue, remove, incrementRetries } = useSyncQueue();

  useEffect(() => {
    const unsubscribe = NetInfo.addEventListener(async (state) => {
      if (!state.isConnected || queue.length === 0) return;

      for (const mutation of queue) {
        if (mutation.retries >= 3) {
          // Dead letter — remove e notifica o usuário
          remove(mutation.id);
          notifyUser('Algumas alterações offline não puderam ser sincronizadas');
          continue;
        }

        try {
          await processMutation(mutation);
          remove(mutation.id);
          queryClient.invalidateQueries(); // atualiza todos os dados
        } catch {
          incrementRetries(mutation.id);
        }
      }
    });

    return unsubscribe;
  }, [queue]);
}

async function processMutation(mutation: PendingMutation) {
  switch (mutation.type) {
    case 'CREATE_COMMENT':
      return api.createComment(mutation.payload as CreateCommentPayload);
    case 'UPDATE_PROFILE':
      return api.updateProfile(mutation.payload as UpdateProfilePayload);
    case 'TOGGLE_FAVORITE':
      return api.toggleFavorite((mutation.payload as ToggleFavoritePayload).articleId, (mutation.payload as ToggleFavoritePayload).favorited);
    default:
      throw new Error(`Tipo de mutation desconhecido: ${mutation.type}`);
  }
}
```

---

## Padrões Comuns e Boas Práticas

### AppState e Tratamento em Segundo Plano

```tsx
import { AppState, AppStateStatus } from 'react-native';

// Rebusca queries desatualizadas quando o app volta ao primeiro plano
function useAppStateFocusEffect() {
  const queryClient = useQueryClient();
  const appState = useRef<AppStateStatus>(AppState.currentState);

  useEffect(() => {
    const subscription = AppState.addEventListener('change', (nextAppState) => {
      if (
        appState.current.match(/inactive|background/) &&
        nextAppState === 'active'
      ) {
        queryClient.invalidateQueries({ stale: true });
      }
      appState.current = nextAppState;
    });

    return () => subscription.remove();
  }, [queryClient]);
}
```

### Hook de Status de Rede

```bash
npx expo install @react-native-community/netinfo
```

```tsx
import NetInfo from '@react-native-community/netinfo';

export function useNetworkStatus() {
  const [isConnected, setIsConnected] = useState(true);
  const [connectionType, setConnectionType] = useState<string | null>(null);

  useEffect(() => {
    const unsubscribe = NetInfo.addEventListener((state) => {
      setIsConnected(state.isConnected ?? true);
      setConnectionType(state.type);
    });
    return unsubscribe;
  }, []);

  return { isConnected, connectionType, isWifi: connectionType === 'wifi' };
}

// Banner offline
function OfflineBanner() {
  const { isConnected } = useNetworkStatus();

  if (isConnected) return null;

  return (
    <View style={styles.banner}>
      <Text style={styles.bannerText}>Sem conexão com a internet</Text>
    </View>
  );
}
```

### Padrão de Selector para Performance

```tsx
// Evite assinar o store completo — selecione apenas o que você precisa
function CartBadge() {
  // Só re-renderiza quando a contagem de itens muda, não o preço total
  const itemCount = useCart((state) => state.items.length);
  return <Badge count={itemCount} />;
}

function CartTotal() {
  // Só re-renderiza quando o total muda
  const total = useCart((state) => state.total());
  return <Text>R$ {total.toFixed(2)}</Text>;
}
```

---

## Anti-Padrões a Evitar

**Armazenar tokens no Zustand ou MMKV sem criptografia.** O armazenamento de tokens deve usar SecureStore (Keychain/Keystore). O MMKV pode ser criptografado, mas requer gerenciar a chave com segurança.

**Usar AsyncStorage para leituras frequentes.** AsyncStorage é assíncrono — cada leitura retorna uma Promise. MMKV é síncrono e ~10x mais rápido. Evite AsyncStorage em código novo.

**Invalidar todas as queries em cada mutation.** `queryClient.invalidateQueries()` sem uma chave rebusca tudo. Seja cirúrgico: invalide apenas as query keys afetadas.

**Construir sua própria fila de sync sem idempotência.** Se seu processador de sync executar a mesma mutation duas vezes (ex.: após um crash no meio da sync), não deve criar dados duplicados. Use chaves de idempotência na sua API.

**Não tratar o estado `stale` na UI.** Quando React Query serve dados em cache enquanto busca, `isStale` é verdadeiro. Mostre um indicador de carregamento sutil em vez de um spinner completo para que os usuários saibam que os dados podem estar sendo atualizados.

**Persistir dados sensíveis no Zustand.** Qualquer state persistido via Zustand para MMKV é legível sem descriptografia em dispositivos com root. Nunca persista tokens, senhas ou dados pessoais no Zustand. Use SecureStore.

---

## Depuração e Resolução de Problemas

### React Query DevTools (Mobile)

```tsx
import { ReactQueryDevtools } from '@tanstack/react-query-devtools';

// Apenas em desenvolvimento
if (__DEV__) {
  // Use flipper-plugin-react-query ou reactotron-react-query
}
```

### Zustand DevTools

```tsx
import { devtools } from 'zustand/middleware';

const useStore = create<State>()(
  devtools(
    persist(/* ... */),
    { name: 'CartStore', enabled: __DEV__ }
  )
);
```

### Depuração MMKV

```tsx
// Inspecionar valores armazenados
if (__DEV__) {
  const keys = storage.getAllKeys();
  keys.forEach((key) => {
    console.log(key, storage.getString(key));
  });
}
```

### Problemas Comuns

**State do Zustand reseta no hot reload**
State está na memória — hot reload reavalia os módulos. Use o middleware `persist` com MMKV para state que deve sobreviver ao hot reload.

**React Query `networkMode: 'offlineFirst'` ainda exibindo estado de carregamento**
A query inicial não tem dados em cache. Mostre um estado de vazio/carregamento adequado para o primeiro lançamento, depois os dados em cache populam instantaneamente nos lançamentos subsequentes.

**`encryptionKey` do MMKV perdida após reinstalação do app**
A chave de criptografia deve vir do SecureStore (persistida no Keychain), não hardcoded. Chaves hardcoded anulam o propósito da criptografia.

---

## Cenários do Mundo Real

### App de Notas com Suporte Offline

```
Arquitetura de state:
- notes[] → MMKV via Zustand persist (acesso offline completo)
- mutations pendentes → SyncQueue Zustand store (sobrevive a reinicializações)
- sincronização com servidor → listener NetInfo processa fila na reconexão
- resolução de conflitos → servidor ganha, alterações locais mescladas com timestamp

Fluxo:
1. Usuário cria nota → adicionada ao Zustand + enfileirada na SyncQueue
2. App fica offline → nota visível imediatamente, sync ignorada
3. App se reconecta → SyncQueue processa CREATE_NOTE pendente
4. Servidor retorna nota salva com ID atribuído → atualiza ID local
5. React Query invalida lista de notas → UI reflete estado do servidor
```

---

## WatermelonDB para Dados Relacionais Complexos

WatermelonDB é um banco de dados reativo de alta performance para React Native, construído sobre SQLite. É a escolha certa quando seu app tem dados relacionais complexos (mensagens, posts com comentários, transações com itens) e milhares de registros que tornariam MMKV + JSON impraticável.

```bash
npx expo install @nozbe/watermelondb
npx expo install @nozbe/with-observables
```

```typescript
// database/schema.ts
import { appSchema, tableSchema } from '@nozbe/watermelondb';

export const schema = appSchema({
  version: 1,
  tables: [
    tableSchema({
      name: 'posts',
      columns: [
        { name: 'title', type: 'string' },
        { name: 'body', type: 'string' },
        { name: 'author_id', type: 'string', isIndexed: true },
        { name: 'created_at', type: 'number' },
        { name: 'is_published', type: 'boolean' },
      ],
    }),
    tableSchema({
      name: 'comments',
      columns: [
        { name: 'post_id', type: 'string', isIndexed: true },
        { name: 'body', type: 'string' },
        { name: 'author_id', type: 'string' },
        { name: 'created_at', type: 'number' },
      ],
    }),
  ],
});
```

```typescript
// models/Post.ts
import { Model, Query, Relation } from '@nozbe/watermelondb';
import { field, date, relation, children } from '@nozbe/watermelondb/decorators';
import { Comment } from './Comment';
import { User } from './User';

export class Post extends Model {
  static table = 'posts';
  static associations = {
    comments: { type: 'has_many' as const, foreignKey: 'post_id' },
    author: { type: 'belongs_to' as const, key: 'author_id' },
  };

  @field('title') title!: string;
  @field('body') body!: string;
  @field('is_published') isPublished!: boolean;
  @date('created_at') createdAt!: Date;
  @children('comments') comments!: Query<Comment>;
  @relation('users', 'author_id') author!: Relation<User>;
}
```

```tsx
// Componente reativo — re-renderiza apenas quando o post ou seus comentários mudam
import { withObservables } from '@nozbe/with-observables';
import { Post } from '../models/Post';
import { Comment } from '../models/Comment';

interface Props {
  post: Post;
  comments: Comment[];
}

const PostCard = ({ post, comments }: Props) => (
  <View>
    <Text style={styles.title}>{post.title}</Text>
    <Text>{comments.length} comentários</Text>
  </View>
);

// HOC conecta os streams observáveis do model às props
const enhance = withObservables(['post'], ({ post }: { post: Post }) => ({
  post,
  comments: post.comments,
}));

export const EnhancedPostCard = enhance(PostCard);
```

**Quando usar WatermelonDB vs MMKV:**

| Cenário | Ferramenta |
|---|---|
| Tokens de autenticação, preferências, flags simples | MMKV |
| Listas paginadas, cache de respostas da API | React Query + MMKV persister |
| Dados relacionais, joins, 10k+ registros | WatermelonDB |
| Dados colaborativos em tempo real | WatermelonDB + CRDT |

---

## Background Fetch com expo-background-fetch

Execute lógica periodicamente quando o app está em segundo plano — sincronize dados, atualize conteúdo, verifique atualizações.

```bash
npx expo install expo-background-fetch expo-task-manager
```

```typescript
// tasks/backgroundSync.ts
import * as BackgroundFetch from 'expo-background-fetch';
import * as TaskManager from 'expo-task-manager';
import { syncWithServer } from '../lib/sync';

export const BACKGROUND_SYNC_TASK = 'background-data-sync';

// Defina no nível do módulo (fora de qualquer componente ou hook)
TaskManager.defineTask(BACKGROUND_SYNC_TASK, async () => {
  try {
    const result = await syncWithServer();
    return result.hasNewData
      ? BackgroundFetch.BackgroundFetchResult.NewData
      : BackgroundFetch.BackgroundFetchResult.NoData;
  } catch (error) {
    console.error('Background sync failed:', error);
    return BackgroundFetch.BackgroundFetchResult.Failed;
  }
});

export async function registerBackgroundSync(): Promise<void> {
  const status = await BackgroundFetch.getStatusAsync();

  if (
    status === BackgroundFetch.BackgroundFetchStatus.Restricted ||
    status === BackgroundFetch.BackgroundFetchStatus.Denied
  ) {
    return; // Usuário/SO restringiu tarefas em segundo plano
  }

  const isRegistered = await TaskManager.isTaskRegisteredAsync(BACKGROUND_SYNC_TASK);
  if (isRegistered) return;

  await BackgroundFetch.registerTaskAsync(BACKGROUND_SYNC_TASK, {
    minimumInterval: 15 * 60,  // 15 minutos (SO pode atrasar significativamente)
    stopOnTerminate: false,     // continua após o usuário fechar o app
    startOnBoot: true,          // registra quando o dispositivo reinicia
  });
}

export async function unregisterBackgroundSync(): Promise<void> {
  await BackgroundFetch.unregisterTaskAsync(BACKGROUND_SYNC_TASK);
}
```

```tsx
// Chame registerBackgroundSync() uma vez na inicialização do app
useEffect(() => {
  registerBackgroundSync();
}, []);
```

**Restrições:**

- **iOS:** O SO controla o timing real. Seu `minimumInterval` é apenas uma sugestão — tarefas em segundo plano normalmente executam a cada 30–60 min dependendo da bateria e padrões de uso. Não é adequado para dados em tempo real.
- **Android:** Sujeito ao modo Doze. As tarefas podem ser agrupadas. Oriente o usuário a desativar a otimização de bateria para o seu app se sync frequente for crítico.
- **Alternativa para tempo real:** Use mensagens de dados FCM (push silencioso) para acionar sync em primeiro plano sob demanda.

---

## Estratégias de Resolução de Conflitos

Escritas offline que sincronizam com o servidor podem conflitar com mudanças concorrentes no servidor. Três estratégias cobrem a maioria dos casos.

**Estratégia 1: Last-Write-Wins (LWW)**

A abordagem mais simples — qualquer escrita com o timestamp mais recente vence.

```typescript
type SyncRecord = {
  id: string;
  data: Record<string, unknown>;
  updatedAt: number; // Unix ms
};

function mergeWithLWW(server: SyncRecord, client: SyncRecord): SyncRecord {
  return client.updatedAt > server.updatedAt ? client : server;
}
```

Boa para: preferências do usuário, configurações de perfil, feature toggles. Ruim para: dados colaborativos, contadores (incrementos concorrentes são perdidos).

**Estratégia 2: Merge por Campo**

Rastreie um snapshot base. Campos alterados apenas pelo cliente vencem; campos alterados por ambos são sinalizados para resolução pelo usuário.

```typescript
function mergeFields(
  base: Record<string, unknown>,
  server: Record<string, unknown>,
  client: Record<string, unknown>,
): { merged: Record<string, unknown>; conflicts: string[] } {
  const merged = { ...server };
  const conflicts: string[] = [];

  for (const key of Object.keys(client)) {
    const clientChanged = client[key] !== base[key];
    const serverChanged = server[key] !== base[key];

    if (clientChanged && !serverChanged) {
      merged[key] = client[key]; // cliente vence sem contestação
    } else if (clientChanged && serverChanged) {
      conflicts.push(key); // ambos mudaram — sinaliza para resolução na UI
    }
    // serverChanged && !clientChanged: servidor vence (já está em merged)
  }

  return { merged, conflicts };
}
```

**Estratégia 3: CRDTs (Conflict-free Replicated Data Types)**

Para edição colaborativa de texto, contadores compartilhados ou conjuntos — use uma biblioteca CRDT (`automerge`, `yjs`). CRDTs garantem consistência eventual sem um resolvedor central de conflitos.

```typescript
import * as Y from 'yjs';

// Documento compartilhado — mudanças de qualquer peer se mesclam automaticamente
const ydoc = new Y.Doc();
const ytext = ydoc.getText('content');

// Peer A digita "hello"
ytext.insert(0, 'hello');

// Peer B digita "world" na mesma posição concorrentemente
// Após sync: ambas as inserções são preservadas em ordem determinística
// Sem conflitos, sem perda de dados
```

Use CRDTs apenas para funcionalidades que realmente precisam deles (colaboração em tempo real, listas compartilhadas offline-first). Eles adicionam complexidade significativa — LWW ou merge por campo cobre 90% dos cenários de sync mobile.

---

## Leitura Adicional

- [Documentação do Zustand](https://zustand.docs.pmnd.rs/)
- [react-native-mmkv](https://github.com/mrousavy/react-native-mmkv)
- [TanStack Query — Suporte offline](https://tanstack.com/query/latest/docs/framework/react/guides/network-mode)
- [TanStack Query — Persistência](https://tanstack.com/query/latest/docs/framework/react/plugins/persistQueryClient)
- [Documentação do NetInfo](https://github.com/react-native-netinfo/react-native-netinfo)

---

## Resumo

Use Zustand para estado do cliente e React Query para estado do servidor — eles têm diferentes ciclos de vida e semânticas de cache. Persista slices do Zustand no MMKV via middleware `persist` e um adaptador de storage síncrono. MMKV é a escolha correta para armazenamento de state do app: síncrono, rápido e opcionalmente criptografado. Para escritas offline, implemente uma fila de sync persistente no Zustand que reprocessa mutations quando a conectividade for restaurada. Armazene tokens e segredos exclusivamente no SecureStore (Keychain/Keystore). Use selectors para assinar apenas o state que você precisa — evite assinaturas ao store completo em componentes que renderizam frequentemente.
