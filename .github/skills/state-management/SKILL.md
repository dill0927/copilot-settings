---
name: state-management
description: >
  Reactの状態管理の設計を支援するスキル。
  useState・useReducer・Context・SWR・React Query・Redux Toolkit・Zustand・Jotai などの
  使い分けや設計に関する質問があるときは必ずこのスキルを参照する。
  「状態どこに持つべき？」「グローバル状態にすべき？」などの明示的な依頼だけでなく、
  状態管理・データフェッチ・キャッシュ・フォーム状態を含むコードを扱う場合も積極的に適用する。
---

# React 状態管理ガイド

状態管理の鉄則は「**必要以上に広げない**」こと。
ローカルで完結できる状態をグローバルに持ち出すと、変更の影響範囲が読めなくなり、バグとパフォーマンス劣化の温床になる。

---

## 状態の種類と選択フロー

状態は性質によって4種類に分類できる。まずどの種類かを判断してから実装手段を選ぶ。

```
UI 状態（開閉・ホバー・入力値など）
  └─ 1コンポーネント内 → useState / useReducer
  └─ 複数コンポーネント間 → Context（更新頻度が低い場合）
               または props を整理して親に持ち上げる

サーバー状態（API から取得するデータ）
  └─ SWR または React Query
     キャッシュ・再取得・ローディング・エラーを任せる

フォーム状態
  └─ React Hook Form（バリデーション・サブミット込み）
     シンプルな場合は useState で十分

グローバル UI 状態（ツールバー設定・カートなど）
  └─ Zustand または Jotai
     サーバー状態でもローカル状態でも賄えない場合のみ使う
```

---

## ローカル状態: useState / useReducer

```tsx
// useState: 独立したシンプルな値
const [isOpen, setIsOpen] = useState(false)

// useReducer: 複数の値が連動して変わる場合
type State = { count: number; status: 'idle' | 'loading' | 'error' }
type Action = { type: 'increment' } | { type: 'fetch' } | { type: 'error' }

const reducer = (state: State, action: Action): State => {
  switch (action.type) {
    case 'increment': return { ...state, count: state.count + 1 }
    case 'fetch':     return { ...state, status: 'loading' }
    case 'error':     return { ...state, status: 'error' }
  }
}
const [state, dispatch] = useReducer(reducer, { count: 0, status: 'idle' })
```

**useReducer を選ぶタイミング:** useState が3つ以上連動する、状態遷移に条件分岐が多い、ロジックをテストしたい。

---

## サーバー状態: SWR / React Query

API データは「サーバーキャッシュ」として扱う。useState に詰め込まない。

```tsx
// SWR
import useSWR from 'swr'

const fetcher = (url: string) => fetch(url).then(r => r.json())

const UserProfile = ({ id }: { id: string }) => {
  const { data, error, isLoading } = useSWR(`/api/users/${id}`, fetcher)

  if (isLoading) return <Skeleton />
  if (error)     return <ErrorMessage />
  return <Profile user={data} />
}

// React Query
import { useQuery, useMutation } from '@tanstack/react-query'

const { data, isPending } = useQuery({
  queryKey: ['user', id],
  queryFn: () => fetchUser(id),
})

const mutation = useMutation({
  mutationFn: updateUser,
  onSuccess: () => queryClient.invalidateQueries({ queryKey: ['user', id] }),
})
```

**SWR vs React Query の選択:**
- シンプルな GET 中心 → SWR（軽量・設定少ない）
- mutation・楽観的更新・複雑なキャッシュ制御が必要 → React Query

---

## Context: ツリー内の共有値

Context は「頻繁に更新されない値の共有」に向いている。高頻度更新には使わない（再レンダリングが全子孫に波及するため）。

```tsx
// 向いているもの: テーマ・ロケール・認証ユーザー・機能フラグ
const AuthContext = createContext<User | null>(null)

export const AuthProvider = ({ children }: { children: ReactNode }) => {
  const [user, setUser] = useState<User | null>(null)
  return <AuthContext.Provider value={user}>{children}</AuthContext.Provider>
}

export const useAuth = () => {
  const ctx = useContext(AuthContext)
  if (ctx === undefined) throw new Error('useAuth must be used within AuthProvider')
  return ctx
}
```

**Context を使うべきでないサイン:** カートの数量・入力中のフォーム値など頻繁に変わる値 → Zustand か React Hook Form へ。

---

## グローバル状態: Zustand / Jotai

サーバー状態でも Context でも賄えない「クライアントのグローバル UI 状態」に使う。

```tsx
// Zustand: ストア単位で管理
import { create } from 'zustand'

type CartStore = {
  items: CartItem[]
  add: (item: CartItem) => void
  remove: (id: string) => void
}

const useCartStore = create<CartStore>((set) => ({
  items: [],
  add: (item) => set((s) => ({ items: [...s.items, item] })),
  remove: (id) => set((s) => ({ items: s.items.filter(i => i.id !== id) })),
}))

// コンポーネントから使う
const { items, add } = useCartStore()

// Jotai: atom 単位で管理（より細粒度）
import { atom, useAtom } from 'jotai'

const countAtom = atom(0)
const [count, setCount] = useAtom(countAtom)
```

**Zustand vs Jotai:**
- 関連する状態・アクションをまとめて管理したい → Zustand
- 独立した小さな状態を複数管理したい、派生状態が多い → Jotai

---

## Redux Toolkit（既存プロジェクト）

新規プロジェクトでは Zustand/Jotai が軽量で済むが、既存の Redux コードベースでは Redux Toolkit（RTK）を使う。

```tsx
import { createSlice, PayloadAction, configureStore } from '@reduxjs/toolkit'

// slice: reducer + action を一元定義
const counterSlice = createSlice({
  name: 'counter',
  initialState: { value: 0 },
  reducers: {
    increment: (state) => { state.value += 1 }, // Immer で immutable に書ける
    setValue: (state, action: PayloadAction<number>) => { state.value = action.payload },
  },
})

export const { increment, setValue } = counterSlice.actions

const store = configureStore({ reducer: { counter: counterSlice.reducer } })

// 型付きフック（src/store.ts に定義しておくと便利）
type RootState = ReturnType<typeof store.getState>
type AppDispatch = typeof store.dispatch
export const useAppSelector = useSelector.withTypes<RootState>()
export const useAppDispatch = useDispatch.withTypes<AppDispatch>()

// コンポーネントから使う
const value = useAppSelector((s) => s.counter.value)
const dispatch = useAppDispatch()
dispatch(increment())
```

**RTK Query（Redux のサーバー状態管理）:** Redux をすでに使っているプロジェクトでは、React Query の代わりに RTK Query を使うと Store との統合がシンプルになる。

```tsx
import { createApi, fetchBaseQuery } from '@reduxjs/toolkit/query/react'

const api = createApi({
  baseQuery: fetchBaseQuery({ baseUrl: '/api' }),
  endpoints: (build) => ({
    getUser: build.query<User, string>({
      query: (id) => `users/${id}`,
    }),
  }),
})

export const { useGetUserQuery } = api
// コンポーネントで: const { data, isLoading } = useGetUserQuery(userId)
```

**Redux を使い続けるべきサイン:**
- 既存コードが Redux で動いている（移行コストが高い）
- Redux DevTools の時間旅行デバッグを活用している
- チームに Redux の知見が十分ある

---

## フォーム状態: React Hook Form

バリデーション・送信・エラー表示が絡むフォームは React Hook Form に任せる。

```tsx
import { useForm } from 'react-hook-form'

type FormValues = { email: string; password: string }

const LoginForm = () => {
  const { register, handleSubmit, formState: { errors } } = useForm<FormValues>()

  const onSubmit = (data: FormValues) => login(data)

  return (
    <form onSubmit={handleSubmit(onSubmit)}>
      <input
        {...register('email', {
          required: 'メールアドレスを入力してください',
          pattern: { value: /\S+@\S+\.\S+/, message: '形式が正しくありません' },
        })}
      />
      {errors.email && <p role="alert">{errors.email.message}</p>}
    </form>
  )
}
```

---

## よくある設計ミスとその対処

### API レスポンスを useState に詰め込む

```tsx
// NG: ローディング・エラー・データを全部自前管理
const [data, setData]       = useState(null)
const [loading, setLoading] = useState(false)
const [error, setError]     = useState(null)

useEffect(() => {
  setLoading(true)
  fetch('/api/users').then(r => r.json()).then(setData).catch(setError).finally(() => setLoading(false))
}, [])

// OK: SWR / React Query に任せる
const { data, isLoading, error } = useSWR('/api/users', fetcher)
```

### グローバル状態の乱用

```tsx
// NG: モーダルの開閉をグローバルストアに持つ
const useModalStore = create(() => ({ isOpen: false, open: () => ..., close: () => ... }))

// OK: モーダルを使うコンポーネントのローカル状態で十分
const [isOpen, setIsOpen] = useState(false)
```

### Props バケツリレーを Context で解決しようとする

```tsx
// 3階層の Props バケツリレー → Context は大げさな場合がある
// まず「children を使った構成の逆転」を検討する

// OK: children で中間コンポーネントを素通りさせる
const Page = ({ user }: { user: User }) => (
  <Layout>
    <Header>{/* user を直接渡さず */}</Header>
    <UserCard user={user} /> {/* 必要な場所だけ渡す */}
  </Layout>
)
```

---

## チェックリスト

状態設計をレビューするときの確認観点：

1. **最小スコープ** — ローカルで完結できるのにグローバルに持ち出していないか
2. **サーバー状態の分離** — API データを useState で管理していないか
3. **Context の更新頻度** — 高頻度更新の値を Context に入れていないか
4. **フォームの一元管理** — バリデーション・エラー表示がバラバラになっていないか
5. **派生状態の計算** — state から導出できる値を別の state に持っていないか（`useMemo` で計算する）
