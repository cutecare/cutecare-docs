[Умная квартира или дом своими руками](http://cutecare.ru)

# Умный сенсор JDY-10

Разработку прототипа умного сенсора на базе BLE-модуля JDY-10 (китайском SoC TLSR8266) выполним на примере климатического датчика, сообщающего о температуре и давлении в помещении.

Измеренные параметры температуры можно использовать в сценариях управления микроклиматом, включения теплого пола и т.п. Параметры давления интересны в сценариях для метеочувствительных людей: синизть шум, включить расслабляющую музыку и т.п.

Как обычно наши схемы ориентированы на любителей, но могут быть легко усовершенствованы специалистами. Особенных требований к оборудованию нет.

Итак, компоненты умного датчика:

|Название|Назначение|Цена, руб.|
| :----------- |:----------- |:----------- |
|[BLE JDY-10](https://rover.ebay.com/rover/1/711-53200-19255-0/1?icep_id=114&ipn=icep&toolid=20004&campid=5338218090&mpre=https%3A%2F%2Fwww.ebay.com%2Fitm%2FHM-11-BLE-Bluetooth-4-0-Transceiver-CC2541-JDY-10-Low-Power-for-Apple-Android%2F401324230954%3FssPageName%3DSTRK%253AMEBIDX%253AIT%26_trksid%3Dp2057872.m2749.l2649)|Радиомодуль Bluetooth LE|110|
|[Arduino Pro Mini](https://rover.ebay.com/rover/1/711-53200-19255-0/1?icep_id=114&ipn=icep&toolid=20004&campid=5338218090&mpre=https%3A%2F%2Fwww.ebay.com%2Fitm%2F2PCS-New-Pro-Mini-atmega328-Board-5V-16M-Arduino-Compatible-Nano%2F191674251828%3FssPageName%3DSTRK%253AMEBIDX%253AIT%26_trksid%3Dp2057872.m2749.l2649)|Микроконтроллер для снятия показаний датчика|135|
|[MPL3115A2](https://rover.ebay.com/rover/1/711-53200-19255-0/1?icep_id=114&ipn=icep&toolid=20004&campid=5338218090&mpre=https%3A%2F%2Fwww.ebay.com%2Fitm%2FMPL3115A2-IIC-I2C-Intelligent-Temperature-Pressure-Altitude-Sensor-For-Arduino%2F401347699449%3FssPageName%3DSTRK%253AMEBIDX%253AIT%26_trksid%3Dp2057872.m2749.l2649)|Датчик температуры, давления, высоты|170|
|[Breadboard 170](https://rover.ebay.com/rover/1/711-53200-19255-0/1?icep_id=114&ipn=icep&toolid=20004&campid=5338218090&mpre=https%3A%2F%2Fwww.ebay.com%2Fitm%2F5-Color-Mini-Solderless-Prototype-Breadboard-170-Tie-points-For-Arduino-Shield%2F201677166530%3FssPageName%3DSTRK%253AMEBIDX%253AIT%26_trksid%3Dp2057872.m2749.l2649)|Небольшая макетка для компоновки модулей датчика|30|
|[Держатель батареек 2x AAA](https://rover.ebay.com/rover/1/711-53200-19255-0/1?icep_id=114&ipn=icep&toolid=20004&campid=5338218090&mpre=https%3A%2F%2Fwww.ebay.com%2Fitm%2FPlastic-Battery-Storage-Case-Box-Holder-1xAA-2xAA-3xAA-4xAA-5xAA-With-Wires-US%2F253322432114%3FssPageName%3DSTRK%253AMEBIDX%253AIT%26var%3D552480399363%26_trksid%3Dp2057872.m2749.l2649)|Питание датчика|60|

Итоговая стоимость: 505 руб. На Я.Маркете самая дешевая метеостанция с функцией отображения атмосферного давления вам обойдется в 1500 руб. и она никак не будет интегрирована в вашу "умную" квартиру.

# Характеристики устройства

Ваше собранное устройство может выглядеть так:

![Умный климатический датчик](https://github.com/cutecare/cutecare-docs/blob/master/images/ClimateSmartSensor.jpg?raw=true)

Поскольку внизу комнаты воздух холоднее, чем сверху, датчики температуры размещают на стене на высоте, равной половине высоты комнаты, чтобы усреднить показание температуры.

В режиме ожидания сенсор потребляет 0.1мА, измеряет и передает данные только в случае их изменений (этот алгоритм вы легко можете изменить под свою задачу).
Пары батареек AAA должно хватить на долго.

# Программирование

О том как настроить Arduino IDE и подключить микроконтроллер к ПК читайте в этой коротенькой [инструкции](http://cutecare.readthedocs.io/ru/master/%D0%9C%D0%B8%D0%BA%D1%80%D0%BE%D0%BA%D0%BE%D0%BD%D1%82%D1%80%D0%BE%D0%BB%D0%BB%D0%B5%D1%80%D1%8B/#arduino-pro-mini).
Еще нужно установить библиотеку для работы с датчиком MPL3115A2, ее нужно скачать [здесь](https://github.com/adafruit/Adafruit_MPL3115A2_Library/archive/master.zip) и затем добавить через меню Sketch - Install library.
Теперь заливайте программу:
```
#include <SoftwareSerial.h>
#include <LowPower.h>
#include <Adafruit_MPL3115A2.h>

#define RX_PIN 4
#define TX_PIN 11
#define BLE_PIN 10
#define BLE_BAUD 115200

unsigned int lastPressure = 0;
unsigned int lastTemperature = 0;
int secondsPassed = 0;

void setup() 
{
  // configure BLE device
  pinMode(BLE_PIN, OUTPUT);
  pinMode(TX_PIN, OUTPUT);

  SoftwareSerial serial(RX_PIN, TX_PIN);
  serial.begin(BLE_BAUD);

  sendCommand(&serial, "AT+DEFAULT");
  sendCommand(&serial, "AT+RESET");
  delay(300);
  sendCommand(&serial, "AT+NAMECLM-TLS-01");
  sendCommand(&serial, "AT+POWE0");
  sendCommand(&serial, "AT+ADVINT4");
  sendCommand(&serial, "AT+CLSSA0");

  measureValues(&lastPressure, &lastTemperature);
  sendSensorData(&serial, lastPressure, lastTemperature, false);
}

void loop() 
{
  // prepare for sleep, clear pins
  pinMode(TX_PIN, INPUT);
  pinMode(BLE_PIN, INPUT);
  pinMode(A4, INPUT); // SDA
  pinMode(A5, INPUT); // SCL
  LowPower.powerDown(SLEEP_8S, ADC_OFF, BOD_OFF);

  // check if 5 minutes have passed
  secondsPassed += 8;
  if ( secondsPassed < 300 ) return;
  secondsPassed = 0;

  // measure sensor value
  delay(200);

  unsigned int nowPressure = 0;
  unsigned int nowTemperature = 0;
  measureValues(&nowPressure, &nowTemperature);
  if ( nowPressure == lastPressure && nowTemperature == lastTemperature ) return;

  // configure BLE if sensor values have changed since the last measure
  lastPressure = nowPressure;
  lastTemperature = nowTemperature;

  SoftwareSerial bleSerial(RX_PIN, TX_PIN);
  bleSerial.begin(BLE_BAUD);
  sendSensorData(&bleSerial, lastPressure, lastTemperature, true);
}

void sendSensorData( SoftwareSerial * bleSerial, int pressure, int temperature, bool wakeUp ) 
{
  char buff[32] = "";
  unsigned int value = pressure << 6;
  value += temperature;
  sprintf(buff, "AT+CHRUUID%04X", value);

  if ( wakeUp ) { // wake up BLE if required
    pinMode(TX_PIN, OUTPUT);
    pinMode(BLE_PIN, OUTPUT);
    digitalWrite(BLE_PIN, LOW);
    delay(800);
    pinMode(BLE_PIN, INPUT);
  }

  sendCommand(bleSerial, buff);
  sendCommand(bleSerial, buff);
  sendCommand(bleSerial, "AT+SLEEP1");
}

void measureValues(unsigned int * pressure, unsigned int * temperature) 
{
  Adafruit_MPL3115A2 sensor;
  sensor.begin();
  float barCoeff = 0.00750063755419211; // millimeter of mercury [0 °C]
  *pressure = sensor.getPressure() * barCoeff;
  *temperature = sensor.getTemperature();
}

void sendCommand(SoftwareSerial * bleSerial, const char * data) {
  delay(200);
  bleSerial->println(data);
}
```

Итак, по порядку:

1. Pro Mini стартует, конфигурирует BLE для работы в режиме iBeacon, дает ему имя CLM-TLS-01, погружает в спящий режим и сам засыпает.
2. Каждые 5 минут микроконтроллер просыпается, снимает показания с датчика и, если они изменились по сравнению с предыдущими значениями, настраивает BLE-модуль на передачу этих значений.
3. BLE-модуль периодически рассылает настройки, HASS считывает их и использует как показания датчиков температуры и давления.

# Схема устройства

![схема](https://github.com/cutecare/cutecare-docs/blob/master/images/ProMini-TLS-MPL_bb.png?raw=true)

# Настройка HASS

Файл: /config/configuration.yaml
```
sensor:
  - platform: cutecare
    scan_interval: 60
    mac: A4:C1:38:77:04:59
    monitored_conditions:
      - temperature
      - pressure
    name: climate_1
    type: jdy10
```

Файл: /config/customize.yaml
```
sensor.climate_1_temperature:
  friendly_name: Температура
  icon: mdi:thermometer
sensor.climate_1_pressure:
  friendly_name: Атмосферное давление
  icon: mdi:nature
```

Файл: /config/groups.yaml
```
cabinet:
  name: Кабинет
  entities:
    - sensor.climate_1_temperature
    - sensor.climate_1_pressure
```

Со временем вы получите примерно такой график:

<img src="https://github.com/cutecare/cutecare-docs/blob/master/images/hass-climate-sensor.png?raw=true" width="400">

# Ссылки

1. [Спецификация модуля JDY-10](https://www.electro-tech-online.com/attachments/jdy-10%E8%93%9D%E7%89%994-0%E6%A8%A1%E5%9D%97v2-4-pdf.109589/)
2. [Открытая платформа для создания заботливого умного дома](http://cutecare.ru)
