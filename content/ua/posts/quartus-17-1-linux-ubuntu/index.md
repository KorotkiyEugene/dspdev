---
title: "Встановлюємо Quartus 17.1 на Linux Ubuntu 24.04 LTS"
date: 2024-05-19
summary: "Встановлюємо Quartus 17.1 на Linux Ubuntu 24.04 LTS"
tags: ["Linux", "Ubuntu", "Quartus", "FPGA", "USB-Blaster"]
author: "Євген Короткий"
weight: 1
draft: false
---
Окей, можливо не кожному знадобиться інфа, як ставити Quartus 17.1 на Ubuntu 24.04 LTS, але така потреба іноді може виникнути. До того ж деякі описані моменти будуть актуальні і для новіших версій Quartus. Тому поїхали! :)

# Встановлюємо додаткові бібліотеки

Для роботи Quartus 17.1 знадобиться libpng12, а для роботи USB Blaster потрібен libudev1:i386. Маємо їх встановити.

Встановлюємо libpng12 (в новіших версіях Quartus цієї проблеми не має бути):

- Встановлюємо build-essential: `sudo apt install build-essential`

- Встановлюємо zlib1g-dev: `sudo apt-get install zlib1g-dev`

- Викачуємо вихідний код з [sourceforge](https://sourceforge.net/projects/libpng/files/libpng12/) (нам потрібен архів libpng-1.2.59.tar.gz)

- Розпаковуємо архів: `tar -xzf libpng-1.2.59.tar.gz`

- Компілимо і інсталимо лібу:

  ```
  cd ./libpng-1.2.59
  ./configure --prefix=/usr/local
  make
  sudo make install
  sudo ldconfig
  ```

Для роботи USB-Blaster (код девайсу 09fb:6001) потрібна 32-розрядна ліба libudev1:i386. Без цієї ліби при спробі сканування JTAG ланцюжка отримаєте "Unable to read device chain - JTAG chain broken". При цьому новіший програматор USB-Blaster 2 (код девайсу 09fb:6010) працює без проблем.

```
sudo dpkg --add-architecture i386
sudo apt update
sudo apt-get install libudev1:i386
sudo ln -sf /lib/x86_64-linux-gnu/libudev.so.1 /lib/x86_64-linux-gnu/libudev.so.0
```

# Встановлюємо Quartus Prime Standard

Викачуєте архів Quartus-17.1.0.590-linux-complete.tar за [посиланням](https://www.intel.com/content/www/us/en/software-kit/669392/intel-quartus-prime-standard-edition-design-software-version-17-1-for-linux.html) (обираєте Complete Download). Розмір 26.8 ГБ. Хеш: `981329178fb7efb2ef326e2d9ee876fdfccef674`

Розпаковуєте архів.

Якщо запустити інсталер версії 17.1 з GUI, він зависне. Тому ставите комоненти з командного рядка окремо.

Інсталите Quartus виконавши в терміналі:

```
$DOWNLOADDIR/components/QuartusLiteSetup-$VER-linux.run \
  --mode unattended \
  --unattendedmodeui none \
  --installdir $INSTALLDIR \
  --disable-components quartus_help,modelsim_ase,modelsim_ae \
  --accept_eula 1
```

Замість $DOWNLOADDIR, $VER та $INSTALLDIR підставляєте відповідні значення. В моєму випадку це було:

```
./QuartusSetup-17.1.0.590-linux.run \  
  --mode unattended \  
  --unattendedmodeui none \
  --installdir ~/intelFPGA/17.1 \
  --disable-components quartus_help,modelsim_ase,modelsim_ae \
  --accept_eula 1
```

Під час встановлення скрипт може зависнути. Контролюйте виконання скрипта за допомогою консольної утиліти `top`. Якщо побачите, що скрипт і архіватор не використовують процесорний час, а скрипт все ще не завершився, можете його завершити за допомогою CTRL+C. Це стосується і виконання двох наступних команд.

Далі встановлюєте ModelSim та QuartusHelp:

```
./ModelSimSetup-17.1.0.590-linux.run \
  --mode unattended \
  --unattendedmodeui none \
  --installdir ~/intelFPGA/17.1 \
  --modelsim_edition modelsim_ase \
  --launch_from_quartus 1 \
  --accept_eula 1
  
./QuartusHelpSetup-17.1.0.590-linux.run \
  --mode unattended \
  --unattendedmodeui none \
  --installdir ~/intelFPGA/17.1 \
  --accept_eula 1
```

Щоб Quartus запускався з командного рядка, можна додати шлях до бінарників Quartus в PATH. А можна виконати скрипт nios2_command_shell.sh в каталозі intelFPGA/17.1/nios2eds, який запустить сесію терміналу з уже налаштованими необхідними змінними оточення.

У вікні налаштування ліцензії, що відкриється після першого зісля запуску Quartus, вказуєте шлях до файлу з ліцензією.

Якщо ваша ліцензія Quartus привязана до іншої MAC адреси мережевого адаптера, змінити MAC адресу можна або в графічному інтерфейсі налаштувань мережевого адаптера netplan, або за допомогою `sudo macchanger --mac=xx:xx:xx:xx:xx:xx enp59s0`, де `xx:xx:xx:xx:xx:xx` - нова MAC адреса, а `enp59s0` - назва відповідного мережевого інтерфейсу. Зміну за допомогою macchanger доведеться робити після кожного перезавантаження системи.

# Налаштовуємо USB-Blaster

Тут все просто. Як вказано вище встановлюєте libudev1:i386, а потім налаштовуєте правила udev створивши файл /etc/udev/rules.d/51-usbblaster.rules з наступним вмістом (для цього знадобляться привілегії root):

```
# USB Blaster
SUBSYSTEM=="usb", ENV{DEVTYPE}=="usb_device", ATTR{idVendor}=="09fb", ATTR{idProduct}=="6001", MODE="0666", NAME="bus/usb/$env{BUSNUM}/$env{DEVNUM}", RUN+="/bin/chmod 0666 %c"
SUBSYSTEM=="usb", ENV{DEVTYPE}=="usb_device", ATTR{idVendor}=="09fb", ATTR{idProduct}=="6002", MODE="0666", NAME="bus/usb/$env{BUSNUM}/$env{DEVNUM}", RUN+="/bin/chmod 0666 %c"
SUBSYSTEM=="usb", ENV{DEVTYPE}=="usb_device", ATTR{idVendor}=="09fb", ATTR{idProduct}=="6003", MODE="0666", NAME="bus/usb/$env{BUSNUM}/$env{DEVNUM}", RUN+="/bin/chmod 0666 %c"

# USB Blaster II
SUBSYSTEM=="usb", ENV{DEVTYPE}=="usb_device", ATTR{idVendor}=="09fb", ATTR{idProduct}=="6010", MODE="0666", NAME="bus/usb/$env{BUSNUM}/$env{DEVNUM}", RUN+="/bin/chmod 0666 %c"
SUBSYSTEM=="usb", ENV{DEVTYPE}=="usb_device", ATTR{idVendor}=="09fb", ATTR{idProduct}=="6810", MODE="0666", NAME="bus/usb/$env{BUSNUM}/$env{DEVNUM}", RUN+="/bin/chmod 0666 %c"
```

Після створення і збереження файлу перепідключаєте USB-Blaster.

От і все, кінець :)