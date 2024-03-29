# WIP: Artillery SideWinder X4 Pro custom firmware

This document shows how the Artillery Sidewinder X4 Pro can be equipped with the latest Klipper version. In addition, problems with the wifi or the Klipper connection can be resolved.

DISCLAIMER: These instructions should only be followed if you have technical knowledge. I am not liable for any damage caused to a device as a result. There is currently no Artillery firmware available for download and it is not known how to reflash it. Every step must therefore be taken with caution!

# Contents

 - Requirements
 - Flashing
 - Configuration
 - Kliauh, Klipper, Moonraker and Mainsail installation
 - Klipper configuration

## Requirements

- To flash a new image to the printer, the printer must be opened. This will break the warranty seal.
You need an `MKS EMMC adapter` to flash the EMMC storage on the board.
The adapter can be ordered [here](https://de.aliexpress.com/item/1005005614719377.html), for example.
- In addition, software is required to flash the image. I use Etcher (https://etcher.balena.io/) in these instructions

## Flashing
The Artillery Sidewinder X4 Pro board is similar to an MKS Pi. There are images from Makerbase for the MKS Pi, but these are outdated. That's why I use a pure Armbian image (Linux distribution) and install Klipper, Moonraker and Mainsail myself.
Go to https://github.com/redrathnure/armbian-mkspi and download the latest image under Releases on the right-hand side. In my case `Armbian-unofficial_24.2.0-trunk_Mkspi_bookworm_current_6.6.17.img.xz`

Once the image has been downloaded, it can be written to the EMMC. To do this, connect the EMMC to your computer via USB using the EMMC adapter.
1. select the file that has just been downloaded
2. select the EMMC
3. press "Flash" and wait until the process is completed -> Sometimes "Flash failed" may be displayed. However, as long as the Flash process has run through and the error only occurs during validation, you can continue
![etcher example](https://i.imgur.com/IDD2Ax7.png)

When the flash is completed, open the file explorer and select the EMMC (If it is not displayed, unplug the adapter and plug it in again).
Open the file `armbian_first_run.txt.template` and change the following settings:
```
FR_net_change_defaults=1
FR_net_wifi_enabled=1
FR_net_wifi_ssid='MySSID'
FR_net_wifi_key='MyWiFiKEY'
FR_net_wifi_countrycode='MyCountyCode'
```
Now save the changes and rename the file to `armbian_first_run.txt`
Go to the folder `dtb\rockchip` and replace the file `rk3328-roc-cc.dtb`  with the one from this repo (**Note**: Be sure to use the **.dtb** file and not the .dts file unless you know what you are doing). 

Now your printer is ready to start for the first time. Install the EMMC and start the printer. Then wait a little and check via your router which IP address the printer has.

## Armbian Configuration

Open PuTTY or another SSH program and connect to the IP. Log in with `root` and `1234`. Now go through the setup wizard.
Now update all packages with `apt update && apt upgrade`.
Some packages should be displayed as kept back here. Whenever a kernel update is made, our modified .dtb file from this repo must be inserted again, otherwise the wifi will no longer work. I have saved this in my root directory.
```
cd && wget https://github.com/diglos/userpatches/blob/master/overlay/armbian_first_run.txt
```
The kept packages can now be installed.
```
apt-get --with-new-pkgs upgrade <list of packages kept back>
```
> Note: If this command does not work for you, you can also make the updates via `armbian-config`

After the packages have been installed, we need to copy our **.dtb** file back into the correct directory.
```
cd && cp rk3328-roc-cc.dtb /boot/dtb/rockchip/rk3328-roc-cc.dtb
```
> Note: You can also use the `.dts` file and build the `.dtb` file by yourself
> ``
> dtc -I dts -O dtb -o /boot/dtb/rockchip/rk3328-roc-cc.dtb rk3328-roc-cc.dts
> ``

You can run `reboot` now. If the printer then no longer connects to the wifi, something probably went wrong when copying the `.dtb` file. To fix this, the printer must be connected to the computer via USB-C and a serial connection must be established. You can then try to copy the file again or build it yourself.

## Kiauh, Klipper, Moonraker and Mainsail installation

> Note: root rights are no longer required for the following steps. Make sure that you are working with the user that you created when setting up Armbian.

First, install Kiauh.
```
cd ~ && git clone https://github.com/dw-0/kiauh.git
```
Run Kiauh
```
./kiauh/kiauh.sh
```
Now select `Install` (1) and then enter the numbers 1 (`Klipper`), 2 (`Moonraker`) and 3 (`Mainsail`) one after the other. 
> Note: the numbers can change with updates, so check that you are installing the correct one.

Once Klipper, Moonraker, and Mainsail have been installed, we can proceed to update the firmware on the printer board. To do this, you first need to put the STM32F401 on the board into DFU mode. This is achieved by briefly pressing the `H-RST button` on the mainboard. Afterward, the the SSH connection will be closed and the printer restarts.

![Mainboard H-RST Button](https://i.imgur.com/ysIb2z8.png)

Now you can check whether DFU mode is active. To do this, enter `lsusb`. One of the entries should be this:
```
Bus 001 Device 002: ID 0483:df11 STMicroelectronics STM Device in DFU Mode
```
> Note: If the entry is listed, you have to try again. Without DFU mode we cannot flash the chip

Next we install our bootloader.
```
cd ~
git clone https://github.com/Arksine/katapult
```
Install pyserial
```
sudo apt install python3-serial
```
Create the firmware
```
cd katapult
make menuconfig
```
![Bootloader config](https://i.imgur.com/mAyfBTM.png)

Exit using `Q`  and confirm with `Y`
Now build and flash the firmware
```
make clean
make
sudo dfu-util -a 0 -D ~/katapult/out/katapult.bin --dfuse-address 0x08000000:force:mass-erase:leave -d 0483:df11
```
Then we can build the Klipper firmware.
```
cd ~
./kiauh/kiauh.sh
```
Select `4 [Advanced]` and `4 [Build + Flash]`

![Klipper firmware config](https://i.imgur.com/qXt8uVz.png)

Exit using `Q`  and confirm with `Y`
Now always choose number `1`. Make sure that "Flashing successfull" is displayed in green at the end.


## Printer Configuration
This part is still a work in progress. Use the current printer.cfg from Artillery here, but comment out the following:
```
#[mcu rpi]
#serial: /tmp/klipper_host_mcu

#[adxl345]
#cs_pin: rpi:None
#spi_bus: spidev0.0
#axes_map: x, z, -y

#[resonance_tester]
#accel_chip: adxl345
#probe_points:
#    100, 100, 20  # an example
```
