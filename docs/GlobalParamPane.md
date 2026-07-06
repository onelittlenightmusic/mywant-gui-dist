# Global Param Pane — 設計書

## 概要

Want Canvas の右端（Minimap とグリッドの間）に、**Global Param Pane** を追加する。
このペインは "asGlobalParam" が設定された Want だけを縦1列のグリッドとして表示し、
全 Want が参照可能なグローバルパラメータとして機能する Want を可視化・管理する。

---

## 配置・レイアウト

```
┌─────────────────────────────────────────────────────────────────┐
│  Header                                                         │
├──────────────┬──────────────────────────┬──────────┬───────────┤
│              │                          │  Global  │           │
│    Sidebar   │   WantCanvas (main grid) │  Param   │  Minimap  │
│   (nav)      │                          │  Pane    │  (480px)  │
│              │                          │  (130px) │           │
└──────────────┴──────────────────────────┴──────────┴───────────┘
```

- **位置**: Fixed、画面右端から 480px (minimap幅) の左側。つまり right: 480px。
- **幅**: `130px` 固定（1列分のグリッドセル + パディング）
- **高さ**: header 下から画面下まで（canvas と同じ高さ）
- **z-index**: minimap より低く、canvas より高い（z-40 程度）

### Dashboard 側の変更

```tsx
// 既存
<div className="... lg:pr-[480px]">
  <WantCanvas ... />
</div>

// 変更後
<div className="... lg:pr-[480px]">
  <WantCanvas ... viewportInsetRight={minimapInsetRight} />
</div>

// 追加: GlobalParamPane（固定位置）
<GlobalParamPane
  wants={topLevelWants}
  isMinimapOpen={minimapOpen}
  onDropFromCanvas={handleGlobalParamDrop}
  onDragOutToCanvas={handleGlobalParamDragOut}
/>
```

---

## Global Param Pane コンポーネント仕様

### ファイル

`web/src/components/dashboard/GlobalParamPane.tsx`

### Props

```ts
interface GlobalParamPaneProps {
  /** 全 Want リスト（asGlobalParam 有無に関わらず渡す） */
  wants: Want[];
  /** ミニマップが開いているか（開いている時は right: 480px、閉じている時は right: 0） */
  isMinimapOpen: boolean;
  /** Canvas → GlobalParamPane へのドロップ時コールバック（recommend 表示用） */
  onDropFromCanvas: (wantId: string, dropPosition: { slotIndex: number }) => void;
  /** GlobalParamPane → Canvas へのドラッグアウト時コールバック（asGlobalParam 解除） */
  onDragOutToCanvas: (wantId: string) => void;
}
```

### 表示ロジック

```ts
// asGlobalParam を持つ Want のみ表示
const globalParamWants = wants.filter(w =>
  w.spec?.exposes?.some(e => !!e.asGlobalParam)
);
```

### 見た目

```
┌────────────────┐
│ 🌐 Global      │  ← ペインヘッダー
│ Params         │    (小さいラベル)
├────────────────┤
│  ┌──────────┐  │
│  │ TimerA   │  │  ← WantCardFace (context="global-param")
│  │   🕐     │  │
│  └──────────┘  │
│       ↓ 🌐    │  ← 緑の下向き三角 + グローバルアイコン
├────────────────┤
│  ┌──────────┐  │
│  │ WeatherB │  │
│  │   ☁     │  │
│  └──────────┘  │
│       ↓ 🌐    │
├────────────────┤
│  ドロップゾーン  │  ← ドラッグ時にハイライト
│  (破線枠)      │
└────────────────┘
```

---

## 下向き三角インジケーター

通常の Canvas 上の接続三角とは異なるデザイン。

```tsx
// GlobalParamConnector コンポーネント
// 位置: WantCardFace の直下（カード外）
<div style={{ display: 'flex', alignItems: 'center', justifyContent: 'center', gap: 4 }}>
  {/* 緑の下向き三角 */}
  <div style={{
    width: 14, height: 14,
    clipPath: 'polygon(0 0, 100% 0, 50% 100%)',  // ▼
    background: 'linear-gradient(135deg, #86efac 0%, #22c55e 50%, #15803d 100%)',
    boxShadow: '0 0 6px rgba(34,197,94,0.9), 0 0 14px rgba(34,197,94,0.5)',
  }} />
  {/* グローバルアイコン (lucide: Globe) */}
  <Globe className="w-3 h-3 text-green-400" />
</div>
```

**色仕様:**
- 通常の expose 三角 → シアン (`#22d3ee`)
- GlobalParam 三角 → **緑** (`#22c55e`)
- グローバルアイコン → `text-green-400`

---

## ドラッグ＆ドロップ仕様

### ① 通常ペイン → Global Param Pane へのドラッグ

**フロー:**
1. WantCanvas から want をドラッグ開始（既存の `dragstart` 処理）
2. GlobalParamPane 上に入った時 (`dragover`) → ペイン内セルをハイライト
3. GlobalParamPane にドロップ (`drop`) → **GlobalParamRecommendBubble** を表示
4. Bubble に各 state フィールドと suggested global param 名を表示
5. ユーザーが選択 → `PATCH /api/v1/wants/{id}` で exposes に `asGlobalParam` を追加
6. ユーザーがキャンセル → want を元の canvas 位置に戻す（localOverride を元に戻す）

**ドロップ処理の分離:**

```tsx
// GlobalParamPane.tsx の onDragOver/onDrop
const handleDragOver = (e: React.DragEvent) => {
  const isWantMove = e.dataTransfer.types.includes('application/mywant-canvas-id');
  if (!isWantMove) return;
  e.preventDefault();
  setIsDragOver(true);
};

const handleDrop = (e: React.DragEvent) => {
  const wantId = e.dataTransfer.getData('application/mywant-canvas-id');
  if (!wantId) return;
  e.preventDefault();
  e.stopPropagation();  // ← WantCanvas の drop に伝播させない（重要！）
  onDropFromCanvas(wantId, { slotIndex: getSlotIndexFromEvent(e) });
};
```

### ② Global Param Pane → 通常ペインへのドラッグアウト

**フロー:**
1. GlobalParamPane 内の want をドラッグ開始
2. WantCanvas 上でドロップ → `onDragOutToCanvas(wantId)` を呼ぶ
3. Dashboard 側で `PATCH /api/v1/wants/{id}` → exposes から asGlobalParam を持つエントリを削除
4. want は通常の canvas グリッドに移動

**実装ポイント:**
- GlobalParamPane 内のドラッグでも `application/mywant-canvas-id` の dataTransfer を設定
- ただし追加で `application/mywant-from-global-pane: true` を設定して区別
- WantCanvas の `handleDrop` で `from-global-pane` フラグを検出したら `onDragOutToCanvas` を呼ぶ

```tsx
// WantCanvas の handleDrop 内
const fromGlobalPane = e.dataTransfer.getData('application/mywant-from-global-pane') === 'true';
if (fromGlobalPane && wantIdMove) {
  onDragOutFromGlobalPane?.(wantIdMove);  // 新しい prop
  applyMove(wantIdMove, { x: cx, y: cy }, dragClusterOffsets);
}
```

### ③ Global Param Pane 内でのドラッグ（並び替え）

- 並び替えは可能（スロットの順番を変える）
- ただし **FieldMatchBubble（接続recommend）は表示しない**
- 他の want との接続処理は一切行わない

---

## GlobalParamRecommendBubble コンポーネント

### ファイル

`web/src/components/dashboard/GlobalParamRecommendBubble.tsx`

### 動作

WantCanvas から GlobalParamPane にドロップされたとき表示される。
`FieldMatchBubble` と同じ見た目・動作だが、対象が「global param への expose 設定」に限定される。

```tsx
interface GlobalParamRecommendBubbleProps {
  /** ドロップされた want */
  want: Want;
  /** 表示位置（canvas 座標系） */
  position: { x: number; y: number };
  onApply: (entry: { currentState: string; asGlobalParam: string }) => Promise<void>;
  onDismiss: () => void;
}
```

### 推奨候補の取得

want type の定義から state フィールドを取得。または新しい API エンドポイントを追加：

```
GET /api/v1/wants/{id}/global-param-options
```

**レスポンス:**
```json
{
  "options": [
    {
      "currentState": "schedule",
      "suggestedName": "global_timer_schedule",
      "description": "Timer schedule (every/at)"
    }
  ]
}
```

または既存の want type 定義（`wantTypes[].currentStates`）から候補を生成してもよい。

### 表示例

```
┌─────────────────────────────────────┐
│ 🌐 Global Param として公開          │
│                                     │
│ Timer_A の state を global param    │
│ として設定します。                   │
│                                     │
│ ○ schedule → global_timer_schedule  │
│                                     │
│ グローバル名: [global_timer_1    ]  │
│              (編集可能)             │
│                                     │
│  [キャンセル]       [適用 ✓]       │
└─────────────────────────────────────┘
```

### apply 時の API call

```ts
// PATCH /api/v1/wants/{id}
// Spec.Exposes に新エントリを追加
{
  spec: {
    exposes: [
      ...existing_exposes,
      { currentState: "schedule", asGlobalParam: "global_timer_1" }
    ]
  }
}
```

既存の PATCH want API を使用する（サーバー側変更不要）。

---

## ペイン内での接続処理の無効化

### 設計

- GlobalParamPane 内では want 同士の接続ヒント（`recommendationHintCells`）を表示しない
- GlobalParamPane 内での want ドロップ時に `useFieldMatchProximity.checkOnDrop` を呼ばない
- GlobalParamPane → GlobalParamPane のドロップでは FieldMatchBubble を表示しない

### 実装方法

`WantCanvas` には `useFieldMatchProximity` hook が含まれているが、GlobalParamPane は別コンポーネントなので自然に分離される。GlobalParamPane は独自の `onDrop` ハンドラを持ち、proximity check を行わない。

---

## 接続 Triangle の差分

| 種別 | 形状 | 色 | アイコン | 表示場所 |
|------|------|-----|---------|---------|
| 通常 expose | ▶◀▼▲ (方向可変) | シアン | なし | canvas ギャップ |
| global param | ▼ (常に下向き) | **緑** | 🌐 Globe | global param ペイン、各カードの下 |

---

## 状態管理

### Dashboard 側で管理する状態

```ts
// グローバルパラメータドロップのペンディング状態
const [pendingGlobalParamDrop, setPendingGlobalParamDrop] = useState<{
  wantId: string;
  slotIndex: number;
} | null>(null);
```

### フロー

```
drop → setPendingGlobalParamDrop → GlobalParamRecommendBubble 表示
  ↓ apply
PATCH want (exposes追加) → fetchWants() → globalParamWants 更新
  ↓ dismiss (キャンセル)
setPendingGlobalParamDrop(null) → want は元の位置のまま
```

---

## API 変更

### 新規エンドポイント（任意）

```
GET /api/v1/wants/{id}/global-param-options
```

want type の定義から expose 可能な state フィールドのリストを返す。
フロントエンド側で want type 情報から計算する方法もあり、実装コストを削減できる。

### 既存エンドポイントの利用

asGlobalParam の設定・解除は既存の `PATCH /api/v1/wants/{id}` で対応可能。
サーバー側の追加変更は最小限で済む。

---

## 実装ファイル一覧

| ファイル | 変更種別 | 内容 |
|---------|---------|------|
| `web/src/components/dashboard/GlobalParamPane.tsx` | **新規** | グローバルパラメータペイン本体 |
| `web/src/components/dashboard/GlobalParamRecommendBubble.tsx` | **新規** | ドロップ時の recommend bubble |
| `web/src/components/dashboard/WantCanvas.tsx` | **修正** | `from-global-pane` フラグ検出、`onDragOutFromGlobalPane` prop 追加 |
| `web/src/pages/Dashboard.tsx` | **修正** | GlobalParamPane 追加、ハンドラ追加 |

---

## 実装優先順位

1. **Phase 1 (Core)**: GlobalParamPane コンポーネント（表示のみ）
   - asGlobalParam を持つ want を縦1列で表示
   - 緑の下向き三角 + Globe アイコン

2. **Phase 2 (Drag)**: ドラッグ処理
   - Canvas → GlobalParamPane ドロップで recommend bubble 表示
   - GlobalParamPane → Canvas ドラッグアウトで asGlobalParam 解除

3. **Phase 3 (Polish)**: スロット並び替え、アニメーション、空状態の UX
