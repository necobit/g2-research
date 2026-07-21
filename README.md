# Even Realities G2 調査まとめ

調査日: 2026-06-19 / 最終更新: 2026-07-21（STTはEvenクラウド処理と判明、オンデバイスSTT記述を訂正）

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
  └→ 音声をBLEでスマホへ → EvenクラウドでSTT（テキスト化）
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

## STTはEvenクラウド処理（オンデバイスではない）

**訂正（2026-07-21）**: 以前「音声→テキスト変換がデバイス内で完結」と記載していたが誤り。

- 出典を追跡した結果、[OpenClawブリッジ記事](https://blog.juchunko.com/en/even-realities-g2-openclaw-bridge/)の「Voice → text happens on-device — G2 sends transcribed text, not audio」という記述が根拠だった。しかしこれは「カスタムエンドポイントに音声ではなくテキストが届く」という観測からの筆者の推測で、エンドポイント側からはグラス内STTかクラウドSTTかは区別できない。
- **実機実験（2026-07-21）**: スマホのWi-Fiを切ると標準搭載の会話サポート（Conversate）が起動しなくなることを確認。
- **公式ドキュメントで裏付け**: [Conversateサポート記事](https://support.evenrealities.com/hc/en-us/articles/14273795154319-Conversate)に「スマホ経由のインターネット接続が必要。オフラインでは利用不可」と明記。[公式ブログ](https://www.evenrealities.com/blogs/even-insider/context-without-compromise)でも文字起こしは「Evenのセキュアなクラウド」で処理し、音声は保存せずテキストのみスマホに残すと説明。

### 結論

- **オンデバイス（グラス内）なのは「Hey Even」のウェイクワード検出のみ**
- STTの流れ: グラスで集音 → BLEでスマホ → **EvenクラウドでSTT** → テキスト化
- したがってカスタムエージェント連携もSTT部分でネットワーク必須（エンドポイントへのPOSTも当然必要）
- Even App自体やBLE完結の機能（ダッシュボード・テレプロンプター・プラグイン描画等）はオフラインでも動作する
- SDKのuseSTTフック（外部STTプロバイダ利用）はこれとは別系統

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

- **幅の実上限は約144px**。docsは「幅20-288 / 高さ20-144」だが、**幅200は `result=1`（invalid）で拒否**された。72×72・144×64 は動作確認。**幅>144 は事実上不可**と考えるべき。画像コンテナは**1ページ最大4枚**なので、144×144を横に4枚並べた **576×144 の全幅帯が最大**（g2-horizontal-lineで実機確認）。
- `imageData` の形式は **SDKバージョンで異なる（重要・実機確認）**:
  - **〜0.0.11**: PNG/JPEGのバイト列（Uint8Array）。ホストがデコード→gray4変換。`canvas.toBlob('image/png')` のバイトをそのまま渡せる。
  - **0.0.12〜**: **生の4bitグレースケール**（2px/バイト、上位ニブル=左px、行優先、`width*height/2` バイト。幅は偶数にする）。転送時はSDKがLZ4圧縮する。**0.0.12にPNGを送ると `imageSizeInvalid` になるか、グラスにノイズが描画される**。RGBA→gray4は自前で変換（輝度 `0.299R+0.587G+0.114B` を16で割って0-15に丸め）。
  - 生gray4化で**フレームレートが大幅に向上**した（下表）。16階調を直接制御できるので、薄いグレーの補助線などもきれいに出せる。
- 画像データは `createStartUpPageContainer` 時には送れない。**コンテナ生成後に `updateImageRawData`**。
- **送信は重複不可**。前の送信が解決してから次を送る（送信ロック必須）。
- 結果コード:
  - `createStartUpPageContainer`: `0`成功 / `1`無効な設定 / `2`オーバーサイズ / `3`メモリ不足。**createが0以外を返すとホストがアプリを終了させる**（＝起動直後に落ちる挙動になる）。
  - `updateImageRawData`: 文字列enum（`success` / `imageException` / `imageSizeInvalid` / `imageToGray4Failed` / `sendFailed`）。

## BLE帯域とフレームレート（実機計測）

**1フレームのデータ量 ≒ 画像コンテナのピクセル数**（ホストがgray4化してBLE送信するため）。面積が小さいほど速い。

| サイズ | 画像形式 | おおよそのfps | 出典 |
|--------|---------|--------------|------|
| 30×30 | PNG (〜0.0.11) | 約9fps | bigdra記事 |
| 40×58 | PNG (〜0.0.11) | 約4.5fps | ねこさんぽ計測 |
| 96〜120×58 | PNG (〜0.0.11) | 約2〜2.8fps | ねこさんぽ計測 |
| 144×64 | PNG (〜0.0.11) | 約1.4fps | ねこさんぽ計測 |
| **144×58** | **生gray4 (0.0.12)** | **約7〜9fps** | ねこさんぽ計測 |

- **0.0.12の生gray4形式で同面積のfpsが約5倍**になった（144×58で1.4→7-9fps、ブリッジ往復は約110-200ms/フレーム）。PNGエンコード/デコードが消えたぶんと思われる。
- 送信間隔(`setInterval`)を大きくすると**そこで頭打ち**になる。タイマー駆動よりも**送信のawait自体をペースメーカーにする自走ループ**が最速（後述のロック対策も兼ねる）。
- **静止中（同じ絵）は送らない**でBLEを節約（直前フレームのシグネチャ比較でスキップ）。

## スマホ画面ロック対策（実機確認・重要）

iPhoneの画面をロックすると描画が2fps程度まで落ちる問題の調査結果。

- **原因はJSタイマーのスロットリング**。ロック中（WebView非表示中）、iOSは `setInterval`/`setTimeout` を**約1Hzに間引く**。一方で**ブリッジ呼び出し（`updateImageRawData` 等）の往復速度はロック中も変わらない**（実測110-220msのまま）。BLEは遅くなっていない。
- **効かなかった対策**（実機で全滅を確認）:
  - Screen Wake Lock … アプリ前面時の画面消灯防止にしかならない
  - 無音オシレータ（WebAudio）でのオーディオセッション維持 … **ロックの瞬間にAudioContextが `interrupted` にされる**。ロック中はresumeも不可（iOSはバックグラウンドでのオーディオセッション開始を許さない）
  - ちなみにホストWebViewでは **DOMのジェスチャーイベント（pointerdown/touchstart/mousedown/click）が一切発火しない**（実機確認）。AudioContextの自動再生は許可されているのでジェスチャー無しで `running` にできるが、上記の通りロックで殺される
- **効いた対策: タイマーを使わない自走ループ**。`while (active) { await push(); }` の形にして、**各フレームのブリッジ往復（〜130ms）自体をペースメーカー**にする。タイマーに依存しないのでスロットリングの影響を受けず、**ロック中も7fps以上を維持**できた。送るものが無いとき（静止中）だけ短い `setTimeout` を挟む（ロック中は〜1秒に伸びるが、静止中なので実害なし）。
- 補助: **キャッチアップ補正**。ステップ駆動（1プッシュ=固定px）だと低fps時にキャラがスローモーションになるので、プッシュ間隔が開いたら複数ステップまとめて進める（上限付き）と見かけの速度が一定になる。

## アニメーションを滑らかにする設計（ハマりどころ）

- **グラス側でアニメは回せない**。プラグインができるのはコンテナ更新だけ＝**見た目の変化は毎回BLE送信**。これがfpsの上限。
- **時間ベースで座標を動かすとテレポートする**。BLEが遅いとプッシュ間隔の隙間で一気に進む。→ **「1プッシュ＝固定px進む」フレーム駆動**にすると小刻みで滑らか（速度はfps次第で落ちる）。
- **広い範囲を動かす手段は全部グリッチを伴う**:
  - コンテナを動かす（`rebuildPageContainer`）→ 移動の瞬間コンテナが空になり**スプライトが消える**。毎フレームrebuildは不安定/クラッシュ。
  - コンテナを複数並べる（タイル）→ スプライトが境界をまたぐと2コンテナを別々に更新するため**ズレる/裂ける**。
  - **結論：固定コンテナ（動かさない）が一番きれい**。スプライトサイズの最小コンテナにすると最速・最滑らか。広い徘徊が欲しい時は速度orグリッチとのトレードオフ。
- **実用解**: 固定コンテナの中でキャラを動かす。PNG時代（〜0.0.11）は帯域が足りずキャラサイズの小窓＋その場アニメが限界だったが、**0.0.12の生gray4なら幅上限いっぱいの144px固定窓で7fps以上出る**ので、窓の中を実際に歩き回らせられる（ねこさんぽ実機確認）。窓は動かさないのでグリッチ皆無。地面ライン等の静的な目印を窓内に描くと相対移動が見やすい。
- ターンの“つぶし”表現で**頭まで横につぶすと嘘くさい**。頭は同サイズのまま胴だけつぶすと自然。

## 入力イベント（実機確認）

- `onEvenHubEvent` で tap / double-tap / scroll / IMU / audio などが届く。
- **protobufのゼロ値省略の罠**: `CLICK`(0) やデフォルトの `eventSource` は **`undefined`** で届く。`?? 0` で吸収する。
- **誤発火対策**: IMU等の雑多な `sysEvent` も流れてくる。`eventType` が `undefined→0=CLICK` になるため全部タップ扱いすると暴発する。**本物のタップは `eventSource`（GLASSES_L/R=1/3, RING=2）が付いているものだけ**採用すると安定。
- ルート画面の**ダブルタップ → `shutDownPageContainer(1)`** は審査必須。

## app.json / パッケージング

- 必須: `package_id`, `edition`, `name`, `version`, `min_app_version`, `entrypoint`(`index.html`)。任意: `min_sdk_version`, `permissions`, `supported_languages`。
- **生gray4形式（0.0.12ネイティブ）を使うアプリは `min_sdk_version: "0.0.12"` にする**こと（旧ホストにraw gray4を送ると壊れた絵になるため）。依存も `"@evenrealities/even_hub_sdk": "0.0.12"` と厳密固定が安全。
- `package_id` は**小文字英数のみ・ハイフン不可・逆ドメイン**（例 `com.necobit.necosanpo`）。
- 上記の通り **`version` を上げないとキャッシュで更新されない**。

## ストア公開まわり（ねこさんぽ公開作業で確認）

- **アプリの同一性は `package_id`**。表示名（app.jsonの`name`）はビルド毎に自由に変えられる。package_idを変えると別アプリ扱い（いいね・既存ユーザーがリセット）なので変えないこと。
- **アイコン**: 24×24 px モノクロPNG（公式デザインガイドライン、Figma公開ファイルに記載）。単色＋透過で作る。512pxなどの大きいアイコン枠は無い。
- **ストアのスクリーンショット**: **576×288 px の透過PNG**（＝ディスプレイのフレームバッファそのもの）。背景の部屋写真はポータルが合成する（Environment: Home/Office/Store/Cafe、Interior/Exterior切替）。**黒背景で上げると背景を覆ってしまうので透過必須**。「光っていない部分＝透明、輝度＝緑のアルファ」にすると実機（加算表示）に忠実。even-g2-cat の `/promo.html` に生成機能あり（`?shot=N`でヘッドレス出力も可）。カバー画像枠は別途あり（サイズ自由っぽい）。
- **ストア掲載フォーム**: Category / 説明 / タグ（英語で cat, pet, kawaii等）/ Permissionsチェックリスト（Mic/Location/Push/Local network/Bluetooth/Background services — プラグインが自分で使うものだけ。グラスとのBLEはホストの仕事なのでBluetoothは不要）/ Privacy and terms（自由記述）。Changelogはビルド毎に500字。
- **PCシミュレータ**: 公式には無し（SDKはEven App WebView必須）。コミュニティ製の [BxNxM/even-dev](https://github.com/BxNxM/even-dev)（Even Hub Simulator）が存在する（未検証）。

## 参考リンク

- [Even Hub 公式ドキュメント](https://hub.evenrealities.com/docs)
- [Even Hub テンプレート (GitHub)](https://github.com/even-realities/evenhub-templates)
- [SDK機能検証記事 (Zenn)](https://zenn.dev/bigdra/articles/eveng2-sdk-features)
- [コミュニティ仕様メモ docs/ (nickustinov/even-g2-notes)](https://github.com/nickustinov/even-g2-notes) — display / error-codes / page-lifecycle / ui-patterns / packaging
- [G2 × OpenClawブリッジ実装例](https://blog.juchunko.com/en/even-realities-g2-openclaw-bridge/)
- [G2プロトコルリバースエンジニアリング (GitHub)](https://github.com/i-soxi/even-g2-protocol)
