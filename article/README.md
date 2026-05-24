# article/ — Qiita記事ソース（qiita-cli運用）

このフォルダは [qiita-cli](https://github.com/increments/qiita-cli) で管理する Qiita 記事のソース置き場。

## セットアップ手順（初回のみ）

```bash
cd /Users/suzukimasato/Project/Arduino/ble-lab/article

# 1. qiita-cli プロジェクトとして初期化
npx @qiita/qiita-cli@latest init

# 2. トークン設定（bad-usb で既にログイン済みならスキップ可）
npx qiita login

# 3. プレビューサーバー起動（任意）
npx qiita preview
# → http://localhost:8888 で記事を確認
```

## 記事を書く

```bash
# 既存ドラフトをプレビューで確認
npx qiita preview

# Qiita に下書きとして同期（private: true のまま）
npx qiita publish 01-esp32s3-applejuice-core3-trap
```

## 公開予定 3部作

| # | ファイル | 状態 |
|---|---|---|
| ① | [public/01-esp32s3-applejuice-core3-trap.md](public/01-esp32s3-applejuice-core3-trap.md) | draft |
| ② | [public/02-ble-scanner-home-bledevices.md](public/02-ble-scanner-home-bledevices.md) | draft |
| ③ | [public/03-apple-continuity-iphone-state-leak.md](public/03-apple-continuity-iphone-state-leak.md) | draft |
