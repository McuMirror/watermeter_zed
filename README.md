# Watermeter Zigbee Telink TLSR8258 (E-BYTE E180-Z5812SP)

[Repository watermeter_zed](https://github.com/slacky1965/watermeter_zed)

---

**Описание**

* Рассчитано на два счетчика воды.
* Не работает с системой namur и счетчиками, где применен датчик "холла". Только замыкание-размыкание, например геркон.
* Ведет подсчет замыканий-размыканий, увеличивая каждый раз количество литров на заданное значение от 1 до 10 литров (по умолчанию 10 литров на один импульс).
* Сохраняет показания в энергонезависимой памяти модуля.
* Передает показания по сети Zigbee.
* Взаимодейстивие с "умными домами" через zigbee2mqtt.
* Первоначальная настройка происходит через web-интерфейс zigbee2mqtt.
* Сбросить устройство до заводских значений (zigbee) - нажать кнопку 5 раз.
* Сделать restart модуля - зажать кнопку бюолее чем на 5 секунд.
* При одиночном нажатии кнопки модуль просыпается.

---

**Железо**

В проекте используется модуль от компании E-BYTE на чипе TLSR8258 - E180-Z5812SP.

<img src="https://raw.githubusercontent.com/slacky1965/watermeter_zed/main/doc/images/E180-Z5812SP.jpg" alt="E-BYTE E180-Z5812SP"/>

Испытывалось все на вот таком dongle от Telink

<img src="https://raw.githubusercontent.com/slacky1965/watermeter_zed/main/doc/images/telink_tlsr8258_dongle.jpg" alt="Telink TLSR8258 dongle"/>

---

**Корпус**

Корпус взят от Battery Holder Box на 3 батарейки АА. Переделываем на 2 батарейки и получаем питание 3 вольта и один сегмент под плату.
<img src="https://raw.githubusercontent.com/slacky1965/watermeter_zed/main/doc/box/box1.jpg" alt="BOX 3AA"/>
<img src="https://raw.githubusercontent.com/slacky1965/watermeter_zed/main/doc/box/box2.jpg" alt="BOX 3AA"/>
<img src="https://raw.githubusercontent.com/slacky1965/watermeter_zed/main/doc/box/box3.jpg" alt="BOX 3AA"/>
<img src="https://raw.githubusercontent.com/slacky1965/watermeter_zed/main/doc/box/box4.jpg" alt="BOX 3AA"/>

---

**Плата**

Плата разведена, но пока не заказана и не проверена.
<img src="https://raw.githubusercontent.com/slacky1965/watermeter_zed/main/doc/images/board_top.jpg" alt="Board top"/>


---

**Готовое устройство**

Пока нет.

---

**Софт**

[Последнюю прошивку](https://github.com/slacky1965/watermeter_zed/raw/main/watermeter_zed_V1.2.bin) нужно залить в модуль с помощью [github.com/pvvx/TLSRPGM](https://github.com/pvvx/TLSRPGM) или оригинального программатора от Telink
<img src="https://raw.githubusercontent.com/slacky1965/watermeter_zed/main/doc/images/telink_pgm.jpg" alt="Telink PGM"/>

---

**Описание работы модуля**

В первый старт происходит попытка подключения к сети Zigbee. Если попытка удалась, модуль переходит в штатный режим работы. 

* Polling
	
> Первые 30 секунд модуль просыпается раз в 3 секунды. По истечению 30 секунд модуль засыпает на 5 минут. Через 5 минут опять просыпается раз в 3 секунды последующие 30 секунд. И опять засыпает на 5 минут. И так по кругу. Можно было сделать просыпание модуля раз в 5 минут, но zigbee2mqtt поллит устройства примерно раз в 10 минут. И начинает ругаться в логе, что устройство не найдено и выставляет статус offline. При такой неровной схеме эта проблема устраняется. Сделать же период просыпания всегда раз в 3 секунды - необоснованное расходование ресурса батарейки.

* Reporting

> Модуль высылает четыре разных отчета. Два для батарейки и еще два для счетчика холодной и горячей воды. Период отправки у них разный. 

> Напряжение на батарейке модуль проверяет один раз в 15 минут. Отчеты о состоянии батарейки (напряжение в мВ и остаточный ресурс в %) высылается не ранее, чем через 5 минут от предыдущего, но и не позднее 1 часа.

> Отчет о значении счетчика воды высылается сразу при увеличении этого значения. И не позднее 5 минут от предыдущего отчета. Если 5 минут кому-то покажется слишком частым, это всегда можно исправить через web-интерфейс zigbee2mqtt.

* Память модуля и где хранится конфиг

> Согласно спецификации на чип TLST8258 память распределена следующим образом

		0x00000 Old Firmware bin
		0x34000 NV_1
		0x40000 OTA New bin storage Area
		0x76000 MAC address
		0x77000 C_Cfg_Info
		0x78000 U_Cfg_Info
		0x7A000 NV_2
		0x80000 End Flash

> Получается, что прошивка не может быть больше, чем 0x34000 (что собственно и подтверждается, если проверить SDK на предмет определения размера заливаемого файла при обновлении OTA), но при использовании прошивки с адреса 0x40000 видно, что под нее отведено не 0x34000, а 0x36000. Выходит, что 0x2000 никогда не используются. Этим мы и воспользуется для хранения промежуточного конфига.

> После аппаратной заливки прошивки в модуль, он всегда стартует с адреса 0x00000. После обновления OTA, адрес старта меняется. Если до обновления он был 0x00000, то после он становится 0x40000. Если до обновления он был 0x40000, то после - 0x00000. И так по кругу после каждого обновления OTA.

> В момент старта модуля происходит проверка, с какого адреса загружается прошивка - с 0x00000 или с 0x40000. Если она грузится с адреса 0x00000, то область с 0x40000 до 0x74000 мы используем длял хранения конфига. Если прошивка грузится с адреса 0x40000, то для хранения конфига мы используем уже облать с 0x00000 до 0x34000.

> Конфиг пишется в выбранную область каждый раз при срабатывании счетчика воды с шагом 0x100. Т.е. первый раз конфиг записался по адресу 0x40000 (0x00000), вот второй раз 0x40100 (0x00100), в третий - 0x40200 (0x00200) и т.д. пока не достигнет границы 0x74000 (0x34000). И далее начинает опять записываться с начального адреса 0x40000 (0x00000).

> В момент обновления OTA конфиг сохраняется в неиспользуемую область по адресу 0x74000. И будет там сохранятся, пока обновление OTA удачно не завершится. При неудаче (это будет видно через логи и web-интерфейс zigbee2mqtt) нужно обязательно запустить обновление снова до успешного завершения. Иначе конфиг так и продолжит сохраняться только по адресу 0x74000.

> После удачного завершения обновления OTA модуль перезагружается, считывает конфиг с адреса 0x74000, проверяет по какому адресу нужно записывать конфиг в штатном режиме и сохраняет его уже по адресу 0x00000 или 0x40000. И так до следующего обновления.

---

**Настройка**

Продолжение следует.

---

**История версий**

- 1.0 - Начало.
- 1.1 - Прошивка без ОТА. 
- 1.2 - Добавлена возможность обновления OTA. 
