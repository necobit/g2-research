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

## 参考リンク

- [Even Hub 公式ドキュメント](https://hub.evenrealities.com/docs)
- [Even Hub テンプレート (GitHub)](https://github.com/even-realities/evenhub-templates)
- [SDK機能検証記事 (Zenn)](https://zenn.dev/bigdra/articles/eveng2-sdk-features)
- [G2 × OpenClawブリッジ実装例](https://blog.juchunko.com/en/even-realities-g2-openclaw-bridge/)
- [G2プロトコルリバースエンジニアリング (GitHub)](https://github.com/i-soxi/even-g2-protocol)
