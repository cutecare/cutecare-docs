# Умное реле JDY-08

Разработку умного реле на базе BLE-модуля JDY-08 осуществим на примере умной розетки, которая будет управлять включением и отключением новогодней гирлянды.

Помимо управления умной розеткой с телефона вы можете настроить сценарий автоматизации, по которому гирлянда автоматически будет включаться утром и выключаться на ночь.

Как обычно наши схемы ориентированы на любителей, но могут быть легко усовершенствованы специалистами. Для сборки умной розетки вам потребуется паяльник и немного смекалки. 

Итак, компоненты умного реле:

|Название|Назначение|Цена, руб.|
| :----------- |:----------- |:----------- |
|BLE JDY-08|Радиомодуль Bluetooth LE|140|
|Hi-Link HLK PM-03 3.3V|Изолированный модуль питания 3.3В|150|
|Songle SRD-03VDC-SL-C|Реле управления нагрузкой до 10А|80|
|Розетка|Электрический разветвитель (двойник или тройник)|60|

Итоговая стоимость: 430 руб - это в два раза дешевле, чем самая дешевая умная розетка, кроме Sonoff конечно.
Также вам потребуются перемычки и любая имеющаяся Arduino, либо купите клон Aruino Nano для эмуляции USB-RS232 за 160 руб.

# Характеристики устройства

Ваше собранное устройство может выглядеть так:

![Умная розетка]()

Если у вас в наличии реле на 5В, то его также можно использовать. Поскольку BLE-модуль построен на 3-х вольтовой логике, в данном случае, вам потребуется добавить в схему простой транзисторный ключ для управления 5В реле.

Для управления 220В вам потребуется переделать саму розетку - нужно будет разорвать одну линию (желательно фазу). Чтобы припаять к контактам розетки провода используйте флюс.

# Программирование BLE-модуля

Для настройки BLE-модуля через Arduino подключите его к Arduino Nano по схеме описанной [здесь](http://cutecare.readthedocs.io/ru/master/%D0%AD%D0%BB%D0%B5%D0%BC%D0%B5%D0%BD%D1%82%D0%BD%D0%B0%D1%8F%20%D0%B1%D0%B0%D0%B7%D0%B0/#_4). Загрузите в Arduino Nano программу:
```
#include <SoftwareSerial.h>

SoftwareSerial mySerial(2, 3); // RX, TX
String deviceName = "Smart socket";

void setup() {
  mySerial.begin(115200);
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
  sendCommand("AT+MAC");
  sendCommand(String("AT+NAME").concat(deviceName)); 
  sendCommand("AT+HOSTEN0");
  sendCommand("AT+POWR0");
  sendCommand("AT+RST");
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

# Схема устройства

![схема](https://github.com/cutecare/cutecare-docs/blob/master/images/Relay-JDY-08_bb.png?raw=true)

# Настройка HASS

Файл: /config/configuration.yaml
```
light:
  - platform: cutecare
    mac: <укажите тут адрес вашего BLE-модуля>
    name: christmas_tree
```

Файл: /config/customize.yaml
```
light.christmas_tree:
  friendly_name: Новогодняя ёлка
  entity_picture: http://hello-halloween.com/wp-content/uploads/2016/12/Christmas-Tree1.png
```

Файл: /config/groups.yaml
```
default_view:
  view: yes
  icon: mdi:home
  entities:
    - group.hall
hall:
  name: Зал
  entities:
    - light.christmas_tree
```

Для управления гирляндой используйте пиктограммы, расположенные рядом с "Новогодней ёлкой":

<img src="https://github.com/cutecare/cutecare-docs/blob/master/images/relay_jdy08.jpg?raw=true" width="400">

