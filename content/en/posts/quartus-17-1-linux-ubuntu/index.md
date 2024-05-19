---
title: "Installing Quartus 17.1 on Linux Ubuntu 24.04 LTS"
date: 2024-05-19
summary: "Installing Quartus 17.1 on Linux Ubuntu 24.04 LTS"
tags: ["Linux", "Ubuntu", "Quartus", "FPGA", "USB-Blaster"]
author: "Ievgen Korotkyi"
weight: 1
---
Maybe not everyone needs to know how to install Quartus 17.1 on Ubuntu 24.04 LTS, but sometimes such info can be useful. Besides, some of the issues described will be relevant to newer versions of Quartus. So let's go! :)

# Installing Additional Libraries

You need libpng12 to work with Quartus 17.1, and libudev1:i386 for the USB Blaster. So we have to install them.

Installing libpng12 (this issue should not occur in newer versions of Quartus):

- Install build-essential: `sudo apt install build-essential`

- Install zlib1g-dev: `sudo apt-get install zlib1g-dev`

- Download the source code from [SourceForge](https://sourceforge.net/projects/libpng/files/libpng12/) (we need the libpng-1.2.59.tar.gz archive)

- Unpack the archive: `tar -xzf libpng-1.2.59.tar.gz`

- Compile and install the library:

  ```bash
  cd ./libpng-1.2.59
  ./configure --prefix=/usr/local
  make
  sudo make install
  sudo ldconfig
  ```

To work with USB-Blaster (device code 09fb:6001), you will need the 32-bit library libudev1:i386. Without this library, when scanning the JTAG chain, you will get "Unable to read device chain - JTAG chain broken". At the same time, the newer USB-Blaster 2 (device code 09fb:6010) works without problems.

```bash
sudo dpkg --add-architecture i386
sudo apt update
sudo apt-get install libudev1:i386
sudo ln -sf /lib/x86_64-linux-gnu/libudev.so.1 /lib/x86_64-linux-gnu/libudev.so.0
```

# Installing Quartus Prime Standard

Download the Quartus-17.1.0.590-linux-complete.tar archive from [this link](https://www.intel.com/content/www/us/en/software-kit/669392/intel-quartus-prime-standard-edition-design-software-version-17-1-for-linux.html) (choose Complete Download). The size is 26.8 GB. Hash: `981329178fb7efb2ef326e2d9ee876fdfccef674`

Unpack the archive.

If you run the installer for version 17.1 with the GUI, it will hang. Therefore, install the components separately from the command line.

Install Quartus by running in the terminal:

```bash
$DOWNLOADDIR/components/QuartusLiteSetup-$VER-linux.run \
  --mode unattended \
  --unattendedmodeui none \
  --installdir $INSTALLDIR \
  --disable-components quartus_help,modelsim_ase,modelsim_ae \
  --accept_eula 1
```

Replace $DOWNLOADDIR, $VER, and $INSTALLDIR with the appropriate values. In my case, it was:

```bash
./QuartusSetup-17.1.0.590-linux.run \  
  --mode unattended \  
  --unattendedmodeui none \
  --installdir ~/intelFPGA/17.1 \
  --disable-components quartus_help,modelsim_ase,modelsim_ae \
  --accept_eula 1
```

During installation, the script may hang. Monitor the script execution using the console util `top`. If you see that the script and the archiver are not using CPU time, but the script has not finished yet, you can terminate it with CTRL+C. This applies to the execution of the following two commands as well.

Next, install ModelSim and QuartusHelp:

```bash
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

You can add Quartus binaries to the PATH to launch Quartus from the command line. Or you can run the script nios2_command_shell.sh in the intelFPGA/17.1/nios2eds directory, which will start a terminal session with all necessary environment variables already set.

Specify the path to the license file in the license setup window that opens after Quartus first time launch.

If your Quartus license is tied to another network adapter's MAC address, you can change the MAC address either in the graphical interface of the netplan network adapter settings or using `sudo macchanger --mac=xx:xx:xx:xx:xx:xx enp59s0`, where `xx:xx:xx:xx:xx:xx` is the new MAC address, and `enp59s0` is the name of the corresponding network interface. You will have to change it using macchanger after each system reboot.

# Configuring USB-Blaster

It's simple here. As mentioned above, install libudev1:i386, and then configure the udev rules by creating the file /etc/udev/rules.d/51-usbblaster.rules with the following content (you will need root privileges for this):

```bash
# USB Blaster
SUBSYSTEM=="usb", ENV{DEVTYPE}=="usb_device", ATTR{idVendor}=="09fb", ATTR{idProduct}=="6001", MODE="0666", NAME="bus/usb/$env{BUSNUM}/$env{DEVNUM}", RUN+="/bin/chmod 0666 %c"
SUBSYSTEM=="usb", ENV{DEVTYPE}=="usb_device", ATTR{idVendor}=="09fb", ATTR{idProduct}=="6002", MODE="0666", NAME="bus/usb/$env{BUSNUM}/$env{DEVNUM}", RUN+="/bin/chmod 0666 %c"
SUBSYSTEM=="usb", ENV{DEVTYPE}=="usb_device", ATTR{idVendor}=="09fb", ATTR{idProduct}=="6003", MODE="0666", NAME="bus/usb/$env{BUSNUM}/$env{DEVNUM}", RUN+="/bin/chmod 0666 %c"

# USB Blaster II
SUBSYSTEM=="usb", ENV{DEVTYPE}=="usb_device", ATTR{idVendor}=="09fb", ATTR{idProduct}=="6010", MODE="0666", NAME="bus/usb/$env{BUSNUM}/$env{DEVNUM}", RUN+="/bin/chmod 0666 %c"
SUBSYSTEM=="usb", ENV{DEVTYPE}=="usb_device", ATTR{idVendor}=="09fb", ATTR{idProduct}=="6810", MODE="0666", NAME="bus/usb/$env{BUSNUM}/$env{DEVNUM}", RUN+="/bin/chmod 0666 %c"
```

After creating and saving the file, reconnect the USB-Blaster.

That's it, the end :)
