# Web Inspector — iPhoneでの使い方

Web Want Inspector をiPhone（Safari / Chrome）で使う方法。iOS上の全ブラウザはWebKitエンジンを強制されるため、Safariで動く方式はChromeでも同様に動く。

## なぜブックマークレット方式か

当初はiOS Shortcuts（Webページ上でJavaScriptを実行アクション + 共有シート）経由の案も検討したが、以下の理由でブックマークレットに一本化した。

- Shortcutsの当該アクションは、スクリプト末尾で`completion(result)`を呼ばないと「無効なJavaScript」エラーになる、Shortcuts固有の制約がある
- Safari App Extensionとして実装されているため、Chromeの共有シートには出てこない可能性がある
- Shortcuts特有のメリットは「共有ボタンからワンタップで実行できる」点のみで、他の機能（LANアドレスへの到達性・オーバーレイの動作・タップ選択）はブックマークレットでも全く同じ
- ブックマークレットは実機検証済みで確実に動作する

## 前提1：LANアドレスの設定

iPhoneは`localhost`を使えない（iPhoneから見ると「iPhone自身」を指してしまう）。Web InspectorモーダルでWant作成 →「手動（ブックマークレット / QR）」→「iPhone」タブを開くと、mywantサーバーの LAN IP が自動検出されて候補表示される。正しければそのまま**保存**（複数ネットワークインターフェースがある場合は誤検出することがあるので、`システム設定 > Wi-Fi` 等で実際のIPを確認すること）。保存すると `~/.mywant/config.yaml` に永続化され、以降このタブに表示されるブックマークレットのコードは自動的にこのアドレスを使う。

## 前提2：Caddyによる内部HTTPS化（必須）

**この設定をしないと、HTTPS化された実サイト（Googleなど、ほぼ全ての実サイト）ではブックマークレットが「Mixed Content」としてブラウザにブロックされ、何も起きない。** `localhost`はMixed Content判定の例外だが、実際のLAN IPは例外にならないため、HTTPS化が必須。

### セットアップ（Mac側、1回だけ）

```sh
brew install caddy
```

`/opt/homebrew/etc/Caddyfile` を編集（`192.168.2.211`の部分は実際のLAN IPに置き換え。GUIのiPhoneタブに保存したLANアドレスと同じ値にすること）:

```
localhost:8443, 127.0.0.1:8443, 192.168.2.211:8443 {
	tls internal
	reverse_proxy localhost:8081
}
```

**注意**: ホスト名を明示的に列挙すること（`:8443`のようにホスト名を省略した「全ホスト受け付け」設定にすると、Caddyがオンデマンド証明書発行をブロックし`tlsv1 alert internal error`で接続できなくなる）。LAN IPが変わった場合は、この設定ファイルとGUI側のLANアドレス設定を両方更新すること。

```sh
brew services start caddy
```

### iPhone側の証明書信頼設定（1回だけ）

Caddyの内部CA証明書は `/opt/homebrew/var/lib/caddy/pki/authorities/local/root.crt` に生成される。これをiPhoneに転送して信頼する必要がある。

**方法A（推奨）**: GUIのiPhoneタブ「Caddy CA証明書のパス」欄に上記パスを入力して保存すると、「証明書をダウンロード」リンクが表示される。**iPhoneのSafariでこのGUI（`http://<LANアドレス>:8081/web-wants`）を直接開き**、そのリンクをタップするだけでダウンロードできる（AirDrop不要。証明書自体は公開鍵のみで機密情報ではないため、この一度きりのダウンロードはHTTPS化前の平文8081ポート経由で行っても問題ない）。

**方法B（Macから転送する場合）**: Finderで`root.crt`を選択 → 共有 → AirDrop → 自分のiPhoneへ送信。

どちらの方法でファイルを受け取っても、以降の手順は同じ。

1. iPhoneで受信した`root.crt`（または`mywant-ca.crt`）をタップ →「プロファイルがダウンロード済み」の案内が出るので、設定アプリで「プロファイルがインストールされていません」→ **インストール**（パスコード入力）
2. 設定 → 一般 → 情報 → **証明書信頼設定** → 追加したルート証明書を**完全に信頼**するようトグルをON

これでiPhoneのSafariが`https://<LANアドレス>:8443/...`を正規の証明書として扱うようになる。

## 手順

1. GUIのiPhoneタブに表示されているブックマークレットのコード（`https://<LANアドレス>:8443/...`）をコピー
2. iPhoneの**Safari**（Chromeは動作しないため非推奨 — 下記「既知の制約」参照）で適当なページをブックマークに追加
3. そのブックマークを編集し、URL欄を手順1のコードに置き換えて保存
4. **iPhoneのSafariで対象URL（例: google.com）を実際に開く**（GUI側の「対象URL」欄への入力は、あくまでバックエンドのwant設定であり、iPhoneの画面には反映されない。別デバイスなので当然だが、見落としやすい）
5. その対象URLが表示されているタブのまま、ブックマーク一覧から手順3のブックマークをタップして実行
6. オーバーレイが表示されたら、ハイライトされた要素を直接タップして選択（キーボード/ゲームパッドと同様、サイドパネルの「✓ 完了」ボタンで確定）

## 既知の制約：Chromeでは動作しない

実機検証の結果、**Chrome for iOSは`javascript:`ブックマークレットの実行をブロックする**ことを確認済み（同一コードでSafariは正常動作、Chromeは無反応）。iOS上の全ブラウザがWebKitエンジンを共有していても、ブックマークレットの実行可否はブラウザアプリ側の実装依存であり、Chromeはこれを許可していない。**iPhoneでは必ずSafariを使うこと。**

一度ブックマークを作れば、以降のcreate/reviewセッションでも同じブックマークを使い回せる（`WebInspectorModal.tsx`の設計により、コード自体にセッション固有の情報を一切含めていないため — クリック時に `GET /api/v1/web-wants/active-inspection` に問い合わせて、今アクティブなセッションを都度発見する）。

## 関連

- 実装の背景・設計全体は memory の `project_web_inspector_manual_launch` を参照
- デスクトップ（Safari等、同一Mac上）向けは同モーダルの「デスクトップ」タブを使用（LANアドレス設定不要、`localhost`のまま）
