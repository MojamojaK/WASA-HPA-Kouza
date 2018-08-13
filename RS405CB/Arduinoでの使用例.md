# Arduinoでの使用例

操舵系統の[レポジトリ](https://github.com/MojamojaK/WASA-Control/blob/master/futaba_servo.h)をみてね

自分で解読してみるとええよ(きれいに書いたつもり)

#### 回路例

公式のシステム構成例は[こちら](http://www.futaba.co.jp/img/uploads/files/robot/download/software/RS485接続例.pdf)

注意点は[こちら](http://www.futaba.co.jp/img/uploads/files/robot/command_type_servos/RS405CB_406CB_108.pdf#page=15)

#### 送信例
```C++

#define TAIL_COMM_ENABLE_PIN 8
#define SERVO_SERIAL Serial2

uint8_t servo_tx_packet[256];

void setup() {
  SERVO_SERIAL.begin(9600);
  pinMode(TAIL_COMM_ENABLE_PIN, OUTPUT);
  servo_reboot(1); // 起動する
  servo_torque_value(1, 1); // トルクモードをONにする
}

void loop() {
  servo_move(1, -1500, 200);
  delay(2000);
  servo_move(1, 1500, 200);
  delay(2000);
}

void servo_reboot(uint8_t id) {
  servo_tx_packet[0] = 0xFA;                                     //Header
  servo_tx_packet[1] = 0xAF;                                     //Header
  servo_tx_packet[2] = id;                                       //ID
  servo_tx_packet[3] = 0x20;                                     //Flags
  servo_tx_packet[4] = 0xFF;                                     //Address
  servo_tx_packet[5] = 0x00;                                     //Length
  servo_tx_packet[6] = 0x00;                                     //Count
  servo_tx_packet[7] = checksum(servo_tx_packet, 7);             //sum

  transmit_packet(8);
  delay(30);
}

void servo_set_torque_mode(uint8_t id, uint8_t state) {
  servo_tx_packet[0] = 0xFA;                                     //Header
  servo_tx_packet[1] = 0xAF;                                     //Header
  servo_tx_packet[2] = id;                                       //ID
  servo_tx_packet[3] = 0x00;                                     //Flags
  servo_tx_packet[4] = 0x24;                                     //Address
  servo_tx_packet[5] = 0x01;                                     //Length
  servo_tx_packet[6] = 0x01;                                     //Count
  servo_tx_packet[7] = (uint8_t) state & 0x00FF;                    //ON/OFF
  servo_tx_packet[8] = checksum(servo_tx_packet, 8);               //sum

  if (!(state == 0 || state == 1 || state == 2)) return;
  transmit_packet(9);
}

void servo_move(uint8_t id, uint16_t o_angle, uint16_t o_time) {
  servo_tx_packet[0] = 0xFA;                            //Header
  servo_tx_packet[1] = 0xAF;                            //Header
  servo_tx_packet[2] = id;                              //ID
  servo_tx_packet[3] = 0x00;                            //Flags
  servo_tx_packet[4] = 0x1E;                            //Address
  servo_tx_packet[5] = 0x04;                            //Length
  servo_tx_packet[6] = 0x01;                            //Count
  servo_tx_packet[7] = lowByte(o_angle);                //目標位置データ(下位バイト)
  servo_tx_packet[8] = highByte(o_angle);               //目標位置データ(上位バイト)
  servo_tx_packet[9] = lowByte(o_time);                 //目標時間データ(下位バイト)
  servo_tx_packet[10] = highByte(o_time);               //目標時間データ(上位バイト)
  servo_tx_packet[11] = checksum(servo_tx_packet, 11);    //Checksum

  transmit_packet(12);
}

void transmit_packet(uint8_t len) {
  digitalWrite(TAIL_COMM_ENABLE_PIN, HIGH);       //送信許可
  SERVO_SERIAL.write(servo_tx_packet, len);       //サーボに送信
  SERVO_SERIAL.flush();                           //リードバッファを初期化(送信データがすべて送信されるまで待つ)
  digitalWrite(TAIL_COMM_ENABLE_PIN, LOW);        //送信禁止
}

uint8_t checksum(uint8_t * data, uint8_t len) {
  uint8_t sum = 0;
  for (uint8_t i = 2; i < len; i++) sum ^= data[i];
  return sum;
}
```

受信については操舵系統の [レポジトリ](https://github.com/MojamojaK/WASA-Control/blob/master/futaba_servo.h) \(268〜353行目\)でみてね。

安全性を保ちながらまあ効率的に受信していると思う。