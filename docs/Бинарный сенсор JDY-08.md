[Умная квартира или дом своими руками](http://cutecare.ru)

# Бинарный сенсор JDY-08

Это сенсор, который принимает два состояния: включено/выключено. Разработку бинарного сенсора осуществим на примере датчика движения. С использованием такого сенсора можно организовать сценарий, который будет управлять автоматическим включением и отключением света в комнате.

Как обычно наши схемы ориентированы на любителей, но могут быть легко усовершенствованы специалистами. Для сборки бинарного сенсора вам не потребуется даже паяльник.

Итак, компоненты умного бинарного сенсора:

|Название|Назначение|Цена, руб.|
| :----------- |:----------- |:----------- |
|[BLE JDY-08](https://rover.ebay.com/rover/1/711-53200-19255-0/1?icep_id=114&ipn=icep&toolid=20004&campid=5338218090&mpre=https%3A%2F%2Fwww.ebay.com%2Fitm%2FBluetooth-4-0-BLE-Low-Power-CC2541-JDY-08-Support-Airsync-iBeacon-Module%2F322511962233%3FssPageName%3DSTRK%253AMEBIDX%253AIT%26_trksid%3Dp2057872.m2749.l2649)|Радиомодуль Bluetooth LE|140|
|[HC-SR505](https://rover.ebay.com/rover/1/711-53200-19255-0/1?icep_id=114&ipn=icep&toolid=20004&campid=5338218090&mpre=https%3A%2F%2Fwww.ebay.com%2Fitm%2FHC-SR501-SR04-SR505-Mini-PIR-Infrarot-Sensor-Module-forArduino-Raspberry-Pi%2F401232752070%3FssPageName%3DSTRK%253AMEBIDX%253AIT%26var%3D670821966681%26_trksid%3Dp2057872.m2749.l2649)|Мини-датчик движения инфракрасный|75|
|[Digispark ATTiny85](https://rover.ebay.com/rover/1/711-53200-19255-0/1?icep_id=114&ipn=icep&toolid=20004&campid=5338218090&mpre=https%3A%2F%2Fwww.ebay.com%2Fitm%2FDigispark-Kickstarter-Attiny85-USB-Development-Board-for-arduino-NEW%2F311076127758%3FssPageName%3DSTRK%253AMEBIDX%253AIT%26_trksid%3Dp2057872.m2749.l2649)|Микроконтроллер для снятия показаний датчика движения|80|
|[Breadboard 170](https://rover.ebay.com/rover/1/711-53200-19255-0/1?icep_id=114&ipn=icep&toolid=20004&campid=5338218090&mpre=https%3A%2F%2Fwww.ebay.com%2Fitm%2F5-Color-Mini-Solderless-Prototype-Breadboard-170-Tie-points-For-Arduino-Shield%2F201677166530%3FssPageName%3DSTRK%253AMEBIDX%253AIT%26_trksid%3Dp2057872.m2749.l2649)|Небольшая макетка для компоновки модулей датчика|30|
|[Держатель батареек](https://www.ebay.com/itm/2xBlack-Plastic-Battery-Storage-Case-Box-Holder-for-4xAAA-1-5V-with-wire-2-layer/252377887792?hash=item3ac2e4f430:g:6SAAAOSwYmZXJx5G)|Питание датчика 5В|30|

Итоговая стоимость: 355 руб - это в три раза дешевле, чем готовый китайский датчик движения.
Также вам потребуются перемычки для соединения модулей.

# Характеристики устройства

Ваше собранное устройство может выглядеть так:

![Умный датчик движения](https://github.com/cutecare/cutecare-docs/blob/master/images/MotionDetectorSensor.jpg?raw=true)

Располагайте датчик движения с учетом того, что он отслеживает движение на расстоянии 4-5 метров. 
В режиме ожидания датчик потребляет около 3мА, так что батареек должно хватить на продолжительную работу устройства. Питание 5В необходимо для устойчивой работы датчика движения. Иначе можно было бы обойтись 2-мя элементами питания.

# Программирование

Для программирования платы DigiSpark ATTiny85 нужно будет установить Arduino IDE и дополнительные драйверы и библиотеки, подробно это описано [здесь](http://digistump.com/wiki/digispark/tutorials/connecting), также советуем почитать об особенностях работы с этими модулями [здесь](https://medium.com/@evgeny.savitsky/%D0%BE%D1%81%D0%B2%D0%B0%D0%B8%D0%B2%D0%B0%D0%B5%D0%BC-attiny85-%D0%B8-lilypad-digispark-%D0%B2-%D1%87%D0%B0%D1%81%D1%82%D0%BD%D0%BE%D1%81%D1%82%D0%B8-c6e955957d53).
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
#define TX_DELAY 200

int wasState = LOW;

void setup() 
{
  setupPins();
  set_sleep_mode(SLEEP_MODE_PWR_DOWN);
  initializeWatchdogTimer(WDTO_1S);

  configureBLEDevice();
}

void loop() 
{
  setupPins();
  adc_disable();
  power_all_disable();
  sleep_mode();

  int nowState = digitalRead(SWITCH_PIN);
  if ( nowState == wasState ) return;
  wasState = nowState;

  power_all_enable();
  wakeupBLE();

  SoftSerial bleSerial(PB4, TX_PIN); // RX, TX
  bleSerial.begin(115200);
  sendCommand(&bleSerial, nowState == HIGH ? "AT+MAJOR0001" : "AT+MAJOR0000");
  sendCommand(&bleSerial, "AT+SLEEP1");
  delay(TX_DELAY);
}

void configureBLEDevice() 
{ 
  wakeupBLE();

  SoftSerial bleSerial(PB4, TX_PIN); // RX, TX
  bleSerial.begin(115200);
  
  sendCommand(&bleSerial, "AT+RESTORE");
  sendCommand(&bleSerial, "AT+NAMEMotionJDY-08"); 
  sendCommand(&bleSerial, "AT+STRUUID686DF1642351471E9F4167F0B2E5EABE"); // this UUID should be unique per device
  sendCommand(&bleSerial, "AT+HOSTEN3");    // app transparent mode
  sendCommand(&bleSerial, "AT+CLSSE0");     // iBeacon mode
  sendCommand(&bleSerial, "AT+RST");        // apply settings
  sendCommand(&bleSerial, "AT+POWR0");      // max RF power
  sendCommand(&bleSerial, "AT+ADVEN1");     // advertising enabled
  sendCommand(&bleSerial, "AT+ADVIN1");     // setup advertising interval
  sendCommand(&bleSerial, "AT+PWMOPEN0");   // turn off PWM
  sendCommand(&bleSerial, "AT+RTCOPEN0");   // disable RTC
  sendCommand(&bleSerial, "AT+MAJOR0000");  // switch initial state is off
  sendCommand(&bleSerial, "AT+SLEEP1");     // power down mode + advertising
}
 
void wakeupBLE()
{
  digitalWrite(BLE_PIN, HIGH);
  delay(100);
  digitalWrite(BLE_PIN, LOW);
}

void sendCommand(SoftSerial * bleSerial, const char * data) {
  delay(TX_DELAY);
  bleSerial->print(data);
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
  digitalWrite(BLE_PIN, LOW);
  pinMode(TX_PIN, OUTPUT);
  digitalWrite(TX_PIN, LOW);
  pinMode(A1,INPUT);  
  pinMode(A2,INPUT);
  pinMode(A3,INPUT);
}
```

Алгоритм работы очень простой:

1. Датчик движения HC-SR505 - самостоятельное устройство, которое отслеживает движение человека. Если движение обнаружено, то на выходе датчика будет установлено напряжение 3В.
2. Микроконтроллер при подаче питания настраивает BLE-модуль и переводит его в режим с пониженным энергопотреблением. Затем он отслеживает изменения состояния датчика движения и выполняет настройку BLE-модуля.
3. BLE-модуль работает в широковещательном режиме и таким образом передает HASS-контроллеру информацию о состоянии датчика движения.

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

Здесь приведен достаточно простой пример автоматизации, который включает свет на кухне при обнаружении движения и отключает его через 10 минут после последнего обнаруженного движения. Конечно, более реалистичный сценарий потребует некоторого усложнения.

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
5. [Открытая платформа для создания заботливого умного дома](http://cutecare.ru)
