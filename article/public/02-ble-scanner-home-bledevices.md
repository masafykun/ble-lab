---
title: ESP32-S3 で BLE スキャナを自作したら、自宅周辺に 40 台の Apple 機器が見えた
tags:
  - Arduino
  - Security
  - bluetooth
  - BLE
  - ESP32
private: true
updated_at: '2026-05-24T23:44:02+09:00'
id: 2efbf20f5b9f03a2f3dc
organization_url_name: null
slide: false
ignorePublish: false
---

> BLEスキャナを 100 行書いただけで、自分の部屋から見える機器の数に驚いた個人検証ノート。

ESP32-S3-DevKitC-1 と Arduino IDE で受動的な BLE スキャナを書いて、自宅から見える周辺デバイスをひたすら列挙してみました。
結果は **5 秒スキャンで 39 台ヒット**。内訳が予想以上にカオスで、自分の Apple TV や iPhone はもちろん、近所の Apple 機器・VR ベースステーション・自動車・トラッカー類まで丸見えでした。

BLE 検証ラボシリーズの **2 本目**（前回 [① Core 3.x の罠](https://qiita.com/masafy/private/e50254ad76c801422a29) で送信側、今回は受信側）。

## TL;DR

- ESP32-S3-DevKitC-1 と Arduino IDE で 100 行程度の BLE スキャナを自作
- 自宅の 1 部屋から 5 秒スキャンしただけで **39 台の BLE デバイス**を検出
- 内訳がカオス：近所の iPhone 10 台以上、Vive ベースステーション、Volvo 2 台、Tile、Find My、PLAUD AIレコーダ、iBeacon 各種…
- **Apple 機器は manufacturer data の `0x4C 0x00` で識別可能**、AirPods 等は modelId で機種特定までできる

## ✨ この記事で書くこと

- **100 行で動く BLE スキャナの全文** — `BLEDevice` / `BLEScan` / `BLEAdvertisedDevice` だけで完結
- **Apple Continuity の入り口** — Manufacturer Data の `0x4C 0x00` を判定して subtype と modelId を抜き出す
- **RSSI で雑に距離推定** — `-50 dBm = ~1m` 程度の目安テーブル
- **自宅で実際に見えた 39 台の正体** — Vive / Volvo / Tile / Find My / PLAUD など内訳と Company ID デコード

## ⚠️ 倫理ライン（毎回意識する）

- ✅ 自宅でパッシブに受信するだけ（送信は一切なし）
- ✅ 結果のスクショ・ログは MAC を OUI のみ残して伏字化してから公開
- ✅ 「**身近にこれだけ見えている**」という事実を共有する個人ノート
- ❌ 特定個人 / 特定車両 / 特定 AirTag の **時系列追跡は実施しない**（ストーカー規制法・個人情報保護法）
- ❌ 商業用途（店舗内マーケティング追跡など）にも転用しない

迷ったら「これは記事で公開して恥ずかしくないか？」で判断します。

## 環境

| 項目 | 値 |
|---|---|
| ボード | ESP32-S3-DevKitC-1-N8R8（技適 201-220052） |
| Arduino IDE | 2.3.8 |
| ESP32 Arduino Core | **2.0.17**（3.x 系で動かない罠あり、前回記事参照） |
| シリアル | `/dev/cu.usbserial-XXX`（CP2102N） |

## スケッチ全文

`~/Documents/Arduino/BLEScanner/BLEScanner.ino`:

```cpp
// ESP32-S3 BLE Scanner - 近所のBLEデバイスを可視化
#include <BLEDevice.h>
#include <BLEScan.h>
#include <BLEAdvertisedDevice.h>

#define SCAN_TIME_SEC 5
BLEScan* pBLEScan;

const char* identifyAppleModel(uint8_t modelId) {
  switch (modelId) {
    case 0x02: return "AirPods Gen 1";
    case 0x0F: return "AirPods Gen 2";
    case 0x13: return "AirPods Gen 3";
    case 0x0E: return "AirPods Pro";
    case 0x14: return "AirPods Pro Gen 2";
    case 0x0A: return "AirPods Max";
    case 0x03: return "PowerBeats";
    case 0x0B: return "PowerBeats Pro";
    case 0x0C: return "Beats Solo Pro";
    case 0x10: return "Beats Flex";
    case 0x05: return "BeatsX";
    case 0x06: return "Beats Solo 3";
    case 0x09: return "Beats Studio 3";
    case 0x11: return "Beats Studio Buds";
    case 0x12: return "Beats Fit Pro";
    case 0x16: return "Beats Studio Buds+";
    case 0x17: return "Beats Studio Pro";
    default:   return "Unknown Beats/AirPods";
  }
}

const char* identifyAppleSubtype(uint8_t type) {
  switch (type) {
    case 0x02: return "iBeacon";
    case 0x05: return "AirDrop";
    case 0x07: return "Proximity Pair (AirPods/Beats)";
    case 0x09: return "AirPlay";
    case 0x0A: return "AirPlay Target";
    case 0x0C: return "Handoff";
    case 0x10: return "Nearby Info (iPhone/iPad/Mac status)";
    case 0x12: return "Find My (AirTag/iPhone)";
    default:   return "Unknown Apple service";
  }
}

const char* distanceLabel(int rssi) {
  if (rssi > -50) return "very close (<1m)";
  if (rssi > -60) return "close (1-2m)";
  if (rssi > -70) return "nearby (3-5m)";
  if (rssi > -80) return "room (5-10m)";
  return "far (>10m)";
}

class AdvCallback: public BLEAdvertisedDeviceCallbacks {
  void onResult(BLEAdvertisedDevice dev) override {
    String mac  = dev.getAddress().toString().c_str();
    int    rssi = dev.getRSSI();
    String name = dev.haveName() ? dev.getName().c_str() : String("(no name)");

    Serial.printf("[%4d dBm | %-17s] %s | %s",
                  rssi, distanceLabel(rssi), mac.c_str(), name.c_str());

    if (dev.haveManufacturerData()) {
      std::string mfg = dev.getManufacturerData();
      if (mfg.length() >= 3 &&
          (uint8_t)mfg[0] == 0x4C && (uint8_t)mfg[1] == 0x00) {
        uint8_t subtype = (uint8_t)mfg[2];
        Serial.printf("  [Apple: %s]", identifyAppleSubtype(subtype));
        if (subtype == 0x07 && mfg.length() >= 6) {
          Serial.printf(" -> %s", identifyAppleModel((uint8_t)mfg[5]));
        }
      } else if (mfg.length() >= 2) {
        uint16_t companyId = ((uint8_t)mfg[1] << 8) | (uint8_t)mfg[0];
        Serial.printf("  [MfgID 0x%04X]", companyId);
      }
    }
    Serial.println();
  }
};

void setup() {
  Serial.begin(115200);
  delay(500);
  Serial.println("\n=== ESP32-S3 BLE Scanner ===\n");
  BLEDevice::init("");
  pBLEScan = BLEDevice::getScan();
  pBLEScan->setAdvertisedDeviceCallbacks(new AdvCallback());
  pBLEScan->setActiveScan(true);
  pBLEScan->setInterval(100);
  pBLEScan->setWindow(99);
}

void loop() {
  Serial.println("------ Scanning... ------");
  BLEScanResults results = pBLEScan->start(SCAN_TIME_SEC, false);
  Serial.printf(">>> %d devices in this round.\n\n", results.getCount());
  pBLEScan->clearResults();
  delay(2000);
}
```

## 仕組み

### 1. Active Scan

```cpp
pBLEScan->setActiveScan(true);
```

`Active Scan` にすると、広告を受信したあとに `SCAN_REQ` を投げて `SCAN_RSP`（追加情報）も取得します。デバイス名（haveName）はここで返ってくることが多いので、`true` にしておくと情報量が増えます。

### 2. Apple Manufacturer Data 識別

BLE 広告の Manufacturer Specific Data は最初の 2 バイトが Bluetooth SIG が割り当てた Company ID（little-endian）。Apple は **`0x004C`** が割り当てられているので、生バイト列の先頭が `4C 00` なら Apple 機器確定。

### 3. AirPods/Beats の機種特定

Apple Proximity Pair (subtype `0x07`) の場合、`mfgData[5]` のバイトが modelId。これで AirPods Pro Gen 2、Beats Studio 3 等まで一意特定できます（modelId の対応関係は AppleJuice 系 OSS の解析結果に依拠）。

### 4. 距離推定（雑に）

RSSI（受信信号強度）は対数スケールで距離に対応します。本格的にやるならパスロスモデルで計算しますが、雑に：

| RSSI | 推定距離 |
|---|---|
| > -50 dBm | ~1m 以内 |
| -50 〜 -60 | 1-2m |
| -60 〜 -70 | 3-5m |
| -70 〜 -80 | 5-10m |
| < -80 | 10m 以上（壁向こうの可能性大） |

## 自宅 5 秒スキャンの結果

実際に自分の部屋で動かした結果（39 台ヒット、抜粋）:

```
[ -37 dBm | very close (<1m) ] 7c:fb:**:**:**:**  [Apple: AirPlay]
[ -41 dBm | very close (<1m) ] 41:91:**:**:**:**  [Apple: Nearby Info (iPhone/iPad/Mac status)]
[ -62 dBm | nearby (3-5m)    ] 7e:4e:**:**:**:**  [Apple: Nearby Info (iPhone/iPad/Mac status)]
[ -66 dBm | nearby (3-5m)    ] 52:ff:**:**:**:**  [Apple: Nearby Info]
[ -68 dBm | nearby (3-5m)    ] 40:4e:36:**:**:** | HTC BS ******
[ -73 dBm | room (5-10m)    ] 40:4e:36:**:**:** | HTC BS ******
[ -77 dBm | room (5-10m)    ] 9c:06:cf:**:**:** | PLAUD NotePin  [MfgID 0x005D]
[ -80 dBm | far (>10m)      ] d8:9c:67:**:**:**  [Apple: iBeacon]
[ -86 dBm | far (>10m)      ] cb:02:79:**:**:**  [Apple: Find My (AirTag/iPhone)]
[ -87 dBm | far (>10m)      ] d4:6c:b2:**:**:**  [MfgID 0x055A]
[ -82 dBm | far (>10m)      ] fe:27:ce:**:**:**  [MfgID 0x0969]
>>> 39 devices in this round.
```

(MAC は個別特定を防ぐため OUI 以外を伏字にしています。先頭3バイト = メーカー識別部分のみ残存)

## 内訳の解読

### 🏠 自分のデバイス（強信号）

- `7c:fb:**:**:**:** -37 dBm AirPlay`: 同室の Apple TV か HomePod
- `41:91:**:**:**:** -41 dBm Nearby Info`: 自分の iPhone

### 🎮 想定外の発見 — Vive ベースステーション 2 台

```
40:4e:36:**:**:** | HTC BS ******
40:4e:36:**:**:** | HTC BS ******
```

`40:4e:36:` は HTC の OUI で、`BS XXXXXX` という名前は **Vive Base Station 2.0**。VR の電源を切り忘れていたことが発覚しました。BLEスキャナがホームインベントリ管理ツールになる瞬間。

### 🚗 まさかの自動車

`MfgID 0x055A` を Bluetooth SIG の [Assigned Numbers](https://www.bluetooth.com/specifications/assigned-numbers/) で引くと **Volvo Cars Corporation**。

つまり**近隣に駐車中の Volvo 2 台が常時 BLE を撒いている**ことが判明。最近の車はキーレスエントリーや車内 Bluetooth Audio のために常時広告するため、車種・台数まで筒抜けです（私の検証範囲では）。

### 📍 Find My / Tile トラッカー

```
cb:02:79:**:**:**  [Apple: Find My]
fe:27:ce:**:**:**  [MfgID 0x0969]   ← 0x0969 = Tile, Inc.
```

近所のどこかに **AirTag が落ちている / 持ち主と一緒に動いている** か、**Tile タグを誰かが持っている** ことが分かります。詳しくは次回記事で。

### 👥 集合住宅の隣人 iPhone（最低 10 台）

```
-62 dBm から -88 dBm まで、Apple: Nearby Info が 10 件以上
```

距離分布から、**隣の部屋から3軒先まで**の iPhone が見えている、というのが私の観察結果。集合住宅の密度がリアルに数値化されて、ちょっと不思議な気分になります。

### 🎙️ AIガジェット

```
9c:06:cf:**:**:** | PLAUD NotePin  [MfgID 0x005D = Nordic Semiconductor]
```

PLAUD NotePin は録音→AI文字起こしのデバイス。`0x005D` は Nordic Semiconductor（チップベンダ）なので「Nordic 製 BLE チップを採用」という事実だけが分かります。

## よく見かける Manufacturer ID 一覧

| Company ID | 会社 | 何 |
|---|---|---|
| 0x004C | Apple | iPhone/AirPods/AirTag/etc. |
| 0x0006 | Microsoft | Surface / Swift Pair |
| 0x0075 | Samsung | Galaxy 系 |
| 0x0157 | Anhui Huami（Mi Band） | Xiaomi スマートウォッチ |
| 0x0499 | Ruuvi Innovations | RuuviTag |
| 0x005D | Nordic Semiconductor | チップ採用各種 |
| 0x055A | Volvo Cars | 車 |
| 0x0969 | Tile | 紛失防止タグ |
| 0x037F | Hangzhou Shengfei | 中華 IoT |

完全な一覧は [Bluetooth SIG Assigned Numbers - Company Identifiers](https://www.bluetooth.com/specifications/assigned-numbers/) を参照。

## ハードウェアファインダープリンティング

これらの広告データから、**RF パッシブ観測のみで以下が推測できる**ことになります（私の検証範囲では）：

- 周辺人物の Apple 機器所有率
- 隣家の在宅状況（時系列で見れば外出パターンも）
- 周辺住人の所有車種
- 紛失防止タグの存在
- VR / IoT 機器の有無

すべて公開広告として撒かれている情報なので**受信自体は合法**ですが、「身近な人ですらこれが見えていることを知らない」というのが BLE プライバシーの本質的な問題だと感じました。

## 改造アイデア

- **RSSI -60 以上だけフィルタ** → 自分の部屋のデバイスだけ抽出
- **MAC を時系列ログに記録** → 入退室検知ホームオートメーション（自分の機器の範囲で）
- **未知 MAC のアラート** → 不審 BLE 機器の侵入検知
- **WebSocket で送信** → ダッシュボード化

## 次回予告

Apple の `Nearby Info` 広告（subtype `0x10`）には **iPhone の現在の状態**（ロック画面 / ホーム画面 / 通話中 / Wi-Fi ON/OFF / AirDrop ON/OFF / Apple Watch 連携の有無）がほぼ平文で入っています。次回はこの中身を解析して、Apple Continuity プロトコルがどれだけ情報を撒いているか可視化します。

→ [Apple Continuity 解析で iPhone がどれだけ状態を BLE で撒いているか可視化した話](https://qiita.com/masafy/private/fd4813cb861a80214eed)

## 参考リンク

- [Bluetooth SIG Assigned Numbers](https://www.bluetooth.com/specifications/assigned-numbers/)
- [ESP32 BLE Arduino library docs](https://github.com/espressif/arduino-esp32/tree/master/libraries/BLE)
- [furiousMAC/continuity（Apple Continuity 解析）](https://github.com/furiousMAC/continuity)
