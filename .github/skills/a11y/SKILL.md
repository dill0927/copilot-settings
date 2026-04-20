---
name: a11y
description: >
  Webアクセシビリティ（a11y）の実装を支援するスキル。
  UIコンポーネントの作成・編集・レビューを行うとき、またはアクセシビリティ・WCAG・ARIA・
  キーボード操作・スクリーンリーダー・色コントラストに関する質問があるときは必ずこのスキルを参照する。
  「アクセシブルにして」「a11y チェック」「スクリーンリーダー対応」などの明示的な依頼だけでなく、
  button・form・modal・nav・img など UI 要素を含むコードを扱う場合も積極的に適用する。
---

# Webアクセシビリティ（a11y）実装ガイド

a11y 対応は後付けではなくコードを書く段階から組み込む。
このスキルは WCAG 2.1 AA 準拠を基準とし、実装可能な具体的コードパターンを提供する。

## 基本原則

アクセシビリティの問題は大きく4つに分類される（POUR）：

- **Perceivable**（知覚可能）: 視覚・聴覚に依存しない情報伝達
- **Operable**（操作可能）: キーボードのみで全操作が完結する
- **Understandable**（理解可能）: 予測可能な動作、明確なエラーメッセージ
- **Robust**（堅牢）: スクリーンリーダーなどの支援技術で正しく解釈される

コードを書く・レビューするとき、この4軸で問題がないか確認する。

---

## よく使うパターン集

### ボタン・インタラクティブ要素

```jsx
// NG: div や span をクリッカブルにする
<div onClick={handleClick}>送信</div>

// OK: button を使う（フォーカス・キーボード操作・role が自動で付く）
<button type="button" onClick={handleClick}>送信</button>

// アイコンのみのボタン: 視覚的なラベルがない場合は aria-label を付ける
<button type="button" aria-label="メニューを閉じる">
  <CloseIcon aria-hidden="true" />
</button>

// 状態を持つボタン（トグル・展開）
<button
  type="button"
  aria-expanded={isOpen}
  aria-controls="menu-list"
>
  メニュー
</button>
```

### フォーム

```jsx
// NG: placeholder をラベル代わりにする（フォーカス時に消える）
<input type="text" placeholder="メールアドレス" />

// OK: label を明示的に紐付ける
<label htmlFor="email">メールアドレス</label>
<input id="email" type="email" />

// エラーメッセージの紐付け
<input
  id="email"
  type="email"
  aria-describedby="email-error"
  aria-invalid={hasError}
/>
{hasError && (
  <p id="email-error" role="alert">
    有効なメールアドレスを入力してください
  </p>
)}

// 必須フィールド
<label htmlFor="name">
  名前 <span aria-hidden="true">*</span>
</label>
<input id="name" type="text" required aria-required="true" />
```

### モーダル・ダイアログ

```jsx
// フォーカストラップとキーボード操作が必須
<div
  role="dialog"
  aria-modal="true"
  aria-labelledby="dialog-title"
  // 開いたとき最初のフォーカス可能要素にフォーカスを移す
>
  <h2 id="dialog-title">確認</h2>
  <p>削除してよいですか？</p>
  <button onClick={onConfirm}>削除</button>
  <button onClick={onClose}>キャンセル</button>
</div>
```

モーダル実装時の必須チェック：
- 開いたとき最初のフォーカス可能要素にフォーカスを移す
- `Tab` / `Shift+Tab` がモーダル内に閉じ込められている（フォーカストラップ）
- `Escape` で閉じられる
- 閉じたとき元のトリガー要素にフォーカスを戻す

### 画像・メディア

```jsx
// 意味のある画像: alt に内容を説明する文言
<img src="chart.png" alt="2024年Q4売上グラフ：前年比120%増" />

// 装飾目的の画像: alt="" でスクリーンリーダーがスキップ
<img src="divider.png" alt="" aria-hidden="true" />

// アイコン: テキストと並んでいる場合は aria-hidden
<span aria-hidden="true">🔍</span> 検索
```

### ナビゲーション

```jsx
// landmark を使って構造を伝える
<header>...</header>
<nav aria-label="メインナビゲーション">...</nav>
<main>...</main>
<aside aria-label="関連リンク">...</aside>
<footer>...</footer>

// 現在のページをナビ内で示す
<nav>
  <a href="/home">ホーム</a>
  <a href="/about" aria-current="page">About</a>
</nav>

// スキップリンク（キーボードユーザー向け）
// ページ先頭に配置し、普段は非表示にする
<a href="#main-content" className="skip-link">
  メインコンテンツへスキップ
</a>
<main id="main-content">...</main>
```

### 動的コンテンツ・通知

```jsx
// 操作結果などの即時通知: role="alert" は自動で読み上げられる
<div role="alert">保存しました</div>

// ライブリージョン: 継続的に更新される情報（カウンター・ステータス等）
<div aria-live="polite" aria-atomic="true">
  {statusMessage}
</div>

// ローディング中
<div aria-busy="true" aria-label="読み込み中">
  <Spinner aria-hidden="true" />
</div>
```

---

## レビュー時のチェックリスト

コードレビューで a11y 問題を確認するときは以下を順番に見る：

1. **セマンティック HTML** — `div`/`span` で代替していないか。`button`, `a`, `input`, `label` 等の本来の要素を使っているか
2. **キーボード操作** — Tab でフォーカスできるか。Enter/Space で操作できるか。フォーカス順序が自然か
3. **ラベル** — 全インタラクティブ要素にアクセシブルな名前があるか（`aria-label`, `aria-labelledby`, または可視テキスト）
4. **色コントラスト** — テキストと背景のコントラスト比が 4.5:1（通常）または 3:1（大文字・太字）以上か
5. **フォーカス可視性** — フォーカスインジケータを CSS で消していないか（`outline: none` の乱用）
6. **動的変更の通知** — DOM の変化がスクリーンリーダーに伝わるか（`aria-live`, `role="alert"`）
7. **画像の代替テキスト** — 意味のある画像に `alt` があるか。装飾画像に `alt=""` があるか

---

## よくある間違いパターン

| NG | OK | 理由 |
|---|---|---|
| `<div onClick={...}>` | `<button>` | div はキーボード操作・role・フォーカスを持たない |
| `placeholder` のみ | `<label>` で紐付け | フォーカス時に消えてメモリ負荷が増す |
| `aria-label="ボタン"` | `aria-label="削除"` | 役割でなく目的を伝える |
| `color: red` でエラー表示 | テキスト＋色で表示 | 色覚異常のユーザーに伝わらない |
| モーダルにフォーカストラップなし | フォーカストラップ実装 | 背景にフォーカスが逃げる |
| `aria-hidden` を focusable 要素に付ける | focusable を除去してから付ける | フォーカスが消えて操作不能になる |

---

## 参照基準

- [WCAG 2.1](https://www.w3.org/TR/WCAG21/) — AA 準拠を最低ライン
- [WAI-ARIA Authoring Practices](https://www.w3.org/WAI/ARIA/apg/) — ウィジェットのパターン集
- [MDN Accessibility](https://developer.mozilla.org/en-US/docs/Web/Accessibility) — HTML 要素別の a11y リファレンス
