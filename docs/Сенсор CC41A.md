[Умная квартира или дом своими руками](http://cutecare.ru)

# Умный сенсор CC41-A

Разработку прототипа умного сенсора на базе BLE-модуля CC41-A осуществим на примере датчика влажности, который можно установить в цветочный горшок любимого растения вашей любимой половинки.

Умный датчик влажности в составе умной квартиры напомнит о необходимости полить растение, с учетом его влаголюбивости или засухоустойчивости, например, отправив сообщение на телефон.

Как обычно наши схемы ориентированы на любителей, но могут быть легко усовершенствованы специалистами. Особенных требований к оборудованию нет. Вам даже не потребуется паяльник. 

Итак, компоненты умного датчика:

|Название|Назначение|Цена, руб.|
| :----------- |:----------- |:----------- |
|[BLE CC41-A](https://rover.ebay.com/rover/1/711-53200-19255-0/1?icep_id=114&ipn=icep&toolid=20004&campid=5338218090&mpre=https%3A%2F%2Fwww.ebay.com%2Fitm%2FArduino-Android-IOS-HM-10-BLE-Bluetooth-4-0-CC2540-CC2541-Serial-Wireless-Module%2F311567433651%3FssPageName%3DSTRK%253AMEBIDX%253AIT%26_trksid%3Dp2057872.m2749.l2649)|Радиомодуль Bluetooth LE|150|
|[Digispark ATTiny85](https://rover.ebay.com/rover/1/711-53200-19255-0/1?icep_id=114&ipn=icep&toolid=20004&campid=5338218090&mpre=https%3A%2F%2Fwww.ebay.com%2Fitm%2FDigispark-Kickstarter-Attiny85-USB-Development-Board-for-arduino-NEW%2F311076127758%3FssPageName%3DSTRK%253AMEBIDX%253AIT%26_trksid%3Dp2057872.m2749.l2649)|Микроконтроллер для снятия показаний датчика|80|
|[YL-38](https://rover.ebay.com/rover/1/711-53200-19255-0/1?icep_id=114&ipn=icep&toolid=20004&campid=5338218090&mpre=https%3A%2F%2Fwww.ebay.com%2Fitm%2F5PCS-Soil-Hygrometer-Detection-Module-Soil-Moisture-Sensor-For-arduino-Smart-car%2F400385860375%3FssPageName%3DSTRK%253AMEBIDX%253AIT%26_trksid%3Dp2057872.m2749.l2649)|Датчик влажности почвы|40|
|[Breadboard 170](https://rover.ebay.com/rover/1/711-53200-19255-0/1?icep_id=114&ipn=icep&toolid=20004&campid=5338218090&mpre=https%3A%2F%2Fwww.ebay.com%2Fitm%2F5-Color-Mini-Solderless-Prototype-Breadboard-170-Tie-points-For-Arduino-Shield%2F201677166530%3FssPageName%3DSTRK%253AMEBIDX%253AIT%26_trksid%3Dp2057872.m2749.l2649)|Небольшая макетка для компоновки модулей датчика|30|
|[Держатель батареек 3x AAA](https://rover.ebay.com/rover/1/711-53200-19255-0/1?icep_id=114&ipn=icep&toolid=20004&campid=5338218090&mpre=https%3A%2F%2Fwww.ebay.com%2Fitm%2F2-3-4xAAA-Battery-Holder-Plastic-Batteries-Box-Battery-Storage-Case-With-Wire-TS%2F152748115174%3Fhash%3Ditem23907f44e6%3Am%3AmR_En8kk8yOo9vvMl-QhG0g)|Питание датчика|80|

Итоговая стоимость: 380 руб - это в два раза дешевле, чем скромный букет цветов. Также вам потребуются перемычки.

# Характеристики устройства

Ваше собранное устройство может выглядеть так:

![Умный датчик влажности почвы](https://github.com/cutecare/cutecare-docs/blob/master/images/SmartPlantSensorCC41A.jpg?raw=true)

Необходимо установить сенсор в цветочном горшке таким образом, чтобы контакты датчика влажности погрузились в почву. Убедитесь, чтобы батарейный блок не касался влажной почвы.

После подключения сенсора к Home Assistant (HASS), периодически (раз в час, но можно и реже) запрашивается показатель влажности почвы. HASS хранит измеренные показания и отображает график изменения уровня влажности. 

Вы также можете настроить удобные уведомления при превышении порогового значения влажности.

В режиме ожидания сенсор потребляет 0.6мА, в режиме считывания уровня влажности - 12мА. Продолжительность считывания составляет около 2-х секунд. Считывание параметров осуществляется один раз в час. Пары батареек AAA должно хватить почти на три месяца.

# Программирование

Подготовьте рабочее место для работы с DigiSpark ATTiny85 по [этой инструкции](http://cutecare.readthedocs.io/ru/master/%D0%9C%D0%B8%D0%BA%D1%80%D0%BE%D0%BA%D0%BE%D0%BD%D1%82%D1%80%D0%BE%D0%BB%D0%BB%D0%B5%D1%80%D1%8B/) и запрограммируйте модуль ATTiny85 с использованием кода:
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
#define TX_PIN PB1
#define RX_PIN PB4
#define BLE_BAUD 9600
#define TX_DELAY 200
#define MEASURE_INTERVAL 3600

volatile int seconds = MEASURE_INTERVAL;

void setup() {
  set_sleep_mode(SLEEP_MODE_PWR_DOWN);
  initializeWatchdogTimer(WDTO_8S);
  setupPins();
  configureBLEDevice(0, 0);
}

void loop() {
  // prepearing for sleep
  adc_disable();
  power_all_disable();
  sleep_mode();

  if ( seconds < MEASURE_INTERVAL ) return;
  seconds = 0; 

  // wake up each hour
  power_all_enable();
  adc_enable();
  
  // turn on sensor VCC and read value
  digitalWrite(SENSOR_VCC_PIN, HIGH); 
  delay(TX_DELAY * 2);
  int sensorValue = analogRead(SENSOR_DATA_PIN);
  setupPins();

  // send sensor data over UART
  configureBLEDevice(sensorValue, 0);
}

void configureBLEDevice(int major, int minor) 
{ 
  SoftSerial bleSerial(RX_PIN, TX_PIN); // RX, TX
  bleSerial.begin(BLE_BAUD);
  
  sendCommand(&bleSerial, "AT");
  sendCommand(&bleSerial, "AT+RENEW"); // restore default settings
  sendCommand(&bleSerial, "AT+ROLE0"); // slave mode
  sendCommand(&bleSerial, "AT+TYPE0"); // unsecure, no pin required
  sendCommand(&bleSerial, "AT+POWE3"); // max RF power

  char buffer[64] = "";
  sprintf(buffer, "AT+MARJ0x%04X\r\n", major);
  sendCommand(&bleSerial, buffer);
  sprintf(buffer, "AT+MINO0x%04X\r\n", minor);
  sendCommand(&bleSerial, buffer);
  
  sendCommand(&bleSerial, "AT+ADVI9"); // long advertising interval
  sendCommand(&bleSerial, "AT+PWRM0"); // auto-sleep
  sendCommand(&bleSerial, "AT+IBEA1"); // iBeacon mode
  sendCommand(&bleSerial, "AT+RESET");
}

void sendCommand(SoftSerial * bleSerial, const char * data) {
  delay(TX_DELAY);
  bleSerial->println(data);
}

void initializeWatchdogTimer(byte sleep_time)
{
  wdt_reset();
  wdt_enable(sleep_time);
  WDTCR |= _BV(WDIE);
}

ISR(WDT_vect) {
  WDTCR |= _BV(WDIE);
  seconds += 8;
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

Вкратце, что тут вообще происходит:

1. Модуль ATTiny85 стартует, конфигурирует BLE для работы в режиме iBeacon, то есть переключает в широковещательный режим работы, затем переходит в спящий режим.
2. При первом включении и затем раз в час микроконтроллер подает напряжение на модуль датчика влажности, считывает показания с датчика влажности и передает их CC41-A по UART-интерфейсу.
3. CC41-A периодически рассылает полученные данные, HASS считывает их и использует как показания датчика влажности.

# Схема устройства

![схема](https://github.com/cutecare/cutecare-docs/blob/master/images/sensor-cc41-a_bb.png?raw=true)

# Настройка HASS

Файл: /config/configuration.yaml
```
sensor:
  - platform: cutecare
    mac: <укажите тут адрес вашего BLE-модуля>
    scan_interval: 3600
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

<img src="https://github.com/cutecare/cutecare-docs/blob/master/images/hass-ficus.jpeg?raw=true" width="400">

## Автоматизация

Самый простой способ отправить уведомление о необходимости полить растение - отправить письмо на электронный адрес. Есть масса других вариантов, например, push-уведомления на телефоне, но в этом случае потребуется установить доп. ПО на телефон.

Файл: /config/configuration.yaml
```
binary_sensor:
  - platform: template
    sensors:
      moisture_low:
        value_template: '{{ states.sensor.plant1_moisture.state|int < 20 }}'
        friendly_name: 'Необходимо полить растение'
notify:
  - name: email
    platform: smtp
    server: smtp.gmail.com
    port: 587
    timeout: 15
    sender: youremail@gmail.com
    encryption: starttls
    username: youremail@gmail.com
    password: *********
    recipient:
      - youremail@gmail.com
    sender_name: Умная квартира
alert:
  low_moisture:
    name: "Нужно полить растение"
    entity_id: binary_sensor.moisture_low
    repeat: 180
    notifiers:
      - email
```

Этими настройками мы создаем бинарный сенсор, который реагирует на достижение порогового значения влажности, измеряемого умным сенсором plant1_moisture. Также создаем канал уведомлений по почте с названием email. Правило "low_moisture" срабатывает при уровне влажности почвы < 20% и отправляет об этом уведомление на почту.

# Ссылки

1. [Введение в использование BLE-модуля](https://medium.com/@yostane/using-the-at-09-ble-module-with-the-arduino-3bc7d5cb0ac2)
2. [Спецификация модуля CC41-A](http://img.banggood.com/file/products/20150104013145BLE-CC41-A%20Spefication.pdf)
3. [Перечень команд, поддерживаемых модулем](http://4ipov.net/bluetooth_4.0_BLE_module_CC2541_CC41-A)
4. [Переключение в режим iBeacon](https://rememberdontsearch.wordpress.com/2017/04/19/ble-cc41-a-hm10-clone-ibeacon/)
5. [Открытая платформа для создания заботливого умного дома](http://cutecare.ru)
