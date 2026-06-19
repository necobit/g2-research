# Even Realities G2 調査まとめ

調査日: 2026-06-19

## ハードウェア仕様

- マイク: **4個**（4マイクアレイ）
- カメラ: **なし**（マイク入力特化設計）
- ディスプレイ: Micro-LED, 576×288px, 4ビットグレースケール
- Bluetooth: 5.4
- ウェイクワード: "Hey Even"（オンデバイス検出）

## オーディオ処理アーキテクチャ

4マイクアレイのビームフォーミング処理がグラス本体側かスマホSDK側かは**未確定**（公式ソースで未確認）。

ただしSDKに届く時点でモノラルPCMになっていることは確認済み。BLE帯域（4ch生データでは約1Mbpsが必要）の観点からはグラス側DSP処理の可能性が高い（ネビィの憶測）。

## SDK 音声API

### raw PCMストリーム取得

```typescript
// app.json に permission 宣言が必要
// { "permissions": ["g2-microphone"] }

await bridge.audioControl(true);  // マイクON

bridge.onEvenHubEvent((event) => {
  if (event.audioEvent) {
    const pcm: Uint8Array = event.audioEvent.audioPcm;
    // PCM: 16kHz / mono / 16bit
  }
});
```

- フォーマット: **16kHz, mono, 16bit PCM**
- BLE経由のため大きなチャンクは転送に時間がかかる
- STTは自前で用意（Deepgram / AssemblyAI / Whisper / Soniox等に投げる）

### STTモジュール（even-toolkit）

```typescript
import { useSTT } from 'even-toolkit';
// プロバイダを差し込むだけでSTTパイプライン完成
// APIキーは別途必要
```

## カスタムエージェント経由の音声テキスト取得

Even Hubアプリの「カスタムエージェント」機能を使うと、**STT済みテキストを直接受け取れる**。

### フロー

```
「Hey Even、電気消して」
  └→ G2オンデバイスSTT（テキスト化）
      └→ カスタムエンドポイントにOpenAI互換APIとしてPOST
          └→ 自作サーバーで処理
              └→ SwitchBot API 等を呼ぶ
```

### 設定方法

Even HubアプリでカスタムエージェントのURL・名前・トークンを登録するだけ。
エンドポイントは **OpenAI互換API** として実装する必要がある。

Cloudflare Worker / FastAPI / LM Studio などで実装可能。

### SwitchBot制御への応用

```
「Hey Even、リビングの電気消して」
  └→ カスタムサーバー受信
      └→ Function Callingで switchbot_control ツール呼び出し
          └→ SwitchBot API: POST /v1.1/devices/{deviceId}/commands
              └→ 電気が消える
```

## 副産物: G2のオンデバイスSTT確認

カスタムエージェント連携のブログ記事により、
「Hey Even」後に音声→テキスト変換がデバイス内で完結していることが確認できた。
（SDKのuseSTTフック（外部STT利用）とは別の処理）

---

# 画面描画・BLE・パッケージング（サンプルアプリ「ねこさんぽ」開発で判明）

下端／画面内を歩く猫アプリ（ネッコサーフィン的なもの）を `@evenrealities/even_hub_sdk` で実装する過程で得た、グラフィック描画まわりの実機知見。**「実機確認」と書いたものは G2 実機で確認済み**、それ以外は docs / コミュニティ情報。

## SDKは2種類ある（混同注意）

- **`@evenrealities/even_hub_sdk`** … グラスネイティブの「プラグイン」を作るSDK。Webアプリ（Vite+TS）をEven AppのWebViewで動かし、コンテナ更新をBLEでグラスへ送る。アプリを作るならこっち。
- **`@evenrealities/even-terminal`** … ノートPCのCLI画面（Claude Code/Codex等）をグラスにミラーする別物。アプリ開発用ではない。

## 開発ワークフロー：dev/QR と .ehpk

- **開発（推奨）**: `npm run dev`（Vite :5173）＋ `evenhub qr` でQR生成 → Even Appの**プロトタイプモード**でライブ読み込み。**毎回最新が即反映・キャッシュ無し**。スマホとPCが届くネットワークが必要（同一LAN or Tailscale。Viteは `host: true` で全IFにbind）。
- **配布**: `npm run pack`（`evenhub pack app.json dist -o x.ehpk`）で `.ehpk` 生成。フォーマットは独自（"EHPK"マジック＋WASM圧縮、標準zipではない）。docs曰く **`.ehpk` は将来のEven Hubポータル配信用**。
- **`.ehpk` 更新が反映されない罠（実機確認）**: `app.json` の `version` を上げないと、端末/ハブが**JSバンドルをキャッシュ**して古いまま動く（version表示だけ新しく中身が古い、が実際に起きた）。確実に反映するには **version を上げる＋旧アプリをアンインストール＋再起動**。確認用に**画面にバージョン文字列を表示**しておくと新旧判別が一瞬。

## 描画モデル（コンテナ）

座標系は原点左上・+x右・+y下、576×288、4bitグレースケール（緑16階調）。

| API | 用途 | 備考 |
|-----|------|------|
| `createStartUpPageContainer` | 起動時のページ生成 | **1回のみ**。戻り値 0=成功 |
| `rebuildPageContainer` | 画面遷移・再構築 | 全コンテナ破棄→再生成。**チラつく**。ナビ用 |
| `updateImageRawData` | 画像更新 | 画像コンテナにビットマップを流す |
| `textContainerUpgrade` | テキスト更新 | **フリッカー無し**・軽い。テキストは積極利用 |
| `shutDownPageContainer(mode)` | 終了 | ルート画面のダブルタップは `(1)` 必須（審査要件） |

- `containerTotalNum` は 1〜12。画像コンテナ最大4・テキスト最大8。
- **`isEventCapture=1` はページ内ちょうど1つ**（画像はイベント捕捉不可なのでテキストに持たせる）。
- コンテナの**描画順はID順（IDが大きいほど前面）**。重なる場合は注意（重ならない配置が無難）。

## 画像コンテナの実機制約（重要・実機確認）

- **幅の実上限は約144px**。docsは「幅20-288 / 高さ20-144」だが、**幅200は `result=1`（invalid）で拒否**された。72×72・144×64 は動作確認。**幅>144 は事実上不可**と考えるべき。
- `imageData` は **PNG/JPEGのバイト列（Uint8Array）でOK**。ホストがデコード→4bitグレースケール変換してくれる（自前で1bpp/gray4化は不要）。`canvas.toBlob('image/png')` のバイトをそのまま渡せる。
- 画像データは `createStartUpPageContainer` 時には送れない。**コンテナ生成後に `updateImageRawData`**。
- **送信は重複不可**。前の送信が解決してから次を送る（送信ロック必須）。
- 結果コード:
  - `createStartUpPageContainer`: `0`成功 / `1`無効な設定 / `2`オーバーサイズ / `3`メモリ不足。**createが0以外を返すとホストがアプリを終了させる**（＝起動直後に落ちる挙動になる）。
  - `updateImageRawData`: 文字列enum（`success` / `imageException` / `imageSizeInvalid` / `imageToGray4Failed` / `sendFailed`）。

## BLE帯域とフレームレート（実機計測）

**1フレームのデータ量 ≒ 画像コンテナのピクセル数**（ホストがgray4化してBLE送信するため）。面積が小さいほど速い。

| サイズ | おおよそのfps | 出典 |
|--------|--------------|------|
| 30×30 | 約9fps | bigdra記事 |
| 40×58 | 約4.5fps | ねこさんぽ計測 |
| 96〜120×58 | 約2〜2.8fps | ねこさんぽ計測 |
| 144×64 | 約1.4fps | ねこさんぽ計測 |

- 送信間隔(`setInterval`)を大きくすると**そこで頭打ち**になる。「送信ロック＋短いポーリング間隔」で“BLEが許す限り送る”のが最速。
- **静止中（同じ絵）は送らない**でBLEを節約（直前フレームのシグネチャ比較でスキップ）。

## アニメーションを滑らかにする設計（ハマりどころ）

- **グラス側でアニメは回せない**。プラグインができるのはコンテナ更新だけ＝**見た目の変化は毎回BLE送信**。これがfpsの上限。
- **時間ベースで座標を動かすとテレポートする**。BLEが遅いとプッシュ間隔の隙間で一気に進む。→ **「1プッシュ＝固定px進む」フレーム駆動**にすると小刻みで滑らか（速度はfps次第で落ちる）。
- **広い範囲を動かす手段は全部グリッチを伴う**:
  - コンテナを動かす（`rebuildPageContainer`）→ 移動の瞬間コンテナが空になり**スプライトが消える**。毎フレームrebuildは不安定/クラッシュ。
  - コンテナを複数並べる（タイル）→ スプライトが境界をまたぐと2コンテナを別々に更新するため**ズレる/裂ける**。
  - **結論：固定コンテナ（動かさない）が一番きれい**。スプライトサイズの最小コンテナにすると最速・最滑らか。広い徘徊が欲しい時は速度orグリッチとのトレードオフ。
- **実用解**: キャラサイズの小窓に固定し、**移動せずその場でアニメ**（足踏み・座る・ジャンプ等）、向き変えはランダム——にすると、最小データ＝最速＝グリッチ皆無で破綻しない。
- ターンの“つぶし”表現で**頭まで横につぶすと嘘くさい**。頭は同サイズのまま胴だけつぶすと自然。

## 入力イベント（実機確認）

- `onEvenHubEvent` で tap / double-tap / scroll / IMU / audio などが届く。
- **protobufのゼロ値省略の罠**: `CLICK`(0) やデフォルトの `eventSource` は **`undefined`** で届く。`?? 0` で吸収する。
- **誤発火対策**: IMU等の雑多な `sysEvent` も流れてくる。`eventType` が `undefined→0=CLICK` になるため全部タップ扱いすると暴発する。**本物のタップは `eventSource`（GLASSES_L/R=1/3, RING=2）が付いているものだけ**採用すると安定。
- ルート画面の**ダブルタップ → `shutDownPageContainer(1)`** は審査必須。

## app.json / パッケージング

- 必須: `package_id`, `edition`, `name`, `version`, `min_app_version`, `entrypoint`(`index.html`)。任意: `min_sdk_version`, `permissions`, `supported_languages`。
- `package_id` は**小文字英数のみ・ハイフン不可・逆ドメイン**（例 `com.necobit.necosanpo`）。
- 上記の通り **`version` を上げないとキャッシュで更新されない**。

## 参考リンク

- [Even Hub 公式ドキュメント](https://hub.evenrealities.com/docs)
- [Even Hub テンプレート (GitHub)](https://github.com/even-realities/evenhub-templates)
- [SDK機能検証記事 (Zenn)](https://zenn.dev/bigdra/articles/eveng2-sdk-features)
- [コミュニティ仕様メモ docs/ (nickustinov/even-g2-notes)](https://github.com/nickustinov/even-g2-notes) — display / error-codes / page-lifecycle / ui-patterns / packaging
- [G2 × OpenClawブリッジ実装例](https://blog.juchunko.com/en/even-realities-g2-openclaw-bridge/)
- [G2プロトコルリバースエンジニアリング (GitHub)](https://github.com/i-soxi/even-g2-protocol)
