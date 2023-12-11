openwrt官方源代码RTC时钟不工作问题会导致luci-app-statistics工作不正常，修改target/linux/mvebu/

config-5.10
config-5.15

22.03分支添加或者修改以下行

```
# CONFIG_RTC_DRV_ARMADA38X is not set
  CONFIG_RTC_INTF_DEV_UIE_EMUL=y
  CONFIG_RTC_DRV_DS1307=y

```

make menuconfig选中moduleKernel modules > Other modules --> kmod-rtc-ds1307

官方更新了903补丁和dts文件来控制风扇转速，风扇转速档位分别为80 102 170 230 255 链接如下

https://github.com/openwrt/openwrt/commit/ea33a5def506b0ba647f779e60e6ccca03c29a17
https://github.com/openwrt/openwrt/commit/aa41f4a395bda69cb6ef0ef423e2a4077284fbcd

拷贝master分支以下文件到22.03目录下编译

patches-5.10/903-drivers-hwmon-Add-the-IEI-WT61P803-PUZZLE-HWMON-driv.patch

target/linux/mvebu/files/arch/arm64/boot/dts/marvell/cn9131-puzzle-m901.dts

target/linux/mvebu/files/arch/arm64/boot/dts/marvell/cn9132-puzzle-m902.dts

target/linux/mvebu/files/arch/arm64/boot/dts/marvell/puzzle-thermal.dtsi

22.03和master风扇控制没有问题，但是iei-wt61p803-puzzle serial0-0: Command reply receive timeout，仍然弹出，LED报错，

```
[167591.374374] iei-wt61p803-puzzle serial0-0: Command reply receive timeout
[167591.381197] leds green:cloud: Setting an LED's brightness failed (-110)
```

可以拷贝911的补丁到 
target/linux/mvebu/patches-5.10/
target/linux/mvebu/patches-5.15/
目录下编译后，测试不会弹出iei-wt61p803-puzzle serial0-0: Command reply receive timeout

可以使用cat /sys/class/hwmon/hwmon6/pwm1来查看档位

```
root@Puzzle-M901:~# cat /sys/class/hwmon/hwmon6/pwm1
80
```
sensors命令来查看温度以及风扇转速

```
root@Puzzle-M901:~# sensors
iei_wt61p803_puzzle-isa-f2702000
Adapter: ISA adapter
fan1:        1560 RPM
fan2:           0 RPM
fan3:           0 RPM
fan4:           0 RPM
fan5:           0 RPM
temp1:        +40.0°C
temp2:        +36.0°C
```
设备树控制风扇转速问题也已经修复
6.1.64
5.15.140内内核正式修复

mvebu: fix PXA I2C bus with recovery
Since kernel 5.11, the PXA I2C driver has been converted to generic I2C
recovery, which makes the I2C bus completely lock up if recovery pinctrl
is present in the DT and I2C recovery is enabled.

This effectively completely broke I2C on Methode uDPU and eDPU boards
as both of them rely on I2C recovery.

After a discussion upstream, it was concluded that there is no simple fix
and that the blamed upstream commit:
0b01392c18b9993a584f36ace1d61118772ad0ca ("i2c: pxa: move to generic GPIO
recovery") should be reverted.
I have sent the revert upstream, it should be merged soon so lets "fix"
OpenWrt as well.


https://github.com/openwrt/openwrt/commit/f74f5b29948aa9303bf94045cf938cee49897944

系统时间设置，编译时候选择hwclock程序编译   刷入后连接网络后同步浏览器时间后

执行hwclock -w -v即可，查找rtc是否正常，执行hwclcok -w -v 后未出现以下错误，说明rtc工作正常


```
Waiting for clock tick...
ioctl(3, RTC_UIE_ON, 0): Invalid argument

```

设置usb SSD MMC启动顺序如下 902修改cn9131-puzzle-m901为cn9132-puzzle-m902

```
setenv ieiusb 'ext4load usb 0:1 0x6500000 Image; ext4load sub 0:1 0x6000000 cn9131-puzzle-m901.dtb;setenv bootargs root=/dev/sda2 rw rootwait rootfstype=ext4 $console cpuidle.off=1; booti 0x6500000 - 0x6000000'

setenv ieinvme 'ext4load nvme 0:1 0x6500000 Image; ext4load nvme 0:1 0x6000000 cn9131-puzzle-m901.dtb;setenv bootargs root=/dev/nvme0n1p2 rw rootwait rootfstype=ext4 $console cpuidle.off=1; booti 0x6500000 - 0x6000000'

setenv ieimmc 'ext4load mmc 0:1 0x6500000 Image; ext4load mmc 0:1 0x6000000 cn9131-puzzle-m901.dtb;setenv bootargs root=/dev/mmcblk0p3 rw rootwait rootfstype=ext4 $console cpuidle.off=1; booti 0x6500000 - 0x6000000'

setenv bootcmd 'usb reset;nvme scan;if ext4load usb 0:1 0x6000000 cn9131-puzzle-m901.dtb;then run ieiusb;fi;if ext4load nvme 0:1 0x6000000 cn9131-puzzle-m901.dtb;then run ieinvme;else run ieimmc;fi'

saveenv

```

测试usb 启动代码  如下  rufus-3.22写入 openwrt-22.03-mvebu-cortexa72-iei_puzzle-m901-ext4-sdcard.img.gz文件 usb或者ssd
ssd参照usb以下代码测试启动

```
usb reset
ext4load usb 0:1 0x6000000 cn9131-puzzle-m901.dtb
ext4load usb 0:1 0x6500000 Image 
setenv bootargs root=/dev/sda2 rw rootwait rootfstype=ext4 $console cpuidle.off=1; booti 0x6500000 - 0x6000000

```

RTC硬件时钟修复后luci-app-statistics也自动恢复正常，，无数据跟RTC时间有关，项目中风扇控制脚本作废

luci-app-statistics无数据发生的原因链接如下，可参考

https://github.com/openwrt/packages/pull/10383

https://forum.openwrt.org/t/collectd-rrdtool-data-corruption-warning-due-to-system-time-change/47223/11

 

iEi-Puzzle-M901升级OpenWrt22.03后内核日志提示iei-wt61p803-puzzle serial0-0: Command reply receive timeout

导致内核日志LED报错

本目录中的补丁是客服给的，拷贝911补丁至target/linux/mvebu/目录下编译即可，iei开发组已经向openwrt官方提交补丁，，官方已经审核批准合并
链接如下
https://github.com/openwrt/openwrt/pull/12370
https://git.openwrt.org/?p=openwrt/openwrt.git


经过测试，升级固件尽量使用ext4格式固件，squashfs格式有时候会造成系统启动时候卡住无法启动，iEi-Puzzle-M901更新uboot方法

```
cp flash-image_m901-mmc.bin 到ext4 的 USB 里

Marvell>> usb start

Marvell>> ext4ls usb 0:1 确认文件有在USB 中

<DIR>       4096 .
<DIR>       4096 ..
<DIR>      16384 lost+found

1805808 flash-image_m901-mmc.bin

以下为刷写uboot 到spi flash 里

Marvell>> bubt flash-image_m901-mmc.bin spi usb

重开机后会看到日期是新的

U-Boot 2019.10-10.3.4.0-4 (Mar 27 2023 - 20:00:46 +0800)
```





