---
title: Apple Continuity 解析で iPhone がどれだけ状態を BLE で撒いているか可視化した話
tags:
  - iOS
  - Security
  - bluetooth
  - BLE
  - ESP32
private: true
updated_at: '2026-05-24T23:44:04+09:00'
id: fd4813cb861a80214eed
organization_url_name: null
slide: false
ignorePublish: false
---

> iPhone が BLE で何を撒いているかをデコードしたら、状態情報の細かさに気づいた個人検証メモ。

ESP32-S3-DevKitC-1 で Apple Continuity プロトコルの **Nearby Info (subtype `0x10`)** を解析するスケッチを書いてみました。
受信したパケットをデコードすると、自分の iPhone のロック状態・ホーム画面か通話中か・Wi-Fi/AirDrop の有効状態・Apple Watch 連携の有無 などが**ほぼ平文**で読み取れます。
ペアリングも認証も不要で、誰でも受信可能（受信自体は合法）。

BLE 検証ラボシリーズの **3 本目**（[① Core 3.x 罠](https://qiita.com/masafy/private/e50254ad76c801422a29) / [② BLE Scanner](https://qiita.com/masafy/private/2efbf20f5b9f03a2f3dc) の続き、完結編）。

## TL;DR

- iPhone / iPad / Mac は常に **Apple Continuity** という独自BLEプロトコルで広告を撒いている
- その中の **Nearby Info (subtype `0x10`)** には現在のロック状態・ホーム画面か・通話中か・Wi-Fi/AirDrop の有効状態・Apple Watch 連携の有無 などが**ほぼ平文**で入っている
- 公開広告のため誰でも受信可能（受信自体は合法）
- iOS 18 現在も**仕様上止められない**（Continuity 全体が成り立たなくなるため）
- Apple ユーザーが自分のデバイスがどれだけ多くの状態を BLE で外部に流しているかを理解しておく価値がある

## ✨ この記事で書くこと

- **Apple Continuity プロトコル超概要** — 主要 subtype 一覧と Nearby Info (0x10) のパケット構造
- **Status Byte / Data Byte のビット解釈テーブル** — OSS 研究プロジェクトの解析結果に基づく（後述の注釈参照）
- **ContinuityAnalyzer スケッチ全文** — Nearby Info だけを抜き出して人間が読める形に整形
- **自分の iPhone での挙動観察** — ロック / 通話 / 音楽再生 / マップ で出力がどう変わるか
- **「漏れる項目」と「漏れない項目」の整理** — 何が見えて、何が見えないか

## ⚠️ 倫理ライン（毎回意識する）

- ✅ 自宅で受動的にスキャンするだけ（送信は一切なし）
- ✅ 自分の iPhone を意図的に操作して、出力がどう変わるかを確認する形式
- ✅ 公開ログの MAC は OUI のみ残して伏字化
- ❌ 特定個人の iPhone 状態を**時系列で追跡する解析は実施しない**（ストーカー規制法）
- ❌ 商業利用（店舗内マーケティング追跡など）への転用も対象外
- ❌ 取得情報を本人特定や行動分析に活用する用途は **絶対 NG**

迷ったら「これは記事で公開して恥ずかしくないか？」で判断します。

## Apple Continuity プロトコル超概要

Apple デバイス同士の「自然な連携」（AirDrop / Handoff / Auto Unlock / Universal Clipboard / 近接ペアリング）を支えているのが Continuity。BLE Advertising で常に近隣にステータスを撒くことで実現されている、と理解しています。

主要なサブタイプ一覧（Manufacturer Data の `0x4C 0x00` の直後）：

| Subtype | 意味 |
|---|---|
| `0x05` | AirDrop（受信モード公告） |
| `0x07` | AirPods/Beats Proximity Pairing |
| `0x09` | AirPlay Source |
| `0x0A` | AirPlay Target |
| `0x0C` | Handoff |
| `0x0F` | Nearby Action |
| **`0x10`** | **Nearby Info（iPhone 等の状態）← 今回の主役** |
| `0x12` | Find My（AirTag 含む） |

## Nearby Info (0x10) のパケット構造

Apple BLE manufacturer data の中の Nearby Info メッセージは：

```
Offset  Size   内容
------  ----   ------------------------
[0]     1      0x4C            Company ID (Apple) LSB
[1]     1      0x00            Company ID MSB
[2]     1      0x10            Subtype: Nearby Info
[3]     1      length          以降のサイズ（typically 0x05 or 0x07）
[4]     1      Status Byte 1   高4bit=Status flags / 低4bit=Action code
[5]     1      Data Byte 2     Wi-Fi/AirDrop/Auth tag 等
[6+]    3      Auth Tag        （optional）
```

### Status Byte 1（mfgData[4]）

高 4 bit がフラグ、低 4 bit がアクションコード。

> ⚠️ **以下のビットフラグ・アクションコードの意味は Apple 非公開のため、OSS 研究プロジェクト（[furiousMAC/continuity](https://github.com/furiousMAC/continuity) / [hexway/apple_bleee](https://github.com/hexway/apple_bleee)）による解析結果に依拠しています。一部は推定や解釈差がある可能性があるため、最新の研究状況はそれぞれのリポジトリを参照してください。**

#### 高 4bit フラグ

| bit | 値 | 意味 |
|---|---|---|
| 4 | 0x10 | Primary iCloud account |
| 5 | 0x20 | AirPods/Audio Device 接続中 |
| 6 | 0x40 | Auto Unlock enabled |
| 7 | 0x80 | Apple Watch Auto Unlock |

#### 低 4bit アクションコード（**ユーザーの今の挙動が読める**）

| 値 | 意味 |
|---|---|
| 0x00 | Activity reporting disabled |
| 0x01 | Apple Watch / Apple TV setup |
| 0x03 | 📞 Locked + Audio/Call |
| 0x05 | 🔒 **LOCK SCREEN（ロック中・無操作）** |
| 0x07 | 🏠 **HOME SCREEN（ロック解除・アクティブ）** |
| 0x09 | 🎵 Audio playing（画面オフでバックグラウンド再生） |
| 0x0A | 🗺️ Maps directions active |
| 0x0B | 📞 Phone/FaceTime call active |
| 0x0D | ✅ Active user（Unlocked & in use） |
| 0x0E | 📶 Wi-Fi setup screen |

### Data Byte 2（mfgData[5]）

| bit | 値 | 意味 |
|---|---|---|
| 0 | 0x01 | Auth tag present（このあと 3 バイト続く） |
| 1 | 0x02 | Wi-Fi ON |
| 2 | 0x04 | AirDrop active |

## ContinuityAnalyzer スケッチ

`~/Documents/Arduino/ContinuityAnalyzer/ContinuityAnalyzer.ino`:

```cpp
#include <BLEDevice.h>
#include <BLEScan.h>
#include <BLEAdvertisedDevice.h>

#define SCAN_TIME_SEC 6
#define MIN_RSSI -85  // 弱すぎる信号は除外

BLEScan* pBLEScan;

const char* actionCode(uint8_t code) {
  switch (code & 0x0F) {
    case 0x00: return "Activity reporting disabled";
    case 0x01: return "Apple Watch / Apple TV setup";
    case 0x03: return "📞 Locked + Audio/Call";
    case 0x05: return "🔒 LOCK SCREEN (idle)";
    case 0x07: return "🏠 HOME SCREEN (unlocked, active)";
    case 0x09: return "🎵 Audio playing (screen off)";
    case 0x0A: return "🗺️ Maps directions active";
    case 0x0B: return "📞 Phone/FaceTime call active";
    case 0x0D: return "✅ Active user (Unlocked & in use)";
    case 0x0E: return "📶 Wi-Fi setup screen";
    default:   return "Unknown activity";
  }
}

void printStatusBits(uint8_t b1, uint8_t b2) {
  uint8_t hi = b1 & 0xF0;
  Serial.printf("       Flags (b1=0x%02X b2=0x%02X):\n", b1, b2);
  Serial.printf("         %s Primary iCloud account\n",        (hi & 0x10) ? "✅" : "  ");
  Serial.printf("         %s AirPods/Audio device connected\n", (hi & 0x20) ? "🎧" : "  ");
  Serial.printf("         %s Auto Unlock enabled\n",            (hi & 0x40) ? "🔓" : "  ");
  Serial.printf("         %s Apple Watch Auto Unlock\n",        (hi & 0x80) ? "⌚" : "  ");
  Serial.println("       Data flags:");
  Serial.printf("         %s Auth tag present\n",  (b2 & 0x01) ? "🔐" : "  ");
  Serial.printf("         %s Wi-Fi ON\n",          (b2 & 0x02) ? "📶" : "📵");
  Serial.printf("         %s AirDrop active\n",    (b2 & 0x04) ? "✈️" : "  ");
}

class AdvCallback: public BLEAdvertisedDeviceCallbacks {
  void onResult(BLEAdvertisedDevice dev) override {
    int rssi = dev.getRSSI();
    if (rssi < MIN_RSSI) return;
    if (!dev.haveManufacturerData()) return;

    std::string mfg = dev.getManufacturerData();
    if (mfg.length() < 6) return;
    if ((uint8_t)mfg[0] != 0x4C || (uint8_t)mfg[1] != 0x00) return;
    if ((uint8_t)mfg[2] != 0x10) return;  // Nearby Info only

    uint8_t b1 = (uint8_t)mfg[4];
    uint8_t b2 = (uint8_t)mfg[5];

    Serial.println("=========================================================");
    Serial.printf("[%4d dBm] %s  📱 Nearby Info\n", rssi, dev.getAddress().toString().c_str());
    Serial.printf("       Activity: %s\n", actionCode(b1));
    printStatusBits(b1, b2);
    Serial.println();
  }
};

void setup() {
  Serial.begin(115200);
  delay(500);
  Serial.println("\n=== Apple Continuity Analyzer ===\n");
  BLEDevice::init("");
  pBLEScan = BLEDevice::getScan();
  pBLEScan->setAdvertisedDeviceCallbacks(new AdvCallback());
  pBLEScan->setActiveScan(true);
  pBLEScan->setInterval(100);
  pBLEScan->setWindow(99);
}

void loop() {
  Serial.println("\n████████ Scan round ████████\n");
  pBLEScan->start(SCAN_TIME_SEC, false);
  pBLEScan->clearResults();
  delay(1000);
}
```

## 自分の iPhone で挙動を観察

書き込んでシリアルモニタを開き、以下を順番にやって出力を見ます。`MIN_RSSI` を `-60` 程度にしておくと自分の部屋の機器に絞れます。

| 操作 | 出力の変化 |
|---|---|
| iPhone をロック画面のまま放置 | `Activity: 🔒 LOCK SCREEN (idle)` |
| ロック解除してホーム画面 | `Activity: 🏠 HOME SCREEN` or `✅ Active user` |
| 自分に FaceTime 発信（テスト） | `Activity: 📞 Phone/FaceTime call active` |
| Apple Music で再生→画面オフ | `Activity: 🎵 Audio playing (screen off)` |
| マップで経路検索開始 | `Activity: 🗺️ Maps directions active` |
| Wi-Fi をオフにする | Data flags の `📶 Wi-Fi ON` が消える |
| 設定 → AirDrop → 受信オフ | `AirDrop active` が消える |

**結論から言うと、私の検証範囲ではほぼテーブルどおりに状態が反映されました。** Continuity が普段からどれだけ細かい状態をブロードキャストしているかが実感できます。

## 実測ログ

自分の iPhone（強信号）と、より遠い位置からの別の iPhone（中信号）が同時に見えた瞬間：

```
=========================================================
[ -41 dBm] 41:91:**:**:**:**  📱 Nearby Info
       Activity: ✅ Active user (Unlocked & in use)
       Flags (b1=0x1D b2=0x12):
         ✅ Primary iCloud account
            AirPods/Audio device connected
         🔓 Auto Unlock enabled
            Apple Watch Auto Unlock
       Data flags:
         🔐 Auth tag present
         📶 Wi-Fi ON
         ✈️ AirDrop active

=========================================================
[ -62 dBm] 7e:4e:**:**:**:**  📱 Nearby Info
       Activity: 🔒 LOCK SCREEN (idle)
       Flags (b1=0x95 b2=0x02):
         ✅ Primary iCloud account
            AirPods/Audio device connected
            Auto Unlock enabled
         ⌚ Apple Watch Auto Unlock
       Data flags:
            Auth tag present
         📶 Wi-Fi ON
            AirDrop active
```

下のエントリ（-62 dBm = 数メートル離れた位置）でも以下が読み取れます：

- メインiCloudアカウント保持機（=本人機）であること
- Apple Watch を装着している
- いま画面ロック中
- Wi-Fi は ON
- AirDrop は OFF

ペアリングしていない他人のデバイスからこのレベルの状態情報が読み取れる、というのが Continuity プロトコルの現実です。**自分の iPhone も同じレベルで他者から見えている**ということでもあります。

## なぜ漏れているのか（仕様上の理由）

Apple は **「近接デバイス同士が自動連携する魔法のような体験」** を売りにしているため：

- iPhone のロック解除 → 近くの Apple Watch / Mac が自動連動
- AirDrop → 近接の Apple 機器を探索
- Universal Clipboard → 近隣デバイス間でクリップボード共有
- AirPlay → 近くの HomePod / Apple TV に即座にキャスト

これらを実現するには、**周囲のデバイスが互いの状態を知っている必要がある**。だから常に BLE で状態を撒いている、というのが私の理解です。

つまり、**この「漏洩」は機能の副作用ではなく、Continuity という UX の本質そのもの**。OS バグや設定で止められる類のものではなく、`設定 → Bluetooth → OFF` でしか停止できない（= AirPods も Apple Watch も使えなくなる）。

## 何が漏れて、何が漏れないか

| 項目 | 漏れる？ |
|---|---|
| ロック / アンロック状態 | ✅ 漏れる |
| ホーム画面 / アプリ使用中 | ✅ 漏れる（アクションコードで） |
| 通話中 / FaceTime 中 | ✅ 漏れる |
| Maps ナビゲーション中 | ✅ 漏れる |
| 音楽再生中（バックグラウンド） | ✅ 漏れる |
| Wi-Fi の ON/OFF | ✅ 漏れる |
| AirDrop の有効状態 | ✅ 漏れる |
| Apple Watch との連携 | ✅ 漏れる |
| メイン iCloud アカウントかどうか | ✅ 漏れる |
| 実際の iCloud アカウント名 | ❌ 漏れない（Auth Tag で匿名化） |
| 表示中のアプリ名 | ❌ 漏れない |
| 会話内容 | ❌ 漏れない |
| 写真・連絡先 | ❌ 漏れない |
| MAC アドレス（永続） | ❌ 漏れない（RPA = Random Private Address でローテーション） |

## プライバシー側の防御策

完全に止めるなら **Bluetooth を OFF にする** しかありません。ただしそうすると：

- AirPods が使えない
- Apple Watch 連携不可
- AirDrop 不可
- Auto Unlock 不可
- Universal Clipboard 不可
- Find My（自分が紛失したとき）への露出減少

Apple エコシステムの利便性と引き換えなので、現実的には「**漏れているという事実を理解した上で使う**」が落としどころかなと感じています。

公共の場で機密性高いオペレーション（重要な経路移動、特定の人への FaceTime 発信タイミング）を秘匿したい場合は、**Bluetooth を一時的にオフ**する判断が必要なケースもありそうです。

## 倫理ライン（再掲）

- 自分の iPhone で挙動を確認する → ✅ 完全合法
- 周辺デバイスが結果として観測対象になる → ⚠️ 受信自体は合法だが、**個人特定や時系列追跡まで踏み込めば別問題**
- 特定の個人を狙って常時監視する → ❌ ストーカー規制法
- 商用利用（マーケティング目的の店舗内追跡） → ❌ 個人情報保護法・電子通信プライバシー

「**広告として撒かれている = 誰でも自由に使っていい**」ではない、というのが私の立場です。

## まとめ

- Apple Continuity は近接BLE広告で状態を撒く設計
- Nearby Info (subtype `0x10`) で iPhone の現在状態がほぼ平文で取得可能
- ペアリングも認証も不要で、誰でも受信できる
- iOS の仕様であり、機能を維持したまま止めることはできない
- 自分の iPhone がどれだけ多くの状態を BLE で外部に流しているかを知ることは、プライバシー意識を高める良い材料になる

## 次にやりたいこと

- **AirTag（Find My subtype 0x12）の中身解析** → 自分の周辺に未知の AirTag が存在しないか、防御目的でモニタリング
- **自分の iPhone 向け Continuity 状態ロガー** → 自分の機器が時間帯ごとにどう情報を出しているか自己観察ダッシュボード化
- **Continuity 偽造実験** → 自分の iPhone に対して「Mac が Auto Unlock 要求してる」風の広告を投げて挙動を確認

## 参考リンク

- [furiousMAC/continuity — Apple Continuity Protocol Documentation](https://github.com/furiousMAC/continuity)
- [hexway/apple_bleee — Apple BLE traffic analyzer](https://github.com/hexway/apple_bleee)
- [BAStille Research — Apple's Bleeding Bluetooth (BlackHat 2020)](https://www.bastille.net/research)
- [前回記事: BLE スキャナで自宅 40 台可視化](https://qiita.com/masafy/private/2efbf20f5b9f03a2f3dc)
