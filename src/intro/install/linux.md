# Linux

这是一些Linux发行版的安装命令。

## 安装包

- Ubuntu 18.04或更高版本
- Debian Stretch或更高版本

> **注意**`gdb-multiarch`是用于调试ARM Cortex-M程序的GDB命令
> 

 <!-- Debian stretch -->
<!-- GDB 7.12 -->
<!-- OpenOCD 0.9.0 -->
<!-- QEMU 2.8.1 -->

<!-- Ubuntu 18.04 -->
<!-- GDB 8.1 -->
<!-- OpenOCD 0.10.0 -->
<!-- QEMU 2.11.1 -->

```sh
sudo apt install gdb-multiarch openocd qemu-system-arm
```

- Ubuntu 14.04和16.04

<!--Ubuntu 14.04-->
<!--GDB 7.6(！)-->
<!--OpenOCD 0.7.0(？)-->
<!--QEMU 2.0.0(？)-->

```sh
sudo apt install gdb-arm-none-eabi openocd qemu-system-arm
```

- Fedora 27或更高版本



<!-- Fedora 27 -->
<!-- GDB 7.6 (!) -->
<!-- OpenOCD 0.10.0 -->
<!-- QEMU 2.10.2 -->

```sh
sudo dnf install arm-none-eabi-gdb openocd qemu-system-arm
```

- Arch Linux


``` console
sudo pacman -S arm-none-eabi-gdb qemu-arch-extra openocd
```

## udev规则

该规则使您可以在不需要root特权的情况下将OpenOCD与Discovery开发板一起使用。

创建文件`/etc/udev/rules.d/70-st-link.rules`，内容如下所示。

``` text
# STM32F3DISCOVERY rev A/B - ST-LINK/V2
ATTRS{idVendor}=="0483", ATTRS{idProduct}=="3748", TAG+="uaccess"

# STM32F3DISCOVERY rev C+ - ST-LINK/V2-1
ATTRS{idVendor}=="0483", ATTRS{idProduct}=="374b", TAG+="uaccess"
```


然后使用以下命令重新加载所有udev规则：

``` console
sudo udevadm control --reload-rules
```

如果您将主板插入笔记本电脑，请先拔下电源，然后再重新插入。

您可以通过运行以下命令来检查权限：

```sh
lsusb
```

应该显示类似结果:

```text
(..)
Bus 001 Device 018: ID 0483:374b STMicroelectronics ST-LINK/V2.1
(..)
```

记下总线和设备号,然后像这样使用路径`/dev/bus/usb/<bus>/<device>`：

``` console
ls -l /dev/bus/usb/001/018
```

```text
crw-------+ 1 root root 189, 17 Sep 13 12:34 /dev/bus/usb/001/018
```

```console
getfacl /dev/bus/usb/001/018 | grep user
```

```text
user::rw-
user:you:rw-
```

权限后面的“ +”表示存在扩展权限。 “getfacl”命令显示当且用户可以使用此设备。

现在，转到[下一部分]。

[下一部分]:verify.md