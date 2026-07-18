# Руководство по сборке हाई-End сетевого аудио-транспорта

Полное руководство по интеграции **Orange Pi PC+ (Allwinner H3)** в режиме **I2S Slave**, DSP-процессора **ADAU1467** (Тактовый мастер через ASRC) и внешнего Android-устройства (с Kodi) для автоматического воспроизведения аудиофайлов и фильмов в качестве до **24 бит / 192 кГц** без рассинхронизации звука и видео (Lipsync).

---

## 1. Архитектура системы и коммутация

Система использует аппаратный асинхронный конвертер частоты дискретизации (**ASRC**) внутри ADAU1467. Это позволяет процессору Orange Pi PC+ всегда работать в режиме **I2S Slave**, а тактовым мастером для всей системы выступает высококачественная сетка генераторов 11.2896 / 12.288 МГц, подключенная к DSP.

### Схема прохождения сигналов
1. **Звук фильма (до 24/192)** транслируется из Kodi (Android) по локальной сети через протокол *Slimproto* в плеер Squeezelite на Orange Pi PC+.
2. **Входной порт ADAU1467** (в режиме Master) генерирует опорные BCLK/LRCK для Orange Pi.
3. **Orange Pi PC+** (в режиме Slave) принимает клоки и возвращает поток данных (SDATA) обратно в DSP.
4. **Блок ASRC** внутри ADAU1467 на лету пересчитывает входную частоту фильма в стабильную частоту вашего ЦАП PCM5242 (например, фиксированные 96 кГц или 192 кГц), тактируемую напрямую от генераторов.

### Схема физического подключения (Распиновка)

| Контакт Orange Pi PC+ (40-pin) | Сигнал | Направление | Контакт ADAU1467 |
| :--- | :--- | :--- | :--- |
| **Pin 39 (GND)** | Общая земля | — | GND (DSP) |
| **Pin 12 (PA18 / I2S0_BCLK)** | Bit Clock (BCLK) | **Вход** (←) | BCLK_OUT (Входной порт DSP) |
| **Pin 35 (PA19 / I2S0_LRCK)** | Frame Clock (LRCK) | **Вход** (←) | LRCLK_OUT (Входной порт DSP) |
| **Pin 40 (PA20 / I2S0_DOUT)**| Serial Data (SDATA)| **Выход** (→)| SDATA_IN (Входной порт DSP) |

*Важно: Пин MCLK на Orange Pi PC+ оставляйте пустым. Ваши генераторы 11.2896 / 12.288 МГц припаиваются к аппаратным входам выбора клока ADAU1467.*

---

## 2. Настройка ОС Armbian на Orange Pi PC+

Для корректной работы I2S в режиме Slave требуется использовать актуальную версию **Armbian (ядро 6.x и выше)**, где исправлены ошибки полярности тактовых сигналов в основном драйвере `sun4i-i2s.c`.

### Шаг 2.1. Создание Device Tree Overlay
Для принудительного переключения аудио-контроллера процессора Allwinner H3 в режим ведомого (Slave) необходимо создать кастомный оверлей дерева устройств.

1. Подключитесь к Orange Pi по SSH.
2. Создайте файл оверлея:
   ```bash
   nano h3-i2s-slave.dts
   ```
3. Вставьте следующий исходный код конфигурации:
   ```dts
   /dts-v1/;
   /plugin/;

   / {
       compatible = "allwinner,sun8i-h3";

       fragment@0 {
           target = <&i2s0>;
           __overlay__ {
               status = "okay";
               /* Указываем процессору слушать внешние BCLK и LRCK */
               bitclock-master = <&adau1467_codec>;
               frame-master = <&adau1467_codec>;
           };
       };

       fragment@1 {
           target-path = "/";
           __overlay__ {
               /* Виртуальный кодек для инициализации связки */
               adau1467_codec: adau1467-codec {
                   #sound-dai-cells = <0>;
                   compatible = "linux,spdif-dit";
                   status = "okay";
               };

               /* Создание звуковой карты ALSA */
               sound_i2s {
                   compatible = "simple-audio-card";
                   simple-audio-card,name = "ADAU1467-I2S-Slave";
                   simple-audio-card,format = "i2s";

                   simple-audio-card,bitclock-master = <&sound_master>;
                   simple-audio-card,frame-master = <&sound_master>;

                   sound_master: simple-audio-card,cpu {
                       sound-dai = <&i2s0>;
                   };

                   simple-audio-card,codec {
                       sound-dai = <&adau1467_codec>;
                   };
               };
           };
       };
   };
   ```
4. Сохраните файл (`Ctrl+O`, затем `Enter`) и выйдите (`Ctrl+X`).

### Шаг 2.2. Компиляция и активация оверлея
1. Скомпилируйте исходный текст `.dts` в бинарный формат `.dtbo`:
   ```bash
   dtc -I dts -O dtb h3-i2s-slave.dts -o h3-i2s-slave.dtbo
   ```
2. Скопируйте скомпилированный оверлей в системную директорию пользовательских надстроек:
   ```bash
   sudo cp h3-i2s-slave.dtbo /boot/overlay-user/
   ```
3. Откройте конфигурационный файл загрузки Armbian:
   ```bash
   sudo nano /boot/armbianEnv.txt
   ```
4. Добавьте или отредактируйте строку для активации вашего оверлея при старте:
   ```text
   user_overlays=h3-i2s-slave
   ```
5. Перезагрузите Orange Pi PC+:
   ```bash
   sudo reboot
   ```
6. После перезагрузки проверьте успешное появление новой звуковой карты ALSA в системе:
   ```bash
   aplay -l
   ```
   *В списке должна отобразиться карта с именем `ADAU1467-I2S-Slave`.*

---

## 3. Настройка сетевого плеера Squeezelite

Плеер Squeezelite будет принимать High-Res аудиопоток по сети и выводить его напрямую в сконфигурированную карту ALSA Slave.

1. Установите Squeezelite из официального репозитория:
   ```bash
   sudo apt update
   sudo apt install squeezelite
   ```
2. Откройте конфигурационный файл сервиса:
   ```bash
   sudo nano /etc/default/squeezelite
   ```
3. Приведите файл к следующему виду, настроив параметры буферизации для исключения задержек (Lipsync) и разрешив вывод частот до 192 кГц:
   ```text
   # Имя плеера в сети LMS
   SLURM_NAME="OPI-Audio-Transport"

   # Настройки вывода: hw:1,0 (замените на индекс вашей карты из aplay -l)
   # -b 500:2000 минимизирует внутренний буфер ALSA для синхронизации с видео
   # -u vH::0 отключает программный апсемплинг, сохраняя Bit-perfect 24/192
   SB_EXTRA_ARG="-d=hw:1,0 -b 500:2000 -u vH::0"
   ```
4. Перезапустите службу Squeezelite для применения настроек:
   ```bash
   sudo systemctl restart squeezelite
   ```

---

## 4. Конфигурация Android и Kodi (Автоматическая синхронизация)

Чтобы высокобитрейтный звук (24/192) фильмов автоматически синхронизировался с видеорядом на внешнем Android без ручной корректировки ползунков задержки, используется сетевой протокол **Slimproto (LMS)**. Он привязывает пакеты аудио и видео к единым системным часам времени (Network Wall-Clock).

1. Запустите ваше внешнее Android-устройство и откройте **Kodi**.
2. Перейдите в раздел **Дополнения (Add-ons)** ➔ **Установка из репозитория** ➔ **Аудио-дополнения**.
3. Найдите и установите плагин **Squeezebox Player** (или *XSqueeze / KodiSqueeze* в зависимости от сборки).
4. Зайдите в настройки установленного плагина и укажите IP-адрес вашего главного сервера **Logitech Media Server (LMS)**, который управляет плеером Squeezelite.
5. Перейдите в общие *Настройки Kodi ➔ Система ➔ Аудио*:
   * Активируйте пункт **«Разрешить сквозной вывод (Passthrough)»**.
   * В качестве основного аудиовыхода выберите виртуальное устройство, созданное плагином Squeezebox.

### Как работает автоматика:
Когда вы включаете фильм с дорожкой 24/192 в Kodi на Android, плагин перехватывает аудиопоток. Протокол Slimproto автоматически транслирует аудио на Orange Pi PC+ в Squeezelite, непрерывно подстраивая скорость воспроизведения кадров на экране Android под тактовую частоту сэмплов, которую Orange Pi запрашивает от аппаратных часов ADAU1467.

---

## 5. Конфигурация ADAU1467 в SigmaStudio

Для успешного приема прыгающей частоты от Orange Pi без зависания звуковой карты, входной порт DSP должен аппаратно пересчитывать сэмплы через внутренний блок ASRC.

1. Откройте **SigmaStudio** и перейдите на вкладку **Hardware Configuration**.
2. В дереве регистров найдите настройки входного I2S-порта, к которому подключен Orange Pi (например, **Serial Input Port 0**):
   * Установите режим порта в **Master Mode** (чтобы DSP генерировал клоки для Orange Pi).
   * Выберите формат данных: **I2S, 24 bit**.
3. Перейдите на вкладку **Routing Matrix** (Матрица маршрутизации):
   * Направьте физические каналы выбранного *Serial Input* на входы аппаратного блока **ASRC 0** (**ASRC Input Selector**).
4. Перейдите в окно построения схемы проекта (**Schematic**):
   * Добавьте блок входа **ASRC Input** (вместо стандартного *Input*).
   * Подключите выходы блока *ASRC Input* к вашей схеме цифровой обработки (кроссоверы, эквалайзеры).
5. Настройки выходного порта (связка **ADAU1467 ➔ PCM5242**):
   * Сконфигурируйте выходной порт (**Serial Output Port**) в режим **Master Mode**.
   * Укажите тактирование от ядра DSP (**Clock Domain**), жестко привязанного к вашему физическому мастер-клоку 12.288 МГц.
6. Скомпилируйте проект (**Link-Compile-Download**) и зашейте его в EEPROM платы для автозагрузки (Self-Boot).

# Дополнительное руководство: Сборка Bluetooth High-Res (LDAC / aptX HD) на Orange Pi PC+

В данном руководстве описан процесс полной очистки системы от встроенного Bluetooth и сборка из исходных кодов продвинутого аудио-демона `BlueALSA` с поддержкой аудиофильских кодеков **Sony LDAC** (24 бит / 96 кГц) и **Qualcomm aptX HD** (24 бит / 48 кГц).

---

## 1. Аппаратная подготовка и отключение встроенного радиомодуля

Встроенный чип *AP6212* на Orange Pi PC+ физически не способен удерживать поток LDAC (990 кбит/с). Необходимо использовать внешний USB-адаптер на чипе **Realtek RTL8761B** (Bluetooth 5.0). Чтобы они не конфликтовали, встроенный чип нужно отключить.

1. Откройте черный список модулей ядра:
   ```bash
   sudo nano /etc/modprobe.d/blacklist-internal-bt.conf
   ```
2. Добавьте строки для блокировки встроенных драйверов связи Allwinner:
   ```text
   blacklist btsdio
   blacklist hci_uart
   blacklist sunxi_bt
   ```
3. Подключите внешний USB-адаптер RTL8761B в любой свободный USB-порт Orange Pi PC+ и перезагрузите плату:
   ```bash
   sudo reboot
   ```
4. Убедитесь, что система видит новый адаптер как `hci0`:
   ```bash
   hciconfig -a
   ```

---

## 2. Сборка библиотек декодеров из исходных кодов

Официальные репозитории дистрибутивов Linux не содержат бинарные пакеты LDAC и aptX из-за лицензионных ограничений, поэтому мы компилируем их открытые исходные коды (AOSP) прямо на плате.

### Шаг 2.1. Установка базовых инструментов сборки
```bash
sudo apt update
sudo apt install -y git build-essential cmake debhelper pkg-config \
libasound2-dev libbluetooth-dev libdbus-1-dev libglib2.0-dev \
libtool autoconf automake libsbc-dev
```

### Шаг 2.2. Сборка кодека Sony LDAC
```bash
git clone https://github.com
cd ldacBT
git submodule update --init
mkdir build && cd build
cmake -DCMAKE_INSTALL_PREFIX=/usr -DINSTALL_LIBDIR=lib ..
make
sudo make install
cd ../..
```

### Шаг 2.3. Сборка кодека Qualcomm aptX / aptX HD
```bash
git clone https://github.com
cd libfreeaptx
make
sudo make install
cd ..
```
*После установки выполните команду обновления ссылок на разделяемые библиотеки:*
```bash
sudo ldconfig
```

---

## 3. Компиляция и установка BlueALSA с поддержкой High-Res

Теперь мы собираем сам аудио-демон, указывая компилятору флаги активации установленных библиотек.

1. Клонируйте официальный репозиторий проекта `bluez-alsa`:
   ```bash
   git clone https://github.com
   cd bluez-alsa
   ```
2. Сгенерируйте скрипты конфигурации:
   ```bash
   autoreconf --install
   mkdir build && cd build
   ```
3. Запустите конфигуратор с **полным набором High-Res флагов**:
   ```bash
   ../configure --enable-ldac --enable-aptx --enable-aptx-hd --with-libfreeaptx --enable-systemd --enable-cli
   ```
   *Убедитесь, что в финальном выводе скрипта напротив строк `LDAC`, `aptX` и `aptX HD` стоит значение `yes`.*
4. Скомпилируйте и установите демон в систему:
   ```bash
   make
   sudo make install
   ```

---

## 4. Автоматизация служб и сопряжение со смартфоном

### Шаг 4.1. Настройка автозапуска BlueALSA
1. Создайте системную службу для демона:
   ```bash
   sudo nano /etc/systemd/system/bluealsa.service
   ```
2. Вставьте конфигурацию, принудительно активирующую кодеки в режиме аудио-приемника (Sink):
   ```ini
   [Unit]
   Description=BlueALSA High-Res Audio Daemon
   After=bluetooth.target
   Requires=bluetooth.target

   [Service]
   Type=dbus
   BusName=org.bluealsa
   ExecStart=/usr/local/bin/bluealsad -p a2dp-sink -c ldac -c aptx-hd
   Restart=on-failure

   [Install]
   WantedBy=multi-user.target
   ```

### Шаг 4.2. Настройка автоматической трансляции в I2S Slave
Чтобы поток данных, принятый по Bluetooth, автоматически перенаправлялся на пины I2S вашей звуковой карты без ручного ввода команд, создаем вторую службу-транслятор.

1. Создайте файл службы:
   ```bash
   sudo nano /etc/systemd/system/bluealsa-aplay.service
   ```
2. Вставьте конфигурацию (где `hw:1,0` — это имя вашей карты I2S Slave):
   ```ini
   [Unit]
   Description=BlueALSA Player to I2S Slave
   After=bluealsa.service
   Requires=bluealsa.service

   [Service]
   ExecStart=/usr/local/bin/bluealsa-aplay --device=hw:1,0 00:00:00:00:00:00
   Restart=on-failure

   [Install]
   WantedBy=multi-user.target
   ```
3. Активируйте и запустите обе службы:
   ```bash
   sudo systemctl daemon-reload
   sudo systemctl enable bluealsa bluealsa-aplay
   sudo systemctl start bluealsa bluealsa-aplay
   ```

### Шаг 4.3. Первичное сопряжение телефона
1. Войдите в консоль управления Bluetooth:
   ```bash
   sudo bluetoothctl
   ```
2. Введите команды для перевода USB-донгла в режим сопряжения:
   ```text
   power on
   agent on
   default-agent
   discoverable on
   pairable on
   ```
3. Откройте настройки Bluetooth на смартфоне, найдите Orange Pi и нажмите «Подключиться».
4. В консоли платы появится запрос на сопряжение, введите `yes` и нажмите `Enter`.
5. Добавьте ваш телефон в доверенные устройства, чтобы плата подключалась к нему автоматически (замените `XX:XX...` на реальный MAC-адрес вашего телефона, который отобразится в консоли):
   ```text
   trust XX:XX:XX:XX:XX:XX
   exit
   ```

После этого в шторке смартфона (в разделе настроек Bluetooth-устройства) активируйте ползунок **LDAC** или **aptX HD**. Теперь при запуске любого плеера на телефоне аудиопоток высокого разрешения полетит напрямую в процессор, а оттуда — в Slave-режиме через I2S в ADAU1467 [см. пред. ответ].
