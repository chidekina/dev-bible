# Mobile State Management

## Overview

State management on mobile has unique constraints that web apps rarely face: storage is synchronous on some platforms and async on others, apps go to background and resume, and users expect offline functionality. A well-architected mobile state layer separates server state (React Query) from client state (Zustand), persists what needs persistence (MMKV), and handles network transitions gracefully.

This chapter covers the trio that makes mobile state robust: Zustand for client state, MMKV for fast synchronous storage, and React Query for server state with offline support.

---

## Prerequisites

- [React Native Basics](react-native-basics.md) — core components and hooks
- [Mobile Navigation](mobile-navigation.md) — understanding app lifecycle
- Familiarity with React Query from Track 02 or 03

---

## Core Concepts

### Storage Options Comparison

| Storage          | Sync  | Encrypted | Size limit | Use case                     |
|------------------|-------|-----------|------------|------------------------------|
| MMKV             | Yes   | Optional  | Large      | App state, user prefs, cache |
| AsyncStorage     | No    | No        | 6MB        | Legacy; avoid in new code    |
| SecureStore      | No    | Yes       | 2KB        | Tokens, secrets              |
| SQLite (Expo)    | No    | Optional  | Large      | Relational data, queries     |
| RNFS (filesystem)| No    | No        | Disk       | Files, images, documents     |

**MMKV** is the right default for most persistent state: it is synchronous (backed by mmap), 10x faster than AsyncStorage, supports encryption, and has a simple key-value API.

**SecureStore** (Expo) uses the iOS Keychain and Android Keystore for credentials. Max 2KB per value.

```tsx
// Token storage pattern
import * as SecureStore from 'expo-secure-store';

export const tokenStorage = {
  get: (key: string) => SecureStore.getItemAsync(key),
  set: (key: string, value: string) => SecureStore.setItemAsync(key, value),
  delete: (key: string) => SecureStore.deleteItemAsync(key),
};
```

### Client State vs Server State

```
Client State                    Server State
─────────────────────           ─────────────────────────────────
UI state (modal open/closed)    Data from your API
User preferences                Lists, paginated feeds
Form draft                      User profile
Cart / selection                Notifications
Navigation history              Search results

Managed by: Zustand             Managed by: React Query
Persisted in: MMKV              Cached in: React Query cache
                                Persisted in: MMKV (optional)
```

---

## Hands-On Examples

### Zustand Store with MMKV Persistence

```bash
npx expo install react-native-mmkv zustand
```

```tsx
// lib/storage.ts
import { MMKV } from 'react-native-mmkv';

export const storage = new MMKV({
  id: 'app-storage',
  // encryptionKey: 'your-encryption-key', // for sensitive data
});

// Zustand storage adapter
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
// stores/cart.ts — ephemeral state (no persistence)
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

### React Query on Mobile

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
      staleTime: 1000 * 60 * 5,    // 5 minutes
      gcTime: 1000 * 60 * 60 * 24, // 24 hours (persist cache)
      retry: (failureCount, error) => {
        // Don't retry on 4xx errors
        if (error instanceof ApiError && error.status < 500) return false;
        return failureCount < 3;
      },
      networkMode: 'offlineFirst', // serve from cache when offline
    },
    mutations: {
      networkMode: 'offlineFirst',
    },
  },
});

// Persist query cache to MMKV
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
  maxAge: 1000 * 60 * 60 * 24, // 24 hours
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
    placeholderData: (previousData) => previousData, // keep old data while fetching
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

    // Optimistic update
    onMutate: async ({ articleId, favorited }) => {
      await queryClient.cancelQueries({ queryKey: ['articles', articleId] });

      const previousArticle = queryClient.getQueryData<Article>(['articles', articleId]);

      queryClient.setQueryData<Article>(['articles', articleId], (old) =>
        old ? { ...old, favorited } : old
      );

      return { previousArticle };
    },

    onError: (_err, { articleId }, context) => {
      // Roll back on error
      if (context?.previousArticle) {
        queryClient.setQueryData(['articles', articleId], context.previousArticle);
      }
    },

    onSettled: (_data, _err, { articleId }) => {
      // Always refetch to sync with server
      queryClient.invalidateQueries({ queryKey: ['articles', articleId] });
    },
  });
}
```

### Offline Sync Pattern

For apps that need full offline writes (not just reads):

```tsx
// stores/syncQueue.ts — persist pending mutations
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

// hooks/useSyncProcessor.ts — process queue when online
import NetInfo from '@react-native-community/netinfo';

export function useSyncProcessor() {
  const { queue, remove, incrementRetries } = useSyncQueue();

  useEffect(() => {
    const unsubscribe = NetInfo.addEventListener(async (state) => {
      if (!state.isConnected || queue.length === 0) return;

      for (const mutation of queue) {
        if (mutation.retries >= 3) {
          // Dead letter — remove and notify user
          remove(mutation.id);
          notifyUser('Some offline changes could not be synced');
          continue;
        }

        try {
          await processMutation(mutation);
          remove(mutation.id);
          queryClient.invalidateQueries(); // refresh all data
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
      throw new Error(`Unknown mutation type: ${mutation.type}`);
  }
}
```

---

## Common Patterns & Best Practices

### App State and Background Handling

```tsx
import { AppState, AppStateStatus } from 'react-native';

// Refetch stale queries when app comes to foreground
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

### Network Status Hook

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

// Offline banner
function OfflineBanner() {
  const { isConnected } = useNetworkStatus();

  if (isConnected) return null;

  return (
    <View style={styles.banner}>
      <Text style={styles.bannerText}>No internet connection</Text>
    </View>
  );
}
```

### Selector Pattern for Performance

```tsx
// Avoid subscribing to the full store — select only what you need
function CartBadge() {
  // Only re-renders when item count changes, not total price
  const itemCount = useCart((state) => state.items.length);
  return <Badge count={itemCount} />;
}

function CartTotal() {
  // Only re-renders when total changes
  const total = useCart((state) => state.total());
  return <Text>${total.toFixed(2)}</Text>;
}
```

---

## Anti-Patterns to Avoid

**Storing tokens in Zustand or MMKV without encryption.** Token storage must use SecureStore (Keychain/Keystore). MMKV can be encrypted but requires managing the key securely.

**Using AsyncStorage for frequent reads.** AsyncStorage is async — every read returns a Promise. MMKV is synchronous and ~10x faster. Avoid AsyncStorage in new code.

**Invalidating all queries on every mutation.** `queryClient.invalidateQueries()` without a key refetches everything. Be surgical: invalidate only the affected query keys.

**Building your own sync queue without idempotency.** If your sync processor runs the same mutation twice (e.g., after a crash mid-sync), it should not create duplicate data. Use idempotency keys in your API.

**Not handling the `stale` state in the UI.** When React Query serves cached data while fetching, `isStale` is true. Show a subtle loading indicator rather than a full spinner so users know the data might be updating.

**Persisting sensitive data in Zustand.** Any state persisted via Zustand to MMKV is readable without decryption on rooted devices. Never persist tokens, passwords, or PII in Zustand. Use SecureStore.

---

## Debugging & Troubleshooting

### React Query DevTools (Mobile)

```tsx
import { ReactQueryDevtools } from '@tanstack/react-query-devtools';

// Only in development
if (__DEV__) {
  // Use flipper-plugin-react-query or reactotron-react-query
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

### MMKV Debugging

```tsx
// Inspect stored values
if (__DEV__) {
  const keys = storage.getAllKeys();
  keys.forEach((key) => {
    console.log(key, storage.getString(key));
  });
}
```

### Common Issues

**Zustand state resets on hot reload**
State is in memory — hot reload re-evaluates modules. Use `persist` middleware with MMKV for state that should survive hot reload.

**React Query `networkMode: 'offlineFirst'` still showing loading state**
The initial query has no cached data. Show a proper empty/loading state for first launch, then the cached data populates instantly on subsequent launches.

**MMKV `encryptionKey` lost after app reinstall**
The encryption key must come from SecureStore (persisted in Keychain), not hardcoded. Hardcoded keys defeat the purpose of encryption.

---

## Real-World Scenarios

### Offline-Capable Note App

```
State architecture:
- notes[] → MMKV via Zustand persist (full offline access)
- pending mutations → SyncQueue Zustand store (survives app restart)
- server sync → NetInfo listener processes queue on reconnect
- conflict resolution → server wins, local changes merged with timestamp

Flow:
1. User creates note → added to Zustand + enqueued in SyncQueue
2. App goes offline → note visible immediately, sync skipped
3. App reconnects → SyncQueue processes pending CREATE_NOTE
4. Server returns saved note with server-assigned ID → update local ID
5. React Query invalidates notes list → UI reflects server state
```

---

## WatermelonDB for Complex Relational Data

WatermelonDB is a high-performance reactive database for React Native, built on SQLite. It's the right choice when your app has complex relational data (messages, posts with comments, transactions with line items) and thousands of records that would make MMKV + JSON impractical.

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
// Reactive component — re-renders only when post or its comments change
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
    <Text>{comments.length} comments</Text>
  </View>
);

// HOC wires the model's observable streams to props
const enhance = withObservables(['post'], ({ post }: { post: Post }) => ({
  post,
  comments: post.comments,
}));

export const EnhancedPostCard = enhance(PostCard);
```

**When to use WatermelonDB vs MMKV:**

| Scenario | Tool |
|---|---|
| Auth tokens, user preferences, simple flags | MMKV |
| Paginated lists, API response caching | React Query + MMKV persister |
| Relational data, joins, 10k+ records | WatermelonDB |
| Real-time collaborative data | WatermelonDB + CRDT |

---

## Background Fetch with expo-background-fetch

Periodically run logic when the app is backgrounded — sync data, refresh content, check for updates.

```bash
npx expo install expo-background-fetch expo-task-manager
```

```typescript
// tasks/backgroundSync.ts
import * as BackgroundFetch from 'expo-background-fetch';
import * as TaskManager from 'expo-task-manager';
import { syncWithServer } from '../lib/sync';

export const BACKGROUND_SYNC_TASK = 'background-data-sync';

// Define at module level (outside any component or hook)
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
    return; // User/OS has restricted background tasks
  }

  const isRegistered = await TaskManager.isTaskRegisteredAsync(BACKGROUND_SYNC_TASK);
  if (isRegistered) return;

  await BackgroundFetch.registerTaskAsync(BACKGROUND_SYNC_TASK, {
    minimumInterval: 15 * 60,  // 15 minutes (OS may delay significantly)
    stopOnTerminate: false,     // continue after user closes app
    startOnBoot: true,          // register when device reboots
  });
}

export async function unregisterBackgroundSync(): Promise<void> {
  await BackgroundFetch.unregisterTaskAsync(BACKGROUND_SYNC_TASK);
}
```

```tsx
// Call registerBackgroundSync() once on app startup
useEffect(() => {
  registerBackgroundSync();
}, []);
```

**Constraints:**

- **iOS:** OS controls actual timing. Your `minimumInterval` is a hint — background tasks typically run every 30–60 min depending on battery and usage patterns. Not suitable for real-time data.
- **Android:** Subject to Doze mode. Tasks may be batched. Request the user to disable battery optimization for your app if frequent sync is critical.
- **Alternative for real-time:** Use FCM data messages (silent push) to trigger foreground sync on demand.

---

## Conflict Resolution Strategies

Offline writes that sync to the server can conflict with concurrent server changes. Three strategies cover most cases.

**Strategy 1: Last-Write-Wins (LWW)**

The simplest approach — whichever write has the later timestamp wins.

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

Good for: user preferences, profile settings, feature toggles. Bad for: collaborative data, counters (concurrent increments get lost).

**Strategy 2: Field-Level Merge**

Track a base snapshot. Fields changed only by the client win; fields changed by both are flagged for user resolution.

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
      merged[key] = client[key]; // client wins uncontested
    } else if (clientChanged && serverChanged) {
      conflicts.push(key); // both changed — flag for UI resolution
    }
    // serverChanged && !clientChanged: server wins (already in merged)
  }

  return { merged, conflicts };
}
```

**Strategy 3: CRDTs (Conflict-free Replicated Data Types)**

For collaborative text editing, shared counters, or sets — use a CRDT library (`automerge`, `yjs`). CRDTs guarantee eventual consistency without a central conflict resolver.

```typescript
import * as Y from 'yjs';

// Shared document — changes from any peer merge automatically
const ydoc = new Y.Doc();
const ytext = ydoc.getText('content');

// Peer A types "hello"
ytext.insert(0, 'hello');

// Peer B types "world" at the same position concurrently
// After sync: both insertions are preserved in a deterministic order
// No conflicts, no data loss
```

Use CRDTs only for features that truly need them (real-time collab, offline-first shared lists). They add significant complexity — LWW or field-merge covers 90% of mobile sync scenarios.

---

## Further Reading

- [Zustand docs](https://zustand.docs.pmnd.rs/)
- [react-native-mmkv](https://github.com/mrousavy/react-native-mmkv)
- [TanStack Query — Offline support](https://tanstack.com/query/latest/docs/framework/react/guides/network-mode)
- [TanStack Query — Persisting](https://tanstack.com/query/latest/docs/framework/react/plugins/persistQueryClient)
- [NetInfo docs](https://github.com/react-native-netinfo/react-native-netinfo)

---

## Summary

Use Zustand for client state and React Query for server state — they have different lifecycles and caching semantics. Persist Zustand slices to MMKV via the `persist` middleware and a synchronous storage adapter. MMKV is the correct choice for app state storage: synchronous, fast, and optionally encrypted. For offline writes, implement a persistent sync queue in Zustand that replays mutations when connectivity is restored. Store tokens and secrets exclusively in SecureStore (Keychain/Keystore). Use selectors to subscribe to only the state you need — avoid full-store subscriptions in frequently-rendering components.
