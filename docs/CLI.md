# mywant-gui CLI リファレンス

`mywant-gui` は MyWant の Web フロントエンドを配信し、`/api/*` をバックエンド
（既定 `http://localhost:8080`）へプロキシする CLI バイナリです。サーバーの起動／
停止に加えて、**ロボットカーソル（CursorMan）を使った GUI 操作をプログラムから
制御する**コマンドを備えています。

> ⚠️ コマンド名は `mywant-gui`（独立バイナリ）です。`mywant gui ...` ではありません。
> （`make gui-restart` など一部の内部スクリプトは `mywant gui` ラッパーを使いますが、
> CLI として直接叩くときは `mywant-gui` を使ってください。）

- バイナリ: `mywant-gui`（`make install` で `~/.local/bin` に配置）
- バージョン確認: `mywant-gui --version`
- ヘルプ: `mywant-gui --help` / `mywant-gui <command> --help`

---

## グローバルフラグ

| フラグ | 説明 |
|---|---|
| `--backend <URL>` | バックエンド API の URL を上書き（既定は `~/.mywant/config.yaml` の `server_host`/`server_port`、なければ `http://localhost:8080`） |
| `-v, --version` | バージョン表示 |
| `-h, --help` | ヘルプ表示 |

設定ファイル `~/.mywant/config.yaml` のキー: `server_host`(=localhost) / `server_port`(=8080) /
`gui_port`(=8081) / `gui_host`。環境変数は `MYWANT_` プレフィックスで上書き可能。

---

## サーバー管理

### `start` — GUI サーバーを起動

```bash
mywant-gui start                                   # フォアグラウンド起動
mywant-gui start -D                                # バックグラウンド起動（detach）
mywant-gui start --port 8081 --backend http://myserver:8080
```

| フラグ | 既定 | 説明 |
|---|---|---|
| `-p, --port` | `8081` | 待ち受けポート（未指定時は config の `gui_port`） |
| `-H, --host` | `localhost` | バインドするホスト |
| `-D, --detach` | `false` | バックグラウンド実行。PID を `~/.mywant/gui.pid`、ログを `~/.mywant/gui.log` に記録 |

起動前に IPv4／IPv6 両方のワイルドカードでポート空きを確認し、使用中なら中断します。

### `stop` — GUI サーバーを停止

```bash
mywant-gui stop
mywant-gui stop --port 8081
```

| フラグ | 既定 | 説明 |
|---|---|---|
| `-p, --port` | `8081` | 停止対象ポート（最終手段としてポート占有プロセスも掃除） |

PID ファイル → プロセス名（`pgrep -f "mywant-gui start"`）→ ポート占有プロセス
の順に SIGTERM（5s 待って SIGKILL）で停止します。

### `get` — 現在の GUI 状態を表示

```bash
mywant-gui get
```

`source` / ダッシュボードのフィルタ・検索 / サイドバーの開閉・対象 want・タブ /
ロボット表示状態を表示します。

---

## ロボット発話（`say`）

ロボットカーソルに吹き出しを表示させ、任意の DOM 要素を指し示します。

```bash
mywant-gui say "こんにちは！"
mywant-gui say "完了！" --hide                       # 表示後すぐ消す
mywant-gui say "処理中..." --duration 3              # 3 秒後に自動で消す
mywant-gui say "フォームを開きます" --target add_want_btn --action click
```

| フラグ | 既定 | 説明 |
|---|---|---|
| `--hide` | `false` | メッセージ表示後すぐに非表示 |
| `--duration <N>` | `0` | N 秒後に自動非表示（0 = 出しっぱなし） |
| `--target <type>` | `none` | 指し示す DOM 要素の種類（下表） |
| `--target-id <id>` | — | 対象の識別子（want ID・タイプ名など） |
| `--action <hint>` | — | アクションヒント `click` / `hover` / `type` |

**`--target` に指定できる種類:**

| target | 意味 | `--target-id` |
|---|---|---|
| `add_want_btn` | ヘッダーの Add Want ボタン | 不要 |
| `want_type_card` | タイプ選択カード | 必須（タイプ名） |
| `form_deploy_btn` | Add Want フォームの Deploy ボタン | 不要 |
| `want_card` | want カード | 必須（want ID） |
| `nav_wants` | Wants ナビリンク | 不要 |
| `none` | 特定の対象なし（既定） | — |

---

## ページナビゲーション

ロボットカーソルを対応するナビ項目へアニメーションさせつつページ遷移します。
すべて `--message <文字列>` で吹き出し文言を上書きできます。

| コマンド | 遷移先 |
|---|---|
| `mywant-gui dashboard` | Wants ダッシュボード |
| `mywant-gui agents` | Agents ページ |
| `mywant-gui recipes` | Recipes ページ |
| `mywant-gui types` | Want Types ページ |

```bash
mywant-gui dashboard
mywant-gui agents --message "エージェント一覧を見てみましょう"
```

---

## Want カード操作（`wants`）

ロボットカーソルアニメーション付きで want カードを操作します。
`<name-or-id>` は want 名でも ID でも指定可能（内部で ID に解決）。

### `wants open <ID>` — サイドバーを開く

```bash
mywant-gui wants open my-want
mywant-gui wants open abc-123 --tab settings
mywant-gui wants open abc-123 --message "この want を見せます"
```

| フラグ | 既定 | 説明 |
|---|---|---|
| `--tab` | `results` | 開くタブ `settings` / `results` / `logs` / `agents` / `chat` |
| `--message` | — | 吹き出し文言 |

### `wants close` — サイドバーを閉じる

```bash
mywant-gui wants close
mywant-gui wants close --message "閉じます"
```

### `wants maximize [ID]` — カードを全画面表示（別名 `max`）

ID 省略時は現在サイドバーで開いている want を最大化します。

```bash
mywant-gui wants maximize abc-123
mywant-gui wants max choice-instance
mywant-gui wants maximize --collapse        # 全画面を解除して通常表示へ
```

| フラグ | 既定 | 説明 |
|---|---|---|
| `--collapse` | `false` | 最大化を解除して通常カード表示に戻す |
| `--message` | — | 吹き出し文言 |

### `wants latest --type <type>` — 最新作成 want の ID を取得

該当 want がなければ終了コード 1。テストスクリプトでデプロイ直後の want を
特定するのに使います（標準出力に ID のみを出力）。

```bash
WANT_ID=$(mywant-gui wants latest --type weather)
mywant-gui say "デプロイしました！" --target want_card --target-id "$WANT_ID"
```

| フラグ | 必須 | 説明 |
|---|---|---|
| `--type` | ✓ | want タイプで絞り込み |

---

## ビュー遷移（`show`、ロボットの有無あり）

### `show want <ID>` — 特定 want のサイドバーを開く（ロボットなし）

```bash
mywant-gui show want abc-123 --tab settings
```

| フラグ | 既定 | 説明 |
|---|---|---|
| `--tab` | `results` | `settings` / `results` / `logs` / `agents` / `versions` / `chat` |
| `--filter` | — | ダッシュボードのステータスフィルタ |
| `--search` | — | ダッシュボードの検索クエリ |

### `show dashboard` — ダッシュボードへ（サイドバーを閉じる）

```bash
mywant-gui show dashboard --filter reaching --search weather
```

| フラグ | 説明 |
|---|---|
| `--filter` | ステータスフィルタ（例 `reaching` / `stopped` / `achieved`） |
| `--search` | 検索クエリ |

### `show global` — Global サイドバーを開く（ロボットあり）

```bash
mywant-gui show global
mywant-gui show global --param otp_data_dir
mywant-gui show global --param otp_data_dir --message "このパラメータを確認しましょう"
```

| フラグ | 説明 |
|---|---|
| `--param` | Global サイドバー内でハイライトするパラメータキー |
| `--message` | 吹き出し文言 |

---

## Add Want フォーム操作（`form`）

`form` コマンド単体ではロボットは動きません。アニメーションさせたい場合は
**直前に `say --target ... --action click` を実行**してから（`sleep 0.8` を挟んで）
フォーム操作を行います。

| コマンド | 説明 |
|---|---|
| `form open` | Add Want フォームを開く（タイプ選択ビュー） |
| `form select <type>` | タイプを選択して params ビューへ移動 |
| `form suggest-deploy` | ロボットを Deploy ボタンへ向け、クリックをユーザーに促す |

> **`form suggest-deploy` は意図的にクリックを自動化しません。** Deploy は
> 非冪等な操作なので、最終クリックは人間のジェスチャーが必要です
> （誤った重複デプロイ防止のため）。自動化テストでは CDP 等で
> `button:has-text("Add")` をクリックします。

```bash
mywant-gui say "Add Want ボタンを押します！" --target add_want_btn --action click
sleep 0.8
mywant-gui form open

mywant-gui say "weather を選択します！" --target want_type_card --target-id weather --action click
sleep 0.8
mywant-gui form select weather

mywant-gui say "設定を確認してください" --target form_deploy_btn
sleep 1.5
mywant-gui form suggest-deploy
```

---

## パラメータ操作（`params`）

want の settings タブを開き、対象パラメータフィールドへロボットカーソルを
向けます。`--want` 省略時は現在サイドバーで開いている want を対象にします。
キーは `params.` プレフィックスを付けても可（自動で除去）。

### `params show <key>` — パラメータをハイライト

```bash
mywant-gui params show temperature --want abc-123
mywant-gui params show interval_seconds
```

### `params set <key> <value>` — パラメータを設定

値は自動で型推論されます（`true`/`false` → bool、整数 → int、小数 → float、
それ以外 → string）。API 経由で更新後、変更フィールドをハイライトします。

```bash
mywant-gui params set temperature 0.8 --want abc-123
mywant-gui params set interval_seconds 30 --want abc-123 --message "ポーリング間隔を更新"
```

| フラグ | 既定 | 説明 |
|---|---|---|
| `--want` | 現在のサイドバー want | 対象 want ID |
| `--message` | — | 吹き出し文言 |

---

## スクリーンショット（`capture`）

GUI で want カードを表示した状態を PNG 画像として保存します。
内部的に headless Chrome を起動し、GUI サーバー（既定 `http://localhost:8081`）に
接続してスクリーンショットを撮影します。

### `capture want <ID>` — カードをキャプチャ

```bash
mywant-gui capture want abc-123                               # サイドバー表示でキャプチャ
mywant-gui capture want abc-123 --max                        # 最大化ビューでキャプチャ
mywant-gui capture want my-want --output ~/Desktop/want.png  # 出力先を指定
mywant-gui capture want abc-123 --max --wait 2000            # アニメーション待機を延長
```

| フラグ | 既定 | 説明 |
|---|---|---|
| `--max` | `false` | 最大化（全画面）ビューをキャプチャ |
| `--output <file>` | `<id>.png` / `<id>_max.png` | 保存先 PNG ファイルパス |
| `--wait <ms>` | `1500` | 要素が表示された後の追加待機時間（アニメーション完了待ち） |

- `--max` なし: サイドバーパネル全体をキャプチャ（`results` タブ表示状態）
- `--max` あり: 全画面最大化カードをキャプチャ
- Chrome / Chromium が必要（macOS では `/Applications/Google Chrome.app` を自動検出）

---

## CursorMan 操作（`cursor`、Want Canvas）

Canvas モードで CursorMan のグリッド座標を制御します。
`x`/`dx` は正=右・負=左、`y`/`dy` は正=下・負=上。

| コマンド | 説明 |
|---|---|
| `cursor get` | 現在座標を表示 |
| `cursor set [x] [y]` | 絶対座標へ移動 |
| `cursor move [dx] [dy]` | 現在位置から相対移動 |

```bash
mywant-gui cursor get
mywant-gui cursor set 5 3
mywant-gui cursor move 3 0
```

**負の値**は位置引数では渡せません。フラグを使います。

```bash
mywant-gui cursor set --x -5 --y -3
mywant-gui cursor move --dx -3 --dy 2
```

---

## タイル操作（`tile`、Want Canvas）

### `tile set <name-or-id> <x> <y>` — タイルを移動

want タイルを指定座標へ移動し、CursorMan も同座標へ同時移動させます。

```bash
mywant-gui tile set workshop-hand 12 5
```

内部的には want のラベル `mywant.io/canvas-x` / `mywant.io/canvas-y` を更新します。

---

## Want 間の接続（`connect` / `disconnect`）

expose/import リンクを CLI から作成・削除します。provider/consumer の役割と
最適な対応キーは API が自動判定（field-match recommendations）します。
`<name-or-id>` は名前でも ID でも可。

```bash
mywant-gui connect workshop-eye workshop-brain      # eye → brain を接続
mywant-gui disconnect workshop-hand workshop-eye    # 誤配線を削除
```

| コマンド | 説明 |
|---|---|
| `connect <A> <B>` | 2 つの want 間に expose/import 接続を作成 |
| `disconnect <A> <B>` | 2 つの want 間の expose/import 接続を削除 |

---

## シェル補完

```bash
mywant-gui completion zsh   # bash / zsh / fish / powershell に対応
```

`wants open`・`show want`・`params --want` などは want ID を、`form select` は
want タイプ ID を、`--tab` はタブ名を補完します。

---

## 典型ワークフロー

### Add Want を canvas に配置してデプロイ後に検証

```bash
# 1. 現在の cursor 位置を確認
mywant-gui get

# 2. canvas モードでフォームを開く（pendingCanvasPos がセットされる）
mywant-gui form open

# 3. タイプ選択 → params ビューへ
mywant-gui form select switch

# 4. 最終 submit は CDP 等で button:has-text("Add") をクリック
#    （form suggest-deploy はユーザー操作前提でクリックを自動化しない）

# 5. 作成された want の ID を取得して検証
WANT_ID=$(mywant-gui wants latest --type switch)
curl http://localhost:8080/api/v1/wants/$WANT_ID | jq '.metadata.labels'
```

---

## エラー時

- 失敗時は `Error: ...` 形式で標準出力／標準エラーに出力し、終了コード 1 で終了します。
- GUI サーバーが起動していない場合は `mywant-gui start` で起動してください。
</content>
</invoke>
