# TinkoffBackendTestTask
Tinkoff Intern Backend Python Test task, Smart hub (Iot), all system test passed
Ru Task description 
Предисловие

Вам предлагается на выполнение задание на моделирование “Умного дома”.

Если вы пишите на python, java, go, вам доступна только стандартная библиотека соответствующих языков. Если вы пишете на scala, помимо стандартной библиотеки разрешается использовать sttp (https://sttp.softwaremill.com/en/stable/examples.html#use-th…).

В задании не потребуется знание каких-то языково-специфичных конструкций или особенностей стандартной библиотеки или API, за исключением работы с HTTP POST запросами на сервер. От вас потребуется знание основ программирования, таких как работа со строками, последовательностями байт и битами внутри байта, и базовых структур данных (массивов, списков).

Текст задания может сначала показаться большим и страшным, и чтобы справиться со сложностью вы можете разделить выполнение задания на несколько этапов.

Научитесь подключаться к серверу и выполнять HTTP POST запросы на сервер, передавая ему уже готовые закодированные сообщения.
Научитесь кодировать и декодировать URL-encoded unpadded base64 для отправки на сервер и приёма данных от сервера.
Научитесь кодировать и декодировать пакеты из бинарной формы в ваше внутреннее представление.
Научитесь выявлять структуру сети устройств с помощью WHOISHERE.
Научитесь управлять одной лампой и одним выключателем.
Реализуйте весь требуемый функционал.
Для вашего удобства вам предоставляется готовая (консольная) программа - модельный сервер, на котором вы можете отлаживаться. Кроме того эта программа умеет кодировать и декодировать пакеты. Для того, чтобы посмотреть доступные опции работы программы, воспользуйтесь опцией –help. Модельный сервер располагается здесь: https://github.com/blackav/smart-home-binary

Условие задачи

Напишите программу, которая реализует функции хаба умного дома. В умном доме размещены сенсоры, то есть устройства, измеряющие параметры окружающей среды, актуаторы, то есть устройства, воздействующие на среду в доме, таймер, передающий показания текущего времени, и хаб, то есть устройство, которое управляет всеми остальными устройствами в умном доме.

Все устройства находятся в общей коммуникационной среде (сети). Информация по сети передаётся с помощью пакетов. Пакет может отправляться одному устройству, либо всем устройствам одновременно (broadcast). Каждое устройство, включая хаб, имеет свой уникальный 14-битный адрес в сети. Коммуникационная среда ненадёжна, то есть пакеты могут в сети теряться либо портиться, но пакеты не дублируются, то есть ситуация, когда получатель получает один и тот же пакет несколько раз, невозможна.

Для моделирования функциональности хаба умного дома ваша программа должна получать данные из сети, и в качестве реакции на полученные данные отправлять данные в сеть. В рамках данной задачи получение данных из сети и отправка данных в сеть моделируется с помощью HTTP POST запроса на специальный сервер, который моделирует функционирование остальных устройств умного дома.

Описание формата пакетов

Для описания формата данных, передаваемых по сети, используются следующие типы данных:

byte — беззнаковое 8-битное значение (октет).
bytes — массив байтов переменного размера, этот тип должен конкретизироваться в описании конкретных форматов пакетов в зависимости от типа пакета и типа устройства.
string — строка. Первый байт строки хранит ее длину, затем идут символы строки, в строке допустимы символы (байты) с кодами 32-126 (включительно).
varuint — беззнаковое целое число в формате ULEB128.
[]T — массив элементов типа T, первый байт памяти, отводимой под массив, это длина массива, далее следуют байты, кодирующие элементы массива.
struct — структура, состоящая из полей произвольного типа. Структура может иметь переменный размер, поскольку поля структуры могут иметь переменный размер, например varuint, string, и т. п.
Каждый пакет, передаваемый по сети, описывается следующим образом:

type packet struct {
  length byte
  payload bytes
  crc8 byte
};
Где:

length — это размер поля payload в октетах (байтах);
payload — данные, передаваемые в пакете, конкретный формат данных для каждого типа пакета описывается ниже;
GitHub
GitHub - blackav/smart-home-binary: Smart-home problem demo server binary releases
Smart-home problem demo server binary releases. Contribute to blackav/smart-home-binary development by creating an account on GitHub.
crc8 — контрольная сумма поля payload, вычисленная по алгоритму cyclic redundancy check 8 (http://www.sunshine2k.de/articles/coding/crc/understanding_c…).
Полезная нагрузка (поле payload) имеет следующую структуру:

type payload struct {
  src varuint
  dst varuint
  serial varuint
  dev_type byte
  cmd byte
  cmd_body bytes
};
Где:

src - это 14-битный “адрес” устройства-отправителя;
dst - 14-битный “адрес” устройства-получателя, причем адреса 0x0000 и 0x3FFF (16383) зарезервированы. Адрес 0x3FFF означает “широковещательную” рассылку, то есть данные адресованы всем устройствам одновременно;
serial - это порядковый номер пакета, отправленного устройством, от момента его включения. serial нумеруется с 1;
dev_type - это тип устройства, отправившего пакет;
cmd - это команда протокола;
cmd_body - формат которого зависит от команды протокола.
Тип устройства dev_type и команда cmd в совокупности определяют данные, которые передаются в cmd_body, как будет описано далее.

Описание команд протокола (cmd)

0x01 - WHOISHERE — отправляется устройством, желающим обнаружить своих соседей в сети. Адрес рассылки dst должен быть широковещательным 0x3FFF. Поле cmd_body описывает характеристики самого устройства в виде структуры:
type device struct {
  dev_name string
  dev_props bytes
};
Содержимое dev_props определяется в зависимости от типа устройства.

0x02 - IAMHERE — отправляется устройством, получившим команду WHOISHERE, и содержит информацию о самом устройстве в поле cmd_body. Команда IAMHERE отправляется строго в ответ на WHOISHERE. Команда отправляется на широковещательный адрес.
0x03 - GETSTATUS — отправляется хабом какому-либо устройству для чтения состояния устройства. Если устройство не поддерживает команду GETSTATUS (например, таймер), команда игнорируется.
0x04 - STATUS — отправляется устройством хабу и как ответ на запросы GETSTATUS, SETSTATUS, и самостоятельно при изменении состояния устройства. Например, переключатель отправляет сообщение STATUS в момент переключения. В этом случае адресом получателя устанавливается устройство, которое последнее отправило данному устройству команду GETSTATUS. Если такой команды ещё не поступало, сообщение STATUS не отправляется никому.
0x05 - SETSTATUS — отправляется хабом какому-либо устройству, чтобы устройство изменило свое состояние, например, чтобы включилась лампа. Если устройство не поддерживает изменение состояния (например, таймер), команда игнорируется.
0x06 - TICK — тик таймера, отправляется таймером. Периодичность отправления не гарантируется, но если на некоторый момент времени запланировано событие, то срабатывание события должно наступать, когда время, передаваемое в сообщении TICK становится больше или равно запланированному. Поле cmd_body содержит следующие данные:
type timer_cmd_body struct {
  timestamp varuint
};
Поле timestamp содержит текущее время в миллисекундах от 1970-01-01T00:00:00Z (java timestamp). Таким образом точность измерения времени составляет 1 мс, но тики таймера могут отправляться значительно реже, например, раз в 100 мс. Обратите внимание, что таймер показывает модельное время, а не реальное астрономическое время. Работа вашей программы, моделирующей хаб умного дома, должна быть привязана к модельному, а не к астрономическому времени.

Командами GETSTATUS, STATUS, SETSTATUS обмениваются два устройства, одно из которых - ваш хаб. Широковещательные адреса не допускаются.

Устройство должно ответить на широковещательный запрос WHOISHERE не позднее, чем через 300 мс. Устройство должно ответить на запросы GETSTATUS или SETSTATUS, адресованные данному устройству, не позднее, чем через 300 мс. Команда TICK не требует ответа.

Типы устройств

0x01 - SmartHub — это устройство, которое моделирует ваша программа, оно единственное устройство этого типа в сети;
0x02 - EnvSensor — датчик характеристик окружающей среды (температура, влажность, освещенность, загрязнение воздуха);
0x03 - Switch — переключатель;
0x04 - Lamp — лампа;
0x05 - Socket — розетка;
0x06 - Clock — часы, которые широковещательно рассылают сообщения TICK.
Часы гарантрированно присутствуют в сети и только в одном экземпляре.
Обработка команд в зависимости от типа устройства

SmartHub
WHOISHERE, IAMHERE — dev_props пуст
EnvSensor
WHOISHERE, IAMHERE — dev_props имеет следующую структуру:
type env_sensor_props struct {
  sensors byte
  triggers [] struct {
    op byte
    value varuint
    name string
  }
};
Поле sensors содержит битовую маску поддерживаемых сенсоров, где значения каждого бита обозначают следующее:
0x1 — имеется датчик температуры (сенсор 0);
0x2 — имеется датчик влажности (сенсор 1);
0x4 — имеется датчик освещенности (сенсор 2);
0x8 — имеется датчик загрязнения воздуха (сенсор 3).
Поле triggers — массив пороговых значений сенсоров для срабатывания. Здесь операция op имеет следующий формат:
Бит 0 (младший) — включить или выключить устройство;
Бит 1 — сравнивать по условию меньше (0) или больше (1);
Биты 2-3 — тип сенсора
value — это пороговое значение сенсора;
name — имя устройства, которое должно быть включено или выключено.
Обратите внимание, что EnvSensor не управляет устройствами непосредственно, функция управления переложена на хаб (вашу программу). Когда значение какого-либо сенсора переходит через пороговое, EnvSensor сам отправляет сообщение STATUS по адресу устройства, которое последним запрашивало GETSTATUS у EnvSensor. Таким образом нет необходимости в постоянном опросе значений сенсоров со стороны хаба.

STATUS — поле cmd_body содержит показания всех поддерживаемых устройством сенсоров как массив целых чисел.
type env_sensor_status_cmd_body struct {
  values []varuint
};
Показания всегда идут в порядке температура-влажность-освещенность-загрязнение воздуха. Например, если сенсор поддерживает датчики температуры и освещенности, то сначала всегда передаётся температура, а затем освещенность. Физические измерения передаются в следующей форме:
температура измеряется в 0.1 Кельвина, то есть температура -273.1 кодируется значением 0, а температура 0 по Цельсию кодируется значением 2731. Примечание: температура в Кельвинах и в Цельсиях связана следующим соотношением: ﻿C C=K+273.1C=K+273.1﻿, где ﻿C﻿ — температура в Цельсиях, а ﻿K﻿ — температура в Кельвинах;
относительная влажность измеряется в промилле, то есть принимает значения от 0 до 1000;
освещенность измеряется в десятых долях люкса;
загрязнение воздуха измеряется в десятых долях показателя PM2.5.
Switch
WHOISHERE, IAMHERE — dev_props является массивом строк. Каждая строка — это имя (dev_name) устройства, которое подключено к данному выключателю. Включение выключателя должно включать все устройства, а выключение должно выключать. За это отвечает хаб (ваша программа).
STATUS — поле cmd_body имеет размер 1 байт и содержит значение 0, если переключатель находится в положении OFF, и 1, если переключатель находится в положении ON.
Lamp и Socket
WHOISHERE, IAMHERE — массив dev_props пустой (имеет нулевой размер);
STATUS — поле cmd_body имеет размер 1 байт и содержит значение 0, если переключатель находится в положении OFF, и 1, если переключатель находится в положении ON;
SETSTATUS — поле cmd_body должно иметь размер 1 байт и содержать 0 для выключения устройства и 1 для включения устройства.
Работа вашей программы

Вашей программе в аргументе командной строки передается URL, который нужно использовать для обмена данными с сетью, и 14-битный адрес вашего устройства-хаба в сети в шестнадцатеричном виде. Для отправки данных в сеть необходимо выполнить POST-запрос передав в теле запроса выдаваемые в сеть пакеты данных в виде URL-encoded Base64 строки. В ответ POST-запрос вернет пакеты данных, принятые из сети. Каждую порцию данных таким образом можно прочитать только один раз. В случае успешного чтения очередной порции данных сервер вернет код ответа HTTP 200. В случае, когда данных больше нет, сервер вернет код ответа HTTP 204. При получении кода 204 ваша программа должна завершить свою работу с кодом завершения 0.

Пример запуска вашей программы:

solution http://localhost:12183 ef0
Если при взаимодействии с сервером возникла какая-либо ошибка на уровне сети или протокола HTTP, или сервер вернул код ответа, отличный от 200 или 204, ваша программа должна завершить выполнение с кодом завершения 99.

Каждая порция входных данных закодирована с помощью URL-encoded unpadded Base64, причем все пробельные символы (пробелы, табуляции, переводы строк) являются незначимыми и должны быть проигнорированы. Если порция данных закодирована в base64 некорректно, она должна быть пропущена, и ваша программа должна перейти к обработке следующей порции данных.

Ваша программа должна моделировать работу хаба по следующему алгоритму:

В начале работы хаб собирает информацию о сети с помощью запроса WHOISHERE и запрашивает начальное состояние всех устройств
Далее хаб ожидает сообщений от сенсоров, и для каждого сообщения от сенсора выполняются действия, прописанные в dev_props сенсора
Все подключенные устройства кроме хаба и таймера в процессе работы могут быть отключены. Отключенное устройство перестает отвечать на сообщения, адресованные этому устройству. Устройства могут быть включены в процессе работы, и могут появиться новые устройства. Повторно или вновь включенное устройство посылает в сеть запрос WHOISHERE, на который хаб должен ответить. Конфигурация устройства (dev_props) может отличаться от конфигурации, которая была в прошлый раз, в этом случае хаб должен начать работать с новой конфигурацией.

Если хаб отправил команду какому-то конкретному устройству и не получил от него ответа в течение 300мс, это устройство считается выключенным и на него больше не отправляется команд, а сообщения от него игнорируются, пока это устройство само не объявит о своем включении с помощью запроса WHOISHERE.

Замечание

Ваша программа должна полностью размещаться в одном файле исходного кода. Ваша программа не должна ничего считывать со стандартного потока ввода или выводить на стандартный поток вывода, кроме того закрывать эти потоки нельзя. Весь обмен данными ведется с помощью HTTP POST запросов.

EN task description 

Preface

You are offered a Smart Home modeling assignment to complete.

If you write in python, java, go, you can only use the standard library of the corresponding languages. If you write in scala, you are allowed to use sttp (https://sttp.softwaremill.com/en/stable/examples.html#use-th...) in addition to the standard library.

The assignment will not require knowledge of any language-specific constructs or features of the standard library or API, except for working with HTTP POST requests to the server. You will be required to know the basics of programming, such as working with strings, byte sequences and bits within bytes, and basic data structures (arrays, lists).

The text of the assignment may seem big and scary at first, and to cope with the complexity you can divide the assignment into several steps.

Learn to connect to the server and make HTTP POST requests to the server, passing it already ready encoded messages.
Learn to encode and decode URL-encoded unpadded base64 to send to the server and receive data from the server.
Learn to encode and decode packets from binary form into your internal representation.
Learn to identify the structure of a network of devices using WHOISHERE.
Learn to control one lamp and one switch.
Realize all the required functionality.
For your convenience you are provided with a ready (console) program - a model server, on which you can debug. In addition, this program can encode and decode packets. To see the available options of the program, use the -help option. The model server is located here: https://github.com/blackav/smart-home-binary.

Problem statement

Write a program that implements the functions of a smart home hub. A smart house contains sensors, i.e. devices that measure environmental parameters, actuators, i.e. devices that influence the environment in the house, a timer that transmits current time readings, and a hub, i.e. a device that controls all other devices in the smart house.

All the devices are in a common communication environment (network). Information on the network is transmitted using packets. A packet can be sent to one device, or to all devices at the same time (broadcast). Each device, including the hub, has its own unique 14-bit address on the network. The communication medium is unreliable, i.e. packets can be lost or corrupted in the network, but packets are not duplicated, i.e. the situation when a recipient receives the same packet several times is impossible.
To simulate the functionality of a smart home hub, your program must receive data from the network and send data to the network as a response to the received data. In this task, receiving data from the network and sending data to the network is modeled using an HTTP POST request to a special server that simulates the functioning of the rest of the smart home devices.

Description of packet format

The following data types are used to describe the format of data transmitted over the network:

byte - an unsigned 8-bit value (octet).
bytes - an array of bytes of variable size, this type should be specified in the description of specific packet formats depending on the packet type and device type.
string - string. The first byte of the string stores its length, followed by the characters of the string, characters (bytes) with codes 32-126 (inclusive) are allowed in the string.
varuint - unsigned integer in ULEB128 format.
[]T - an array of elements of type T, the first byte of memory allocated for the array is the length of the array, followed by bytes encoding the array elements.
struct - a structure consisting of fields of arbitrary type. A struct can have a variable size because the fields of a struct can have a variable size, e.g. varuint, string, etc.
Each packet transmitted over the network is described as follows:

type packet struct {
  length byte
  payload bytes
  crc8 byte
};

Where:

length is the size of the payload field in octets (bytes);
payload is the data transmitted in the packet, the specific data format for each packet type is described below;
GitHub
GitHub - blackav/smart-home-binary: Smart-home problem demo server binary releases
Smart-home problem demo server binary releases. Contribute to blackav/smart-home-binary development by creating an account on GitHub.
crc8 - checksum of the payload field calculated by the cyclic redundancy check 8 algorithm (http://www.sunshine2k.de/articles/coding/crc/understanding_c...).
The payload (payload field) has the following structure:

type payload struct {
  src varuint
  dst varuint
  serial varuint
  dev_type byte
  cmd byte
  cmd_body bytes
};
Where:

src is the 14-bit "address" of the sending device;
dst is the 14-bit "address" of the destination device, with addresses 0x0000 and 0x3FFF (16383) reserved. Address 0x3FFF means "broadcasting", i.e. the data is addressed to all devices at the same time;
serial is the serial number of the packet sent by the device from the time it was enabled. Serial is numbered from 1;
dev_type is the type of the device that sent the packet;
cmd is a protocol command;
cmd_body - whose format depends on the protocol command.
The dev_type and cmd together define the data that is passed to cmd_body, as described later.


Description of protocol commands (cmd)

0x01 - WHOISHERE - sent by a device wishing to discover its neighbors on the network. The dst distribution address must be broadcast 0x3FFF. The cmd_body field describes the characteristics of the device itself in the form of a structure:
type device struct {
  dev_name string
  dev_props bytes
};
The content of dev_props is defined depending on the device type.

0x02 - IAMHERE - sent by the device that received the WHOISHERE command and contains information about the device itself in the cmd_body field. The IAMHERE command is sent strictly in response to WHOISHERE. The command is sent to a broadcast address.
0x03 - GETSTATUS - sent by the hub to some device to read the device status. If the device does not support the GETSTATUS command (for example, a timer), the command is ignored.
0x04 - STATUS - sent by the device to the hub both as a response to GETSTATUS, SETSTATUS requests and independently when the device state changes. For example, a switch sends a STATUS message at the moment of switching. In this case, the recipient address is the device that last sent the GETSTATUS command to this device. If no such command has yet been received, the STATUS message is not sent to anyone.
0x05 - SETSTATUS - sent by the hub to some device to make the device change its state, for example, to turn on a lamp. If the device does not support state change (such as a timer), the command is ignored
0x06 - TICK - timer tick, sent by the timer. The frequency of sending is not guaranteed, but if an event is scheduled at some point in time, the event should be triggered when the time transmitted in the TICK message becomes greater than or equal to the scheduled time. The cmd_body field contains the following data:
type timer_cmd_body struct {
  timestamp varuint
};
The timestamp field contains the current time in milliseconds from 1970-01-01T00:00:00:00Z (java timestamp). Thus the timestamp is accurate to 1 ms, but timer ticks can be sent much less frequently, for example once every 100 ms. Note that the timer shows the modeled time, not the real astronomical time. The operation of your program simulating the smart home hub should be tied to model time, not astronomical time.

The GETSTATUS, STATUS, SETSTATUS commands are exchanged between two devices, one of which is your hub. Broadcast addresses are not allowed.

The device must respond to a broadcast WHOISHERE request no later than 300 ms. The device must respond to GETSTATUS or SETSTATUS requests addressed to this device no later than 300 ms. The TICK command does not require a response.

Device types

0x01 - SmartHub - this is the device that your program simulates, it is the only device of this type in the network;
0x02 - EnvSensor - sensor of environmental characteristics (temperature, humidity, light, air pollution);
0x03 - Switch - switch;
0x04 - Lamp - lamp;
0x05 - Socket - socket;
0x06 - Clock - the clock that broadcasts TICK messages.
The clock is guaranteed to be present in the network and only in one copy.
Command processing depending on the device type

SmartHub
WHOISHERE, IAMHERE - dev_props is empty
EnvSensor
WHOISHERE, IAMHERE - dev_props has the following structure:
type env_sensor_props struct {
  sensors byte
  triggers [] struct {
    op byte
    value varuint
    name string
  }
};
The sensors field contains a bitmask of supported sensors, where the values of each bit denote the following:
0x1 - there is a temperature sensor (sensor 0);
0x2 - there is a humidity sensor (sensor 1);
0x4 - there is a light sensor (sensor 2);
0x8 - there is an air pollution sensor (sensor 3).
The triggers field is an array of sensor thresholds for triggering. Here the op operation has the following format:
Bit 0 (lowest) - enable or disable the device;
Bit 1 - compare by the condition less than (0) or greater than (1);
Bits 2-3 are the sensor type
value - is the threshold value of the sensor;
name - the name of the device to be enabled or disabled.

Note that EnvSensor does not control the devices directly, the control function is delegated to the hub (your program). When the value of a sensor crosses a threshold, EnvSensor itself sends a STATUS message to the address of the device that last requested GETSTATUS from EnvSensor. Thus, there is no need for constant polling of sensor values by the hub.

STATUS - cmd_body field contains the readings of all sensors supported by the device as an array of integers.
type env_sensor_status_cmd_body struct {
  values []varuint
};
Readings always go in the order temperature-humidity-luminescence-pollution-air pollution. For example, if the sensor supports temperature and illumination sensors, temperature is always transmitted first, followed by illumination. Physical measurements are transmitted in the following form:
temperature is measured in 0.1 Kelvin, that is, a temperature of -273.1 is coded with a value of 0, and a temperature of 0 Celsius is coded with a value of 2731. Note: temperature in Kelvin and temperature in Celsius are related by the following relationship: C C=K+273.1C=K+273.1 , where C is the temperature in Celsius and K is the temperature in Kelvin;
relative humidity is measured in ppm, i.e. it takes values from 0 to 1000;
illuminance is measured in tenths of lux;
air pollution is measured in tenths of PM2.5.

Switch
WHOISHERE, IAMHERE - dev_props is an array of strings. Each string is the name (dev_name) of a device that is connected to this switch. Turning the switch on should turn all devices on, and turning it off should turn them off. This is the responsibility of the hub (your program).
STATUS - the cmd_body field is 1 byte in size and contains a value of 0 if the switch is in the OFF position and 1 if the switch is in the ON position.
Lamp and Socket
WHOISHERE, IAMHERE - the dev_props array is empty (has size zero);
STATUS - the cmd_body field is 1 byte in size and contains a value of 0 if the switch is in the OFF position and 1 if the switch is in the ON position;
SETSTATUS - the cmd_body field must be 1 byte in size and contain 0 to turn the device off and 1 to turn the device on.
Operation of your program

Your program is passed the URL to use to communicate with the network and the 14-bit address of your hub device on the network in hexadecimal form in the command line argument. To send data to the network you need to execute a POST-request, passing in the body of the request the data packets to be sent to the network in the form of URL-encoded Base64 string. In response POST-request will return the data packets received from the network. Each data portion can be read only once in this way. If the next portion of data is successfully read, the server will return an HTTP 200 response code. If there is no more data, the server will return an HTTP 204 response code. When you receive the 204 code, your program should start the HTTP 200 response.
When there is no more data, the server will return an HTTP response code 204. Upon receiving the 204 code, your program should terminate with a termination code of 0.

Here is an example of running your program:

solution http://localhost:12183 ef0
If any network or HTTP protocol error occurs while communicating with the server, or if the server returns a response code other than 200 or 204, your program should terminate with a termination code of 99.

Each chunk of input data is encoded using URL-encoded unpadded Base64, with all whitespace characters (spaces, tabs, line feeds) being insignificant and should be ignored. If a portion of data is encoded in base64 incorrectly, it should be skipped and your program should proceed to processing the next portion of data.

Your program should simulate the work of the hub according to the following algorithm:

At the beginning, the hub gathers information about the network using the WHOISHERE query and requests the initial state of all devices
Next, the hub waits for sensor messages, and for each sensor message, the actions defined in the sensor's dev_props are executed
All connected devices except the hub and the timer can be disconnected during operation. A disabled device stops responding to messages addressed to that device. Devices may be enabled during operation, and new devices may appear.
A re- or re-enabled device sends a WHOISHERE request to the network, to which the hub must respond. The configuration of the device (dev_props) may be different from the last time it was configured, in which case the hub should start with the new configuration.

If the hub has sent a command to a particular device and has not received a response from it within 300ms, that device is considered disabled and no more commands are sent to it, and messages from it are ignored until that device itself announces that it is powered on with a WHOISHOISHERE request.

Note

Your program should reside entirely in a single source code file. Your program must not read anything from the standard input stream or output to the standard output stream, and you must not close these streams. All data exchange is done with HTTP POST requests.
