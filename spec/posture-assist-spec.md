# 介護姿勢アシスト — テクニカル仕様書

**プロジェクト名:** Care Posture Assist  
**バージョン:** 0.1.0  
**対象読者:** フロントエンドエンジニア  
**最終更新:** 2026-03-08

---

## 0. 開発方針

### 0.1 タイムボックス

| フェーズ | 時間 | 目標 |
|---------|------|------|
| たたき台 | 〜30分 | 動くものを Cloudflare Pages にデプロイする |
| 改善ループ | 30分〜2時間 | フィードバックを重ね、最終版をデプロイする |

「完璧にしてからデプロイ」ではなく、**動くものを早く公開してフィードバックを得る**ことを最優先とする。
コードの品質よりも動作確認のサイクルを回すスピードを重視すること。

### 0.2 たたき台（〜30分）の基準

- カメラが起動し、MediaPipe が骨格を検出できる
- 体幹前傾角が HUD に数値表示される
- 閾値超過で Claude API が呼ばれ、結果テキストが画面に出る

音声・デザイン・エラーハンドリングの作り込みはこの段階では不要。

### 0.3 改善ループ（30分〜2時間）の進め方

実機（iPhone）で動かしながら気づいた問題を即座に修正し、都度デプロイする。
1サイクルは「修正 → `git push` → Cloudflare 自動デプロイ（〜1分） → 実機確認」。

---

## 1. プロジェクト概要

### 1.1 目的

ベッドから椅子への移乗介助において、介護者の腰部負荷リスクをリアルタイム検出し、
AI が具体的な改善指示を音声で提供する。モック段階では単一 HTML ファイルで完結させ、
開発・デプロイ・フィードバックサイクルを最速化することを優先する。

### 1.2 スコープ（v0.1）

| 対象 | 内容 |
|------|------|
| 場面 | ベッド腰掛け → 椅子への移乗 |
| 分析対象 | 介護者の体幹前傾角・頸部屈曲角 |
| 入力 | iPhone リアカメラ映像（リアルタイム） |
| 出力 | リスク評価テキスト ＋ 音声読み上げ |

### 1.3 スコープ外（v0.1）

- 被介護者の姿勢評価
- 複数人物の追跡
- セッション履歴の永続化
- 骨格ランドマークの可視化オプション設定

---

## 2. アーキテクチャ

### 2.1 全体構成

```
[iPhone Camera]
      │ getUserMedia (HTTPS必須)
      ▼
[<video> element]  ──────────────────────────────────────
      │                                                  │
      │ drawImage (~60fps)                               │ captureCanvas
      ▼                                                  │ (LLMトリガー時のみ)
[<canvas> overlay]                                       │
      │                                                  ▼
      │ detectForVideo (~15fps)              [captureCanvas → base64 JPEG]
      ▼                                                  │
[MediaPipe Pose Landmarker]                              │
      │                                                  │
      │ 33 landmarks                                     │
      ▼                                                  │
[角度計算エンジン]  ─── 閾値超過×N連続フレーム ──────▶ [Claude API call]
      │                                                  │
      │                                                  │ JSON response
      ▼                                                  ▼
[HUD リアルタイム表示]                     [BottomSheet 結果表示]
                                                         │
                                                         ▼
                                               [Web Speech API TTS]
```

### 2.2 デプロイ構成

```
Cloudflare Pages
  └── index.html   (静的ファイル 1本)
        ├── MediaPipe WASM  ← CDN (jsdelivr)
        ├── MediaPipe モデル ← CDN (storage.googleapis.com)
        └── Anthropic API   ← 直接呼び出し (anthropic-dangerous-direct-browser-access: true)
```

**API キーの扱い:**  
ブラウザのメモリにのみ保持し、外部に転送しない。
`localStorage` への保存は行わない（セキュリティリスク回避）。
本番運用時は Cloudflare Workers をプロキシとして挟み、キーをサーバーサイドに移行すること。

---

## 3. 使用技術

| カテゴリ | 技術 | バージョン | 用途 |
|---------|------|---------|------|
| 姿勢推定 | MediaPipe Pose Landmarker | 0.10.14 | 骨格33点検出 |
| AI 分析 | Claude claude-sonnet-4-20250514 | — | 姿勢評価・指示生成 |
| 音声合成 | Web Speech API (SpeechSynthesis) | ブラウザ標準 | 指示の読み上げ |
| カメラ | MediaDevices.getUserMedia | ブラウザ標準 | リアルタイム映像取得 |
| ホスティング | Cloudflare Pages | — | 静的ファイル配信 |
| フレームワーク | なし（Vanilla JS ES Modules） | — | — |

---

## 4. 姿勢推定仕様

### 4.1 使用ランドマーク

MediaPipe が出力する 33 点のうち、以下 6 点のみを使用する。

| Index | 名称 | 用途 |
|-------|------|------|
| 7 | left_ear | 頸部ベクトル計算 |
| 8 | right_ear | 頸部ベクトル計算 |
| 11 | left_shoulder | 体幹ベクトル計算 |
| 12 | right_shoulder | 体幹ベクトル計算 |
| 23 | left_hip | 体幹ベクトル計算 |
| 24 | right_hip | 体幹ベクトル計算 |

### 4.2 中点の定義

```
shoulder_mid = avg(lm[11], lm[12])   # 肩の中点
hip_mid      = avg(lm[23], lm[24])   # 腰の中点
ear_mid      = avg(lm[7],  lm[8])    # 耳の中点（≈ 首の付け根）
```

座標系: MediaPipe の正規化座標（0〜1）。y 軸は**下向き正**。

### 4.3 体幹前傾角の計算

```
trunk_vec = shoulder_mid - hip_mid
         = (tvx, tvy)

trunk_angle = arccos( -tvy / |trunk_vec| )   [rad → deg]
```

| 角度 | 解釈 |
|------|------|
| ≈ 0° | 直立 |
| 30°〜 | 要注意（黄色） |
| 45°〜 | 危険（赤） |

**符号について:** y 軸下向きのため直立時 tvy < 0。
`-tvy / mag` が +1 に近い → arccos ≈ 0°（直立）。
前傾すると tvy の絶対値が減少し角度増大。

### 4.4 頸部屈曲角の計算

```
neck_vec  = ear_mid - shoulder_mid
          = (nvx, nvy)

dot       = (nvx * tvx + nvy * tvy) / (|neck_vec| * |trunk_vec|)
neck_angle = arccos(clamp(dot, -1, 1))   [rad → deg]
```

体幹ベクトルを基準とした頸部の相対角度。0° = 頸部と体幹が一直線。

### 4.5 Visibility フィルタ

```
コア4点（11,12,23,24）の visibility > 0.45 を全て満たす場合のみ計算
耳2点（7,8）は visibility > 0.30 のみ頸部角を計算、未満は null
```

### 4.6 検出レート

```
detectForVideo の呼び出し間隔: 66ms（≈15fps）
requestAnimationFrame によるキャンバス描画: ≈60fps
```

MediaPipe は前回の推論結果をキャッシュするため、描画 fps と検出 fps は独立して制御可能。

---

## 5. LLM トリガー仕様

### 5.1 自動トリガー条件（AND 条件）

```
1. trunk_angle >= DANGER_ANGLE（デフォルト 45°）
2. 上記が consecutive_high_frames >= 18 フレーム連続
3. 前回 LLM 呼び出しから llm_cooldown >= 15,000ms 経過
4. isAnalyzing === false（前回の API 呼び出しが完了済み）
```

18 フレーム × 66ms ≈ **1.2秒** 継続で初めてトリガー。
瞬間的な前傾姿勢（歩行動作など）による誤検出を防ぐ。

### 5.2 連続フレームカウントのリセット

```
trunk_angle < WARN_ANGLE（デフォルト 30°）になると -1/frame でデクリメント
trunk_angle >= WARN_ANGLE かつ < DANGER_ANGLE では現状維持（カウント凍結）
```

### 5.3 手動トリガー

HUD の「分析」ボタンで即時実行。ただし isAnalyzing === true の場合は無効。

---

## 6. Claude API 仕様

### 6.1 エンドポイント

```
POST https://api.anthropic.com/v1/messages
```

### 6.2 リクエストヘッダー

```http
Content-Type: application/json
x-api-key: {API_KEY}
anthropic-version: 2023-06-01
anthropic-dangerous-direct-browser-access: true
```

### 6.3 モデル・パラメータ

```json
{
  "model": "claude-sonnet-4-20250514",
  "max_tokens": 600
}
```

### 6.4 送信コンテンツ

1つのメッセージに2つのコンテンツブロックを含める。

```json
{
  "role": "user",
  "content": [
    {
      "type": "image",
      "source": {
        "type": "base64",
        "media_type": "image/jpeg",
        "data": "{captureCanvas の JPEG base64 (quality: 0.75)}"
      }
    },
    {
      "type": "text",
      "text": "{プロンプト（後述）}"
    }
  ]
}
```

**フレームキャプチャ:** 骨格オーバーレイを描画していない `captureCanvas`（`<video>` から直接 drawImage）を使用し、クリーンな映像を送信する。

### 6.5 システムプロンプト設計

```
状況コンテキスト:   ベッドから椅子への移乗介助場面
計測値の提供:       体幹前傾角・頸部角の数値を文字列で渡す
出力形式の指定:     JSONのみ（マークダウン・前置き文不要）
TTS 適合の指示:     体言止め禁止、完結した文で出力させる
```

### 6.6 レスポンス JSON スキーマ

```typescript
interface PostureAnalysis {
  risk:         "高" | "中" | "低";
  summary:      string;       // 1〜2文の状況説明
  instructions: string[];     // 2〜4項目の具体的指示
}
```

### 6.7 JSONパース

レスポンスの `content[].text` を結合し、正規表現 `/\{[\s\S]*\}/` で JSON 部分を抽出してパース。
API がマークダウンコードブロックを付加した場合も対応可能。

---

## 7. 音声合成仕様

### 7.1 読み上げテキスト

```
{result.summary} {result.instructions.join('。')}。
```

### 7.2 パラメータ

```javascript
utterance.lang  = 'ja-JP'
utterance.rate  = 0.92
utterance.pitch = 1.0
utterance.voice = // speechSynthesis.getVoices() から lang='ja' のボイスを優先選択
```

### 7.3 トリガータイミング

LLM レスポンス受信 → BottomSheet 表示 → **400ms 後に自動再生開始**  
（BottomSheet のアニメーション完了を待つため）

---

## 8. UI コンポーネント

### 8.1 画面構成

```
┌─────────────────────────────┐
│ [体幹角ゲージ]  [頸部角バッジ] │  ← HUD Top
│                             │
│      <canvas> カメラ映像     │
│  + MediaPipe 骨格オーバーレイ │
│                             │
│     [検出ステータスPill]      │
│                             │
│      [トリガープログレスバー]  │  ← 閾値超過時のみ表示
│                             │
│ [⚙設定]  [分析ボタン]  [✕]  │  ← HUD Bottom
└─────────────────────────────┘
         ↕ スライドアップ
┌─────────────────────────────┐
│ BottomSheet                 │
│  ├── [リスクバッジ]  [角度値] │
│  ├── サマリー               │
│  ├── 指示リスト（1〜4項）    │
│  └── [🔊読み上げ] [閉じる]  │
└─────────────────────────────┘
```

### 8.2 リスク色定義

| レベル | 色 | HEX |
|--------|-----|-----|
| 良好 | セーフグリーン | `#00c896` |
| 要注意 | アンバー | `#f59e0b` |
| 危険 | レッド | `#ef4444` |

### 8.3 体幹角アークゲージ

SVG パスによる円弧表示。`stroke-dashoffset` を角度に比例して制御。

```
arc_fill_ratio  = min(trunk_angle / 90°, 1.0)
dash_offset     = ARC_LEN × (1 - arc_fill_ratio)
ARC_LEN         = 160  （SVGパスの総長に合わせた定数）
```

---

## 9. エラーハンドリング

| エラー種別 | 対処 |
|-----------|------|
| カメラ許可拒否 | ローディング画面にエラーメッセージ表示 |
| MediaPipe ロード失敗 | ローディング画面にエラーメッセージ表示 |
| Visibility 不足 | 角度計算をスキップ、ステータスPillに「検出不可」 |
| Claude API エラー | BottomSheet を閉じ、エラートースト表示（5秒） |
| JSON パース失敗 | 上記に同じ |

---

## 10. Cloudflare Pages デプロイ手順

### 10.1 前提

- Cloudflare アカウント
- HTTPS ドメイン（カメラ API 必須要件）
- Anthropic API キー

### 10.2 デプロイ手順

```bash
# 1. ファイルをリネーム
mv posture-assist.html index.html

# 2. Cloudflare Pages にプッシュ
#    Dashboard → Pages → Create project → Direct Upload
#    index.html を含むフォルダをアップロード

# または GitHub 連携でブランチにプッシュするだけで自動デプロイ
git push origin main
```

### 10.3 iPhone アクセス時の注意点

1. Safari で Cloudflare Pages の URL にアクセス
2. 「このサイトにカメラへのアクセスを許可しますか？」→ 許可
3. カメラは `facingMode: { ideal: 'environment' }` でリアカメラ優先起動
4. API キーをセットアップ画面で入力（セッション中のみ保持）

---

## 11. 既知の制約と今後の課題

### 11.1 v0.1 の制約

| 制約 | 内容 |
|------|------|
| API キー管理 | ブラウザ入力のため、共有端末での運用には不向き |
| 照明依存 | 暗所では MediaPipe の検出精度が低下 |
| 横画面対応 | レイアウト未調整 |
| オフライン非対応 | MediaPipe モデルの CDN 取得が必要 |

### 11.2 v0.2 以降の拡張候補

```
優先度 高:
  - Cloudflare Workers によるAPIキーのサーバーサイド管理
  - 縦向き固定・横向きレイアウト対応
  - 検出信頼度インジケータの表示

優先度 中:
  - 分析履歴のセッション内保持（時系列グラフ）
  - 被介護者の姿勢評価（腰・膝の屈曲）
  - 複数フレームの連続スナップショット送信

優先度 低:
  - PWA 化（ホーム画面追加、オフラインキャッシュ）
  - MediaPipe モデルのローカルバンドル
  - 動画ファイルのオフライン解析モード
```

---

## 12. ファイル構成

```
index.html          # アプリ全体（単一ファイル）
  ├── <style>       # CSS（CSS Variables ベース、ダークテーマ）
  ├── <body>        # DOM（Setup / App / BottomSheet / Loading）
  └── <script type="module">
        ├── import  MediaPipe Tasks Vision（CDN）
        ├── init()              # MediaPipe 初期化
        ├── onStart()           # カメラ起動
        ├── renderLoop()        # rAF ループ（描画 + 検出）
        ├── calcAngles(lm)      # 角度計算
        ├── updateHUD(angles)   # HUD 更新 + トリガー判定
        ├── triggerLLM(manual)  # キャプチャ + API 呼び出し
        ├── callClaude(frame, angleInfo)  # Claude API
        ├── renderResult(r)     # BottomSheet 描画
        └── TTS 関連            # speak / handleTTS / updateTTSBtn
```

---

*このドキュメントは v0.1 実装に対応しています。*  
*仕様変更時は本ドキュメントを先に更新し、実装に反映すること。*
