# Install macOS Ventura On AMD 3700X With 5700XT BY Proxmox

本人mbp15寸用了6年，因为囊中羞涩所以决定使用黑苹果，黑苹果很折腾，需要查阅大量资料和测试，走了一些弯路，因为之前直接尝试裸机安装黑苹果发现驱动问题很大，还要做内核Patch，安装成功后Xcode也跑不起来，调试各类驱动特别难，非常容易直接烂掉，后来发现Proxmox这个用KVM虚拟技术安装的教程，直击我的痛点，但秉承维稳的原则，先在VMware尝试了一遍安装（虚拟机套虚拟机套虚拟机），非常的顺利，后面在真机上安装了Proxmox，过程经历了2-3个月，微星主板直接被干坏两次，忍痛换了技嘉主板，顺利很多，这篇文档把我的配置和操作记录下来，供参考和查阅。

## 目标
macOS Venture安装到Proxmox，实现GPU直通、USB直通、蓝牙直通、WiFi直通

## 设备配置

CPU: AMD Ryzen 7 3700X
GPU: AMD 5700XT
内存：金士顿 DDR4 32GB
主板：技嘉 X570 AORUS PRO Wi-Fi
硬盘：三星 EVO 970 1TB

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

