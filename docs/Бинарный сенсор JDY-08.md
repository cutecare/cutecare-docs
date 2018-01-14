# Бинарный сенсор JDY-08

Это сенсор, который принимает два состояния: включено/выключено. Разработку бинарного сенсора осуществим на примере датчика движения. С использованием такого сенсора можно организовать сценарий, который будет управлять автоматическим включением и отключением света в комнате.

Как обычно наши схемы ориентированы на любителей, но могут быть легко усовершенствованы специалистами. Для сборки бинарного сенсора вам не потребуется даже паяльник.

Итак, компоненты умного бинарного сенсора:

|Название|Назначение|Цена, руб.|
| :----------- |:----------- |:----------- |
|BLE JDY-08|Радиомодуль Bluetooth LE|140|
|HC-SR505|Мини-датчик движения инфракрасный|75|
|Digispark ATTiny85|Микроконтроллер для снятия показаний датчика движения|80|
|Breadboard 170|Небольшая макетка для компоновки модулей датчика|30|
|Держатель батареек 3x AAA|Питание датчика 4.5В|30|

Итоговая стоимость: 355 руб - это в три раза дешевле, чем готовый китайский датчик движения.
Также вам потребуются перемычки для соединения модулей.

# Характеристики устройства

Ваше собранное устройство может выглядеть так:

![Умная розетка](https://github.com/cutecare/cutecare-docs/blob/master/images/Smart-Socket-JDY-08.jpg?raw=true)

Располагайте датчик движения с учетом того, что он отслеживает движение на расстоянии 4-5 метров. 
В режиме ожидания датчик потребляет около 1мА, так что батареек должно хватить на продолжительную работу устройства.

# Программирование микроконтроллера

Для программирования модулей вам потребуется установить Arduino IDE. 
Для программирования платы DigiSpark ATTiny85 нужно будет установить дополнительные драйверы и библиотеки, подробно это описано [здесь](http://digistump.com/wiki/digispark/tutorials/connecting), также советуем почитать об особенностях работы с этими модулями [здесь](https://medium.com/@evgeny.savitsky/%D0%BE%D1%81%D0%B2%D0%B0%D0%B8%D0%B2%D0%B0%D0%B5%D0%BC-attiny85-%D0%B8-lilypad-digispark-%D0%B2-%D1%87%D0%B0%D1%81%D1%82%D0%BD%D0%BE%D1%81%D1%82%D0%B8-c6e955957d53).
Переключите Arudino IDE для работы с DigiSpark устройствами. Запрограммируйте модуль ATTiny85 с использованием этого кода:

```
#include <SoftSerial.h>
#include <avr/sleep.h>      
#include <avr/power.h>    
#include <avr/wdt.h>         
#include <avr/io.h>
#include <avr/interrupt.h>

#define adc_disable() (ADCSRA &= ~_BV(ADEN))
#define adc_enable()  (ADCSRA |=  _BV(ADEN))

#define BLE_PIN PB0
#define TX_PIN PB1
#define SWITCH_PIN PB2
#define TX_DELAY 125

int wasState = LOW;

void setup() 
{
  setupPins();
  configureBLEDevice();
  set_sleep_mode(SLEEP_MODE_PWR_DOWN);
  initializeWatchdogTimer(WDTO_1S);
}

void disableModules()
{
  adc_disable();
  power_all_disable();
  PRR &= ~_BV(PRADC);
  PRR &= ~_BV(PRUSI);
}

void loop() 
{
  setupPins();
  disableModules();
  sleep_mode();

  int nowState = digitalRead(SWITCH_PIN);
  if ( nowState == wasState ) return;
  wasState = nowState;

  power_all_enable();

  digitalWrite(BLE_PIN, LOW); // wake up BLE to enable UART
  delay(TX_DELAY);
  digitalWrite(BLE_PIN, HIGH);

  SoftSerial bleSerial(PB4, TX_PIN); // RX, TX
  bleSerial.begin(115200);
  sendCommand(&bleSerial, nowState == HIGH ? "AT+MAJOR0001" : "AT+MAJOR0000");
  sendCommand(&bleSerial, "AT+SLEEP1");
}

void configureBLEDevice() 
{ 
  SoftSerial bleSerial(PB4, TX_PIN); // RX, TX
  bleSerial.begin(115200);
  
  sendCommand(&bleSerial, "AT");
  sendCommand(&bleSerial, "AT+RST");
  sendCommand(&bleSerial, "AT+HOSTEN3");
  sendCommand(&bleSerial, "AT+POWR0");      // max RF power
  sendCommand(&bleSerial, "AT+ADVEN1");     // advertising enabled
  sendCommand(&bleSerial, "AT+ADVIN1");     // setup advertising interval
  sendCommand(&bleSerial, "AT+MAJOR0000");  // switch initial state is off
  sendCommand(&bleSerial, "AT+PWMOPEN0");   // turn off PWM
  sendCommand(&bleSerial, "AT+RTCOPEN0");   // disable RTC
  sendCommand(&bleSerial, "AT+SLEEP1");     // power down mode + advertising
}

void sendCommand(SoftSerial * bleSerial, const char * data) {
  delay(TX_DELAY);
  bleSerial->print(data);
  delay(20);
  while(bleSerial->available()) {
    bleSerial->read();
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
  pinMode(BLE_PIN, OUTPUT); 
  digitalWrite(BLE_PIN, HIGH);
  pinMode(TX_PIN, OUTPUT);
  digitalWrite(TX_PIN, LOW);
  pinMode(SWITCH_PIN,INPUT);  
  pinMode(A2,INPUT);
  pinMode(A3,INPUT);
}
```

Алгоритм работы очень простой:

1. Датчик движения HC-SR505 - самостоятельное устройство, которое отслеживает движение человека. Если движение обнаружено, то на выходе датчика будет установлено напряжение 3В.
2. Микроконтроллер при подаче питания настраивает BLE-модуль и переводит его в режим с пониженным энергопотреблением. Затем он отслеживает изменения состояния датчика движения и выполняет настройку BLE-модуля.
3. BLE-модуль работает в широковещательном режиме и таким образом передает контроллеру информацию о состоянии датчика движения.

# Схема устройства

![схема](https://github.com/cutecare/cutecare-docs/blob/master/images/binarySensorJDY08MotionDetector_bb.png?raw=true)

# Настройка HASS

Файл: /config/configuration.yaml
```
binary_sensor:
  - platform: cutecare
    name: Motion detector
    mac: 3c:a3:08:c6:82:10
```

Файл: /config/customize.yaml
```
binary_sensor.motion_detector:
  friendly_name: Движение
```

Файл: /config/groups.yaml
```
default_view:
  view: yes
  icon: mdi:home
  entities:
    - group.hall
    - group.kitchen
    - group.bedroom

kitchen:
  name: Кухня
  entities:
    - binary_sensor.motion_detector
```

В приложении вы сможете отслеживать изменение состояния датчика:

<img src="https://github.com/cutecare/cutecare-docs/blob/master/images/binary-sensor-jdy08-app.png?raw=true" width="400">

## Автоматизация

Чтобы гирлянда автоматически включалась утром (9:00) и выключалась на ночь (22:00), добавьте следующую автоматизацию:

Файл: /config/automations.yaml
```
- alias: Включить свет на кухне, если обнаружено движение
  trigger:
    platform: state
    entity_id: binary_sensor.motion_detector
    to: 'on'
  action:
    service: homeassistant.turn_on
    entity_id: light.kitchen_light

- alias: Выключить свет на кухне через 10 минут после последнего движения
  trigger:
    platform: state
    entity_id: binary_sensor.motion_detector
    to: 'off'
    for:
      minutes: 10
  action:
    service: homeassistant.turn_off
    entity_id: light.kitchen_light
```

# Ссылки

1. [Адаптированная спецификация на модуль и прошивку в переводе с китайского](https://github.com/kichMan/JDY-08)
2. [Оригинальная спецификация на модуль и прошивку (англ.)](https://fccid.io/2AM2YJDY-08/User-Manual/User-Manual-3511895.pdf)
3. [Утилита для считывания параметров iBeacon](http://developer.radiusnetworks.com/ibeacon/idk/ibeacon_scan)
4. [Описание формата данных iBeacon](https://kvurd.com/blog/tilt-hydrometer-ibeacon-data-format/)
