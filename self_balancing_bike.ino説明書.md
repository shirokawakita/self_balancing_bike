# self_balancing_bike.ino プログラム説明書

## 概要

`self_balancing_bike.ino`は、セルフバランシングバイクのメインプログラムです。MPU6050センサーからの角度データを取得し、PID制御アルゴリズムを使用してモーターを制御することで、バイクのバランスを維持します。また、スマートフォンアプリからのリモート制御信号を受信し、ステアリングと速度を制御します。

## ファイル構成

このファイルは以下の主要な部分で構成されています：

1. **ライブラリのインクルード**
2. **ピン定義と定数定義**
3. **グローバル変数の宣言**
4. **setup()関数** - 初期化処理
5. **loop()関数** - メイン制御ループ

## 1. ライブラリのインクルード

```1:3:01.Arduino Code and Library/self_balancing_bike.ino
#include <Wire.h>
#include <EEPROM.h>
#include <ServoTimer2.h>
```

### 使用ライブラリ

- **`Wire.h`**: I2C通信ライブラリ（MPU6050センサーとの通信に使用）
- **`EEPROM.h`**: EEPROM読み書きライブラリ（キャリブレーションデータとPIDパラメータの保存に使用）
- **`ServoTimer2.h`**: サーボモーター制御ライブラリ（Timer2を使用したサーボ制御）

## 2. ピン定義と定数定義

### 2.1 モーター制御ピン

```5:8:01.Arduino Code and Library/self_balancing_bike.ino
#define PWM_1         9
#define DIR_1         7
#define PWM_2         10
#define DIR_2         5 
```

- **PWM_1 (ピン9)**: モーター1のPWM速度制御
- **DIR_1 (ピン7)**: モーター1の回転方向制御
- **PWM_2 (ピン10)**: モーター2のPWM速度制御
- **DIR_2 (ピン5)**: モーター2の回転方向制御

### 2.2 エンコーダーピン

```10:11:01.Arduino Code and Library/self_balancing_bike.ino
#define ENC_1         2
#define ENC_2         3
```

- **ENC_1 (ピン2)**: エンコーダーA相（割り込み0）
- **ENC_2 (ピン3)**: エンコーダーB相（割り込み1）

### 2.3 その他のピン

```13:15:01.Arduino Code and Library/self_balancing_bike.ino
#define BRAKE         8
#define BUZZER        12
#define VBAT          A7
```

- **BRAKE (ピン8)**: ブレーキ制御（HIGH=ブレーキON、LOW=ブレーキOFF）
- **BUZZER (ピン12)**: ブザー出力
- **VBAT (アナログピンA7)**: バッテリー電圧監視

### 2.4 制御パラメータ定数

```17:21:01.Arduino Code and Library/self_balancing_bike.ino
#define STEERING_MAX    350
#define SPEED_MAX       80
#define STEERING_CENTER 1500
#define ST_LIMIT        5
#define SPEED_LIMIT     4
```

- **STEERING_MAX**: ステアリングの最大値（±350）
- **SPEED_MAX**: 速度の最大値（±80）
- **STEERING_CENTER**: ステアリングの中立位置（1500、サーボの中央値）
- **ST_LIMIT**: ステアリング値の変化率制限（1ループあたり±5）
- **SPEED_LIMIT**: 速度値の変化率制限（1ループあたり±4、現在はコメントアウト）

## 3. グローバル変数の宣言

### 3.1 補完フィルタパラメータ

```23:24:01.Arduino Code and Library/self_balancing_bike.ino
//float Gyro_amount = 0.996;
float Gyro_amount = 0.896;
```

- **Gyro_amount**: 補完フィルタにおけるジャイロの重み（0.896 = 89.6%）
- コメントアウトされた0.996は、よりジャイロに依存する設定

### 3.2 状態フラグ

```26:29:01.Arduino Code and Library/self_balancing_bike.ino
bool vertical = false;
bool calibrating = false;
bool calibrated = false;
bool PIDHas = false;
```

- **vertical**: ロボットが垂直状態（±0.3度以内）かどうか
- **calibrating**: キャリブレーション中かどうか
- **calibrated**: キャリブレーションデータがEEPROMに保存されているかどうか
- **PIDHas**: PIDパラメータがEEPROMに保存されているかどうか

### 3.3 PID制御パラメータ

```31:34:01.Arduino Code and Library/self_balancing_bike.ino
float K1 = 24.00;       //p
float K2 = -20.00;      //i
float K3 = -9.00;       //s
float K4 = -0.00;       //a
```

- **K1 (P項)**: 角度偏差に対する比例項（24.00）
- **K2 (I項)**: 角速度に対する積分項（-20.00）
- **K3 (D項)**: モーター速度に対する微分項（-9.00）
- **K4 (位置項)**: 位置フィードバック項（現在は0で無効）

### 3.4 制御周期

```36:36:01.Arduino Code and Library/self_balancing_bike.ino
float loop_time = 10;
```

- **loop_time**: メイン制御ループの周期（10ms = 100Hz）

### 3.5 データ構造体

#### キャリブレーションオフセット構造体

```38:43:01.Arduino Code and Library/self_balancing_bike.ino
struct OffsetsObj {
  int ID;
  int16_t AcY;
  int16_t AcZ;
};
OffsetsObj offsets;
```

- **ID**: データ識別子（35が設定されている場合、有効なデータ）
- **AcY**: Y軸加速度計のオフセット値
- **AcZ**: Z軸加速度計のオフセット値

#### PIDパラメータ構造体

```45:52:01.Arduino Code and Library/self_balancing_bike.ino
struct PidNum {
  int ID;
  float K1;
  float K2;
  float K3;
  float K4;
};
PidNum MyPid;
```

- **ID**: データ識別子（55が設定されている場合、有効なデータ）
- **K1～K4**: PID制御パラメータ

### 3.6 フィルタパラメータ

```54:54:01.Arduino Code and Library/self_balancing_bike.ino
float alpha = 0.4;
```

- **alpha**: ローパスフィルタの係数（0.4 = 40%の新規データ、60%の過去データ）

### 3.7 センサーデータ変数

```56:63:01.Arduino Code and Library/self_balancing_bike.ino
int16_t  AcY, AcZ, GyX, gyroX, gyroXfilt;

int16_t  AcYc, AcZc;
int16_t  GyX_offset = 0;
int32_t  GyX_offset_sum = 0;

float robot_angle;
float Acc_angle;
```

- **AcY, AcZ**: 加速度計の生データ（Y軸、Z軸）
- **GyX**: ジャイロの生データ（X軸）
- **gyroX**: ジャイロデータを度/秒に変換した値
- **gyroXfilt**: ローパスフィルタ適用後のジャイロデータ
- **AcYc, AcZc**: キャリブレーション補正後の加速度計データ
- **GyX_offset**: ジャイロのオフセット値
- **GyX_offset_sum**: ジャイロオフセット計算用の累積値
- **robot_angle**: 補完フィルタで計算されたロボットの角度
- **Acc_angle**: 加速度計から直接計算された角度

### 3.8 エンコーダー関連変数

```65:68:01.Arduino Code and Library/self_balancing_bike.ino
volatile byte pos;
volatile int motor_counter = 0, enc_count = 0;
int16_t motor_speed;
int32_t motor_pos;
```

- **pos**: エンコーダーの状態を保持する変数（割り込み処理で使用）
- **enc_count**: エンコーダーのカウント値（割り込み処理で更新、volatile指定）
- **motor_speed**: モーターの速度（エンコーダーから計算）
- **motor_pos**: モーターの位置（速度の積分、-110～+110に制限）

### 3.9 リモート制御変数

```69:69:01.Arduino Code and Library/self_balancing_bike.ino
int steering_remote = 0, speed_remote = 0, speed_value = 0, steering_value = STEERING_CENTER;;
```

- **steering_remote**: リモート制御からのステアリング値
- **speed_remote**: リモート制御からの速度値
- **speed_value**: 実際の速度制御値（現在はコメントアウト）
- **steering_value**: 実際のステアリング制御値

### 3.10 その他の変数

```71:76:01.Arduino Code and Library/self_balancing_bike.ino
int bat_divider = 58; // this value needs to be adjusted to measure the battery voltage correctly

long currentT, previousT_1, previousT_2 = 0;  

ServoTimer2 steering_servo;
ServoTimer2 steering_servo2;
```

- **bat_divider**: バッテリー電圧測定用の分圧比（58に調整が必要）
- **currentT**: 現在時刻（millis()）
- **previousT_1**: 前回のメインループ実行時刻
- **previousT_2**: 前回のバッテリー監視実行時刻
- **steering_servo**: ステアリング用サーボモーター（ピンA3）
- **steering_servo2**: 速度制御用サーボモーター（ピン10）

## 4. setup()関数 - 初期化処理

```78:130:01.Arduino Code and Library/self_balancing_bike.ino
void setup() {
  Serial.begin(115200);                       // シリアル通信の初期化

  // ピンD9とD10 - 7.8 kHz
  TCCR1A = 0b00000001;                        // ピン9,10のPWM設定
  TCCR1B = 0b00001010;                        // ピン9,10のPWM設定

  steering_servo.attach(A3);                  // サーボモーターのピン定義
  steering_servo.write(STEERING_CENTER);       // サーボを中立位置に

  steering_servo2.attach(10);                  // サーボモーターのピン定義
  steering_servo2.write(STEERING_CENTER);     // サーボを中立位置に

  pinMode(DIR_1, OUTPUT);                     // 方向1ピンのモード定義
  pinMode(DIR_2, OUTPUT);                     // 方向2ピンのモード定義
  pinMode(BRAKE, OUTPUT);                     // ブレーキピンのモード定義
  pinMode(BUZZER, OUTPUT);                    // ブザーピンのモード定義
  pinMode(ENC_1, INPUT);                      // エンコーダーAピンのモード定義
  pinMode(ENC_2, INPUT);                      // エンコーダーBピンのモード定義
  
  Motor1_control(0);                          // モーター1の速度を0に設定
  Motor2_control(0);                          // モーター2の速度を0に設定

  attachInterrupt(0, ENC_READ, CHANGE);       // 割り込み源0、エンコーダー読み取り関数をトリガー、レベル変化でトリガー
  attachInterrupt(1, ENC_READ, CHANGE);       // 割り込み源1、エンコーダー読み取り関数をトリガー、レベル変化でトリガー

  EEPROM.get(0, offsets);                     // EEPROMからキャリブレーションデータを読み取り
  if (offsets.ID == 35) 
    {
      calibrated = true;                      // データIDが35の場合、calibratedをtrueに（キャリブレーションデータあり）
      Serial.println("Read calibration data!");
    }
  else 
    {
      calibrated = false;                     // 読み取れなかった場合、calibratedをfalseに
      Serial.println("Has no calibration data!");
    }

  EEPROM.get(50, MyPid);                     // EEPROMからPIDデータを読み取り
  if (MyPid.ID == 55)
    {
      PIDHas = true;                          // データIDが55の場合、PIDHasをtrueに（PIDデータあり）
      Serial.println("Read PID data!");
    }
  else
    {
      PIDHas = false;                         // 読み取れなかった場合、PIDHasをfalseに
      Serial.println("Has no PID data!");
    }
  delay(3000);                                // 遅延
  beep();                                     // ブザーで通知
  angle_setup();                              // 角度設定
}
```

### 初期化処理の流れ

1. **シリアル通信の初期化**（115200bps）
   - スマートフォンアプリとの通信に使用

2. **PWM周波数の設定**
   - Timer1を使用してピン9,10のPWM周波数を7.8kHzに設定
   - 可聴域外の周波数でモーターの動作音を低減

3. **サーボモーターの初期化**
   - 両サーボを中立位置（1500）に設定

4. **ピンモードの設定**
   - 各ピンの入出力モードを設定

5. **モーターの停止**
   - 安全のため、両モーターを停止状態に設定

6. **エンコーダー割り込みの設定**
   - エンコーダーの状態変化で割り込み処理を実行

7. **EEPROMからのデータ読み込み**
   - キャリブレーションデータとPIDパラメータを読み込み
   - データの有効性を確認（IDで判定）

8. **3秒待機**
   - システムの安定化とユーザーへの準備時間

9. **ブザー音の再生**
   - 起動準備完了の合図

10. **MPU6050の初期化**
    - `angle_setup()`関数を呼び出してMPU6050を初期化
    - ジャイロオフセットを自動補正

## 5. loop()関数 - メイン制御ループ

```132:196:01.Arduino Code and Library/self_balancing_bike.ino
void loop() {

  currentT = millis();

  if (currentT - previousT_1 >= loop_time) {
    readControlParameters();  
    angle_calc();

    motor_speed = -enc_count;
    enc_count = 0;

    if (vertical && calibrated && !calibrating) {
      digitalWrite(BRAKE, HIGH);
      gyroX = GyX / 131.0; // 度/秒に変換

      gyroXfilt = alpha * gyroX + (1 - alpha) * gyroXfilt;

      motor_pos += motor_speed;
      motor_pos = constrain(motor_pos, -110, 110);
      
      int pwm = constrain(K1 * robot_angle + K2 * gyroXfilt + K3 * motor_speed + K4 * motor_pos, -255, 255); 
      Motor1_control(-pwm);
      
      if ((steering_value - STEERING_CENTER - steering_remote) > ST_LIMIT)
        steering_value -= ST_LIMIT;
      else if ((steering_value - STEERING_CENTER - steering_remote) < -ST_LIMIT)
        steering_value += ST_LIMIT;
      else
        steering_value = STEERING_CENTER + steering_remote;

      steering_servo.write(steering_value);
      
      steering_servo2.write(STEERING_CENTER + speed_remote*7);
      
    } else { 
      digitalWrite(BRAKE, LOW);
      steering_value = STEERING_CENTER;
      steering_servo.write(STEERING_CENTER);
      steering_servo2.write(STEERING_CENTER);
      speed_value = 0;
      Motor1_control(0);
      Motor2_control(0);
      motor_pos = 0;
    }
    previousT_1 = currentT;
  }
  
  if (currentT - previousT_2 >= 2000) {    
    battVoltage((double)analogRead(VBAT) / bat_divider); 
    if (!calibrated && !calibrating) {
      Serial.println("first you need to calibrate the balancing point...");
    }
    previousT_2 = currentT;
  }
}
```

### メインループの処理フロー

#### 10ms周期のメイン制御ループ

1. **リモート制御パラメータの読み取り**
   - `readControlParameters()`: スマートフォンアプリからのジョイスティック/ボタン入力を受信

2. **角度計算**
   - `angle_calc()`: MPU6050からデータを取得し、補完フィルタで角度を計算

3. **エンコーダー速度の取得**
   - `motor_speed = -enc_count`: エンコーダーのカウント値を速度に変換
   - `enc_count = 0`: 次のループに備えてリセット

4. **動作条件のチェック**
   - `vertical && calibrated && !calibrating`: 3つの条件をすべて満たす場合のみ動作

5. **動作時の処理**（条件を満たす場合）
   - **ブレーキ**: HIGH（ブレーキON）
   - **角速度の計算とフィルタリング**: ジャイロデータを度/秒に変換し、ローパスフィルタを適用
   - **位置の更新**: モーター速度を積分して位置を更新（-110～+110に制限）
   - **PID制御計算**: 角度、角速度、速度、位置からPWM値を計算
   - **モーター制御**: 計算されたPWM値でモーターを制御
   - **ステアリング制御**: リモート制御値に応じてステアリングを制御（変化率制限あり）
   - **サーボ制御**: ステアリングと速度に応じてサーボを制御

6. **停止時の処理**（条件を満たさない場合）
   - **ブレーキ**: LOW（ブレーキOFF）
   - **モーター**: 停止
   - **サーボ**: 中立位置に戻す
   - **位置リセット**: `motor_pos = 0`

#### 2秒周期のバッテリー監視ループ

1. **バッテリー電圧の監視**
   - `battVoltage()`: バッテリー電圧を読み取り、低電圧時にブザーを鳴らす

2. **キャリブレーション未実施時の警告**
   - キャリブレーションデータがない場合、シリアルに警告メッセージを出力

## 6. 主要な処理の詳細

### 6.1 PID制御の実装

```153:153:01.Arduino Code and Library/self_balancing_bike.ino
      int pwm = constrain(K1 * robot_angle + K2 * gyroXfilt + K3 * motor_speed + K4 * motor_pos, -255, 255);
```

**制御式**:
```
PWM = K1 × robot_angle + K2 × gyroXfilt + K3 × motor_speed + K4 × motor_pos
```

- **K1 × robot_angle**: 角度偏差に対する比例応答
- **K2 × gyroXfilt**: 角速度フィードバック（ローパスフィルタ適用済み）
- **K3 × motor_speed**: 速度フィードバック（エンコーダーから取得）
- **K4 × motor_pos**: 位置フィードバック（現在は0で無効）

### 6.2 ステアリング制御の実装

```164:169:01.Arduino Code and Library/self_balancing_bike.ino
      if ((steering_value - STEERING_CENTER - steering_remote) > ST_LIMIT)
        steering_value -= ST_LIMIT;
      else if ((steering_value - STEERING_CENTER - steering_remote) < -ST_LIMIT)
        steering_value += ST_LIMIT;
      else
        steering_value = STEERING_CENTER + steering_remote;
```

**動作**:
- リモート制御値と現在値の差分が`ST_LIMIT`（5）を超える場合、1ループあたり5ずつ変化
- これにより、急激なステアリング変化を防ぎ、スムーズな動作を実現

### 6.3 安全機能

**動作条件**:
```144:144:01.Arduino Code and Library/self_balancing_bike.ino
    if (vertical && calibrated && !calibrating) {
```

- **vertical**: ロボットが垂直状態（±0.3度以内）
- **calibrated**: キャリブレーションデータがEEPROMに保存されている
- **!calibrating**: キャリブレーション中ではない

これらの条件をすべて満たす場合のみ、モーターが動作します。

**停止時の処理**:
```176:185:01.Arduino Code and Library/self_balancing_bike.ino
    } else { 
      digitalWrite(BRAKE, LOW);
      steering_value = STEERING_CENTER;
      steering_servo.write(STEERING_CENTER);
      steering_servo2.write(STEERING_CENTER);
      speed_value = 0;
      Motor1_control(0);
      Motor2_control(0);
      motor_pos = 0;
    }
```

条件を満たさない場合、即座に：
- ブレーキを解除
- モーターを停止
- サーボを中立位置に戻す
- 位置をリセット

## 7. 外部関数の呼び出し

このファイルから呼び出される外部関数（`functions.ino`と`remote.ino`に定義）：

- **`angle_calc()`**: MPU6050からデータを取得し、角度を計算
- **`angle_setup()`**: MPU6050の初期化とジャイロオフセット補正
- **`Motor1_control()`**: モーター1の制御
- **`Motor2_control()`**: モーター2の制御
- **`ENC_READ()`**: エンコーダーの読み取り（割り込み処理）
- **`readControlParameters()`**: リモート制御パラメータの読み取り
- **`battVoltage()`**: バッテリー電圧の監視
- **`beep()`**: ブザー音の再生

## 8. まとめ

`self_balancing_bike.ino`は、セルフバランシングバイクの**メイン制御プログラム**です。以下の機能を実現しています：

1. **システムの初期化**: ハードウェアの設定、EEPROMからのデータ読み込み、MPU6050の初期化
2. **角度の計算**: MPU6050センサーからデータを取得し、補完フィルタで角度を計算
3. **PID制御**: 角度、角速度、速度、位置からモーターのPWM値を計算
4. **リモート制御**: スマートフォンアプリからの信号を受信し、ステアリングと速度を制御
5. **安全機能**: 動作条件をチェックし、条件を満たさない場合は即座に停止

10ms周期の制御ループにより、高速で安定したバランス制御を実現しています。

