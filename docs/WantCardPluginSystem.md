# Want Card Plugin System

このガイドは、ダッシュボードの Want カードに新しい表示UIを追加する開発者向けのドキュメントです。

## 概要

Want カードのUI実装は **プラグインシステム** で管理されています。各 Want タイプは自分専用のコンポーネントを持ち、中央の `WantCardContent` に変更を加えることなく独立して追加・修正できます。

### バックエンドとの対称性

バックエンドの Want タイプ登録（`RegisterWantImplementation[T, L]("timer")`）と同じ考え方をフロントエンドに適用しています。

```
バックエンド: engine/types/timer_types.go の init() で自己登録
フロントエンド: web/src/.../TimerCardPlugin.tsx の末尾で自己登録
```

## ファイル構成

```
web/src/components/dashboard/
  WantCardContent.tsx                    ← 薄いディスパッチャー（共通UI + plugin呼び出し）
  WantCard/
    plugins/
      registry.ts                        ← インターフェース定義 + レジストリ
      index.ts                           ← 全プラグインの一括登録（バレルimport）
      types/
        SliderCardPlugin.tsx             ← slider タイプのUI
        ChoiceCardPlugin.tsx             ← choice タイプのUI
        TimerCardPlugin.tsx              ← timer タイプのUI
        ReplayCardPlugin.tsx             ← replay タイプのUI（floating bubble含む）
```

## コアインターフェース

`registry.ts` に定義されています。

```typescript
// プラグインが受け取るプロパティ
export interface WantCardPluginProps {
  want: Want;             // Want オブジェクト全体
  isChild: boolean;       // 子カードとして表示されているか
  isControl: boolean;     // user-control ラベルが付いているか
  isFocused: boolean;     // フォーカス状態か
  isSelectMode: boolean;  // 複数選択モードか
  onView: (want: Want) => void;
  onViewResults?: (want: Want) => void;
  onSliderActiveChange?: (active: boolean) => void;  // slider専用
}

// プラグインの定義
export interface WantCardPlugin {
  types: string[];   // マッチする want type 名（複数指定可）
  ContentSection: React.ComponentType<WantCardPluginProps>;
}
```

## 新しい Want タイプのカードUIを追加する手順

### Step 1: プラグインファイルを作成する

`web/src/components/dashboard/WantCard/plugins/types/XxxCardPlugin.tsx` を作成します。

```typescript
import React, { useState, useEffect } from 'react';
import { WantCardPluginProps, registerWantCardPlugin } from '../registry';

const XxxContentSection: React.FC<WantCardPluginProps> = ({
  want, isChild, isControl, isFocused,
}) => {
  // want.state?.current から必要な値を読む
  const someValue = want.state?.current?.some_key as string | undefined;

  // ユーザー操作 → PUT /api/v1/states/{id}/{key} で状態を更新
  const handleChange = async (newValue: string) => {
    const id = want.metadata?.id;
    if (!id) return;
    await fetch(`/api/v1/states/${id}/some_key`, {
      method: 'PUT',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify(newValue),
    });
  };

  // isChild / isControl / isFocused に応じたマージン調整
  const mt = (isChild || (isControl && !isFocused)) ? 'mt-2' : 'mt-4';

  return (
    <div className={`${mt} ...`}>
      {/* タイプ固有のUI */}
    </div>
  );
};

// ファイル末尾で自己登録する
registerWantCardPlugin({
  types: ['xxx'],           // want.metadata.type と一致する文字列
  ContentSection: XxxContentSection,
});
```

### Step 2: index.ts に1行追加する

`web/src/components/dashboard/WantCard/plugins/index.ts` に import を追加します。

```typescript
import './types/SliderCardPlugin';
import './types/ChoiceCardPlugin';
import './types/TimerCardPlugin';
import './types/ReplayCardPlugin';
import './types/XxxCardPlugin';   // ← 追加するのはこの1行だけ
```

以上で完了です。`WantCardContent.tsx` への変更は不要です。

## WantCardContent の責務分担

`WantCardContent.tsx` は以下の **共通UI** のみを担当します。プラグインはこれらに手を加えません。

| 責務 | 場所 |
|---|---|
| ステータスバッジ | カード右上（`absolute` 配置） |
| Reaction オーバーレイ | goal/reminder の承認待ち時に表示 |
| カードヘッダー | タイプ名・エージェントアイコン・スケジュールアイコン |
| エラー表示 | `status === 'failed'` 時 |
| final_result 表示 | 全タイプ共通のリザルト表示 |
| **プラグイン呼び出し** | `plugin.ContentSection` をレンダリング |

```typescript
// WantCardContent.tsx 内のディスパッチ箇所
const plugin = getWantCardPlugin(wantType);

{plugin && (
  <plugin.ContentSection
    want={want}
    isChild={isChild}
    isControl={isControl}
    isFocused={isFocused}
    isSelectMode={isSelectMode}
    onView={onView}
    onViewResults={onViewResults}
    onSliderActiveChange={onSliderActiveChange}
  />
)}
```

## 状態の読み書きパターン

### 状態を読む

Want の状態は `want.state?.current` に格納されています。バックエンドが `StoreState(key, value)` で書いた値はここから読めます。

```typescript
// 基本的な読み方
const value = want.state?.current?.my_key as string | undefined;
```

### 状態を書く（ユーザー操作）

`PUT /api/v1/states/{id}/{key}` エンドポイントで単一キーを更新します。

```typescript
await fetch(`/api/v1/states/${want.metadata.id}/my_key`, {
  method: 'PUT',
  headers: { 'Content-Type': 'application/json' },
  body: JSON.stringify(newValue),
});
```

高頻度操作（スライダーなど）はデバウンスを挟みます。

```typescript
const debounceRef = useRef<ReturnType<typeof setTimeout> | null>(null);

const handleChange = (value: number) => {
  setLocalValue(value);   // UI即時反映
  if (debounceRef.current) clearTimeout(debounceRef.current);
  debounceRef.current = setTimeout(async () => {
    await fetch(`/api/v1/states/${id}/value`, { method: 'PUT', ... });
  }, 150);
};
```

### グローバルパラメータを書く（timer タイプの例）

グローバルパラメータを制御したい場合は、状態に書いてバックエンドの `Progress()` に伝播させます。

```
フロントエンド → PUT /api/v1/states/{id}/every
                       ↓
バックエンド Progress() が検知 → PropagateParameter(targetParam, WhenSpec)
                       ↓
                グローバルパラメータ更新（fromGlobalParam を使う Want に反映）
```

## 既存プラグイン一覧

| プラグインファイル | Want タイプ | 概要 |
|---|---|---|
| `SliderCardPlugin.tsx` | `slider` | 数値レンジスライダー。`min/max/step/value` を state から読む |
| `ChoiceCardPlugin.tsx` | `choice` | ドロップダウン選択。`choices` 配列と `selected` を state から読む |
| `TimerCardPlugin.tsx` | `timer` | SVGクロックダイアル。グローバルパラメータの WhenSpec（`every`/`at`）を制御 |
| `ReplayCardPlugin.tsx` | `replay` | ブラウザ録画・リプレイ。floating bubble（portal）含む |

## 複数タイプへの対応

1つのプラグインが複数の Want タイプを担当できます。

```typescript
registerWantCardPlugin({
  types: ['reminder', 'goal'],   // どちらにも同じ ContentSection を使う
  ContentSection: ReactionContentSection,
});
```

## portal（フローティングUI）を使う場合

`ReplayCardPlugin` のように、カード外にフローティングUIを表示したい場合は `createPortal` を使います。`ContentSection` 内で呼んでも `document.body` に正しくレンダリングされます。

```typescript
import { createPortal } from 'react-dom';

// ContentSection の return 内に含める
{showBubble && createPortal(
  <div ref={bubbleRef} style={bubbleStyle} className="...">
    {/* フローティングコンテンツ */}
  </div>,
  document.body,
)}
```

## 将来の拡張

| シナリオ | 対応方法 |
|---|---|
| 動的ロード（コード分割） | `React.lazy(() => import('./types/XxxCardPlugin'))` に変更するだけ |
| バックエンドから型定義を取得 | `getWantTypes()` API で取得した型名を元に動的登録 |
| デフォルトUI（未登録タイプ） | `types: ['__default__']` で登録、`getWantCardPlugin` のフォールバックに対応 |

---

## フロントエンドとバックエンドの関係

### アーキテクチャ全体図

```
ブラウザ (React SPA)
    │
    ├─ GET /api/v1/*  ────────────────────────────┐
    │                                             │
    │                                    mywant-gui (Go)
    │                                    server/server.go
    │                                    ポート 8081
    │                                             │
    │                                    リバースプロキシ
    │                                    /api/ → localhost:8080
    │                                             │
    └─ static (/)  ←── 埋め込み SPA              │
                                        mywant backend (Go)
                                        ポート 8080
```

mywant-gui は **静的ファイルサーバー＋リバースプロキシ** の薄いサーバーです。API リクエストはすべて mywant バックエンドへ透過的に転送されます。

---

### データフロー：Want 状態の読み取り（ポーリング）

フロントエンドは Smart Polling によって Want の状態を5秒ごとに同期します。

```
フロントエンド (wantStore.ts)
    │
    ├─ setInterval 5000ms
    │       │
    │       ├─ GET /api/v1/wants
    │       │       └─ レスポンス: Want[] (each with .hash フィールド)
    │       │
    │       └─ hash 比較
    │               変化があった Want のみ state を更新 → 該当カードだけ再レンダリング
    │
    └─ want.state.current.*  ← バックエンドが StoreState() で書いた値がここに入る
```

バックエンドでの状態書き込み例（`engine/types/timer_types.go`）：

```go
t.StoreState("every",         every)
t.StoreState("at",            at)
t.StoreState("timer_mode",    timerMode)
t.StoreState("at_recurrence", atRecurrence)
```

フロントエンドでの読み取り：

```typescript
const every    = want.state?.current?.every        as string | undefined;
const timerMode = want.state?.current?.timer_mode  as string | undefined;
```

---

### データフロー：Want 状態の書き込み（ユーザー操作）

プラグイン内のユーザー操作 → バックエンドの状態を直接更新 → 次のポーリングで全クライアントに伝播。

```
プラグイン UI 操作
    │
    └─ PUT /api/v1/states/{id}/{key}
            body: JSON.stringify(newValue)
            │
            └─ backend: want.StoreState(key, value)
                    │
                    └─ 次のポーリング (5秒以内) で各クライアントに反映
```

高頻度操作（スライダー等）はデバウンスを挟みます（150〜400ms）。

---

### データフロー：インタラクティブ Want のメッセージ送信

`interactive: true` の Want（coding タイプ等）は Webhook 経由でメッセージを受け取ります。

```
CodingCardPlugin
    │
    └─ POST /api/v1/webhooks/{wantName}
            body: { text: "...", sender: "user" }
            │
            └─ backend: WebhookReceiver → cc_messages に追記 → Progress() が処理
```

---

### データフロー：Reaction（承認/否認）

```
WantCardContent (ReactionOverlay)
    │
    └─ PUT /api/v1/reactions/{queueId}
            body: { approved: true/false, comment: "..." }
            │
            └─ backend: ReactionQueue.AddReaction() → Want の Progress() がブロック解除
```

---

### 外部プラグインの読み込みフロー

`~/.mywant/custom-types/` 以下に置いた JSX/TSX ファイルは、ページロード時に自動で読み込まれます。

```
App.tsx (useEffect)
    │
    ├─ GET /api/v1/plugins
    │       └─ レスポンス: ["/api/v1/plugins/rpg-observe.js", ...]
    │
    └─ import("/api/v1/plugins/rpg-observe.js")   ← dynamic ES module import
            │
            mywant backend: handlers_plugins.go
            ├─ ~/.mywant/custom-types/rpg-observe/view/plugin.jsx を読む
            ├─ esbuild で JSX → ESModule JS にコンパイル（JSXFactory: window.React.createElement）
            └─ コンパイル結果を返す（MD5 キャッシュ済み）
                    │
                    plugin.jsx 末尾の実行:
                    window.__mywant.registerPlugin({ types: ['rpg_observe'], ContentSection: ... })
                            │
                            registerWantCardPlugin() (registry.ts) に登録完了
```

**外部プラグインのファイル配置ルール：**

```
~/.mywant/custom-types/
  <want-type>/
    view/
      plugin.jsx   ← JSX が推奨（.tsx, .js, .ts も可）
```

---

### バックエンド Want タイプの自己登録

バックエンドとフロントエンドは同じ「自己登録」パターンを使います。

| 層 | 登録場所 | 登録方法 |
|---|---|---|
| バックエンド | `engine/types/<type>_types.go` の `init()` | `RegisterWantImplementation[T, L]("timer")` |
| フロントエンド（組み込み） | `plugins/types/TimerCardPlugin.tsx` の末尾 | `registerWantCardPlugin({ types: ['timer'], ... })` |
| フロントエンド（外部） | `~/.mywant/custom-types/<type>/view/plugin.jsx` の末尾 | `window.__mywant.registerPlugin({ types: ['rpg_observe'], ... })` |

新しい Want タイプを追加する際は、バックエンドとフロントエンドの両方に登録が必要です。

---

### 変更を反映する方法（再読み込み）

| 変更対象 | 反映方法 |
|----------|----------|
| 組み込みプラグイン（`plugins/types/*.tsx`） | `make install && mywant gui stop && mywant gui start -D` |
| 外部プラグイン（`~/.mywant/custom-types/*/view/plugin.jsx`） | **ブラウザをリロード**（Cmd+R）。バックエンドはファイルの MD5 が変わると自動で再コンパイルする |
| バックエンド Want タイプ定義 | `POST /api/v1/want-types/reload`（= `mywant want-type reload`）で再スキャン。GUI 再起動不要 |
| バックエンドロジック（Go コード） | `make release && mywant stop && mywant start -D` |

---

### API エンドポイント早見表（プラグイン実装で使うもの）

| メソッド | パス | 用途 |
|----------|------|------|
| `GET` | `/api/v1/wants` | Want 一覧取得（ポーリング） |
| `GET` | `/api/v1/wants/{id}` | 単一 Want 取得 |
| `PUT` | `/api/v1/states/{id}/{key}` | Want 状態の単一キーを更新 |
| `PUT` | `/api/v1/states/{id}` | Want 状態をまとめて更新 |
| `POST` | `/api/v1/webhooks/{name}` | メッセージ送信（interactive want） |
| `PUT` | `/api/v1/reactions/{queueId}` | Reaction 送信（承認/否認） |
| `GET` | `/api/v1/plugins` | 外部プラグイン URL 一覧 |
| `GET` | `/api/v1/plugins/{type}.js` | 外部プラグインのコンパイル済み JS |
| `POST` | `/api/v1/want-types/reload` | カスタム Want タイプを再スキャン |
