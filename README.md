# ble-lab

ESP32-S3-DevKitC-1 を母艦にした BLE セキュリティ検証ラボ。

## 構成

- **ハード**: ESP32-S3-DevKitC-1-N8R8（技適 201-220052、秋月電子）
- **ホスト**: MacBook + Arduino IDE 2.3.8
- **ESP32 Arduino Core**: **2.0.17**（3.x系は ESP32-S3 で NimBLE に切り替わり既存BLEスケッチが軒並み動かないため、意図的に固定）
- **シリアルポート**: `/dev/cu.usbserial-140`（CP2102N、macOS標準ドライバで認識）

## 内容物

| ディレクトリ | 内容 |
|---|---|
| [article/](article/) | Qiita 記事ソース（qiita-cli 運用） |
| [sketches/](sketches/) | 検証スケッチ（実体は `~/Documents/Arduino/` 配下） |
| [photos/](photos/) | 記事用スクショ・写真 |

## 検証済みスケッチ

1. **AppleJuice**（[ECTO-1A/AppleJuice](https://github.com/ECTO-1A/AppleJuice)） — 単一機種の Apple BLE Spoofing
2. **EvilAppleJuice-ESP32**（[ckcr4lyf/EvilAppleJuice-ESP32](https://github.com/ckcr4lyf/EvilAppleJuice-ESP32)） — 29機種ローテーション + MAC 自動ランダム化
3. **BLEScanner**（自作） — 周辺 BLE デバイス可視化・Apple機器種別判定
4. **ContinuityAnalyzer**（自作） — Apple Continuity プロトコル解析・iPhone 状態漏洩可視化

## 倫理ライン

- **対象は自宅・自分の所有デバイスのみ**
- 公共の場・他人のデバイスに対する送信は実行しない
- 2.4GHz ジャミングは電波法違反、絶対やらない
- 技適なしハードでの送信も違法（屋内・自宅でもアウト）

## 関連

- [bad-usb](../bad-usb/) — Adafruit Trinkey QT2040 による HID 注入ラボ（BLE ラボの有線版）
