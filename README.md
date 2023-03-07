openwrt官方源代码RTC时钟不工作问题会导致luci-app-statistics工作不正常，修改target/linux/mvebu/

config-5.10
config-5.15

22.03分支添加或者修改以下行

```
# CONFIG_RTC_DRV_ARMADA38X is not set
  CONFIG_RTC_INTF_DEV_UIE_EMUL=y
  CONFIG_RTC_DRV_DS1307=y

```

这个方法在22.03分支中适用

Master修改添加修改以下行， /有疑问要不要加的有空在测试更新本说明，希望大家可以测试回复告诉我

```

  CONFIG_RTC_DRV_DS1307=y
  CONFIG_CLKDEV_LOOKUP=y   /有疑问要不要加，22.03 默认选中
  CONFIG_RTC_INTF_DEV_UIE_EMUL=y
  CONFIG_ARM_ARCH_TIMER=y  /有疑问要不要加
  CONFIG_ARM_ARCH_TIMER_EVTSTREAM=y  /有疑问要不要加
# CONFIG_RTC_DRV_ARMADA38X is not set

```

make menuconfig选中moduleKernel modules > Other modules --> kmod-rtc-ds1307

官方更新了901&902补丁和dts文件来控制风扇转速，风扇转速档位分别为80 102 170 230 255 链接如下

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

可以拷贝902和904的补丁到 
patches-5.10/
patches-5.15/
目录下编译后，过测试不会弹出iei-wt61p803-puzzle serial0-0: Command reply receive timeout

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

系统时间设置，编译时候选择busybox的hwclock 选定Base system  --->Customize busybox options---->Linux System Utilities  ---> hwclock程序编译   刷入后连接网络后同步浏览器时间后

执行hwclock -w

RTC硬件时钟修复后luci-app-statistics也自动恢复正常，，无数据跟RTC时间有关，项目中风扇控制脚本作废

luci-app-statistics无数据发生的原因链接如下，可参考

https://github.com/openwrt/packages/pull/10383

https://forum.openwrt.org/t/collectd-rrdtool-data-corruption-warning-due-to-system-time-change/47223/11

 

iEi-Puzzle-M901升级OpenWrt22.03后内核日志提示iei-wt61p803-puzzle serial0-0: Command reply receive timeout

导致LED和温度传感器工作不正常

本目录中的补丁是客服给的，替换target/linux/mvebu/目录下补丁文件夹中的902和904即可编译，本项目中的补丁因为不符合openwrt官方标准，等待官方提交新的pr

链接如下




fancontrol.sh s 风扇控制脚本 上传到/etc/目录下

chmod +x fancontrol.sh

```

* Test it to make sure that it runs correctly.
```

/etc/fancontrol.sh verbose

```

* Let it run in the background to keep your router cool.
```

/etc/fancontrol.sh &

```

####Disable the orginal fan controller.
*Remove or comment out this line from /etc/crontabs/root (In LuCI, it's System > Scheduled Tasks)
```

 */5* ** * /sbin/fan_ctrl.sh

```

####Optional
* Have this run on boot.
* Add this to /etc/rc.local (In LuCI, it's System > Startup)
```

/etc/fancontrol.sh &

```


luci-app-statistics报错无法正常工作，，经过本人修google解决方案如下


I believe I found the workaround to get rid of those error messages. With previous attempt existance of file created after NTP sync was checked prior to execution of NTP script so in the result neither collectd nor ntp was started.
For it to work START variable for collectd init script should be higher (>98) than the one for NTP script.
/etc/init.d/collectd changed in line 4 and addition of line 342:
#!/bin/sh /etc/rc.common
# Copyright (C) 2006-2016 OpenWrt.org

START=99 /修改为99
STOP=10
/cut/
start_service() {
 [ ! -z "$(uci get dhcp.@dnsmasq[0])" ] && [ ! -z "$(uci get system.ntp.server)" ] && while [ ! -f "/var/state/dnsmasqsec" ]; do sleep 1; done /添加本行
 process_config

 procd_open_instance
 procd_set_param command /usr/sbin/collectd
 procd_append_param command -C "$COLLECTD_CONF"
 procd_append_param command -f # don't daemonize
 procd_set_param nice "$NICEPRIO"
 procd_set_param stderr 1
 procd_set_param respawn
 procd_close_instance
}
/cut/
