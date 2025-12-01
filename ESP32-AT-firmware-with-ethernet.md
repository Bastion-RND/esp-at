## Перекомпиляция AT прошивки для ESP32: Ethernet + hardware connection

## Компиляция в git репозитории
### Краткая выжимка [инструкции](https://docs.espressif.com/projects/esp-at/en/release-v4.1.0.0/esp32/Compile_and_Develop/How_to_build_project_with_web_page.html):
1. Выбрать нужную ветку или создать новую (в моём случае mass-notification-system)
2. Находясь на странице в github нажмите `.` на клавиатуре - откроется редактор github.dev
3. Файл конфигурации находится по пути `esp-at/module_config/<your_module_name>/sdkconfig.defaults` или `esp-at/module_config/<your_module_name>/sdkconfig-silent.defaults`, в зависимости от того, включен ли [silent mode](https://docs.espressif.com/projects/esp-at/en/release-v4.1.0.0/esp32/Compile_and_Develop/How_to_configure_silence_mode.html) (по умолчанию включен)
4. Добавить в файл необходимые конфигурационные настройки - полный список можно посмотреть в любой скомпилированной прошивке в файле `sdkconfig`
5. Таким же образом добавляются или изменяются любые другие файлы прошивки, которые можно поменять локально из `idf.py` - смотреть инструкцию ниже
6. После коммита автоматически запустятся git actions - а именно в разделе actions репозитория появится последний коммит, а компиляция будет поставлена в список ожидания
7. Зайдите на последний коммит. Внизу страницы будет [список скомпилированных прошивок](https://docs.espressif.com/projects/esp-at/en/release-v4.1.0.0/esp32/Compile_and_Develop/How_to_download_the_latest_temporary_version_of_AT_from_github.html), отсюда их можно скачать и залить на устройство с помощью esptool или [flash download tool](https://dl.espressif.com/public/flash_download_tool.zip) ([инструкция](https://docs.espressif.com/projects/esp-test-tools/en/latest/esp32/production_stage/tools/flash_download_tool.html))

## Включение модуля Ethernet в esp-at

## Локальная компиляция
#### Предусловия:
1. Нужен модуль ESP32, который поддерживает Ethernet (в моем случае esp-wroom-32)
2. Лучше всего это делать из под Linux (в моем случае WSL с Ubuntu)
3. Чтобы прокинуть usb в WSL можно использовать [usbip](https://github.com/dorssel/usbipd-win)
4. esp-idf в моем случае размещена в папке `~/esp/esp-idf`, проекты в `~/esp/projects`
5. Почитать про AT прошивку, всевозможные команды и настройки можно в [официальной документации](https://docs.espressif.com/projects/esp-at/en/latest/esp32/?spm=a2ty_o01.29997173.0.0.5c6bc921P9XLFs#)

### Краткая выжимка [инструкции](https://docs.espressif.com/projects/esp-at/en/release-v4.1.0.0/esp32/Compile_and_Develop/How_to_clone_project_and_compile_it.html):
1. Поставить [esp-idf](https://docs.espressif.com/projects/esp-idf/en/release-v5.4/esp32/get-started/linux-macos-setup.html) и собрать первый проект (hello world)
2. Склонировать проект [esp-at](https://github.com/espressif/esp-at) с нужной ветки (обычно master) `git clone --recursive https://github.com/espressif/esp-at.git`
3. ОБЯЗАТЕЛЬНО активировать окружение esp-idf `. $HOME/esp/esp-idf/export.sh`
4. В папке с проектом esp-at выполнить `./build.py install`. 
При первом запуске в консольном меню выбрать платформу (ESP32), контроллер (WROOM-32) и silent mode (обычно надо отключать, позволяет видеть все AT логи, но увеличивает размер прошивки). 
Для перевыбора опций надо удалить из проекта файл `rm build/module_info.json`
5. Далее можно настраивать проект через menuconfig `./build.py menuconfig`. Подробные настройки будут ниже
6. После настройки можно собирать проект `./build.py build`
7. После компиляции можно загрузить проект на модуль `idf.py flash` и проверять через `idf.py monitor`

### Настройки необходимые для включения Ethernet:
1. Включить `./build.py menuconfig` -> `Component config` -> `AT` -> `AT ethernet support`
2. Выбрать PHY `./build.py menuconfig` -> `Component config` -> `AT` -> `AT ethernet support` -> `Ethernet PHY` -> Нужная физика (у меня Microchip LAN8720 PHY)
3. Для конкретного модуля LAN8720 также нужно настроить следующее:
   1. `./build.py menuconfig` -> `Component config` -> `Ethernet` -> `Support ESP32 internal EMAC controller` -> `RMII clock mode` -> `Output RMII clock from internal`
   2. Для тактования будем использовать GPIO16 (EMAC_CLK_OUT) `./build.py menuconfig` -> `Component config` -> `Ethernet` -> `Support ESP32 internal EMAC controller` -> `RMII clock GPIO number` -> `16`
4. Также нужно перенастроить UART AT пины, потому как стандартные пины 16 и 17 используются для тактования Ethernet ([инструкция в которой это указано](https://docs.espressif.com/projects/esp-at/en/latest/esp32/AT_Command_Set/Ethernet_AT_Commands.html))
   1. Для того чтобы отредактировать пины вводим в консоли `nano components/customized_partitions/raw_data/factory_param/factory_param_data.csv`
   2. Находим строку с нашим модулем - `WROOM-32`, и редактируем пины в колонках `description`, `uart_tx_pin`, `uart_rx_pin`, `uart_cts_pin`, `uart_rts_pin`. 
   Например, можно перенести TX на 2, RX на 4, а CTS и RTS выставить в -1 как неиспользуемые

## Подключение модуля к ESP32:
1. Все пины модуля подключить к соответствующим пинам из таблицы [Common Pin Assignments](https://github.com/espressif/esp-idf/blob/v5.5.1/examples/ethernet/README.md#common-pin-assignments)
   
   | GPIO   | RMII Signal | Notes        |
   | ------ | ----------- | ------------ |
   | GPIO21 | TX_EN       | EMAC_TX_EN   |
   | GPIO19 | TX0         | EMAC_TXD0    |
   | GPIO22 | TX1         | EMAC_TXD1    |
   | GPIO25 | RX0         | EMAC_RXD0    |
   | GPIO26 | RX1         | EMAC_RXD1    |
   | GPIO27 | CRS_DV      | EMAC_RX_DRV  |
   | GPIO16 | EMAC_CLK_OUT| output       |
   | GPIO23 | MDC         | Output to PHY|
   | GPIO18 | MDIO        | Bidirectional|

2. Подключить GPIO2 и GPIO4 в качестве AT TX/RX

### Результат:
1. Ethernet разъем должен светиться зеленым без подключенного кабеля и светиться/моргать зеленым и оранжевым с подключенным кабелем. Если ничего не светится - возможно проблема с тактованием модуля - перепроверить пины
2. При подключении кабеля мы должны получить сообщение `+ETH_CONNECTED`, `+ETH_GOT_IP: x.x.x.x`
3. При запросе команды `AT+CIPETHMAC?` мы должны получить MAC адрес - если получаем ERROR значит либо неправильно скомпилирован проект, либо неправильно настроен/подключен пин тактования
   1. Для лучшей отладки можно включить логгирование модуля - `./build.py menuconfig` -> `Component config` -> `Log` -> `Log level` -> `Debug` 
   2. А также включить логгирование AT - `./build.py menuconfig` -> `Component config` -> `AT` -> `AT Log level` -> `Debug`

