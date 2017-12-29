# Умный сенсор CC41-A

Разработку умного сенсора на базе BLE-модуля CC41-A осуществим на примере датчика влажности, который можно установить в цветочный горшок любимого растения вашей любимой половинки.

Умный датчик влажности в составе умной квартиры напомнит о необходимости полить растение, с учетом его влаголюбивости или засухоустойчивости, например, отправив сообщение на телефон.

Как обычно наши схемы ориентированы на любителей, но могут быть легко усовершенствованы специалистами. Особенных требований к оборудованию нет. Вам даже не потребуется паяльник. 

Итак, компоненты умного датчика:

|Название|Назначение|Цена, руб.|
| :----------- |:----------- |:----------- |
|BLE CC41-A|Радиомодуль Bluetooth LE|150|
|Digispark ATTiny85|Микроконтроллер для снятия показаний датчика|80|
|YL-38|Датчик влажности почвы, например, [такой](https://www.ebay.com/itm/5PCS-Soil-Hygrometer-Detection-Module-Soil-Moisture-Sensor-For-arduino-Smart-car/400385860375?ssPageName=STRK%3AMEBIDX%3AIT&_trksid=p2057872.m2749.l2649)|40|
|Breadboard 170|Небольшая макетка для компоновки модулей датчика|30|
|Перемычки|Для соединения модулей на плате: папа-папа, мама-папа|50|
|Держатель батареек 2x AAA|Питание датчика|80|

Итоговая стоимость: 430 руб - это в два раза дешевле, чем скромный букет цветов.
Также вам потребуется любая имеющаяся Arduino, либо купите клон Aruino Nano для эмуляции USB-RS232 за 160 руб.

# Характеристики устройства

Ваше собранное устройство может выглядеть так:
![Умный датчик влажности почвы](https://github.com/cutecare/cutecare-docs/blob/master/images/SmartPlantSensorCC41A.jpg?raw=true)

Необходимо установить сенсор в цветочном горшке таким образом, чтобы контакты датчика влажности погрузились в почву. Убедитесь, чтобы батарейный блок не касался влажной почвы.

После подключения сенсора к Home Assistant (HASS), периодически (раз в час, но можно и реже) запрашивается показатель влажности почвы. HASS хранит измеренные показания и показывает график изменения уровня влажности. 

Вы также можете настроить удобные уведомления при превышении порогового значения влажности.

В режиме ожидания сенсор потребляет 1мА, в режиме считывания уровня влажности - 16мА. Продолжительность считывания составляет около 3-4 секунд. Пары батареек AAA должно хватить почти на два месяца.

CC41-A это конечно сложный случай и все из-за того, что его штатная прошивка не очень дружелюбна к нашей задаче.

# Подготовка рабочего места

Для программирования модулей вам потребуется установить Arduino IDE. Для программирования платы DigiSpark ATTiny85 нужно будет установить дополнительные драйверы и библиотеки, подробно это описано [здесь](http://digistump.com/wiki/digispark/tutorials/connecting), также советуем почитать об особенностях работы с этими модулями [здесь](https://medium.com/@evgeny.savitsky/%D0%BE%D1%81%D0%B2%D0%B0%D0%B8%D0%B2%D0%B0%D0%B5%D0%BC-attiny85-%D0%B8-lilypad-digispark-%D0%B2-%D1%87%D0%B0%D1%81%D1%82%D0%BD%D0%BE%D1%81%D1%82%D0%B8-c6e955957d53).

# Программирование модулей

Для настройки BLE-модуля через Arduino подключите его к Arduino Nano по схеме описанной [здесь](http://cutecare.readthedocs.io/ru/master/%D0%AD%D0%BB%D0%B5%D0%BC%D0%B5%D0%BD%D1%82%D0%BD%D0%B0%D1%8F%20%D0%B1%D0%B0%D0%B7%D0%B0/#_4). Загрузите в Arduino Nano программу:
```
#include <SoftwareSerial.h>

SoftwareSerial mySerial(2, 3); // RX, TX
String deviceName = "Small ficus"; // Укажите название вашего растения

void setup() {
  mySerial.begin(9600);
  mySerial.setTimeout(500);
  Serial.begin(9600);
  configureDevice();
}

void loop() {
  char c;
  if (Serial.available()) {
    c = Serial.read();
    mySerial.print(c);
  }
  if (mySerial.available()) {
    c = mySerial.read();
    Serial.print(c);    
  }
}

void configureDevice() {
  sendCommand("AT");
  sendCommand("AT+VERSION");
  sendCommand("AT+LADDR"); 
  sendCommand("AT+RENEW"); 	// restore default settings
  sendCommand(String("AT+NAME").concat(deviceName)); 
  sendCommand("AT+ROLE0"); 	// slave mode
  sendCommand("AT+TYPE0"); 	// unsecure, no pin required
  sendCommand("AT+POWE3"); 	// max RF power
  sendCommand("AT+UUID0xFFE0");
  sendCommand("AT+CHAR0xFFE1");
  sendCommand("AT+PWRM0"); 	// auto-sleep
  sendCommand("AT+RESET"); 	// reboot to apply settings
}

void sendCommand(const char * data) {
  Serial.println(data);
  mySerial.println(data);
  while(mySerial.available()) {
    Serial.println(mySerial.readStringUntil('\n'));
  }
  delay(20);
}
```

Переключите Arudino IDE для работы с DigiSpark устройствами. Запрограммируйте модуль ATTiny85 с использованием кода:
```
#include <SoftSerial.h>
#include <avr/sleep.h>      
#include <avr/power.h>    
#include <avr/wdt.h>         
#include <avr/io.h>          
#include <avr/interrupt.h>

#define adc_disable() (ADCSRA &= ~_BV(ADEN))
#define adc_enable()  (ADCSRA |=  _BV(ADEN))

#define SENSOR_VCC_PIN PB0
#define SENSOR_DATA_PIN A2
#define RF_LED_PIN PB2
#define TX_PIN PB1
#define RX_PIN PB4

void setup() {
  set_sleep_mode(SLEEP_MODE_PWR_DOWN);
  initializeWatchdogTimer(WDTO_4S);
}

void loop() {
  // prepearing for sleep
  setupPins();
  adc_disable();
  power_all_disable();
  sleep_mode();

  // wait for LED pin on BLE is ON
  if ( digitalRead(RF_LED_PIN) == LOW ) return;
  
  // wake up after high level on INTERRUPT_PIN
  power_all_enable();
  adc_enable();

  digitalWrite(SENSOR_VCC_PIN, HIGH); // turn on sensor VCC
  SoftSerial mySerial(RX_PIN, TX_PIN); // RX, TX (RX is not used)
  mySerial.begin(9600);

  // send sensor data over UART
  for( int retry = 0; retry < 2; retry++ ) {
    int sensorValue = analogRead(SENSOR_DATA_PIN);
    mySerial.write(sensorValue >> 8);
    mySerial.write(sensorValue & 0xFF);
    mySerial.write(13);
    mySerial.write(10);
    delay(200);
  }
}

void initializeWatchdogTimer(byte sleep_time)
{
  wdt_reset();
  wdt_enable(sleep_time);
  WDTCR |= _BV(WDIE);
}

ISR(WDT_vect) {
  WDTCR |= _BV(WDIE);
}

void setupPins()
{
  pinMode(PB0, OUTPUT);
  digitalWrite(PB0, LOW);
  pinMode(PB1, OUTPUT);
  digitalWrite(PB1, LOW);
  pinMode(A1, INPUT);
  analogWrite(A1, 0);
  pinMode(A2, INPUT);
  analogWrite(A2, 0);
  pinMode(A3, INPUT);
  analogWrite(A3, 0);
}
```

Вкратце опишу, что тут происходит:

1. Модуль BLE CC41-A стартует и переходит в спящий режим (потребление 0.6мА)
2. Модуль ATTiny85 стартует и переходит в спящий режим (потребление 0.4мА)
3. При подключении HASS к BLE CC41-A он просыпается, на его LED-пине устанавливается логическая 1
4. ATTiny85 периодически отслеживает состояние BLE CC41-A и, когда обнаруживает его в состоянии подключенности по Bluetooth, подает напряжение на модуль датчика влажности, считывает показания с датчика влажности и передает их CC41-A по UART-интерфейсу.
5. CC41-A передает через Bluetooth LE полученные данные, HASS считывает это событие и использует полученные данные как показания датчика влажности.
6. После обрыва Bluetooth-подключения оба модуля засыпают, пока опять не будет установлено подключение по Bluetooth LE.

# Схема устройства
![схема](https://github.com/cutecare/cutecare-docs/blob/master/images/sensor-cc41-a_bb.png?raw=true)

# Настройка HASS

Файл: /config/configuration.yaml
```
sensor:
  - platform: cutecare
    scan_interval: 3600
    mac: <укажите тут адрес вашего BLE-модуля>
    monitored_conditions:
      - moisture
    name: plant1
```

Файл: /config/customize.yaml
```
sensor.plant1_moisture:
  entity_picture: http://www.plantsguru.com/image/cache/catalog/foliage%20plants/pg-ficus-elastica-600x548.jpg
  friendly_name: Маленький фикус
```

Файл: /config/groups.yaml
```
default_view:
  view: yes
  icon: mdi:home 
  entities:
    - group.hall
hall:
  name: Большая комната
  entities:
    - sensor.plant1_moisture
```

После перезагрузки HASS он начнет опрашивать датчик один раз в час. Со временем вы получите примерно такой график:

![График влажности](https://github.com/cutecare/cutecare-docs/blob/master/images/hass-ficus.jpeg?raw=true)
