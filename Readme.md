# Ubuntu Linux SteamVR Setup Guide on a Windows Virtual Machine

## Motivation
Provide a way to run Linux as the host operating system while running SteamVR on Windows at the same time when necessary to access VR features not yet supported on Linux.

## SteamVR Linux
Valve provides above average support for SteamVR on Linux. SteamVR runs on Linux, There is a [dedicated forum](https://steamcommunity.com/app/250820/discussions/5/) for Linux support, their marquee game Half-Life: Alyx may soon [support Linux](https://www.gamingonlinux.com/articles/it-appears-that-valve-are-preparing-half-life-alyx-for-linux.16503), and Valve is sponsoring [Monado and the OpenXR Desktop](https://www.collabora.com/news-and-blog/news-and-events/xrdesktop-014-with-openxr-support-released.html). While all of these VR advancements on Linux are welcome Linux support still lags behind Windows, most notably with [WebXR](https://developer.mozilla.org/en-US/docs/Web/API/WebXR_Device_API). Current versions of Chrome and Firefox support WebXR on Windows but there is [no support](https://steamcommunity.com/app/250820/discussions/5/1639789306564271606/#c1639789306574958014) on Linux. In order to run the latest VR applications Windows is required unfortunately.

## Linux Virtual Machines
Describing all aspects of [Linux virtualization](https://www.redhat.com/en/topics/virtualization) is beyond the scope of this document. A general summary follows.

Linux is frequently used in data centers to host virtualized systems. Chip makers like AMD and Intel compete in the data center market and many of the virtualization features in their enterprise processors trickle down into the consumer market. With a modern Linux distribution and kernel setting up virtual machines is fairly straight forward with some caveats. [KVM](https://www.redhat.com/en/topics/virtualization/what-is-KVM), [QEMU](https://www.qemu.org/), and [libvirt](https://libvirt.org/) are key software components that are combined together to assist with the execution and management of virtual machines.

### PCI Passthrough
By definition virtualization virtualizes hardware devices. While some devices like CPUs and network adapters have advanced support for virtualization in hardware other devices like GPUs do not. Today operating systems like Windows offload graphics API calls directly to device drivers which in turn interact with native hardware. Despite the gallant efforts of Open Source driver and virtualization developers attempting to create virtualization layers inbetween guest OS's and hardware their efforts fall short due to the complexity and opaqueness of proprietary GPU devices. This causes display adapter virtualization performance to greatly lag behind native performance. While GPU virtualization technologies such as [SR-IOV](https://community.amd.com/community/radeon-instinct-accelerators/blog/2019/11/01/what-is-sr-iov-why-it-s-the-gold-standard-for-gpu-sharing) are on the horizon today only PCI passthrough can provide near native graphics performance needed for gaming and VR.

As the name implies PCI passthrough, and USB passthrough for that matter, pass device control directly from the host OS to the guest VM. While the VM is running the guest VM's OS loads it's own device driver and directly interacts with the device using special [VFIO software](https://www.kernel.org/doc/Documentation/vfio.txt) and [IOMMU hardware](https://terenceli.github.io/%E6%8A%80%E6%9C%AF/2019/08/04/iommu-introduction) virtualization features to provide near bare metal performance.

Intuitively, passing a GPU through from the host to a guest means it will no longer be available in the host. While there techniques for using a [single GPU](https://github.com/joeknock90/Single-GPU-Passthrough) the result has little to no benefit over dual booting so installing two GPUs in the host is the most common configuration. In regards to VR all HMD's require USB interfaces and while passing through individual USB devices could possibly work passing a PCI USB controller with the HMD plugged into it is the most robust solution.

### KVM

It seems strange but one needs to use a KVM with a single computer if the USB controller the mouse and keyboard are plugged into is passed to a guest VM. I already had a KVM for switching inputs between my desktop computer and some work laptops so this was not an issue for me.

Configuring [evdev](https://passthroughpo.st/using-evdev-passthrough-seamless-vm-input/) is an alternative to using a KVM switch although if one is passing a USB controller it would make more sense to achieve a native mouse and keyboard experience. For example I am using a Razer mouse and keyboard which has additional Windows software available for it.



### Dual boot or Virtualization?
Given the PCI pass through hardware requirements above and the known issues listed below virtualization may not be the best solution in all circumstances. [Dual booting](https://en.wikipedia.org/wiki/Multi-booting) using a [UEFI boot manager](https://docs.microsoft.com/en-us/windows-hardware/drivers/bringup/boot-and-uefi) or software boot manager like [Grub](https://itsfoss.com/grub-customizer-ubuntu/) one can run multiple operating systems on the same hardware and achieve optimal performance. On the other hand virtualization saves time switching back and forth between OS's, data can be moved between the running systems, and the guest performance is close to bare metal. Additionally moving Windows VM machines between different hosts becomes possible although the Microsoft [one instance](https://www.microsoft.com/en-us/Useterms/Retail/Windows/10/UseTerms_Retail_Windows_10_English.htm) licensing policy should be adhered to.

### Known Issues
These issues affect the specific AMD host hardware I am using but these problems appear to affect many others as well and may factor into one's hardware purchasing decisions. Despite the frustrations I encountered I am still extremely satisfied with the features and value of AMD products and I will continue to be a loyal customer.

* **AMD X570 motherboard USB 3.0 Controller PCI pass through**
 * This appears to affect all [AMD x570 motherboards and requires a kernel patch](https://www.reddit.com/r/VFIO/comments/dwxxtb/passing_though_a_usb_controller_on_a_gigabyte/) to fix. Since I plan to use the Linux host and Windows guest at the same time and the main USB controller manages all the USB ports on the motherboard I decided to purchase an inexpensive extra PCI-E USB 3.0 controller. This way I could continue to use the host without moving cables around and pass a USB controller with all of the HMD USB devices together. Hopefully this issue will be fixed in a future AGESA update.


* **AMD Vega reset bug**
  * An issue still remains with powering down a guest VM with an AMD GPU pass through device and the GPU [falls into an invalid state](https://www.reddit.com/r/VFIO/comments/ccfe8z/can_someone_explain_the_vega_reset_bug_to_me/etmk0up?utm_source=share&utm_medium=web2x). Once the GPU is in this invalid state the guest OS cannot be restarted. What's more the GPU will throttle up to 100% usage causing the GPU fan/blower to max out. This can be quite an unexpected surprise! The only workaround to this problem currently is to put the host to sleep which ostensibly clears out the invalid state from the GPU device so that it becomes valid again. Infrequently the KVM will return an error 127 then that will require a reboot.I discovered that installing the VFIO guest agent on the Windows guest and then restarting it to activate it seemed to shutdown the guest in an orderly fashion and solve the guest restart issue completely. I am able to restart the the guest multiple times without a host reboot or host suspend. Note this does not affect unbinding the GPU from the VFIO-PCI driver, binding the AMDGPU driver for local Linux host access, and then rebinding the VFIO-PCI driver. that still remains an open AMDGPU driver cleanup issue.


* **AMD driver guest OS installation hangs host**
    * This issue did not occur when I used a BIOS firmware for the guest and only happened when I configure UEFI. Providing a [GPU ROM file](https://www.reddit.com/r/VFIO/comments/c17igy/fix_for_vega_5664_reset_bug_gentoo/) in the guest VM configuration solved the issue but it unfortunately did not solve the reset bug. Note that this solution requires dual booting Windows once so that the AMD drivers will update the GPU and the latest BIOS can be dumped to a file for use by the guest VM.


* **Valve Index camera not working**
    * This probably has [nothing to do](https://steamcommunity.com/app/250820/discussions/2/1752394325937022875/) with virtualization because the same thing happened when running Windows natively.


* **Immediate wake upon sleep**
    * This didn't happen before virtualization configurations were added to the host. I am still investigating.


* **AMD Vega high fan speed while using the VFIO-PCI driver**
    * The Vega blower runs at a slightly above normal fan speed when not passed through to a guest or bound in the host by the AMDGPU driver. The AMD drivers undervolt the GPU and reduce the fan speed when not under load and the kernel VFIO-PCI driver doesn't do that.


* **Windows Device Manager Unknown device**
    * Device path ACPI\\APP0005\\3&2411E6FE&0 - ignore it, it is a [QEMU bug](https://bugs.launchpad.net/qemu/+bug/1856724)


## Hardware
Here are the hardware components for the Linux host OS:

* **CPU** - AMD Ryzen 3900X
* **Motherboard** - ASRock X570 Steel Legend
* **WIFI** -  OKN WiFi 6 AX200 Key E and a M.2 NGFF antenna
* **RAM** - Corsair Vengeance 32GB DDR4-3200
* **Storage** - Inland Performance M.2 PCI3 4.0 1TB SSD
* **Power Supply** - MAINGEAR IGNITION 1000W
* **PCI Expansion Card USB Controller** - FebSmart PCI Express Slot 4-Port USB 3.0 Controller (slot PCIE3)
* **KVM** - UGREEN USB 2.0 switch box hub (USB only, no video)
* **Host GPU** - AMD MSI Radeon RX 480 (slot PCIE4)
* **Pass through GPU** - AMD Saphire Radeon RX Vega 56 (slot PCIE1)
* **Dual Boot SSD** - Old Samsung EVO 860 256GB SATA SSD
* **Monitors** -  Two LG 32UK50T
* **VR Headset** -  Valve Index


## Cabling
* Plug in the primary KVM switch USB cable into the PCI-E card USB port (1)

* Plug the Valve Index headset USB cable into the PCI-E card USB port (2)

* Plug in the two valve index controllers side by side into the PCI-E card USB port (3 & 4)

* Plug in any other host USB devices like a speaker into the motherboard USB port.

* Plug in the other KVM switch USB cable into the host's front case panel USB port.

* Plug the Valve Index headset Display Port cable into the Display Port on the pass through GPU.

* Plug the Display Port cables from both of the monitors into the host GPU (display ports 1 &2)

* Plug the HDMI cable from the primary monitor into the pass through GPU.


## Operating Systems

Host - **Ubuntu Linux 20.04**. Many in the VFIO community favor Arch Linux since it has more documentation and up-to-date support for Linux virtualization. Kernel - 5.6.11

Guest - **Windows 10** November 2019 Update

## Host Setup

### BIOS

Use instaflash and a USB drive to updpate the BIOS to the latest version

Start the host. Hold the BIOS button (F2) to enter the BIOS. Ensure the following settings are enabled:

* Advanced -> CPU Configuration -> SVM mode - Auto

* Advanced -> AMD CBS -> NBIO Common Options -> IOMMU - Auto, PCIe ARI - Auto

### Ubuntu Focal Fossa 20.04 Installation

Download the 64-bit PC (AMD64) desktop image ISO file from the Ubuntu [download page](https://releases.ubuntu.com/20.04/)

[Create a bootable USB drive](https://ubuntu.com/tutorials/tutorial-create-a-usb-stick-on-ubuntu#1-overview), install the OS.

#### List the IOMMU Groups
Record for future reference.
```
shopt -s nullglob
for d in /sys/kernel/iommu_groups/*/devices/*; do
    n=${d#*/iommu_groups/*}; n=${n%%/*}
    printf 'IOMMU Group %s ' "$n"
    lspci -nns "${d##*/}"
done;


IOMMU Group 0 00:01.0 Host bridge [0600]: Advanced Micro Devices, Inc. [AMD] Starship/Matisse PCIe Dummy Host Bridge [1022:1482]
IOMMU Group 10 00:08.0 Host bridge [0600]: Advanced Micro Devices, Inc. [AMD] Starship/Matisse PCIe Dummy Host Bridge [1022:1482]
IOMMU Group 11 00:08.1 PCI bridge [0604]: Advanced Micro Devices, Inc. [AMD] Starship/Matisse Internal PCIe GPP Bridge 0 to bus[E:B] [1022:1484]
IOMMU Group 12 00:08.2 PCI bridge [0604]: Advanced Micro Devices, Inc. [AMD] Starship/Matisse Internal PCIe GPP Bridge 0 to bus[E:B] [1022:1484]
IOMMU Group 13 00:08.3 PCI bridge [0604]: Advanced Micro Devices, Inc. [AMD] Starship/Matisse Internal PCIe GPP Bridge 0 to bus[E:B] [1022:1484]
IOMMU Group 14 00:14.0 SMBus [0c05]: Advanced Micro Devices, Inc. [AMD] FCH SMBus Controller [1022:790b] (rev 61)
IOMMU Group 14 00:14.3 ISA bridge [0601]: Advanced Micro Devices, Inc. [AMD] FCH LPC Bridge [1022:790e] (rev 51)
IOMMU Group 15 00:18.0 Host bridge [0600]: Advanced Micro Devices, Inc. [AMD] Matisse Device 24: Function 0 [1022:1440]
IOMMU Group 15 00:18.1 Host bridge [0600]: Advanced Micro Devices, Inc. [AMD] Matisse Device 24: Function 1 [1022:1441]
IOMMU Group 15 00:18.2 Host bridge [0600]: Advanced Micro Devices, Inc. [AMD] Matisse Device 24: Function 2 [1022:1442]
IOMMU Group 15 00:18.3 Host bridge [0600]: Advanced Micro Devices, Inc. [AMD] Matisse Device 24: Function 3 [1022:1443]
IOMMU Group 15 00:18.4 Host bridge [0600]: Advanced Micro Devices, Inc. [AMD] Matisse Device 24: Function 4 [1022:1444]
IOMMU Group 15 00:18.5 Host bridge [0600]: Advanced Micro Devices, Inc. [AMD] Matisse Device 24: Function 5 [1022:1445]
IOMMU Group 15 00:18.6 Host bridge [0600]: Advanced Micro Devices, Inc. [AMD] Matisse Device 24: Function 6 [1022:1446]
IOMMU Group 15 00:18.7 Host bridge [0600]: Advanced Micro Devices, Inc. [AMD] Matisse Device 24: Function 7 [1022:1447]
IOMMU Group 16 01:00.0 Non-Volatile memory controller [0108]: Phison Electronics Corporation E16 PCIe4 NVMe Controller [1987:5016] (rev 01)
IOMMU Group 17 02:00.0 PCI bridge [0604]: Advanced Micro Devices, Inc. [AMD] Matisse Switch Upstream [1022:57ad]
IOMMU Group 18 03:02.0 PCI bridge [0604]: Advanced Micro Devices, Inc. [AMD] Matisse PCIe GPP Bridge [1022:57a3]
IOMMU Group 19 03:03.0 PCI bridge [0604]: Advanced Micro Devices, Inc. [AMD] Matisse PCIe GPP Bridge [1022:57a3]
IOMMU Group 1 00:01.1 PCI bridge [0604]: Advanced Micro Devices, Inc. [AMD] Starship/Matisse GPP Bridge [1022:1483]
IOMMU Group 20 03:05.0 PCI bridge [0604]: Advanced Micro Devices, Inc. [AMD] Matisse PCIe GPP Bridge [1022:57a3]
IOMMU Group 21 03:08.0 PCI bridge [0604]: Advanced Micro Devices, Inc. [AMD] Matisse PCIe GPP Bridge [1022:57a4]
IOMMU Group 21 0c:00.0 Non-Essential Instrumentation [1300]: Advanced Micro Devices, Inc. [AMD] Starship/Matisse Reserved SPP [1022:1485]
IOMMU Group 21 0c:00.1 USB controller [0c03]: Advanced Micro Devices, Inc. [AMD] Matisse USB 3.0 Host Controller [1022:149c]
IOMMU Group 21 0c:00.3 USB controller [0c03]: Advanced Micro Devices, Inc. [AMD] Matisse USB 3.0 Host Controller [1022:149c]
IOMMU Group 22 03:09.0 PCI bridge [0604]: Advanced Micro Devices, Inc. [AMD] Matisse PCIe GPP Bridge [1022:57a4]
IOMMU Group 22 0d:00.0 SATA controller [0106]: Advanced Micro Devices, Inc. [AMD] FCH SATA Controller [AHCI mode] [1022:7901] (rev 51)
IOMMU Group 23 03:0a.0 PCI bridge [0604]: Advanced Micro Devices, Inc. [AMD] Matisse PCIe GPP Bridge [1022:57a4]
IOMMU Group 23 0e:00.0 SATA controller [0106]: Advanced Micro Devices, Inc. [AMD] FCH SATA Controller [AHCI mode] [1022:7901] (rev 51)
IOMMU Group 24 04:00.0 VGA compatible controller [0300]: Advanced Micro Devices, Inc. [AMD/ATI] Ellesmere [Radeon RX 470/480/570/570X/580/580X/590] [1002:67df] (rev c7)
IOMMU Group 24 04:00.1 Audio device [0403]: Advanced Micro Devices, Inc. [AMD/ATI] Ellesmere HDMI Audio [Radeon RX 470/480 / 570/580/590] [1002:aaf0]
IOMMU Group 25 05:00.0 PCI bridge [0604]: ASMedia Technology Inc. ASM1184e PCIe Switch Port [1b21:1184]
IOMMU Group 26 06:01.0 PCI bridge [0604]: ASMedia Technology Inc. ASM1184e PCIe Switch Port [1b21:1184]
IOMMU Group 26 07:00.0 Network controller [0280]: Intel Corporation Wi-Fi 6 AX200 [8086:2723] (rev 1a)
IOMMU Group 27 06:03.0 PCI bridge [0604]: ASMedia Technology Inc. ASM1184e PCIe Switch Port [1b21:1184]
IOMMU Group 28 06:05.0 PCI bridge [0604]: ASMedia Technology Inc. ASM1184e PCIe Switch Port [1b21:1184]
IOMMU Group 28 09:00.0 Ethernet controller [0200]: Intel Corporation I211 Gigabit Network Connection [8086:1539] (rev 03)
IOMMU Group 29 06:07.0 PCI bridge [0604]: ASMedia Technology Inc. ASM1184e PCIe Switch Port [1b21:1184]
IOMMU Group 2 00:01.2 PCI bridge [0604]: Advanced Micro Devices, Inc. [AMD] Starship/Matisse GPP Bridge [1022:1483]
IOMMU Group 30 0b:00.0 USB controller [0c03]: Renesas Technology Corp. uPD720201 USB 3.0 Host Controller [1912:0014] (rev 03)
IOMMU Group 31 0f:00.0 PCI bridge [0604]: Advanced Micro Devices, Inc. [AMD] Device [1022:1470] (rev c3)
IOMMU Group 32 10:00.0 PCI bridge [0604]: Advanced Micro Devices, Inc. [AMD] Device [1022:1471]
IOMMU Group 33 11:00.0 VGA compatible controller [0300]: Advanced Micro Devices, Inc. [AMD/ATI] Vega 10 XL/XT [Radeon RX Vega 56/64] [1002:687f] (rev c3)
IOMMU Group 34 11:00.1 Audio device [0403]: Advanced Micro Devices, Inc. [AMD/ATI] Vega 10 HDMI Audio [Radeon Vega 56/64] [1002:aaf8]
IOMMU Group 35 12:00.0 Non-Essential Instrumentation [1300]: Advanced Micro Devices, Inc. [AMD] Starship/Matisse PCIe Dummy Function [1022:148a]
IOMMU Group 36 13:00.0 Non-Essential Instrumentation [1300]: Advanced Micro Devices, Inc. [AMD] Starship/Matisse Reserved SPP [1022:1485]
IOMMU Group 37 13:00.1 Encryption controller [1080]: Advanced Micro Devices, Inc. [AMD] Starship/Matisse Cryptographic Coprocessor PSPCPP [1022:1486]
IOMMU Group 38 13:00.3 USB controller [0c03]: Advanced Micro Devices, Inc. [AMD] Matisse USB 3.0 Host Controller [1022:149c]
IOMMU Group 39 13:00.4 Audio device [0403]: Advanced Micro Devices, Inc. [AMD] Starship/Matisse HD Audio Controller [1022:1487]
IOMMU Group 3 00:02.0 Host bridge [0600]: Advanced Micro Devices, Inc. [AMD] Starship/Matisse PCIe Dummy Host Bridge [1022:1482]
IOMMU Group 40 14:00.0 SATA controller [0106]: Advanced Micro Devices, Inc. [AMD] FCH SATA Controller [AHCI mode] [1022:7901] (rev 51)
IOMMU Group 41 15:00.0 SATA controller [0106]: Advanced Micro Devices, Inc. [AMD] FCH SATA Controller [AHCI mode] [1022:7901] (rev 51)
IOMMU Group 4 00:03.0 Host bridge [0600]: Advanced Micro Devices, Inc. [AMD] Starship/Matisse PCIe Dummy Host Bridge [1022:1482]
IOMMU Group 5 00:03.1 PCI bridge [0604]: Advanced Micro Devices, Inc. [AMD] Starship/Matisse GPP Bridge [1022:1483]
IOMMU Group 6 00:04.0 Host bridge [0600]: Advanced Micro Devices, Inc. [AMD] Starship/Matisse PCIe Dummy Host Bridge [1022:1482]
IOMMU Group 7 00:05.0 Host bridge [0600]: Advanced Micro Devices, Inc. [AMD] Starship/Matisse PCIe Dummy Host Bridge [1022:1482]
IOMMU Group 8 00:07.0 Host bridge [0600]: Advanced Micro Devices, Inc. [AMD] Starship/Matisse PCIe Dummy Host Bridge [1022:1482]
IOMMU Group 9 00:07.1 PCI bridge [0604]: Advanced Micro Devices, Inc. [AMD] Starship/Matisse Internal PCIe GPP Bridge 0 to bus[E:B] [1022:1484]

```

Note **IOMMU Group 30**, **IOMMU Group 33**, and **IOMMU Group 34** above. 30 is the USB 3.0 controller PCI expansion card, 33 is the passthrough GPU, and 34 is the pass through GPU HDMI Audio device. All three of these will be passed to the guest VM. Also note the passthrough GPU PCI vendor IDs 1002:687f and 1002:aaf8. These will be used later.

#### Distribution update
Ensure the OS is up-to-date
```
sudo apt-get update
sudo apt-get upgrade
sudo apt-get dist-upgrade
```

#### AMD Vulkan Driver
Not directly related to VFIO but to achieve optimal performance on the host if it uses an AMD GPU install the AMD Vulkan drivers. Ubuntu comes with the Open Source `mesa-vulkan-drivers` which offer the same if not better performance than the AMD driver. To optionally install the AMD Vulkan driver perform the following:

```
sudo wget -qO - http://repo.radeon.com/amdvlk/apt/debian/amdvlk.gpg.key | sudo apt-key add -
sudo sh -c 'echo deb [arch=amd64] http://repo.radeon.com/amdvlk/apt/debian/ bionic main > /etc/apt/sources.list.d/amdvlk.list'
sudo apt update
sudo apt-get install amdvlk
```

#### Kernel Update
Install mainline for Ubuntu kernel updates:
```
sudo apt-add-repository -y ppa:cappelikan/ppa
sudo apt update
sudo apt install mainline
```

Install the latest kernel. Reboot.

#### Install the KVM, QEMU, and Libvirt tools

`sudo apt install qemu-kvm libvirt-clients libvirt-daemon-system bridge-utils virt-manager`

#### Update the Kernel Parameters
In a perfect world with the QEMU/libvirt managed device setting would effortless move a PCI device between the host and guest. This works fine for the PCI expansion USB 3.0 controller for example. However as it stands today the AMDGPU driver will grab the passthrough GPU during the Linux boot process and modify it so that it will not work with VFIO. To avoid that from happening the PCI vendor IDs need to be set as a kernel paramemter so that the VFIO functionality built into the latest Linux kernel version will acquire the device first. Using a guest firmware of BIOS didn't require this setting but when I switched over to UEFI this fixed the guest black sreen issue.

```
sudo vi /etc/default/grub
GRUB_CMDLINE_LINUX_DEFAULT="quiet splash vfio-pci.ids=1002:687f,1002:aaf8"

sudo update-grub
```

Optinally add a second grub menu item without the VFIO parameters so that the pass through GPU will be available in the host for SteamVR Linux gaming or AMD ROCM development. Alternatively, one could create a Linux VM and pass the GPU to that guest which would have access to it.

restart the host.


## Guest Software

* [Windows 10 Installation ISO](https://www.microsoft.com/en-us/software-download/windows10ISO), English, 64bit

* [VirtIO Windows drivers](https://fedorapeople.org/groups/virt/virtio-win/direct-downloads/latest-virtio/) - additional [details](https://docs.fedoraproject.org/en-US/quick-docs/creating-windows-virtual-machines-using-virtio-drivers/index.html).



## Guest Setup - Dual Boot (Optional)
As mentioned in the known issues in order to avoid the guest AMD driver lockup issue the VBIOS of the AMD card needs to be dumped from Windows after a successful driver install. If one has access to another desktop with an adequate PSU that is already running Windows then the passthrough GPU can be moved over to it, the drivers updated, and the bios dumped.

Another alternative is to download a vbios from the [Tech Power Up Collection](https://www.techpowerup.com/vgabios/) Some of the vbios look dated but they may work, I didn't try.


Otherwise,  acquire an extra drive and dual boot Windows.

Create a Windows boot USB drive in Ubuntu.
```
lsblk

 sda           8:0    1  28.5G  0 disk
└─sda1        8:1    1  28.5G  0 part /media/dev/Transcend

umount /media/dev/Transcend

wget https://raw.githubusercontent.com/slacka/WoeUSB/master/src/woeusb
chmod +x woeusb
sudo ./woeusb --target-filesystem NTFS --device ~/Downloads/Win10_1909_English_x64.iso /dev/sda

```

Reboot, press F11 to enter the UEFI boot manager, select the Windows installation USB.

Install Windows 10 Pro. No need to activate it if you are not going to dual boot.  

After it is running, install the [AMD drivers](https://www.amd.com/en/support).

Install [GPU-Z](https://www.techpowerup.com/gpuz/), dump the vbios (be sure to select the pass through card and not the host), copy the vbios to a USB drive.

Boot back into the host.

## Guest Installation

Start QEMU:

`virt-manager`

File ->  New Virtual Machine
  * Local install media
  * Choose ISO -> Browse -> Browse Local - select the Win10_1909_English_x64.iso image
  * Confirm the OS is Microsoft Windows 10

(Adjust these settings for the host as needed)
* Memory: 20480
* CPU: 20

Select or create custom storage (needed to auto-grow qcow2 or allocate upfront raw storage for performance).
* Manage -> create new volume -> Name SteamVRWindows10, format qcow2, max capacity 512GB

* Name: SteamVRWindows10

* Storage: customize configuration before install

* Network selection - default - Virtual network 'default': NAT

Now make the following adjustments to the guest VM virtual hardware:

* Overview - Chipset - Q35

* Overview - Chipset - Firmware -> UEFI /usr/share/OVFM/OVFM_Code.fd

* Sata Disk 1 -> 	advanced options disk bus -> VirtIO

* NIC -> Device model virtio

* Tablet, Remove Hardware

* Console, Remove Hardware

* Channel spice, Remove hardware

* Add Hardware, Channel Device, Name com.redhat.spice.0 (default), Device Type - Unix socket

* Display Spice -> Listen type - none

* Add Hardware -> Storage -> Device type -> CDROM device

* SATA CDROM 2-> Source Path -> Browse -> Browse Local - select virtio-win-0.1.173.iso

* Add Hardware, PCI host device, 11:00.0 Advanced Micro Devices, Inc. [AMD/ATI] Vega 10 XL/XT [Radeon RX Vega 56/64]

* Add Hardware, PCI host device, 11:00.1 Advanced Micro Devices, Inc. [AMD/ATI] Vega 10 HDMI Audio [Radeon Vega 56/64]

* Add Hardware, PCI host device, 0B:00.0 Renesas Technology Corp. uPD720201 USB 3.0 Host Controller

Note 1: Notice the three PCI host devices. These are the passthrough GPU (video & audio) and the USB controller the Valve Index are plugged into.

Note 2: The spice display configuration above has been optimized to use VirtIO and a UNIX socket to avoid network overhead. With GPU passthrough it is doubtful that remote access is needed to the guest.

Note 3: One doesn't necessarily need a spice server and virtualized video display adapter and these can be removed. A virtualized display server allows for accessing the guest from the host without performing a KVM switchover. Also Windows Activation is tied to hardware configurations and the profile should be kept constant during the lifetime of the VM to avoid activation issues. Another popular option is [Looking Glass](https://looking-glass.hostfission.com/) which acts as a software based KVM to bridge the host and guest together using a GPU buffer to sync the display.

Optionally delete the Spice server if it is not needed:

* Display Spice, Remove Hardware

* Channel Spice, Remove Hardware

* Video QXL, Remove Hardware

* USB Redirector 1, Remove Hardware

* USB Redirector 2, Remove Hardware

### Windows installation - Part 1

Press enter when prompted to boot from the CDROM. If you are not fast enough and the UEFI shell is displayed type exit and then either select continue to display the CDROM boot prompt again or select boot manager -> UEFI QEMU DVDROM QM03.

Select 'I don't have a product key' to delay activating Windows. Only when the guest VM is fully setup and stabilized should one active Windows since a reinstallation is not out of the question when installing on an evolving platform.

On the installation screen select Custom: Install Windows Only (advanced). WIndows doesn't have the VirtIO drivers included in the distribution so it won't see the storage drive. Click load driver, browse, CD Drive E: virtio-win-0.1.173, amd64\\w10, then select the first option Red Hat VirtIO SCSI controller.

Try to shutdown the VM before the Windows restart triggers. If the guest gets stuck on a black screen due to the AMD GPU restrat bug then force the guest off, put the host to sleep, wake up the host.


### Guest VBIOS Setup

Now that the VM is created it is a good time to add the vbios ROM to the guest as Windows driver installation on the passthrough GPU may hang the guest.

Copy the passthrough GPU vbios to the host

```
sudo mkdir /etc/firmware
sudo cp Vega56.rom /etc/firmware/
sudo chown root:root /etc/firmware/Vega56.rom
sudo chmod 744 /etc/firmware/Vega56.rom

```

`sudo vi /etc/apparmor.d/abstractions/libvirt-qemu`

append to the end, the leading two spaces are import!
```  
  /etc/firmware/* r,
```

`sudo systemctl restart apparmor.service`


Edit the guest VM's Libvirt XML configuration and add a reference to the vbios. This can be done in QEMU GUI by selecting the PCI device and clicking on the XML tab or by using virsh edit.

`virsh edit SteamVRWindows10`

Scroll to the end where the hostdev entries are. Look for the one with the bus and function values match the passthrough GPU. In this example 11:00.0 equates to bus='0x11' function='0x0'. Add a rom element so that the hostdev entry looks like:

```
<hostdev mode='subsystem' type='pci' managed='yes'>
      <source>
        <address domain='0x0000' bus='0x11' slot='0x00' function='0x0'/>
      </source>
      <address type='pci' domain='0x0000' bus='0x05' slot='0x00' function='0x0'/>
        <rom bar='on' file='/etc/firmware/Vega56.rom'/>
    </hostdev>


```
### Windows installation - Part 2

Start the Guest

Finish the Windows installation. Use the KVM and switch the monitor to the HDMI input connected to the pass through GPU to confirm the display is working.

Open the Windows Search and find Device Manager. There are a couple of missing VirtIO drivers. On can install the spice guest tools for Windows and it should load the missing drivers. Another alternative is to update them individually.

Ignore the unknown device missing driver, tht is a QEMU 4.2 bug.

Select the PCI Device VEN_1AF4&DEV_1045 (balloon), select update driver, browse my computer, select E:\\Balloon\\w10\\amd64

Select the PCI Simple Communications Controller, update driver, E:\\vioserial\\w10

### Backup #1
Now is a good time for a backup. Shutdown the guest. Export the Libvirt XML:

`virsh dumpxml SteamVRWindows10 > SteamVRWindows10.xml`
`sudo cp  /var/lib/libvirt/images/SteamVRWindows10.qcow2 .`

Some aspects of the Linux file system show the compact size of the file others do not. Use a QEMU file utility to shrink all aspects of the file.
```
du -h SteamVRWindows10.qcow2
12G	SteamVRWindows10.qcow2

sudo qemu-img convert -p -f qcow2 ./SteamVRWindows10.qcow2 -O qcow2 ./SteamVRWindows10-shrunk.qcow2
```

### Tune CPU performance

Boost CPU performance by optimizing the CPU configuration. Note these settings are for an AMD 3900x (12 core x2 threads)

`virsh edit SteamVRWindows10`

In the Libvirt XML replace
`<vcpu placement="static">20</vcpu>` with `<vcpu placement='static'>24</vcpu>`, the number of concurrent threads the processor supports.

Replace `<cpu mode="host-model" check="partial"/>` with
```
<cpu mode='host-passthrough' check='partial'>
  <model fallback='allow'/>
  <topology sockets='1' cores='12' threads='2'/>
</cpu>

```

From this point forward use the KVM and Monitor input selector to switch back and forth between the VM.

Optionally remove the spice server and related display configuration.

Start the guest.

### VirtIO Guest Agent (Recommended)

In Windows Explorer open  E: virtio-win-0.1.173 and run guest-agent\\qemu-ga-x86_64

Restart the guest to activate the agent.


Install the [AMD drivers](https://www.amd.com/en/support).

Install [Steam](https://store.steampowered.com).

## SteamVR Setup

* Ensure the HMD and Index controller USB cables are connected to the passthrough PCI extension card USB controller.

* Check to make sure that the Index HMD Display Port cable is plugged into the passthrough GPU.

* Make sure the Index HMD and the two Lighthouse base stations are plugged into an electrical outlet.

* Log into Steam.

* Go to Library, SteamVR, and select install.

* Launch Steam VR

* Perform Room Setup

* From the Home environment observe the performance. It should be near native. Open the [Half-Life: Alyx Russell's Lab](https://uploadvr.com/half-life-alyx-steamvr-home-download/) environment and observe the performance.

* Note that SteamVR [does not turn the Lighthouse Base Stations Off in Linux](https://github.com/ValveSoftware/SteamVR-for-Linux/issues/320). Be sure to unplug them after you are finished playing in VR.

## Firefox WebXR

* [Install Firefox](https://www.mozilla.org/en-US/firefox/new/)

* Start SteamVR

* Start Firefox, go to [Hello WebXR!](https://mixedreality.mozilla.org/hello-webxr/)

* Click 'Allow Virtual Reality access'

* Click the 'Enter VR' button

* Navigate and enjoy!!!


## Chrome WebXR

* [Install Chrome](https://www.google.com/chrome/)

* Start SteamVR

* In Chrome open the URL chrome://flags

* Set XR device sandboxing to disabled

* Set Force WebXr Runtime to SteamVR (OpenVR)

* Open the Hello WebXR! URL above, click allow, Enter VR, and put the HMD on.

## Host and Guest Interaction

Virtualization does not have much of an advantage over dual booting if the host and guest are not utilized at the same time. There are a few ways for the guest to access host services.


### Networking

run `ip addr` in the host OS and identify the host's IP address. ignore the loopback lo and virtual vrb and vnet interfaces. For example, 2: enp9s0 ...  inet 192.168.1.25/24

run `python3 -m http.server 192.168.1.25:8080` in the host OS.

Now in a browser on the host open up the URL http://192.168.1.25:8080 to confirm the web server is accessible.

Finally enter the same URL in Firefox running on the Windows guest. The page should load and the log in the host should record the access.

### File Sharing

VirtIO filesystem support is in Kernel 5.4+ and if one adds hardware -> Filesystem, mode mapped, source & target path, the share actually shows up in the Windows guest as PCI Device /VEN_1AF4&DEV_1049. Unfortunately while there is a [Linux guest driver](https://virtio-fs.gitlab.io/) there isn't a [Windows guest driver available yet](https://github.com/virtio-win/kvm-guest-drivers-windows/issues/126).

Another option is to use spice webdav. However this feature doesn't scale well with large files.

The best solution at this time is to setup a [Samba Server](https://www.iodocs.com/use-virt-manager-share-files-linux-host-windows-guest/).

### Clipboard sharing

This only works if a Spice client (Video QXL) is used.

Verify a  com.redhat.spice.0 Spice change is configured on the guest.

Install [spice guest tools](https://www.spice-space.org/download.html) in the Windows guest.

## Libvirt XML

This is the final [SteamVRWindows10.xml](SteamVRWindows10.xml) file

## Resources

* **[Reddit VFIO](https://www.reddit.com/r/VFIO/)** - invaluable amount of knowledge and open collaboration

* **[SteamVR Forum](https://steamcommunity.com/app/250820/discussions/)**
