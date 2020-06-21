---
title: "MacOS on WSL2(Win10)"
date: 2020-06-21 14:23:00 +0900
categories: VM
---

# Disclaimer
This instruction is only care how to install MacOS on WSL2, but not whether you have right permission to install MacOS on your device or not. (EULA)

# Steps
0. prerequirements
1. Setting WSL2
2. Enable KVM
3. set QEMU VM
4. create macOS-Simple-KVM on QEMU
5. Launch

# prerequirements
1. KVM supporting Intel CPU.
2. Device to install MacOS (that does not conflict with EULA)

# Setting WSL2
1. First of all you need to check your Windows 10 version is greater than 2004 (means it was released at April 2020 in general).
If you are not, then you update your Windows and come back here.
2. Then You can enable Windows Subsystem Linux at `Control` - `Apps` - `Programs and Features` - `Turn Windows features on or off`.
![image](https://user-images.githubusercontent.com/769432/85217426-dd415200-b3cb-11ea-909b-eac868c8c639.png)
3. Then go to Microsoft Store. Install `Ubuntu`.
(If you don't want to use Ubuntu, then you can try any other linux. maybe.. :) )
4. Sure you should set your Linux. Just follow instruction by starting `Ubuntu` app.
At this moment you have the WSL but not WSL2.
5. After all, upgrade WSL to version 2. This official instruction is kind and easy enough.
https://docs.microsoft.com/en-us/windows/wsl/install-win10

```PowerShell
PS C:\WINDOWS\system32> dism.exe /online /enable-feature /featurename:VirtualMachinePlatform /all /norestart

# you may need to restart!

PS C:\WINDOWS\system32> wsl --list --verbose
  NAME      STATE           VERSION
* Ubuntu    Running         2
PS C:\WINDOWS\system32> wsl --set-version Ubuntu 2
변환이 진행 중입니다. 몇 분 정도 걸릴 수 있습니다...
WSL 2와의 주요 차이점에 대한 자세한 내용은 https://aka.ms/wsl2를 참조하세요
```

Congrats, you have WSL2 now! but it is just fist step.

# Enable KVM
1. You may think it was dizzy work to set WSL2, but it is absolutely clear than this step. So, take a deep breath before start.
2. First you should get WSL2 Kernel codes. It is preferred to have code of released version. (I am using 4.19.104)
https://github.com/microsoft/WSL2-Linux-Kernel
3. Before clone or unzip the code, you may need to set `per-directory case sensitivity`.
Its quite hassle but if you skip this, you will fail to build after all!
```
PS C:\Windows\system32> fsutil.exe file SetCaseSensitiveInfo C:\some\path\you\use enable
```
4. Then extract( or clone) code somewhere you set case-sensitive.
5. Following below instruction. It cames from [Accelerated KVM guests on WSL 2](https://boxofcables.dev/accelerated-kvm-guests-on-wsl-2/)
```bash
$ sudo apt update && sudo apt -y upgrade
$ sudo apt -y install build-essential libncurses-dev bison flex libssl-dev libelf-dev cpu-checker qemu-kvm aria2 

$ tar -xf WSL2-Linux-Kernel-4.19.104-microsoft-standard.tar.gz
$ cd WSL2-Linux-Kernel-4.19.104-microsoft-standard/

$ cp Microsoft/config-wsl .config
$ make menuconfig

Because he captured many screenshots about this, it is better to follow original blog's instruction in this stage.

```
6. should check `.config` after setting, you should have followings.
```bash
KVM_GUEST=y
CONFIG_KVM=y
CONFIG_KVM_INTEL=m
CONFIG_VHOST_NET=y
CONFIG_VHOST=y
```
7. Let's build! It takes an hour.
```bash
make -j 8
```
8. After-build setting:
```bash
$ sudo make modules_install
$ cp arch/x86/boot/bzImage /mnt/c/somewhere/windows/can/access
```
9. Are you done? then make a `.wslconfig` on your windows user home directory (`C:\Users\<username>`).
Below is assuming your build image was copied to user home directory.
```
[wsl2]
nestedVirtualization=true
kernel=C:\\Users\\<username>\\bzImage
```
10. Let's get back to PowerShell(administrator).
```PowerShell
PS C:\WINDOWS\system32> wsl --shutdown Ubuntu
PS C:\WINDOWS\system32> wsl -d Ubuntu
```
11. You should check your Kernel(and CPU) support KVM in linux shell, after that.
```bash
$ uname -ar
Linux DESKTOP-E5KKPEB 4.19.104-microsoft-standard #1 SMP Sat Jun 20 14:08:58 KST 2020 x86_64 x86_64 x86_64 GNU/Linux
$ egrep -c "(svm|vmx)" /proc/cpuinfo
8
```
12. If you cannot get some positive number after last line, then you are a unfortunate guy! (I did. haha)
13. If you are not, you can go to next step. Otherwise follow below steps.
14. You need `WinDbg Preview` in Microsoft store.
![image](https://user-images.githubusercontent.com/769432/85217969-da952b80-b3d0-11ea-8d98-5ca9a89be0db.png)
15. Get script from here: https://gist.github.com/steffengy/62a0b5baa124830a4b0fe4334ccc2606 (Thank you Steffengy!)
get `run-wsl.bat` and `script.js` at same directory.
16. If you launch `run-wsl.bat`: then you will see WinDbg window and the WSL terminal after few seconds.
now you can try below again!
```bash
$ uname -ar
Linux DESKTOP-E5KKPEB 4.19.104-microsoft-standard #1 SMP Sat Jun 20 14:08:58 KST 2020 x86_64 x86_64 x86_64 GNU/Linux
$ grep "(svm|vmx)" /proc/cpuinfo
8
```

# set QEMU VM
1. You are using Windows as host, thus you may need to install X server on your Windows.
2. I installed vcXsrv by following this instruction. https://blog.nadekon.net/115
3. It is preferred to add below line in your `.bashrc`.
```
VETHER_IP=$(/usr/bin/grep nameserver /etc/resolv.conf 2> /dev/null | /usr/bin/tr -s ' ' | /usr/bin/cut -d' ' -f2)
export DISPLAY=$VETHER_IP:0.0
```
4. We will use KVM, thus should import KVM modules every running. Adding belows to `.bashrc` are preferred.
```bash
sudo modprobe kvm_intel
sudo chmod 666 /dev/kvm
```
5. Simply check setting is okay:
```
$ kvm-ok
INFO: /dev/kvm exists
KVM acceleration can be used(svm|vmx)
```

# create [macOS-Simple-KVM](https://github.com/foxlet/macOS-Simple-KVM.git) on QEMU
0. this steps are from https://computingforgeeks.com/how-to-run-macos-on-kvm-qemu/ . some are changed but note reference.
1. Let's install requirements and code.
```bash
$ sudo apt -y install qemu-kvm libvirt-daemon qemu-system qemu-utils python3 python3-pip bridge-utils virtinst libvirt-daemon-system virt-manager
$ git clone https://github.com/foxlet/macOS-Simple-KVM.git
```
2. Get install disk from server.
```bash
$ cd macOS-Simple-KVM
$ ./jumpstart.sh --catalina
```
3. make disk image.
```bash
$ qemu-img create -f qcow2 macOS.qcow2 60G # you may need enough disk size for some Applications like Xcode.
```
4. add below disk options to `./basic.sh`.
```bash
-drive id=SystemDisk,if=none,file=macOS.qcow2 \
-device ide-hd,bus=sata.4,drive=SystemDisk \
```
5. You will need to format disk, install macOS from internet, and setting your preferences with 'Clover'.
Details are described at the origin instruction.

# launch
1. when you run `.basic.sh`, you will see a window is opening via vcXsrv!
2. you may take 2 or more hours to install Catallina on your VM, after that, you are free to use MacOS VM.
3. Memory and Display settings might be uncomfortable, but you can find options easily from internet: keyword is `QEMU mem/vga setting`.

<img width="1920" alt="screenshot-macOS vm" src="https://user-images.githubusercontent.com/769432/85218275-b9820a00-b3d3-11ea-873d-32b5f47163fb.png">

# References
1. https://docs.microsoft.com/en-us/windows/wsl/install-win10
2. https://github.com/microsoft/WSL2-Linux-Kernel
3. https://boxofcables.dev/accelerated-kvm-guests-on-wsl-2/
4. https://gist.github.com/steffengy/62a0b5baa124830a4b0fe4334ccc2606
5. https://blog.nadekon.net/115
6. https://computingforgeeks.com/how-to-run-macos-on-kvm-qemu/=
7. https://wiki.gentoo.org/wiki/QEMU/Options
