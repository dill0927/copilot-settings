---
name: component-design
description: >
  Reactコンポーネントの設計を支援するスキル。
  コンポーネントの作成・分割・リファクタリングを行うとき、Props設計・状態管理・責務分離・
  再利用性・コンポーネント構成に関する質問があるときは必ずこのスキルを参照する。
  「コンポーネントに分けて」「設計どうすべき？」などの明示的な依頼だけでなく、
  React コンポーネントのコードを扱う場合も積極的に適用する。
---

# Reactコンポーネント設計ガイド

良いコンポーネント設計の目標は「変更しやすく、テストしやすく、読みやすい」コードにすること。
以下のパターンと判断基準を使い、その場の文脈に合った設計を提案する。

---

## 責務の分離: Presentational / Container

コンポーネントを「見た目」と「ロジック」で分けると、テスト・再利用・変更が容易になる。

```tsx
// Presentational: UI の描画だけを担う。外部依存なし
type UserCardProps = {
  name: string
  avatarUrl: string
  onFollow: () => void
}
const UserCard = ({ name, avatarUrl, onFollow }: UserCardProps) => (
  <div>
    <img src={avatarUrl} alt={name} />
    <p>{name}</p>
    <button onClick={onFollow}>フォロー</button>
  </div>
)

// Container: データ取得・状態管理を担う。UI の詳細を知らない
const UserCardContainer = ({ userId }: { userId: string }) => {
  const { data } = useUser(userId)
  const { follow } = useFollow(userId)
  if (!data) return <Skeleton />
  return <UserCard {...data} onFollow={follow} />
}
```

**いつ分けるか:** Presentational 側を Storybook で単独表示したい、または複数の Container から使い回したいとき。小さなコンポーネントで再利用予定がなければ無理に分けない。

---

## Props 設計

### 必要最小限の Props にする

```tsx
// NG: 呼び出し側に詳細を知らせすぎる
<Button color="blue" size="lg" fontWeight="bold" borderRadius="4px">送信</Button>

// OK: ユースケースを表す variant で抽象化
<Button variant="primary">送信</Button>
```

### Props の型は具体的に

```tsx
// NG: string は何でも受け取れてしまう
type Props = { status: string }

// OK: 取りうる値を列挙する
type Props = { status: 'idle' | 'loading' | 'success' | 'error' }
```

### children vs render props vs コンポーネント Props

```tsx
// children: レイアウト・ラッパー系
<Card>
  <p>内容</p>
</Card>

// コンポーネント Props: 差し替えが必要なスロット
<Dialog header={<CustomTitle />} footer={<ActionButtons />} />

// render props: 内部状態を外に公開したいとき（useXxx フックで代替できる場合はそちらを優先）
<Dropdown renderItem={(item) => <CustomItem {...item} />} />
```

---

## コンポーネントの分割判断

分割するかどうかの基準：

| 判断軸 | 分割すべきサイン |
|---|---|
| **行数** | JSX が 100〜150 行を超えてきた |
| **責務** | 「〜かつ〜する」と説明が必要になった |
| **再利用** | 同じ構造が2箇所以上に現れた |
| **テスト** | テストしたいロジックが UI と混在している |
| **変更頻度** | 変わりやすい部分と安定した部分が混在している |

**分割しなくていいサイン:** 1回しか使わない、分割後のコンポーネントが親なしで意味をなさない（抽象化が漏れている）。

---

## 状態管理の置き場所

状態を置く場所は「最も近い共通の親」が原則。広げすぎない。

```
ローカル state (useState)
  └─ 1コンポーネント内で完結する UI 状態（開閉・入力値）

Context
  └─ ツリー内の複数コンポーネントが参照するが、頻繁に更新されない値
     （テーマ・ロケール・認証ユーザー）

サーバー状態 (SWR / React Query)
  └─ API から取得するデータ。キャッシュ・再取得・楽観的更新を任せる

グローバル状態 (Zustand / Jotai 等)
  └─ サーバー状態でもローカル状態でも賄えない場合のみ
     （ショッピングカート・未保存のフォーム状態など）
```

---

## よくある設計ミスとその対処

### 過度な Props バケツリレー

```tsx
// NG: 3階層以上 props をバケツリレーしている
<Page user={user}>
  <Header user={user}>
    <Avatar user={user} />
  </Header>
</Page>

// OK: Context か children で構成を逆転させる
<UserContext.Provider value={user}>
  <Page>
    <Header>
      <Avatar /> // Context から取得
    </Header>
  </Page>
</UserContext.Provider>
```

### ロジックを useEffect に詰め込む

```tsx
// NG: イベントハンドラで完結できるものを useEffect に書く
useEffect(() => {
  if (submitted) {
    saveData(formData)
  }
}, [submitted])

// OK: イベントハンドラに書く
const handleSubmit = () => {
  saveData(formData)
}
```

### 汎用すぎるコンポーネント

```tsx
// NG: 何でもできる God Component
<DataDisplay
  type="table|chart|list"
  sortable
  filterable
  paginated
  exportable
/>

// OK: 責務を分けて compose する
<FilterBar />
<DataTable />
<Pagination />
```

---

## コンポーネント設計のチェックリスト

コードレビュー・設計相談で確認する観点：

1. **単一責務** — コンポーネントの役割を一文で説明できるか
2. **Props の最小化** — 呼び出し側が知らなくていい詳細を隠せているか
3. **状態の置き場所** — 必要以上に上位に状態が持ち上がっていないか
4. **副作用の分離** — データ取得・購読がカスタムフックに切り出されているか
5. **再利用性** — Storybook で単独表示できる（外部依存なし）か
6. **テスタビリティ** — ロジックを UI と分けてテストできるか
