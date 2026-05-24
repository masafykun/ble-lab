---
title: >-
  ESP32-S3 で AppleJuice / BLE Spam を動かすとき Arduino Core 3.x で詰む話と Core 2.0.17
  への戻し方
tags:
  - Arduino
  - Security
  - bluetooth
  - BLE
  - ESP32
private: true
updated_at: '2026-05-24T23:44:00+09:00'
id: e50254ad76c801422a29
organization_url_name: null
slide: false
ignorePublish: false
---

> Arduino Core 3.x で `setDeviceAddress` が消えたと思ったら、BLE スタックが NimBLE に切り替わっていた。Core 2.0.17 に戻したら全部解決した個人検証メモ。

ESP32-S3-DevKitC-1 で Apple BLE Continuity の送信側（AppleJuice / EvilAppleJuice 系）を動かそうとしたら想像以上にハマった。
原因は ESP32 Arduino Core 3.x が ESP32-S3 のBLEスタックを **Bluedroid → NimBLE** にデフォルト切替したことで、既存の Bluedroid 前提スケッチが軒並みコンパイル不能になっていたこと。
最終的に Core 2.0.17 へダウングレードして解決するまでの作業ログを残しておきます。

BLE 検証ラボシリーズの **1 本目**（→ ②BLE Scanner、③Continuity 解析と続く予定）。

## TL;DR

- ESP32-S3-DevKitC-1 で BLE Spam（AppleJuice / EvilAppleJuice）を動かそうとすると、現行 Arduino ESP32 Core 3.x **だと環境次第で詰む**
- 原因は ESP32-S3 では Core 3.x からデフォルトBLEスタックが **Bluedroid → NimBLE** に切り替わり、既存スケッチが使う API が消失するため
- 一番楽な対処は **ESP32 Arduino Core を 2.0.17 にダウングレード**
- ダウングレードすれば EvilAppleJuice-ESP32 がほぼ無改造で動き、29機種ローテーション + MAC 自動ランダム化により iOS 17.2+ で導入された対策の効きを実機で観察できる

## ✨ この記事で書くこと

- **問題の切り分け手順** — エラーメッセージ → BLE ヘッダ → `sdkconfig.h` まで追って NimBLE 切替を突き止める流れ
- **再現性のある解決手順** — Arduino IDE のボードマネージャで 2.0.17 にダウングレードするだけ
- **応急修理パッチ** — AppleJuice を Core 3.x で無理やり動かす2箇所のパッチも併記
- **ハマり遍歴をすべて残す** — 失敗ルートも書いておくと同じ罠を踏む人の検索に引っかかる

## ⚠️ 倫理ライン（毎回意識する）

- ✅ 自分が所有する ESP32-S3 + 自分の iPhone のみが検証対象
- ✅ 技適番号 201-220052 付きの正規品を使用、屋内で短時間のみ送信
- ✅ Qiita 記事は「個人検証ノート」のスタンス（攻撃ペイロード集ではない）
- ❌ 他人のデバイス・公共の場・カフェ等での実行は **絶対 NG**（電波法・各種法令違反）
- ❌ 技適マークなしの並行輸入 ESP32 は屋内でもアウト

迷ったら「これは記事で公開して恥ずかしくないか？」で判断します。

## やりたかったこと

セキュリティ学習目的で、Apple BLE Continuity プロトコルの挙動を自分の iPhone で観察したい。具体的には：

- 「AirPods を接続しますか？」のペアリングポップアップを意図的に出す
- iOS 17.2+ で導入された連続スパム対策の効きぐあいを確認
- 後続の Continuity 解析の前段として、送信側を理解する

## 用意したもの

| 項目 | 内容 | 価格 |
|---|---|---|
| メイン基板 | ESP32-S3-DevKitC-1-N8R8（秋月電子） | 2,380円 |
| 技適番号 | **201-220052**（基板上のシールド缶に刻印） | — |
| USB-C ケーブル | データ通信対応のもの | 手持ち |
| ホスト | MacBook（Apple Silicon） + Arduino IDE 2.3.8 | — |
| 検証ターゲット | 自分のメイン iPhone（iOS 最新） | — |

技適マークなしの並行輸入品 ESP32 は、自宅・屋内であっても電波を出した時点で電波法違反になります。Amazon の安いやつには技適なしが混ざっているので、**秋月や Switch Science などの国内代理店経由**が安心。

## デバイス認識確認

USB-C ケーブルを **「UART」と書かれている方** のポートに挿します（ESP32-S3-DevKitC-1 には USB-C が2つあるので注意）。

```bash
$ ioreg -p IOUSB -l -w 0 | grep -A 3 "CP2102"
"USB Product Name" = "CP2102N USB to UART Bridge Controller"
"USB Vendor Name" = "Silicon Labs"
```

macOS Sonoma 以降は標準ドライバで認識されるので、Silicon Labs の CP2102N ドライバを別途入れる必要はありません。シリアルポートは `/dev/cu.usbserial-XXX` の形で見えます。

## Arduino IDE ボード設定

`Tools` メニューから：

| 項目 | 値 |
|---|---|
| Board | **ESP32S3 Dev Module** |
| Port | `/dev/cu.usbserial-140`（環境による） |
| USB CDC On Boot | Disabled |
| Flash Size | 8MB (64Mb) |
| Partition Scheme | Default 4MB with spiffs |
| PSRAM | **OPI PSRAM** |
| Upload Speed | 921600 |

## 第一の罠: AppleJuice が Core 3.x でコンパイル不能

[ECTO-1A/AppleJuice](https://github.com/ECTO-1A/AppleJuice) の `ESP32-Arduino/applejuice/applejuice.ino` を Arduino IDE で開いてコンパイル → 即エラー：

```
error: 'ADV_TYPE_IND' was not declared in this scope
error: no matching function for call to 'BLEAdvertisementData::addData(std::string)'
```

これは ESP32 Arduino Core 3.x で BLE ライブラリの API シグネチャが変わっているのが原因。

```cpp
// 旧 (Core 2.x)
oAdvertisementData.addData(std::string((char*)data, sizeof(dataAirpods)));

// 新 (Core 3.x)
oAdvertisementData.addData((char*)data, sizeof(dataAirpods));  // 2引数版
```

`ADV_TYPE_IND` も Core 3.x では `setAdvertisementType()` が `uint8_t` を直接受け取るように変更されているので、数値を直接渡せば動きます。

応急修理して動くようにはなりましたが、`AppleJuice` は1機種固定送信なのでスパム性能が低い。本命の **EvilAppleJuice-ESP32** に乗り換えようとしたら、ここで第二の壁にぶつかりました。

## 第二の罠: EvilAppleJuice が ESP32-S3 で API ごと消滅

[ckcr4lyf/EvilAppleJuice-ESP32](https://github.com/ckcr4lyf/EvilAppleJuice-ESP32) は MAC 自動ローテーション + 29機種ランダム送信を実装した OSS の参考実装。これをコンパイルすると：

```
error: 'class BLEAdvertising' has no member named 'setDeviceAddress'
error: 'BLE_ADDR_TYPE_RANDOM' was not declared in this scope; did you mean 'BLE_ADDR_RANDOM'?
error: 'ADV_TYPE_IND' was not declared in this scope
```

`setDeviceAddress` ごと消えている。これはおかしい。BLE ライブラリのヘッダを覗くと：

```cpp
// libraries/BLE/src/BLEAdvertising.h
#if defined(CONFIG_BLUEDROID_ENABLED)
  void setPrivateAddress(esp_ble_addr_type_t type = BLE_ADDR_TYPE_RANDOM);
  bool setDeviceAddress(esp_bd_addr_t addr, esp_ble_addr_type_t type = BLE_ADDR_TYPE_RANDOM);
#endif

#if defined(CONFIG_NIMBLE_ENABLED)
  void setName(String name);
  // ... setDeviceAddress が無い
#endif
```

**Bluedroid と NimBLE で完全に別 API に分岐している。** そして ESP32-S3 用の `sdkconfig.h` を確認すると：

```bash
$ grep -E "BT_BLUEDROID|BT_NIMBLE" \
  ~/Library/Arduino15/packages/esp32/tools/esp32s3-libs/3.3.8/opi_opi/include/sdkconfig.h
#define CONFIG_BT_NIMBLE_ENABLED 1
```

つまり **ESP32 Core 3.x では ESP32-S3 のBLEスタックが NimBLE に強制切り替えされている**。Bluedroid 専用 API を使っているスケッチは原理的に動かない。

PSRAM 設定（qio_qspi / dio_opi / opi_opi 等）を全部チェックしましたが、**私の検証範囲では ESP32-S3 のどの組み合わせでも NimBLE が選択**されており、Arduino IDE からは切替不能でした。

## 解決策: ESP32 Core を 2.0.17 にダウングレード

Core 2.x までは ESP32-S3 でも Bluedroid がデフォルトだったので、既存のスケッチ群がそのまま動きます。

### 手順

1. Arduino IDE 左サイドバーの **ボードマネージャー** を開く
2. 検索欄に `esp32` と入力
3. 「esp32 by Espressif Systems」カードのバージョン選択で **2.0.17** を選択
4. **インストール** を押す（ダウングレードを確認するダイアログが出たら OK）
5. 5〜15分待つ（toolchain 込みで 200〜400MB ダウンロードされる）
6. 完了後、Arduino IDE を再起動

### ダウングレード後の注意

メニュー項目が一部減ります：

| 項目 | Core 3.x | Core 2.0.17 |
|---|---|---|
| USB CDC On Boot | ある | ない |
| USB DFU On Boot | ある | ない |
| Zigbee Mode | ある | ない |
| **PSRAM** | OPI PSRAM | OPI PSRAM ある |

`Board: ESP32S3 Dev Module` 再選択、`PSRAM: OPI PSRAM` 再設定、`Flash Size: 8MB` 確認、ぐらいやれば OK。

## EvilAppleJuice-ESP32 を実行

ダウングレード後にもう一度 EvilAppleJuice をコンパイル → 通る。書き込み → iPhone の Bluetooth 設定画面を開いて待機。

数秒以内に **AirPods、Beats、HomePod、Apple TV Setup、新しいiPhoneセットアップ** など多種類のポップアップが連続出現します（私の iPhone での観察結果）。

### ESP32-S3 向けの最小パッチ

EvilAppleJuice は元々 AirM2M ESP32-C3 ボード向けに書かれていて、物理ボタンでモード切り替えする想定。ESP32-S3-DevKitC-1 には対応ボタンがないので、ランダムモードに固定する1行を追加します：

```cpp
// EvilAppleJuice-ESP32-INO.ino setup() の preferences.end() 直後
currentMode = 1; // LEFT_OFF_RIGHT_FLASH = random device every cycle
```

これだけで全29機種からランダム選択 + MAC 自動ローテーション + 広告タイプランダム化が動きます。

## なぜ MAC ローテーションが効くのか

iOS 17.2 以降の対策は「**同じMACアドレスから連続して同じ広告を受信したら抑制**」というロジック（公開情報・各種解析より）。EvilAppleJuice は広告のたびに：

```cpp
esp_bd_addr_t dummy_addr;
for (int i = 0; i < 6; i++) {
  dummy_addr[i] = random(256);
  if (i == 0) dummy_addr[i] |= 0xF0;  // 先頭4bitは1に
}
pAdvertising->setDeviceAddress(dummy_addr, BLE_ADDR_TYPE_RANDOM);
```

毎回違う MAC で送るため、iPhone から見ると「毎回別のデバイス」として扱われ、抑制対象から外れます。これが Bluedroid API（`setDeviceAddress` + `BLE_ADDR_TYPE_RANDOM`）を必要とする理由。NimBLE で同じことをやるには別 API を経由する必要があり、そこが互換性問題に直結します。

## まとめ：教訓

| 教訓 | 詳細 |
|---|---|
| **ESP32 Core のバージョンは慎重に選ぶ** | 最新が一番ではない。ESP32-S3 + BLE 系スケッチなら Core 2.0.17 が安牌 |
| **Bluedroid と NimBLE は別物** | 同じ BLE ライブラリのフリをして API が違う。スケッチの想定スタックを確認 |
| **`sdkconfig.h` で実際の有効化スタックを確認** | `~/Library/Arduino15/packages/esp32/tools/esp32s3-libs/<version>/<variant>/include/sdkconfig.h` |
| **技適と倫理は常にセット** | 自宅・自分のデバイス対象に限定 |

## 次回予告

ESP32-S3 で BLE 送信側を理解したら、次は受信側の番。自宅周辺の BLE デバイスを片っ端からスキャンしてみると、想像以上に多くの機器が常時電波を撒いていることが分かります。次回記事では BLEScanner を自作して、近隣 BLE デバイスを可視化してみます。

→ [ESP32-S3 で BLE スキャナを自作したら、自宅周辺に 40 台の Apple 機器が見えた](https://qiita.com/masafy/private/2efbf20f5b9f03a2f3dc)

## 参考リンク

- [ckcr4lyf/EvilAppleJuice-ESP32](https://github.com/ckcr4lyf/EvilAppleJuice-ESP32)
- [ECTO-1A/AppleJuice](https://github.com/ECTO-1A/AppleJuice)
- [秋月電子 ESP32-S3-DevKitC-1-N8R8](https://akizukidenshi.com/catalog/g/g117073/)
- [Espressif ESP32 Arduino Core](https://github.com/espressif/arduino-esp32)
