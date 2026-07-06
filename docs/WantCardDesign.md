# Want Card 設計ドキュメント

## 概要

Want Card は単一の Want（宣言的タスク）を表すグリッドセルです。固定高さ（sm: 6rem / md+: 10rem）のカードと、ポータル展開のフルスクリーンモードを持ちます。

---

## レイヤー構成（z-index）

```
z-30   ReactionOverlay / DragOverTarget / QuickActionsOverlay / DeleteConfirmOverlay
z-[26] Maximize button（常時表示）
z-[25] Card header
z-20   SelectMode checkbox
z-10   WantCardContent（コンテンツ全体）
z-0    ReplayScreenshot background
```

---

## 構成要素

### 1. カード外枠 `WantCard`

**ファイル:** `web/src/components/dashboard/WantCard/WantCard.tsx`

| 要件 | 詳細 |
|------|------|
| 固定高さ | 6rem (mobile) / 10rem (sm以上) |
| インタラクション | click → 詳細表示、右クリック → QuickActions、長押し → QuickActions |
| ドラッグ | ラベルドロップ・Want 並び替え・親 Want へのネスト対応 |
| キーボードナビ | `data-keyboard-nav-selected` / `tabIndex` で選択状態を管理 |
| 選択モード | `isSelectMode` 時はチェックボックス表示、ドラッグ無効 |
| ユーザーコントロール (`isControl`) | 非選択時はボーダー・ヘッダー非表示 |
| 処理中 (`isBeingProcessed`) | opacity-50、pointer-events-none |

**注意:** SVG 要素を含むプラグインがある場合、`handleCardClick` 内で `element.className` が `SVGAnimatedString` になるため `.includes()` は使用不可。`getAttribute('class')` を使用すること。

---

### 2. カードヘッダー

**実装:** `WantCard.tsx` 内 `absolute z-[25]`

| 要件 | 詳細 |
|------|------|
| 位置 | `header_position` 設定に応じて `top-0` または `bottom-0` |
| 表示条件 | 通常カード: 常時表示。`isControl` または `isFullScreen` ラベルあり: 選択時のみ表示 |
| 背景 | `backdrop-blur-[2px]` + 半透明白/紺。選択時は青味を帯びる |
| 左側 | type アイコン（Heart / BottleOnly / HeartInBottle）+ want type 名 |
| 右側（左→右順） | 子 want 数＋ステータスドット / チャットボタン / エージェント実行インジケーター / スケジュールアイコン |
| 子ステータスドット | 各子 want の status を色付きドットで表示。reaching 系は `pulseGlow` アニメーション |

---

### 3. 最大化ボタン

**実装:** `WantCard.tsx` 内 `absolute z-[26]`（ヘッダーから独立）

| 要件 | 詳細 |
|------|------|
| 表示条件 | `isSelectMode` 以外は常時表示（ヘッダーの visibility に依存しない） |
| 位置 | `isHeaderBottom` → `bottom-1 right-1`、それ以外 → `top-1 right-1` |
| アイコン回転 | `isHeaderBottom` 時は `rotate-90` |

---

### 4. プログレスバー `ProgressBars`

**ファイル:** `web/src/components/dashboard/WantCard/parts/ProgressBars.tsx`

| 要件 | 詳細 |
|------|------|
| 表示条件 | `!isControl \|\| selected` |
| データ | `want.state.current.achieving_percentage` (0–100) |

---

### 5. 相関オーバーレイ `CorrelationOverlay`

**ファイル:** `web/src/components/dashboard/WantCard/parts/CorrelationOverlay.tsx`

| 要件 | 詳細 |
|------|------|
| 表示条件 | `!isControl \|\| selected` |
| 用途 | ラベル検索時の関連度スコア表示 |

---

### 6. リプレイスクリーンショット背景

**実装:** `WantCard.tsx` 内 `absolute inset-0 z-0`

| 要件 | 詳細 |
|------|------|
| データ | `want.state.current.replay_screenshot_url` |
| 表示 | opacity 0.12 のカバー画像（pointer-events-none） |
| 表示条件 | `!isControl \|\| selected` |

---

### 7. バージョンバッジ `VersionBadge`

**ファイル:** `web/src/components/dashboard/WantCard/parts/VersionBadge.tsx`

| 要件 | 詳細 |
|------|------|
| 表示条件 | `!isControl \|\| selected` |
| データ | `want.metadata.version` |

---

### 8. コンテンツ領域 `WantCardContent`

**ファイル:** `web/src/components/dashboard/WantCardContent.tsx`（z-10）

内部に以下をこの順で持つ：

| サブ要素 | 表示条件 | 詳細 |
|----------|----------|------|
| **Status Badge** | `!isSelectMode && !isFullScreen` | want.status を右上に絶対配置 |
| **Reaction Overlay** | `shouldShowReactionButtons` | Approve / Deny ボタンを上部に表示（z-[30]）。reminder / goal の承認待ち状態 |
| **Error Display** | `isFailed && hasError && (!isControl \|\| isFocused)` | エラーメッセージを赤背景で表示 |
| **Plugin ContentSection** | プラグイン登録済み | want type に応じたカスタムコンテンツ |
| **Final Result** | `final_result != null && !isFullScreen && !plugin.hideFinalResult` | JSON / テキスト結果を表示。コピーボタン付き |

---

### 9. プラグインシステム

**ファイル:** `web/src/components/dashboard/WantCard/plugins/registry.ts`、`plugins/types/`

プラグインは want type 名でレジストリに登録し、`ContentSection` コンポーネントを提供します。外部プラグイン（`~/.mywant/custom-types/<type>/view/plugin.jsx`）は `window.__mywant.registerPlugin()` 経由でランタイム登録されます。

#### 組み込みプラグイン

| プラグイン | want type | コンパクト表示 | 拡張表示（isExpanded） |
|------------|-----------|----------------|------------------------|
| **TimerCardPlugin** | `timer` | every/at トグル + 0.75倍スケールのダイヤル | フルサイズダイヤル |
| **CodingCardPlugin** | `coding` | 入力欄（上）→ phase バッジ → AI レスポンス | 同左 |
| **WeatherCardPlugin** | `weather` | 天気サマリー | 詳細天気 |
| **SliderCardPlugin** | `slider` | スライダー値 | インタラクティブスライダー |
| **SwitchCardPlugin** | `switch` | ON/OFF 状態 | トグルスイッチ |
| **ButtonCardPlugin** | `button` | ボタン | ボタン |
| **ChoiceCardPlugin** | `choice` | 選択肢 | 選択肢 |
| **ReplayCardPlugin** | `replay` | スクリーンショット | 詳細 |
| **RpgStageViewCardPlugin** | `rpg_stage_view` | タイトルバー（下）+ scene テキスト | タイトルバー（上）+ scene テキスト |

#### 外部プラグイン

| プラグイン | want type | 場所 | コンパクト表示 | 拡張表示 |
|------------|-----------|------|----------------|----------|
| **rpg_observe** | `rpg_observe` | `~/.mywant/custom-types/rpg-observe/view/plugin.jsx` | タイトルバー（ステージ名 + next_goal）+ SVG ステージ図（h-full） | 上記 + event history + next goal テキスト |

#### プラグイン Props

| prop | 型 | 用途 |
|------|----|------|
| `want` | Want | Want データ全体 |
| `isChild` | boolean | 子カードとして表示中か |
| `isControl` | boolean | user-control ラベルありか |
| `isFocused` | boolean | キーボード選択中か |
| `isSelectMode` | boolean | 一括選択モードか |
| `isExpanded` | boolean | 最大化ポータル内か |
| `isInnerFocused` | boolean | コントロール内フォーカス（キーボード操作） |
| `onExitInnerFocus` | fn | 内部フォーカスを抜けるコールバック |
| `onView` | fn | カード詳細表示 |
| `onViewResults` | fn | 結果表示 |
| `onSliderActiveChange` | fn | スライダー操作中フラグ（ドラッグ抑制用） |

#### プラグイン実装上の注意

- コンパクト表示（`!isExpanded`）でインタラクティブ要素（ボタン、スライダー等）を持つ場合、その要素のみ `stopPropagation` を設定する。外側のラッパーに `stopPropagation` を置くとカードクリック（詳細表示）が無効になる。
- `hideFinalResult: true` を設定すると、プラグイン下部の Final Result 表示を抑制できる。

---

### 10. オーバーレイ群

| コンポーネント | ファイル | トリガー | 詳細 |
|----------------|----------|----------|------|
| **DragOverTarget** | WantCard.tsx 内 | Want ドラッグオーバー（target want） | 青オーバーレイ + Plus アイコン（z-30） |
| **SelectMode Checkbox** | WantCard.tsx 内 | `isSelectMode` | 右上にチェックボックス（z-20） |
| **QuickActionsOverlay** | `parts/QuickActionsOverlay.tsx` | 右クリック / 長押し | Start / Stop / Edit / Delete 等のアクションメニュー |
| **DeleteConfirmOverlay** | `parts/DeleteConfirmOverlay.tsx` | QuickActions から Delete 選択 | 削除確認ダイアログ |

---

### 11. 最大化ポータル（Expanded View）

**実装:** `WantCard.tsx` 内 `createPortal → document.body`

| 要件 | 詳細 |
|------|------|
| アニメーション | カード元位置 → 画面中央へ CSS transition（300ms cubic-bezier） |
| サイズ | 幅: min(640px, viewport-32px)、高さ: viewport × 0.82 |
| 閉じ方 | ✕ボタン / Esc キー |
| コンテンツ | `WantCardContent` を `isExpanded=true` で再描画 |
| レイアウト | `isHeaderBottom` に応じて `flex-col` / `flex-col-reverse` |

---

## ラベルによる表示モード切り替え

| ラベル | 値 | 効果 |
|--------|----|------|
| `user-control` | `"true"` | 非選択時: ヘッダー・ProgressBars・CorrelationOverlay を非表示。ドラッグ無効 |
| `full-screen-display` | `"true"` | Status Badge 非表示。非選択時ヘッダー非表示 |
| `recipe-based` | `"true"` | ヘッダーアイコンを 🫙 に変更（子あり: HeartInBottle、なし: BottleOnly） |
