# Install macOS Ventura On AMD 3700X With 5700XT BY Proxmox

本人mbp15寸用了6年，因为囊中羞涩所以决定使用黑苹果，黑苹果很折腾，需要查阅大量资料和测试，走了一些弯路，因为之前直接尝试裸机安装黑苹果发现驱动问题很大，还要做内核Patch，安装成功后Xcode也跑不起来，调试各类驱动特别难，非常容易直接烂掉，后来发现Proxmox这个用KVM虚拟技术安装的教程，直击我的痛点，但秉承维稳的原则，先在VMware尝试了一遍安装（虚拟机套虚拟机套虚拟机），非常的顺利，后面在真机上安装了Proxmox，过程经历了2-3个月，微星主板直接被干坏两次，忍痛换了技嘉主板，顺利很多，这篇文档把我的配置和操作记录下来，供参考和查阅。

## 目标
macOS Venture安装到Proxmox，实现GPU直通、USB直通、蓝牙直通、WiFi直通

## 设备配置

CPU: AMD Ryzen 7 3700X 8-Core Processor
GPU: AMD Radeon RX 5700XT
内存：金士顿 DDR4 32GB
主板：技嘉 Gigabyte Technology Co., Ltd. X570 AORUS PRO WIFI/X570 AORUS PRO WIFI, BIOS F35 01/04/2022
硬盘：Samsung SSD 870 EVO 1TB

之所以选择这么贵的硬盘主要是为了抵消虚拟化的性能损失，能接近苹果上的表现。然后主板有板载WIFi+蓝牙，Intel AX200可以破解使用，我其实只要蓝牙就可以，用来连接苹果蓝牙键盘和触控板。

## 安装Promox

进入proxmox官网下载PVE ISO镜像，我使用的是7.3版本，使用其他工具写到U盘里面，然后安装PVE系统到电脑上，我把整个970都给了PVE。
https://www.proxmox.com/en/downloads/category/iso-images-pve

## 升级到最新内核版本

这是为了后续`vendor-reset`和其他模块运行提供便利，到时候升级可能会出问题，提示header找不到之类的，proxmox被弄崩溃一次。

最好是科学上网，直连容易断，首先注释掉企业版的更新源：

```shell
# 另一台电脑可以进行科学上网
export http_proxy=http://192.168.2.129:1111
export https_proxy=http://192.168.2.129:1111
nano /etc/apt/sources.list.d/pve-enterprise.list
```

这行注释
```shell
# deb https://enterprise.proxmox.com/debian/pve bullseye pve-enterprise
```

然后更新系统，更新后要重启系统：

```shell
apt update && apt dist-upgrade
update-initramfs -u
reboot
```

更新完成后，再按照如下文档安装`vendor-reset`, 更新内核后都要重启系统:

> 参考文档：https://www.nicksherlock.com/2020/11/working-around-the-amd-gpu-reset-bug-on-proxmox/

## 安装macOS Venture

根据下面的文档安装即可，博主写得非常好，下面也有很多帖子讨论一些问题处理方法：

https://www.nicksherlock.com/2022/10/installing-macos-13-ventura-on-proxmox/

注：macOS无法识别核心为单数的CPU，但是Socket数量没有这个限制，3700X有16个逻辑核心，我分配了 3 Socket * 4 Core = 12个核心给macOS，另外4个给proxmox用，防止有后续需求。内存也是只分配28G给macOS。

### 虚拟机配置

我的虚拟机ID是100，配置文件是`/etc/pve/qemu-server/100.conf`，其他参数基本根据不同情况配置，这里po下我的args参数（THE OS KEY需要通过前面文档获取）

```shell
args: -device isa-applesmc,osk="THE OS KEY" -smbios type=2 -device usb-kbd,bus=ehci.0,port=2 -global nec-usb-xhci.msi=off -global ICH9-LPC.acpi-pci-hotplug-with-bridge-support=off -cpu Haswell-noTSX,vendor=GenuineIntel,+invtsc,+hypervisor,kvm=on,vmware-cpuid-freq=on
```

#### GRUB配置

```shell
GRUB_CMDLINE_LINUX_DEFAULT="quiet amd_iommu=on iommu=pt video=vesafb:off,efifb:off initcall_blacklist=sysfb_init rootdelay=10"
```

`initcall_blacklist=sysfb_init` 和 `rootdelay=10` 是必须的，否则单GPU及时配置了blacklist也是会被宿主系统使用。

#### PCI Passthrough配置

`nano /etc/modules`添加如下模块
```shell
vendor-reset
vfio
vfio_iommu_type1
vfio_pci
vfio_virqfd
```

其他配置
```shell
echo "options vfio_iommu_type1 allow_unsafe_interrupts=1" > /etc/modprobe.d/iommu_unsafe_interrupts.conf
echo "options kvm ignore_msrs=1" > /etc/modprobe.d/kvm.conf
echo "options kvm-amd nested=Y" >> /etc/modprobe.d/kvm-amd.conf
# GPU HDMI 音频直通相关
echo "options snd-hda-intel enable_msi=1" >> /etc/modprobe.d/snd-hda-intel.conf

# ids是所有需要直通的PCI设备ID，`lspci -n`命令可以查到
echo "options vfio-pci ids=1002:731f,1002:ab38,8086:2723,1022:1485,1022:1486,1022:149c,1022:1487 disable_vga=1" > /etc/modprobe.d/vfio.conf

# 显卡
echo "blacklist radeon" >> /etc/modprobe.d/blacklist.conf
echo "blacklist nouveau" >> /etc/modprobe.d/blacklist.conf
echo "blacklist nvidia" >> /etc/modprobe.d/blacklist.conf
echo "blacklist amdgpu" >> /etc/modprobe.d/blacklist.conf

# HDMI
echo "blacklist snd_hda_intel" >> /etc/modprobe.d/blacklist.conf

# WiFi
echo "blacklist iwlwifi" >> /etc/modprobe.d/blacklist.conf

# Cryptographic Coprocessor PSPCPP，影响其他IOMMU Group
echo "blacklist ccp" >> /etc/modprobe.d/blacklist.conf
```

注：正确配置后显示器会卡在 `init xxxxxxxxx` 的grub界面，如果没有则gpu被宿主机使用，要重新检查配置

## 驱动

安装成功并且PCI设备直通后，就是要处理macOS的驱动问题了，Wifi、蓝牙是没有驱动的，而显卡又需要一些hack。接下来这部分就是和黑苹果相关，处理OpenCore的配置。

先挂载OpenCore的EFI分区
```shell
~$ diskutil list
/dev/disk0 (external, physical):
   #:                   TYPE NAME              SIZE       IDENTIFIER
   0:  GUID_partition_scheme                  *512.1 GB   disk0
   1:                    EFI EFI               209.7 MB   disk0s1
   2:             Apple_APFS Container disk1   511.9 GB   disk0s2
sudo mkdir /Volumes/EFI
sudo mount -t msdos /dev/disk0s1 /Volumes/EFI
```

5700XT显卡需要正常驱动，需要添加启动参数`NVRAM>Add>7C436110-AB2A-4BBB-A880-FE41995C9F82>boot-args`为`keepsyms=1 agdpmod=pikera`。
> 参考文档：https://dortania.github.io/GPU-Buyers-Guide/modern-gpus/amd-gpu.html#navi-21-series

WiFi驱动，添加方法按照文档来
https://github.com/OpenIntelWireless/itlwm

蓝牙驱动，添加方法按照文档来
https://github.com/OpenIntelWireless/IntelBluetoothFirmware

优先使用 `AirportItlwm`，支持系统原生控制隔空投送等功能（目前还是alpha版本不稳定），`itlwm` 需要配合HeilPort使用。

如果使用 `AirportItlwm` 则需要配置安全启动，中级的就可以，完全的安全启动测试没有通过。`DmgLoading`为`Signed`，`SecureBootModel`为你的设备对应值，修改后重新启动系统。
> SecureBootModel 参考文档 https://dortania.github.io/OpenCore-Post-Install/universal/security/applesecureboot.html#special-notes-with-securebootmodel

## 启动虚拟机

因为显卡的reset bug，启动虚拟机需要两次启动和一些hack，要移除需要直通的PCI设备并重新rescan。

> /sys/bus/pci/devices/xxxx/ 是对应的设备ID，需要通过命令或者proxmox界面上添加PCI设备上查看

### 第一次启动
```shell
export LC_ALL=en_US.UTF-8
# 显卡
echo 1 > /sys/bus/pci/devices/0000:0b:00.0/remove
# 显卡HDMI
echo 1 > /sys/bus/pci/devices/0000:0b:00.1/remove
# 板载 Intel AX 200 无线网络WiFi模块
echo 1 > /sys/bus/pci/devices/0000:04:00.0/remove
# USB控制器
echo 1 > /sys/bus/pci/devices/0000:06:00.1/remove
echo 1 > /sys/bus/pci/devices/0000:06:00.3/remove
echo 1 > /sys/bus/pci/devices/0000:0d:00.3/remove
echo 1 > /sys/bus/pci/rescan
# vendor-reset需要这句特殊配置
echo 'device_specific' > /sys/bus/pci/devices/0000:0b:00.0/reset_method
qm start 100
```

启动后通过`dmesg`可以看到如下log, `No more image in the PCI ROM`说明显卡没有启动成功。
```shell
[   92.007715] vendor-reset-drm: atomfirmware: bios_scratch_reg_offset initialized to 4c
[   92.220640] vfio-pci 0000:0b:00.0: AMD_NAVI10: bus reset disabled? yes
[   92.220644] vfio-pci 0000:0b:00.0: AMD_NAVI10: SMU response reg: 0, sol reg: 0, mp1 intr enabled? no, bl ready? yes
[   92.220648] vfio-pci 0000:0b:00.0: AMD_NAVI10: performing post-reset
[   92.257817] vfio-pci 0000:0b:00.0: AMD_NAVI10: reset result = 0
[   94.696592] vfio-pci 0000:0b:00.0: No more image in the PCI ROM
[   94.696614] vfio-pci 0000:0b:00.0: No more image in the PCI ROM
```

关闭虚拟机
```shell
qm stop 100
```

### 第二次启动
```shell
# 显卡
echo 1 > /sys/bus/pci/devices/0000:0b:00.0/remove
# 显卡HDMI
echo 1 > /sys/bus/pci/devices/0000:0b:00.1/remove
# 板载 Intel AX 200 无线网络WiFi模块
echo 1 > /sys/bus/pci/devices/0000:04:00.0/remove
# USB控制器
echo 1 > /sys/bus/pci/devices/0000:06:00.1/remove
echo 1 > /sys/bus/pci/devices/0000:06:00.3/remove
echo 1 > /sys/bus/pci/devices/0000:0d:00.3/remove
echo 1 > /sys/bus/pci/rescan
# vendor-reset需要这句特殊配置
echo 'device_specific' > /sys/bus/pci/devices/0000:0b:00.0/reset_method
qm start 100
```

代码和第一次一样，但是`dmesg`没有`No more image in the PCI ROM`就说明是成功了，等一会就看到显示器亮了
```shell
[  127.898210] vendor-reset-drm: atomfirmware: bios_scratch_reg_offset initialized to 4c
[  128.111121] vfio-pci 0000:0b:00.0: AMD_NAVI10: bus reset disabled? yes
[  128.111125] vfio-pci 0000:0b:00.0: AMD_NAVI10: SMU response reg: 0, sol reg: 0, mp1 intr enabled? no, bl ready? yes
[  128.111128] vfio-pci 0000:0b:00.0: AMD_NAVI10: performing post-reset
[  128.149985] vfio-pci 0000:0b:00.0: AMD_NAVI10: reset result = 0
```