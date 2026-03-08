# Care Posture Assist

介護者の姿勢をリアルタイム解析し、AI が改善アドバイスを音声で提供する Web アプリケーションです。

## 機能

- iPhone / スマートフォンのカメラで介護者の姿勢をリアルタイム検出（MediaPipe Pose Landmarker）
- 体幹前傾角・頸部屈曲角を HUD に表示
- 危険姿勢が一定時間続くと自動で Claude API に画像を送信し、改善指示を取得
- 結果をボトムシートに表示し、音声で読み上げ

## 必要なもの

- HTTPS でアクセスできるホスティング環境（カメラ API に必須）
- Anthropic API キー（[console.anthropic.com](https://console.anthropic.com/) で取得）

## ローカル開発

HTTPS が必要なため、ローカルサーバーは HTTPS 対応のものを使用してください。

```bash
# 方法 1: npx で簡易 HTTPS サーバーを起動
npx http-server . --ssl -p 8443

# 方法 2: Python の場合（自己署名証明書が必要）
python -m http.server 8080
# ※ localhost は一部ブラウザで HTTPS 不要で動作します

# 方法 3: VS Code の Live Server 拡張機能を使用
```

ブラウザで `https://localhost:8443`（または該当ポート）にアクセスします。

## Cloudflare Pages へのデプロイ

### GitHub 連携（推奨）

1. このリポジトリを GitHub にプッシュ
2. [Cloudflare Dashboard](https://dash.cloudflare.com/) → Pages → Create a project
3. GitHub リポジトリを連携
4. ビルド設定:
   - **Framework preset:** None
   - **Build command:** (空欄)
   - **Build output directory:** `.`（ルート）
5. 「Save and Deploy」

以降は `git push origin main` で自動デプロイされます（約1分）。

### ダイレクトアップロード

1. Cloudflare Dashboard → Pages → Create a project → Direct Upload
2. `index.html` を含むフォルダをアップロード

## iPhone での使い方

1. Safari で Cloudflare Pages の URL にアクセス
2. Anthropic API キーを入力して「開始する」をタップ
3. カメラへのアクセスを許可（リアカメラが優先起動）
4. 介護者の全身が映るようにスマートフォンを設置
5. 体幹前傾角が 45° 以上の状態が約 1.2 秒続くと自動分析
6. 手動で分析したい場合は「分析」ボタンをタップ

## 閾値

| 角度 | 状態 | 色 |
|------|------|-----|
| 0°〜30° | 良好 | 緑 |
| 30°〜45° | 要注意 | 黄 |
| 45°〜 | 危険 | 赤 |

## 注意事項

- API キーはブラウザのメモリにのみ保持され、外部に保存されません
- 本番運用時は Cloudflare Workers をプロキシとして API キーをサーバーサイドに移行してください
- 暗所では骨格検出の精度が低下します

## 技術スタック

- MediaPipe Pose Landmarker v0.10.14（CDN）
- Claude claude-sonnet-4-20250514（Anthropic API）
- Web Speech API（音声読み上げ）
- Vanilla JS（フレームワークなし、単一 HTML ファイル）
