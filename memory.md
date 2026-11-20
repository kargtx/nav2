# BMW EntryNav2 (ENAVEVO / Alpine Linux / DRA72x): Wired CarPlay & Architecture Memory

> Документ консолидирует все технические исследования, анализ аппаратной и программной архитектуры, обнаруженные баги и инженерные решения для реализации **проводного Apple CarPlay (Wired USB CarPlay)** на головных устройствах **BMW EntryNav2** без заводского Wi-Fi (WLAN).

---

## 1. Контекст и аппаратная платформа

* **ГУ**: BMW / MINI EntryNav2 (также известные как EntryEvo / ENAVEVO, карты серии WAY, интерфейс ID6-lite).
* **Производитель железа**: Alpine Electronics.
* **SoC**: Texas Instruments **DRA72x** (архитектура Jacinto 6 Eco / OMAP5), 2 ядра ARM Cortex-A15.
* **ОС**: Alpine Linux (ядро `3.12.25-ti`).
* **Аудио DSP**: NXP **Dirana3** DSP (маршрутизация звука через ALSA `pcm.entertainment`, 44.1 kHz, S32_LE).
* **Bluetooth / Combo-модуль**: Подключён по скоростному аппаратному UART `/dev/ttyS0` (`4806a000.serial`, драйвер `omap8250`) на скорости **3,686,400 bps**. Аппаратная линия сброса чипа заведена на **GPIO 219**.
* **USB-контроллер**: Synopsys DesignWare USB3 core (`48890000.usb` / `dwc3` / `usb3`).
* **Стек проекции**: Cinemo CarPlay middleware (`/usr/local/bin/CARPLAY`) + RHMI плагины.

---

## 2. Первоисточник исследования

* **Форум CarTechnoloGY, тема #124686**:
  *Название*: *«BMW EntryNav2 (Alpine Linux / DRA72x): Wired USB CarPlay & AA causes /dev/ttyS0 Bluetooth module to wedge»*
* **Суть проблемы**: Попытка активировать проводной CarPlay через переключение USB в режим гаджета (`g_ffs_ncm`) на головах без заводского Wi-Fi приводит к зависанию Bluetooth через 1–2 минуты работы, утилизации одного ядра CPU в 100% и падению звука в динамики телефона или BT A2DP.

---

## 3. Детальный разбор трёх взаимосвязанных багов

### Баг №1: Зависание демона `AuxInUCM` (100% CPU lockup)
* **Процесс**: `/usr/local/bin/AuxInUCM` (`USBConnectionManager.service`), рабочий поток `UCMConST3`.
* **Причина**: В стоковом режиме `AuxInUCM` отслеживает статус подключения через sysfs: `/sys/bus/usb/devices/usb3/connection_status`. При переключении USB-порта 1 (`48890000.usb`) из режима `host` в режим `device` (гаджет `g_ffs_ncm` с FunctionFS `/dev/ffs-carlife`) эта нода удаляется ядром.
* **Следствие**: Поток `UCMConST3` не закрывает дескриптор файла. В цикле опроса ошибка `-ENODEV` (возврат `<= 0`) не считается фатальной, из-за чего поток входит в бесконечный tight-loop без пауз, потребляя **100% одного ядра CPU** и засоряя `dmesg` предупреждениями `sysfs_remove_file WARN_ON`.

### Баг №2: Клин Bluetooth UART (`/dev/ttyS0` / IRQ 104)
* **Аппаратные ресурсы**:
  * UART Bluetooth: **IRQ 104** (`4806a000.serial`), скорость 3.686 Мбод.
  * Читающий поток: `evo_hli_socket` (поток `EO_uart_rx`), приоритет `SCHED_FIFO` (-19).
  * USB DWC3: IRQ 108, 109.
  * Сетевой интерфейс CarPlay: `usb0` (NCM-протокол).
* **Причина**: По умолчанию прерывание IRQ 104, прерывания USB DWC3 и сетевые софтирки (`NET_RX_SOFTIRQ` интерфейса `usb0`, прокачивающего видеопоток CarPlay) обрабатываются на **CPU 0**.
* **Следствие**: В условиях 100% загрузки CPU 0 демоном `AuxInUCM` и интенсивного сетевого трафика буферы UART переполняются. Происходит потеря пакетов (например, `HCI_Disconnection_Complete`), ломается H4/HLI фрейминг. Поток `EO_uart_rx` уходит в ~88% CPU в попытках вернуть синхронизацию, демон iDrive HMI `btmbt` зависает со статусом «Connected», а сам BT-контроллер перестает отвечать.
* **Особенность**: В стоке вернуть BT к жизни помогает только глубокий сон авто (*Bus Ruhe*, ~15 мин), при выходе из которого срабатывает аппаратный сброс по GPIO 219.

### Баг №3: Отвал звука и сбой `OpenWifiIAP` в `CARPLAY`
* **Бинарник**: `/usr/local/bin/CARPLAY`.
* **Функция**: `CConnection::ExecuteConnPostProcess` (смещение `0x1B2658`).
* **Причина**: Функция рассчитана на беспроводной CarPlay и вызывает `OpenWifiIAP()`. На железе без Wi-Fi чипа вызов завершается ошибкой, роняя всю сессию с кодом `-6722`.
* **Опасность костыля `POST=0`**: Если принудительно отключить или пропустить `ExecuteConnPostProcess`, ломается вызов стейт-машины:
  $$\text{ExecuteConnPostProcess} \longrightarrow \text{SetConnectionState(7)} \longrightarrow \text{AuxIn\_DeviceInfo2\_Set\_Send} \longrightarrow \text{CCarPlayControl::Connected}$$
* **Следствие**: iDrive не инициализирует цифровой ALSA-вывод на DSP Dirana3 (`pcm.entertainment`). Телефон не видит аудиоканал ГУ и направляет звук либо в свой встроенный динамик, либо пытается транслировать через Bluetooth A2DP, что моментально перегружает UART и ускоряет наступление Бага №2.

---

## 4. Архитектурный план решения (Как бахнуть проводной CarPlay)

### Шаг 1: Точечный бинарный патч `/usr/local/bin/CARPLAY`
* **Цель**: Не отключать `ExecuteConnPostProcess`, а нейтрализовать только Wi-Fi зависимость.
* **Реализация**: Внутри `CConnection::ExecuteConnPostProcess` (`0x1B2658`):
  * Вызов `OpenWifiIAP()` заменяется на возврат успеха (`MOVS R0, #0` / NOP на ветвление с ошибкой `-6722`).
  * Функция продолжает выполнение и штатно вызывает `SetConnectionState(7)`.
  * Звук CarPlay направляется напрямую в ALSA-устройство `pcm.entertainment` (Dirana3 DSP). Никакого fallback на Bluetooth A2DP.

### Шаг 2: Устранение петли в `AuxInUCM`
* **Вариант А (сервисный хук при старте CarPlay)**:
  ```bash
  systemctl stop USBConnectionManager
  # либо: killall -STOP AuxInUCM / killall AuxInUCM
  ```
  При отключении телефона от USB сервис возвращается к жизни.
* **Вариант Б (бинарный патч)**:
  В потоке `UCMConST3` при возврате ошибки `<= 0` (`-ENODEV`) выполнять `close(fd)` и спать, либо корректно завершать поток.

### Шаг 3: Изоляция ядер CPU и прерываний (Affinity & RPS)
DRA72x имеет 2 ядра: `CPU 0` (маска `1`) и `CPU 1` (маска `2`).
1. **Bluetooth UART (IRQ 104)** изолируется на `CPU 0`:
   ```bash
   echo 1 > /proc/irq/104/smp_affinity
   ```
2. **Прерывания USB DWC3 (IRQ 108, 109)** уводятся на `CPU 1`:
   ```bash
   echo 2 > /proc/irq/108/smp_affinity
   echo 2 > /proc/irq/109/smp_affinity
   ```
3. **Разгрузка softirq для сетевого интерфейса `usb0` (RPS)** на `CPU 1`:
   ```bash
   echo 2 > /sys/class/net/usb0/queues/rx-0/rps_cpus
   ```
4. **Привязка тяжелых процессов к CPU 1**:
   ```bash
   taskset -cp 1 $(pgrep CARPLAY)
   ```

### Шаг 4: Аварийный ресет BT через GPIO 219 (Fast Watchdog)
Скрипт для моментального восстановления Bluetooth без ожидания Bus Ruhe:
```bash
if [ ! -d /sys/class/gpio/gpio219 ]; then
    echo 219 > /sys/class/gpio/export
fi
echo out > /sys/class/gpio/gpio219/direction
echo 0 > /sys/class/gpio/gpio219/value
usleep 100000
echo 1 > /sys/class/gpio/gpio219/value
killall -9 evo_hli_socket btmbt
```

---

## 5. Заметки по открытию SSH и окружению

1. **Текущий сетевой статус на ПК**:
   * Интерфейс `Ethernet` поднят с APIPA IP `169.254.35.66`.
   * В ARP-таблице обнаружены адреса ГУ: `169.254.14.102` (DoIP порт 13400) и `169.254.1.0`.
2. **Нюанс с `bavarian-ssh-client.exe`**:
   * Данный клиент ориентирован на **NBT Evo (ID6)** (система на QNX/Intel, использует облачный релей `ssh.bavarian-aa.uk`).
   * На **EntryNav2 (Linux / DRA72x)** SSH открывается иначе — через специализированные скрипты/патчи активации (например, механизмы FeatureInstaller или специализированный эксплойт через DoIP UDS, аналогично проекту `Jiraiya43/entry-aad`).
3. **План дальнейших действий**:
   * Разблокировать доступ по SSH на EntryNav2.
   * Снять дампы `/usr/local/bin/CARPLAY` и `/usr/local/bin/AuxInUCM`.
   * Применить описанные выше патчи и протестировать стабильный проводной CarPlay со звуком через DSP.
