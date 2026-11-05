# 第5回 USB HID

## 1. なぜUSB HIDを学ぶのか

これまで、XIAO RP2040に接続した部品から入力を読み取ってきました。

第3回では、

**タクトスイッチ  
→ デジタル入力  
→ HIGH / LOW**

を扱いました。

第4回では、

**アナログスティックなど  
→ アナログ入力  
→ 数値**

を扱いました。

今回は、それらの入力をUSBを通してPCへ送ります。

今日の全体の流れは、

**物理入力  
→ XIAO RP2040  
→ USB HID  
→ PC**

です。

XIAO RP2040を、PCから見て

- キーボード
- ゲームパッド

として動作させます。

---

# 2. 今日の目標

- USB HIDとは何かを理解する
- HIDのDescriptorとReportの役割を理解する
- XIAO RP2040をUSBキーボードとして動作させる
- デジタル入力をキーボードのキーとして送る
- デジタル入力をゲームパッドのボタンとして送る
- アナログ入力をゲームパッドのX/Y軸として送る
- Windows / Macで入力を確認する

---

# 3. USB HIDとは

HIDは、

**Human Interface Device**

の略です。

人がコンピュータを操作するための入力機器などを扱うUSBの仕組みです。

代表例には、

- キーボード
- マウス
- ゲームパッド
- ジョイスティック

などがあります。

HIDはUSBで標準化されています。

そのため、一般的なHID機器は、機器ごとに専用ドライバをインストールしなくても、OSが持っている標準的な仕組みを使って動作できます。

---

# 4. HIDでは何を送るのか

HIDでは、機器によって送る情報が異なります。

## キーボード

例えば、

**Aキーが押されている**

という状態を送ります。

## ゲームパッド

例えば、

```text
Button 1 = ON
X = 700
Y = 420
```

のような状態を送ります。

今回使うJoystickライブラリの標準設定では、X軸とY軸の値は

**0〜1023**

です。

したがって、

- 0：一方の端
- 約512：中央
- 1023：反対側の端

に対応します。

`X = 700` は、この0〜1023の範囲の中の一例です。

---

# 5. DescriptorとReport

USB HIDを理解するために、2つの言葉を覚えます。

## Descriptor

**この機器には何があるか**

をPCへ説明する情報です。

例えばゲームパッドなら、

- X軸がある
- Y軸がある
- ボタンがある

といった構成をPCへ知らせます。

## Report

**現在、その機器がどのような状態か**

を表すデータです。

例えば、

```text
X = 700
Y = 420
Button 1 = ON
```

という情報です。

まとめると、

**Descriptor：機器の構造**

**Report：現在の状態**

です。

今回の演習では、Descriptorを一から作ることはしません。

Arduino-Picoのライブラリを使い、

**どの入力を、どのHID情報としてPCへ送るか**

に注目します。

---

# 6. Arduino IDEのUSB設定を確認する

HIDプログラムを書く前に、Arduino IDEの設定を確認します。

Arduino IDEで、

**Tools → USB Stack**

を開いてください。

今回使う `Keyboard`、`Mouse`、`Joystick` は、Arduino-Pico標準のUSB機能を使用します。

**Arduino-Pico標準のPico SDK側のUSB Stack**

が選択されていることを確認してください。

設定が異なっていると、今回のサンプルがそのままコンパイル・実行できない場合があります。

---

# 7. サンプル1：タクトスイッチをUSB Keyboardにする

まず、第3回で使ったタクトスイッチを使います。

内部プルアップを使用します。

回路は、

**D2 → タクトスイッチ → GND**

です。

したがって、

- 押していない：HIGH
- 押している：LOW

になります。

## プログラム

```cpp
#include <Keyboard.h>

const int switchPin = D2;

bool previousPressed = false;

void setup() {
  pinMode(switchPin, INPUT_PULLUP);

  Keyboard.begin();
}

void loop() {
  bool pressed = (digitalRead(switchPin) == LOW);

  if (pressed && !previousPressed) {
    Keyboard.press('a');
  }

  if (!pressed && previousPressed) {
    Keyboard.release('a');
  }

  previousPressed = pressed;

  delay(10);
}
```

---

# 8. Keyboard HIDを確認する

1. XIAO RP2040をUSBでPCに接続します。
2. **文字を入力できるアプリ（エディタなど）を開きます。**
   - 例：Windowsのメモ帳、MacのTextEdit
3. 文字を入力できる場所をクリックします。
4. タクトスイッチを押します。
5. `a` が入力されることを確認します。
6. スイッチを離すと入力が止まることを確認します。

スイッチを押し続けると、OSのキーリピート機能によって、

```text
aaaaaaaa...
```

のように入力されることがあります。

これはXIAO RP2040が何度も `Keyboard.press()` を送っているのではなく、

**Aキーが押された状態が続いている**

とPCが判断しているためです。

---

# 9. プログラムを読む

まず、現在のスイッチ状態を読みます。

```cpp
bool pressed = (digitalRead(switchPin) == LOW);
```

`pressed` は、

- 押されている：`true`
- 押されていない：`false`

になります。

次に、

```cpp
if (pressed && !previousPressed) {
  Keyboard.press('a');
}
```

では、

**前回は押されていなかったが、今回は押されている**

場合だけキーを押します。

逆に、

```cpp
if (!pressed && previousPressed) {
  Keyboard.release('a');
}
```

では、

**前回は押されていたが、今回は押されていない**

場合だけキーを離します。

最後に、

```cpp
previousPressed = pressed;
```

で、今回の状態を次回のために覚えておきます。

---

# 10. なぜ previousPressed が必要なのか

もし、

```cpp
if (pressed) {
  Keyboard.press('a');
} else {
  Keyboard.release('a');
}
```

とすると、`loop()` が実行されるたびに同じ処理を繰り返すことになります。

今回は、

**状態が変化したときだけ処理する**

ようにします。

そのため、

- 現在の状態：`pressed`
- 前回の状態：`previousPressed`

を比較しています。

キーを複数使用する場合には、それぞれのキーについて前回の状態を覚える必要があります。

キー数が増えた場合には、配列などを使って管理することもできます。

---

# 11. スイッチのチャタリング

実際のスイッチでは、押した瞬間や離した瞬間に、

ON / OFFが短時間に何度か切り替わったように見えることがあります。

これを、

**チャタリング**

と呼びます。

今回の基本課題では、チャタリング対策までは行いません。

もし意図しない複数回の入力が起きた場合には、

**「スイッチの状態は本当に一度だけ変化しているのか」**

を考えてみてください。

---

# 12. 注意：XIAO RP2040は本当にキーボードになる

HIDプログラムを書き込むと、XIAO RP2040はPCから本物のキーボードとして扱われます。

プログラムを間違えると、

- キーを入力し続ける
- 意図しないショートカットを実行する

ことがあります。

基本課題では、まず `a` などの通常の文字キーを使います。

動作がおかしくなった場合には、USBケーブルを抜けば入力を止められます。

---

# 13. 同じスイッチをGamepad Buttonにする

次は、同じタクトスイッチを

**ゲームパッドのButton 1**

として使います。

回路は変更しません。

変わるのは、

**PCへ何として送るか**

です。

---

# 14. サンプル2：タクトスイッチ → Gamepad Button

```cpp
#include <Joystick.h>

const int switchPin = D2;

bool previousPressed = false;

void setup() {
  pinMode(switchPin, INPUT_PULLUP);

  Joystick.begin();
}

void loop() {
  bool pressed = (digitalRead(switchPin) == LOW);

  if (pressed != previousPressed) {
    Joystick.button(1, pressed);
  }

  previousPressed = pressed;

  delay(10);
}
```

---

# 15. Keyboardとの違いを考える

入力部分はどちらも、

```cpp
digitalRead()
```

です。

物理回路も同じです。

しかしKeyboardでは、

```cpp
Keyboard.press()
Keyboard.release()
```

を使いました。

Gamepadでは、

```cpp
Joystick.button()
```

を使います。

つまり、

**同じ物理入力でも、プログラムによってPC側での意味を変えることができます。**

---

# 16. Gamepadの動作を確認する

Gamepad HIDの動作確認には、次のWebページを使用します。

**Controller Test**

https://controller-test.com/

Windows / Macのどちらでも同じページを使います。

1. XIAO RP2040をUSBでPCに接続します。
2. Controller Testを開きます。
3. タクトスイッチを一度押します。
4. Gamepadが認識されたことを確認します。
5. タクトスイッチを押し、Buttonの表示が変化することを確認します。

ブラウザでは、Gamepadを接続しただけでは表示されず、最初にボタンなどを操作してから認識される場合があります。

---

# 17. デジタル入力からアナログ入力へ

ここまでは、

**押している / 押していない**

という2つの状態を扱いました。

次は、第4回で使ったアナログスティックを使います。

前回すでに、

- アナログスティックを接続する
- X軸とY軸を `analogRead()` で読む
- スティックを動かして値が変化することを確認する

ところまで行っています。

今回は、そのアナログ値を

**PCのGamepadのX/Y軸**

として使います。

---

# 18. サンプル3：アナログスティック → Gamepad X/Y

RP2040のADCは12 bitです。

今回は、

```cpp
analogReadResolution(12);
```

として、

**0〜4095**

の範囲で読み取ります。

一方、Arduino-PicoのJoystick HIDは標準設定では、

**0〜1023**

の軸値を使います。

そこで `map()` を使います。

```cpp
#include <Joystick.h>

const int xPin = A0;
const int yPin = A1;

void setup() {
  analogReadResolution(12);

  Joystick.begin();
}

void loop() {
  int x = analogRead(xPin);
  int y = analogRead(yPin);

  int xHid = map(x, 0, 4095, 0, 1023);
  int yHid = map(y, 0, 4095, 0, 1023);

  Joystick.position(xHid, yHid);

  delay(10);
}
```

---

# 19. プログラムの流れ

今回の処理は、

**スティック位置  
→ 電圧  
→ ADC値  
→ HID軸値  
→ USB  
→ PC**

です。

まず、

```cpp
int x = analogRead(xPin);
int y = analogRead(yPin);
```

で、X軸とY軸を読みます。

これは第4回で行ったことと同じです。

次に、

```cpp
map(x, 0, 4095, 0, 1023)
```

で、

**0〜4095**

を

**0〜1023**

へ変換します。

最後に、

```cpp
Joystick.position(xHid, yHid);
```

で、X軸とY軸の現在値をPCへ送ります。

---

# 20. アナログスティックの動作を確認する

Controller Testを開いたまま、

アナログスティックを

- 左
- 右
- 上
- 下

へ動かしてください。

X軸とY軸の表示が変化することを確認します。

次に、スティックから手を離してください。

中央付近へ戻ることも確認します。

---

# 21. 今回の送信方法

今回のサンプルでは、

### デジタル入力

タクトスイッチの状態が

**変化したときだけ**

HIDへ送ります。

### アナログ入力

現在のX/Y軸の値を

**10 msごとに**

HIDへ送ります。

これはHIDで必ずこの方法にしなければならない、という意味ではありません。

今回は、入力の性質に合わせてこのようなプログラムにしています。

---

# 22. 実際の値は0〜4095いっぱいまで使うとは限らない

プログラムでは、

```text
ADC：0〜4095
HID：0〜1023
```

として変換しました。

しかし、実際のアナログスティックが必ずADCの0や4095まで到達するとは限りません。

例えば、

```text
左端：180
中央：2050
右端：3920
```

のようになることもあります。

前回の実験でも、

**実際の部品がどのような値を出すか**

を確認しました。

仕様上の範囲と、実物で測定した範囲は必ずしも同じではありません。

---

# 23. 実際に測った範囲を使う

例えば、前回確認したX軸の範囲が、

```text
180〜3920
```

だった場合、

```cpp
int xHid = map(x, 180, 3920, 0, 1023);
```

とすれば、実際に使えるスティックの範囲を、HIDの0〜1023へ対応させることができます。

これは、

**キャリブレーション**

の基本的な考え方です。

ただし、個体によって測定値は異なる可能性があります。

自分の部品で確認した値を使う必要があります。

---

# 24. 中央位置も完全には一致しない

スティックから手を離したときも、必ず

```text
X = 512
Y = 512
```

になるとは限りません。

例えば、

```text
X = 508
Y = 520
```

となることがあります。

また、停止していても数値が少し変化することがあります。

これは、

- 部品の個体差
- 機械的な中央位置
- ADC値の変動

などによって起こります。

---

# 25. 発展：デッドゾーン

ゲームのコントローラでは、スティックに触っていないのに少し入力され続けると困ります。

そこで、中央付近を

**入力なし**

として扱うことがあります。

この範囲を

**デッドゾーン**

と呼びます。

例えば、

```text
490〜534 → 512として扱う
```

と決めることができます。

第3回で使った `if` 文を利用すれば実装できます。

---

# 26. 自分で変更してみる

次の中から1つ以上選び、変更してください。

## A. Keyboardのキーを変更する

`a` ではなく、別の文字キーを送ってみてください。

## B. Gamepad Buttonを追加する

タクトスイッチを追加し、

- Button 1
- Button 2

を作ってください。

## C. X軸を逆にする

スティックを左へ動かしたときに値が大きくなるよう変更してください。

## D. Y軸を逆にする

Y軸について同じことを行ってください。

## E. 実測した範囲を使う

自分のアナログスティックで確認した最小値・最大値を使って、`map()` の範囲を変更してください。

## F. デッドゾーンを作る

中央付近では常に中央値を送るようにしてください。

---

# 27. 発展：マイコン内蔵LEDスティックを光らせる

今回使用するLEDスティックには、WS2812Bというマイコン内蔵RGB LEDが8個並んでいます。

通常のLEDとは違い、それぞれのLEDに対して、

- 赤
- 緑
- 青
- 明るさ

を個別に指定できます。

今回使うLEDスティックは、

**AE-WS2812B-STICK8**

です。

## 配線

LEDスティックの入力側を使います。

- 5V → 5V
- GND → GND
- DIN → XIAO RP2040のデジタル出力ピン

例えば、DINをD3へ接続します。

> DOUT側ではなく、DIN側へ接続してください。

---

## ライブラリを準備する

Arduino IDEのライブラリマネージャから、

**Adafruit NeoPixel**

を検索してインストールします。

---

## サンプル：8個のLEDを順番に光らせる

```cpp
#include <Adafruit_NeoPixel.h>

const int ledPin = D3;
const int ledCount = 8;

Adafruit_NeoPixel pixels(ledCount, ledPin, NEO_GRB + NEO_KHZ800);

void setup() {
  pixels.begin();
  pixels.clear();
  pixels.show();
}

void loop() {
  for (int i = 0; i < ledCount; i++) {
    pixels.clear();

    pixels.setPixelColor(
      i,
      pixels.Color(0, 50, 0)
    );

    pixels.show();

    delay(200);
  }
}
```

---

## プログラムを読む

```cpp
Adafruit_NeoPixel pixels(ledCount, ledPin, NEO_GRB + NEO_KHZ800);
```

で、

- LEDの個数
- 使用するGPIO
- LEDの通信方式

を設定しています。

---

### LEDの色を指定する

```cpp
pixels.Color(0, 50, 0)
```

では、

```text
赤 = 0
緑 = 50
青 = 0
```

なので、緑色になります。

例えば、

```cpp
pixels.Color(50, 0, 0)
```

なら赤、

```cpp
pixels.Color(0, 0, 50)
```

なら青になります。

---

### どのLEDを光らせるか指定する

```cpp
pixels.setPixelColor(i, ...);
```

で、LED番号を指定します。

8個のLEDは、

```text
0 1 2 3 4 5 6 7
```

と番号が付いています。

---

### LEDへデータを送る

```cpp
pixels.show();
```

を実行すると、設定した色がLEDへ送られます。

---

## HIDと組み合わせてみる

ここまでできたら、HID入力とLED出力を組み合わせることもできます。

例えば、

- Gamepad Buttonを押す → LEDを赤くする
- スティックを右へ動かす → 右側のLEDを光らせる
- スティック位置に応じて光るLEDを移動させる

といった装置を作ることができます。

例えば、

**アナログスティック**  
**→ `analogRead()`**  
**→ LED番号へ変換**  
**→ WS2812B**

という流れです。

---

## 発展課題

次の中から1つ試してください。

- LEDの色を変える
- 8個全部を別々の色で光らせる
- 左から右へ光を移動させる
- アナログスティックの位置に応じて光るLEDを変える
- タクトスイッチを押したときだけLEDを光らせる
- Gamepad HIDとLEDスティックを同時に動かす

---

# 28. 今日やったことをつなげる

第3回では、

**タクトスイッチ  
→ `digitalRead()`**

を学びました。

第4回では、

**アナログスティック  
→ `analogRead()`  
→ `map()`**

を学びました。

今日は、それらをUSB HIDへつなぎました。

### デジタル入力

**タクトスイッチ  
→ `digitalRead()`  
→ Keyboard / Gamepad Button**

### アナログ入力

**アナログスティック  
→ `analogRead()`  
→ `map()`  
→ Gamepad X/Y**

つまり、新しいものを一から作ったわけではありません。

これまで学んだ機能を組み合わせることで、

**自分で作った回路をPCの入力デバイスにする**

ことができました。

---

# 29. 今日の到達点

ここまでできれば、今日の基本課題は完了です。

- HIDとは何か説明できる
- DescriptorとReportの違いを説明できる
- USB Stackの設定を確認できる
- XIAO RP2040をUSB Keyboardとして使える
- デジタル入力をKeyboardのキーとして送れる
- デジタル入力をGamepad Buttonとして送れる
- アナログ入力をGamepad X/Y軸として送れる
- Windows / Macで入力を確認できる
- 「入力 → 処理 → HID → USB → PC」の流れを説明できる

---

# 30. 振り返り

次のことを自分の言葉で説明してください。

1. HIDとは何ですか。
2. DescriptorとReportは何が違いますか。
3. タクトスイッチをKeyboardにもGamepadにもできるのはなぜですか。
4. デジタル入力では、なぜ前回の状態を記録しましたか。
5. 今回のアナログ入力では、なぜ値を繰り返し送っていますか。
6. `analogRead()` の0〜4095を、そのままJoystickへ渡さなかったのはなぜですか。
7. 実際のアナログスティックが理論上の最大値・最小値まで届くとは限らないのはなぜでしょうか。
8. 今日のプログラムで、自分が変更したところはどこですか。

---

# 次回

次回は、

**ミニプロジェクト**

に取り組みます。

これまでに使った、

- LED
- タクトスイッチ
- 可変抵抗
- CdS
- ジョイスティック
- デジタル入力
- アナログ入力
- PWM
- USB HID

などを組み合わせます。

目的や動作を自分で決め、

**必要なことを調べ、確かめながら、自分の装置を作ります。**
