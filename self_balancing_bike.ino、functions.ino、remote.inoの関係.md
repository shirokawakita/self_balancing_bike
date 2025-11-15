# self_balancing_bike.ino、functions.ino、remote.ino の関係

## 概要

Arduino IDEでは、同じスケッチフォルダ内の複数の`.ino`ファイルは**自動的に1つのプログラムとして統合**されます。このプロジェクトでは、`self_balancing_bike.ino`、`functions.ino`、`remote.ino`の3つのファイルが統合され、機能を分離することでコードの可読性と保守性を向上させています。

## ファイルの役割分担

### self_balancing_bike.ino（メインプログラム）

**役割**: プログラムの**エントリーポイント**と**メイン制御フロー**

- `setup()`関数と`loop()`関数を定義
- グローバル変数の宣言
- ピン定義と定数定義
- メイン制御ループの実装
- 他のファイルで定義された関数を呼び出す

### functions.ino（関数定義ファイル）

**役割**: **機能別の関数群**を定義

- MPU6050センサー関連の関数
- モーター制御関数
- エンコーダー読み取り関数
- キャリブレーション関連の関数
- ユーティリティ関数

### remote.ino（リモート制御ファイル）

**役割**: **スマートフォンアプリとの通信処理**を実装

- シリアル通信からのデータ受信
- ジョイスティック入力の処理
- ボタン入力の処理
- 制御パラメータの送信（現在は未使用）

## 関数の呼び出し関係

### self_balancing_bike.ino → functions.ino の呼び出し

#### setup()関数内での呼び出し

```78:130:01.Arduino Code and Library/self_balancing_bike.ino
void setup() {
  // ... 初期化処理 ...
  
  Motor1_control(0);                          // functions.inoで定義
  Motor2_control(0);                          // functions.inoで定義

  attachInterrupt(0, ENC_READ, CHANGE);       // functions.inoで定義
  attachInterrupt(1, ENC_READ, CHANGE);       // functions.inoで定義

  // ... EEPROM読み込み ...
  
  beep();                                     // functions.inoで定義
  angle_setup();                              // functions.inoで定義
}
```

**呼び出される関数**:
- `Motor1_control()`: モーター1の制御
- `Motor2_control()`: モーター2の制御
- `ENC_READ()`: エンコーダー読み取り（割り込み処理）
- `beep()`: ブザー音の再生
- `angle_setup()`: MPU6050の初期化とジャイロオフセット補正

#### loop()関数内での呼び出し

```132:196:01.Arduino Code and Library/self_balancing_bike.ino
void loop() {
  // ...
  
  if (currentT - previousT_1 >= loop_time) {
    readControlParameters();                  // remote.inoで定義
    angle_calc();                             // functions.inoで定義

    // ...
    
    Motor1_control(-pwm);                      // functions.inoで定義
    Motor2_control(0);                         // functions.inoで定義
  }
  
  if (currentT - previousT_2 >= 2000) {    
    battVoltage((double)analogRead(VBAT) / bat_divider);  // functions.inoで定義
    // ...
  }
}
```

**呼び出される関数**:
- `readControlParameters()`: リモート制御パラメータの読み取り（`remote.ino`で定義）
- `angle_calc()`: MPU6050からデータを取得し、角度を計算（`functions.ino`で定義、10ms周期）
- `Motor1_control()`: モーター1の制御（`functions.ino`で定義、PID制御の結果を適用）
- `Motor2_control()`: モーター2の制御（`functions.ino`で定義、現在は停止のみ）
- `battVoltage()`: バッテリー電圧の監視（`functions.ino`で定義、2秒周期）

### self_balancing_bike.ino → remote.ino の呼び出し

#### loop()関数内での呼び出し

```136:139:01.Arduino Code and Library/self_balancing_bike.ino
  if (currentT - previousT_1 >= loop_time) {
    //Tuning();
    readControlParameters();  
    angle_calc();
```

**呼び出される関数**:
- `readControlParameters()`: スマートフォンアプリからのシリアル通信データを読み取り（10ms周期）

**処理内容**:
- シリアル通信からデータを受信
- ジョイスティックまたはボタンのデータを解析
- `steering_remote`と`speed_remote`を更新

## グローバル変数の共有

### self_balancing_bike.inoで定義された変数

以下の変数は`self_balancing_bike.ino`で定義され、`functions.ino`でも使用されます：

#### 状態フラグ
```26:29:01.Arduino Code and Library/self_balancing_bike.ino
bool vertical = false;
bool calibrating = false;
bool calibrated = false;
bool PIDHas = false;
```

- `functions.ino`の`angle_calc()`で`vertical`を更新
- `functions.ino`の`save()`で`calibrated`と`calibrating`を更新
- `functions.ino`の`Tuning()`で`calibrating`を更新

#### センサーデータ変数
```56:63:01.Arduino Code and Library/self_balancing_bike.ino
int16_t  AcY, AcZ, GyX, gyroX, gyroXfilt;

int16_t  AcYc, AcZc;
int16_t  GyX_offset = 0;
int32_t  GyX_offset_sum = 0;

float robot_angle;
float Acc_angle;
```

- `functions.ino`の`angle_calc()`で読み書き
- `functions.ino`の`angle_setup()`で`GyX_offset`を計算

#### エンコーダー変数
```65:68:01.Arduino Code and Library/self_balancing_bike.ino
volatile byte pos;
volatile int motor_counter = 0, enc_count = 0;
int16_t motor_speed;
int32_t motor_pos;
```

- `functions.ino`の`ENC_READ()`で`enc_count`を更新（割り込み処理）
- `self_balancing_bike.ino`の`loop()`で`motor_speed`を計算

#### PIDパラメータ
```31:34:01.Arduino Code and Library/self_balancing_bike.ino
float K1 = 24.00;       //p
float K2 = -20.00;      //i
float K3 = -9.00;       //s
float K4 = -0.00;       //a
```

- `functions.ino`の`Tuning()`で変更
- `functions.ino`の`PidSave()`でEEPROMに保存

#### キャリブレーションデータ構造体
```38:43:01.Arduino Code and Library/self_balancing_bike.ino
struct OffsetsObj {
  int ID;
  int16_t AcY;
  int16_t AcZ;
};
OffsetsObj offsets;
```

- `functions.ino`の`angle_calc()`で使用
- `functions.ino`の`save()`でEEPROMに保存
- `functions.ino`の`Tuning()`で更新

#### その他の変数
```24:24:01.Arduino Code and Library/self_balancing_bike.ino
float Gyro_amount = 0.896;
```

- `functions.ino`の`angle_calc()`で補完フィルタの重みとして使用

```36:36:01.Arduino Code and Library/self_balancing_bike.ino
float loop_time = 10;
```

- `functions.ino`の`angle_calc()`で角度積分の時間間隔として使用

### remote.inoで定義された変数

以下の変数は`remote.ino`で定義され、内部で使用されます：

```1:4:01.Arduino Code and Library/remote.ino
#define    STX          0x02
#define    ETX          0x03
byte cmd[8] = {0, 0, 0, 0, 0, 0, 0, 0};
byte buttonStatus = 0;
```

- **STX**: 通信開始マーカー（0x02）
- **ETX**: 通信終了マーカー（0x03）
- **cmd**: 受信データバッファ
- **buttonStatus**: ボタンの状態を保持（ビットフラグ）

### remote.inoが使用する変数（self_balancing_bike.inoで定義）

以下の変数は`self_balancing_bike.ino`で定義され、`remote.ino`で更新されます：

```69:69:01.Arduino Code and Library/self_balancing_bike.ino
int steering_remote = 0, speed_remote = 0, speed_value = 0, steering_value = STEERING_CENTER;;
```

- **steering_remote**: リモート制御からのステアリング値（`remote.ino`の`getJoystickState()`で更新）
- **speed_remote**: リモート制御からの速度値（`remote.ino`の`getJoystickState()`で更新）

以下の定数も`remote.ino`で使用されます：

```17:18:01.Arduino Code and Library/self_balancing_bike.ino
#define STEERING_MAX    350
#define SPEED_MAX       80
```

- **STEERING_MAX**: ステアリングの最大値（`remote.ino`の`getJoystickState()`で使用）
- **SPEED_MAX**: 速度の最大値（`remote.ino`の`getJoystickState()`で使用）

### functions.inoで定義された定数

以下の定数は`functions.ino`で定義され、`self_balancing_bike.ino`でも使用されます：

```1:25:01.Arduino Code and Library/functions.ino
#define MPU6050 0x68              // Device address
#define ACCEL_CONFIG 0x1C         // Accelerometer configuration address
#define GYRO_CONFIG  0x1B         // Gyro configuration address

// ... レジスタ定義 ...

#define accSens 0             // 0 = 2g, 1 = 4g, 2 = 8g, 3 = 16g
#define gyroSens 1            // 0 = 250rad/s, 1 = 500rad/s, 2 1000rad/s, 3 = 2000rad/s
```

## データフロー図

```
┌─────────────────────────────────────┐
│  self_balancing_bike.ino            │
│  (メインプログラム)                  │
├─────────────────────────────────────┤
│                                      │
│  setup() {                           │
│    Motor1_control(0) ────────────┐  │
│    Motor2_control(0) ────────────┤  │
│    attachInterrupt(ENC_READ) ────┤  │
│    beep() ──────────────────────┤  │
│    angle_setup() ────────────────┤  │
│  }                                 │  │
│                                      │  │
│  loop() {                            │  │
│    readControlParameters() ────────┤  │
│    angle_calc() ────────────────────┤  │
│    Motor1_control() ────────────────┤  │
│    Motor2_control() ────────────────┤  │
│    battVoltage() ───────────────────┤  │
│  }                                   │  │
│                                      │  │
│  グローバル変数:                     │  │
│  - vertical, calibrated, etc.        │  │
│  - robot_angle, AcY, AcZ, GyX       │  │
│  - K1, K2, K3, K4                   │  │
│  - offsets, MyPid                   │  │
│  - steering_remote, speed_remote     │  │
│  - STEERING_MAX, SPEED_MAX          │  │
└─────────────────────────────────────┘  │
                                          │
                    ┌─────────────────────┘
                    │ 関数呼び出し
                    │ グローバル変数の共有
                    │
        ┌───────────┴───────────┐
        │                       │
        ▼                       ▼
┌──────────────────┐  ┌──────────────────┐
│  functions.ino   │  │  remote.ino      │
│  (関数定義ファイル)│  │  (リモート制御)   │
├──────────────────┤  ├──────────────────┤
│                  │  │                  │
│  angle_calc() {  │◄─│  readControlPara-│◄─┘
│    // MPU6050から │  │  meters() {      │
│    // データ取得   │  │    // シリアル通 │
│    // 補完フィルタ │  │    // 信から受信 │
│    // で角度計算   │  │    getJoystick- │
│    // verticalを  │  │    State()       │
│    // 更新         │  │    getButton-   │
│  }                │  │    State()       │
│                   │  │  }               │
│  angle_setup() {  │◄─┘                  │
│    // MPU6050初期 │  │  getJoystick-   │
│    // 化と補正     │  │  State() {      │
│  }                │  │    // ジョイス   │
│                   │  │    // ティック   │
│  Motor1_control()│◄─┘  │    // 処理     │
│  Motor2_control()│      │    // steering │
│                   │      │    // _remote │
│  ENC_READ() {     │      │    // を更新  │
│    // エンコーダー │      │  }            │
│    // 読み取り     │      │               │
│  }                │      │  getButton-   │
│                   │      │  State() {    │
│  battVoltage() {  │◄─┘  │    // ボタン  │
│    // バッテリー   │      │    // 処理    │
│    // 電圧監視     │      │  }            │
│  }                │      │               │
│                   │      │  使用する変数:│
│  beep() {         │◄─┘  │  - steering_   │
│    // ブザー音再生 │      │    remote     │
│  }                │      │  - speed_remote│
│                   │      │  - STEERING_  │
│  save() {         │      │    MAX        │
│    // キャリブレー │      │  - SPEED_MAX  │
│    // ションデータ │      │               │
│    // 保存         │      │  定義する変数:│
│  }                │      │  - buttonStatus│
│                   │      │  - cmd[]      │
│  Tuning() {       │      │               │
│    // PIDパラメー │      │               │
│    // ータ調整     │      │               │
│  }                │      │               │
└──────────────────┘      └──────────────────┘
```

## 関数の詳細な関係

### 1. 角度計算関連

#### angle_calc() - 角度計算関数

**定義場所**: `functions.ino`

**呼び出し元**: `self_balancing_bike.ino`の`loop()`関数内（10ms周期）

**処理内容**:
- MPU6050から加速度計とジャイロのデータを取得
- キャリブレーション値を適用
- 補完フィルタで角度を計算
- `robot_angle`を更新
- `vertical`フラグを更新

**使用する変数**（`self_balancing_bike.ino`で定義）:
- `AcY`, `AcZ`, `GyX`: センサーデータ
- `offsets`: キャリブレーション値
- `GyX_offset`: ジャイロオフセット
- `robot_angle`: 計算結果を格納
- `Gyro_amount`: 補完フィルタの重み
- `loop_time`: 積分の時間間隔
- `vertical`: 垂直判定フラグ

#### angle_setup() - MPU6050初期化関数

**定義場所**: `functions.ino`

**呼び出し元**: `self_balancing_bike.ino`の`setup()`関数内（起動時1回）

**処理内容**:
- I2C通信の初期化
- MPU6050の設定
- ジャイロオフセットの自動補正（1024サンプル）
- `GyX_offset`を計算

**使用する変数**:
- `GyX_offset_sum`: オフセット計算用の累積値
- `GyX_offset`: 計算結果を格納

### 2. モーター制御関連

#### Motor1_control() / Motor2_control() - モーター制御関数

**定義場所**: `functions.ino`

**呼び出し元**: 
- `self_balancing_bike.ino`の`setup()`: 初期化時に停止
- `self_balancing_bike.ino`の`loop()`: PID制御の結果を適用

**処理内容**:
- PWM値から方向と速度を決定
- DIRピンで回転方向を制御
- PWMピンで速度を制御

**使用する定数**（`self_balancing_bike.ino`で定義）:
- `PWM_1`, `DIR_1`: モーター1のピン
- `PWM_2`, `DIR_2`: モーター2のピン

### 3. エンコーダー関連

#### ENC_READ() - エンコーダー読み取り関数

**定義場所**: `functions.ino`

**呼び出し元**: 割り込み処理（`self_balancing_bike.ino`の`setup()`で登録）

**処理内容**:
- エンコーダーのA/B相を読み取り
- 4倍カウントを実現
- `enc_count`を更新

**使用する変数**:
- `pos`: エンコーダーの状態を保持（volatile）
- `enc_count`: カウント値を更新（volatile）

**使用する定数**:
- `ENC_1`, `ENC_2`: エンコーダーピン

### 4. キャリブレーション関連

#### save() - キャリブレーションデータ保存関数

**定義場所**: `functions.ino`

**呼び出し元**: `functions.ino`の`Tuning()`関数内（`c-`コマンド時）

**処理内容**:
- `offsets`構造体をEEPROMに保存
- `calibrated`フラグを`true`に設定
- `calibrating`フラグを`false`に設定

**使用する変数**:
- `offsets`: キャリブレーションデータ（`self_balancing_bike.ino`で定義）
- `calibrated`, `calibrating`: 状態フラグ

#### Tuning() - PIDパラメータ調整関数

**定義場所**: `functions.ino`

**呼び出し元**: 現在はコメントアウト（`self_balancing_bike.ino`の`loop()`内）

**処理内容**:
- シリアル通信でPIDパラメータを調整
- `K1`, `K2`, `K3`, `K4`を変更
- EEPROMに保存

**使用する変数**:
- `K1`, `K2`, `K3`, `K4`: PIDパラメータ
- `calibrating`: キャリブレーションモードの状態
- `offsets`: キャリブレーションデータ

### 5. ユーティリティ関数

#### beep() - ブザー音再生関数

**定義場所**: `functions.ino`

**呼び出し元**: 
- `self_balancing_bike.ino`の`setup()`: 起動準備完了の合図
- `functions.ino`の`angle_setup()`: 補正完了の合図
- `functions.ino`の`save()`: 保存完了の合図
- `functions.ino`の`PidSave()`: 保存完了の合図

**使用する定数**:
- `BUZZER`: ブザーピン（`self_balancing_bike.ino`で定義）

#### battVoltage() - バッテリー電圧監視関数

**定義場所**: `functions.ino`

**呼び出し元**: `self_balancing_bike.ino`の`loop()`内（2秒周期）

**処理内容**:
- バッテリー電圧を読み取り
- 低電圧時にブザーを鳴らす

**使用する定数**:
- `VBAT`: バッテリー電圧監視ピン
- `BUZZER`: ブザーピン

### 6. リモート制御関連（remote.ino）

#### readControlParameters() - リモート制御パラメータ読み取り関数

**定義場所**: `remote.ino`

**呼び出し元**: `self_balancing_bike.ino`の`loop()`関数内（10ms周期）

**処理内容**:
- シリアル通信からデータを受信
- STX（0x02）で始まるデータパケットを検出
- ETX（0x03）で終わるデータパケットを解析
- ボタンデータ（3バイト）またはジョイスティックデータ（7バイト）を処理

**通信プロトコル**:
- **ボタンデータ**: `<STX> <ボタン文字> <ETX>`（例: `<STX> "C" <ETX>`）
- **ジョイスティックデータ**: `<STX> <X座標3桁> <Y座標3桁> <ETX>`（例: `<STX> "200" "180" <ETX>`）

```6:23:01.Arduino Code and Library/remote.ino
void readControlParameters() {
  if (Serial.available())  {                           // data received from smartphone
    //delay(1);
    cmd[0] =  Serial.read();  
    if(cmd[0] == STX)  {
      int i = 1;      
      while (Serial.available())  {
        //delay(1);
        cmd[i] = Serial.read();
        if(cmd[i] > 127 || i > 7)                 break;     // Communication error
        if((cmd[i] == ETX) && (i == 2 || i == 7)) break;     // Button or Joystick data
        i++;
      }
      if (i == 2) getButtonState(cmd[1]);                  // 3 Bytes  ex: < STX "C" ETX >
      else if (i == 7) getJoystickState(cmd);              // 6 Bytes  ex: < STX "200" "180" ETX >
    }
  } 
}
```

#### getJoystickState() - ジョイスティック状態取得関数

**定義場所**: `remote.ino`

**呼び出し元**: `remote.ino`の`readControlParameters()`関数内

**処理内容**:
- ASCII文字列から数値を抽出（例: "200" → 200）
- オフセット200を引いて-100～+100の範囲に変換
- デッドゾーン処理（±10以内は0）
- 指数関数的な応答特性を適用
- `steering_remote`と`speed_remote`を更新

```25:44:01.Arduino Code and Library/remote.ino
void getJoystickState (byte data[8])    {
  int joyX = (data[1] - 48) * 100 + (data[2] - 48) * 10 + (data[3] - 48);   // obtain the Int from the ASCII representation
  int joyY = (data[4] - 48) * 100 + (data[5] - 48) * 10 + (data[6] - 48);
  joyX = joyX - 200;                                                        // Offset to avoid
  joyY = joyY - 200;                                                        // transmitting negative numbers

  if (joyX < -100 || joyX > 100 || joyY < -100 || joyY > 100) return;       // commmunication error
  if (joyX < - 10 || joyX > 10)  { // dead zone
    if (joyX > 0) // exponential
      steering_remote = (-joyX * joyX + 0.1 * joyX) / 100.0;
    else   
      steering_remote = (joyX * joyX + 0.1 * joyX) / 100.0;
  } else 
      steering_remote = 0;
  steering_remote = -map(steering_remote, -100, 100, -STEERING_MAX, STEERING_MAX);
  if (joyY < - 10 || joyY > 10)  // dead zone 
     speed_remote = map(joyY, 100, -100, SPEED_MAX, -SPEED_MAX);
  else
     speed_remote = 0;        
}
```

**使用する変数**（`self_balancing_bike.ino`で定義）:
- `steering_remote`: ステアリング値を更新
- `speed_remote`: 速度値を更新
- `STEERING_MAX`: ステアリングの最大値
- `SPEED_MAX`: 速度の最大値

**特徴**:
- **デッドゾーン**: ±10以内の入力は無視（ノイズ対策）
- **指数関数的応答**: ジョイスティックの中央付近で感度を下げる
- **範囲マッピング**: -100～+100の値を`STEERING_MAX`/`SPEED_MAX`の範囲にマッピング

#### getButtonState() - ボタン状態取得関数

**定義場所**: `remote.ino`

**呼び出し元**: `remote.ino`の`readControlParameters()`関数内

**処理内容**:
- ボタンのON/OFF状態を`buttonStatus`に記録
- ビットフラグで複数のボタン状態を管理

```46:63:01.Arduino Code and Library/remote.ino
void getButtonState (int bStatus)  {
  switch (bStatus) {
// -----------------  BUTTON #1  -----------------------
   case 'A':
      buttonStatus |= B000001;        // ON
      break;
    case 'B':
      buttonStatus &= B111110;        // OFF
      break;
// -----------------  BUTTON #2  -----------------------
   case 'C':
      buttonStatus |= B000010;        // ON    
      break;
    case 'D':
      buttonStatus &= B111101;        // OFF    
      break;
  }
}
```

**ボタンコマンド**:
- `'A'`: ボタン1をON（ビット0をセット）
- `'B'`: ボタン1をOFF（ビット0をクリア）
- `'C'`: ボタン2をON（ビット1をセット）
- `'D'`: ボタン2をOFF（ビット1をクリア）

#### sendControlParameters() - 制御パラメータ送信関数

**定義場所**: `remote.ino`

**呼び出し元**: 現在は未使用（将来の拡張用）

**処理内容**:
- ボタン状態とその他のデータをシリアル通信で送信
- スマートフォンアプリへのフィードバック用

```74:81:01.Arduino Code and Library/remote.ino
void sendControlParameters() {
  Serial.print((char)STX);                                                // Start of Transmission
  Serial.print(getButtonStatusString());  Serial.print((char)0x1);        // buttons status feedback
  Serial.print(0);                        Serial.print((char)0x4);        // datafield #1
  Serial.print(0);                        Serial.print((char)0x5);        // datafield #2
  Serial.print(0);                                                        // datafield #3
  Serial.print((char)ETX);                                                // End of Transmission
}
```

## コンパイル時の統合

Arduino IDEでは、同じスケッチフォルダ内の`.ino`ファイルは以下の順序で統合されます：

1. **メインファイル**（フォルダ名と同じ名前のファイル、または最初のファイル）
2. **その他のファイル**（アルファベット順）

このプロジェクトの場合：
- `self_balancing_bike.ino`がメインファイルとして扱われる
- `functions.ino`と`remote.ino`が追加ファイルとして統合される（アルファベット順）

**統合順序**:
1. `self_balancing_bike.ino`（メインファイル）
2. `functions.ino`（アルファベット順）
3. `remote.ino`（アルファベット順）

**重要なポイント**:
- 関数の定義順序は関係ない（前方宣言は不要）
- グローバル変数はすべてのファイルで共有される
- `#define`で定義された定数もすべてのファイルで有効

## 設計上の利点

### 1. コードの分離

- **メインロジック**と**機能実装**を分離
- 各ファイルの役割が明確
- コードの可読性が向上

### 2. 保守性の向上

- 機能ごとにファイルを分離することで、修正箇所が特定しやすい
- 関数の追加・削除が容易

### 3. 再利用性

- `functions.ino`の関数は他のプロジェクトでも再利用可能
- モジュール化された設計

### 4. デバッグの容易さ

- 各ファイルを独立して確認できる
- 関数単位でのテストが可能

## ファイル間のデータフロー

### リモート制御のデータフロー

```
スマートフォンアプリ
    ↓ (シリアル通信)
remote.ino
    ├─ readControlParameters()
    ├─ getJoystickState() → steering_remote, speed_remote を更新
    └─ getButtonState() → buttonStatus を更新
    ↓
self_balancing_bike.ino
    ├─ loop() で steering_remote, speed_remote を読み取り
    ├─ ステアリング制御に使用
    └─ サーボ制御に使用
```

### 角度計算のデータフロー

```
MPU6050センサー
    ↓ (I2C通信)
functions.ino
    ├─ angle_calc() → robot_angle, vertical を更新
    └─ angle_setup() → GyX_offset を計算
    ↓
self_balancing_bike.ino
    ├─ loop() で robot_angle を使用してPID制御
    └─ vertical で動作条件をチェック
```

### モーター制御のデータフロー

```
self_balancing_bike.ino
    ├─ loop() でPID制御値を計算
    └─ PWM値を計算
    ↓
functions.ino
    ├─ Motor1_control() → モーター1を制御
    └─ Motor2_control() → モーター2を制御
```

## まとめ

`self_balancing_bike.ino`、`functions.ino`、`remote.ino`の関係は：

1. **`self_balancing_bike.ino`**: メインプログラム
   - プログラムのエントリーポイント
   - メイン制御フロー
   - グローバル変数の定義
   - 他のファイルの関数を呼び出して統合

2. **`functions.ino`**: ハードウェア制御関数定義ファイル
   - MPU6050センサー関連の関数
   - モーター制御関数
   - エンコーダー読み取り関数
   - キャリブレーション関連の関数
   - ユーティリティ関数

3. **`remote.ino`**: リモート制御関数定義ファイル
   - シリアル通信からのデータ受信
   - ジョイスティック入力の処理
   - ボタン入力の処理
   - 制御パラメータの送信（将来の拡張用）

4. **統合**: Arduino IDEが自動的に1つのプログラムとしてコンパイル

5. **データ共有**: グローバル変数と定数がすべてのファイルで共有される

### 設計の利点

- **機能分離**: 各ファイルが明確な役割を持つ
- **保守性**: 機能ごとにファイルが分かれているため、修正箇所が特定しやすい
- **拡張性**: 新しい機能を追加する際に、適切なファイルに追加できる
- **再利用性**: 各ファイルの関数は他のプロジェクトでも再利用可能

この設計により、コードの可読性、保守性、再利用性が向上しています。

