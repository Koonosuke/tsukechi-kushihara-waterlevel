# NVS 連携実装メモ

## ゴールとスコープ
- 通信強度（RSRP/RSRQ）を取得し、送信成功/失敗とともに NVS に保存する。
- 最大 30 件をリングバッファ的に保持し、DeepSleep 後も残るようにする。
- このステップでは「保存」まで。送信側での蓄積データ送信は次フェーズ。

## 現状
- 水位取得 / HTTP 送信 / DeepSleep まで実装済み。
- 追加が必要なもの
  - `AT+CESQ` で通信強度取得
  - NVS への保存
  - 送信成功/失敗フラグを持たせる

## 手順

### STEP0 追加箇所を決める（コードはまだ触らない）
| 機能 | 追加場所 |
| --- | --- |
| 通信強度取得 | `serial_send()` の最初 |
| NVS 保存 | 送信前（成功/失敗に関係なく） |
| 成功/失敗判定 | `AT+SHREQ` の結果 |

### STEP1 ライブラリと構造体を追加
- ヘッダに追加
  ```cpp
  #include <Preferences.h>
  ```
- 通信強度ログ構造体
  ```cpp
  struct SignalLog {
    int rsrp;
    int rsrq;
    uint32_t ts;   // 観測時刻（今回は millis）
    uint8_t result; // 1=成功, 0=失敗
  };
  ```
- グローバルに宣言
  ```cpp
  Preferences prefs;
  ```

### STEP2 NVS を初期化（setup の最後）
```cpp
prefs.begin("signal", false); // NVS namespace
```

### STEP3 `AT+CESQ` で通信強度取得関数を追加
```cpp
bool getSignal(int &rsrp, int &rsrq) {
  MySerial0.println("AT+CESQ");
  delay(500);

  while (MySerial0.available()) {
    String line = MySerial0.readStringUntil('\n');
    line.trim();

    if (line.startsWith("+CESQ:")) {
      int v[6];
      sscanf(line.c_str(),
             "+CESQ: %d,%d,%d,%d,%d,%d",
             &v[0], &v[1], &v[2], &v[3], &v[4], &v[5]);

      // RSRQ
      if (v[4] == 255) rsrq = -999;
      else rsrq = (v[4] - 19.5) / 2;

      // RSRP
      if (v[5] == 255) rsrp = -999;
      else rsrp = -140 + v[5];

      return true;
    }
  }
  return false;
}
```

### STEP4 NVS に保存する関数を追加（最大 30 件の簡易リングバッファ）
```cpp
void saveSignal(int rsrp, int rsrq, uint32_t ts, uint8_t result) {
  int count = prefs.getInt("count", 0);
  if (count >= 30) count = 0; // 上書き

  SignalLog log = {rsrp, rsrq, ts, result};

  String key = "log" + String(count);
  prefs.putBytes(key.c_str(), &log, sizeof(SignalLog));
  prefs.putInt("count", count + 1);
}
```

### STEP5 `serial_send()` の先頭で通信強度を取得
```cpp
int rsrp = -999;
int rsrq = -999;
getSignal(rsrp, rsrq);

Serial.print("RSRP=");
Serial.print(rsrp);
Serial.print(" RSRQ=");
Serial.println(rsrq);
```

### STEP6 送信成功/失敗を判定して保存
- `AT+SHREQ` の結果を判定し、成功/失敗に関係なく保存する。
```cpp
bool success = sendATCommand("AT+SHREQ=\"http://uni.soracom.io\",3\r\n", POSTTIMEOUT);

saveSignal(rsrp, rsrq, millis(), success ? 1 : 0);

if (!success) {
  Serial.println("Error: AT+SHREQ");
  failureCount++;
  continue;
}
```

### STEP7 想定される動き
- 通信失敗時
  - RSRP=-114 / RSRQ=-16
  - NVS に保存 → 送信失敗 → DeepSleep
- 通信成功時
  - RSRP=-98 / RSRQ=-10
  - NVS に保存 → 送信成功 → 次ステップでまとめて送信

### STEP8 今ステップで十分な理由
- 電波状況を取得できる。
- 失敗時もログが残る。
- DeepSleep の影響を切り分けられる。
- コード改変は最小限。

## 超要約
- `AT+CESQ` で電波取得。
- Preferences（NVS）で保存。
- 成功/失敗に関係なく保存。
- 最大 30 件で制限。
