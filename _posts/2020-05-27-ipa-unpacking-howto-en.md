---
title: "ipa unpacking guide"
date: 2020-05-27 15:26:00 +0900
categories: ipa-unpacking
---

## Disclaimer:
1. Below all of this article is discussing about iPhone app's bundling/unbundling, encrypting/decripting only in academic purposes.
2. Readers' can apply this on their work with their own responsibility and I do not guarantee any result including ethical/legal things.

## Summary

### ipa unpacking
1. To destruct iPhone application bundle (IPA) to make them readable
2. Can extract assets and disassembled codes/keywords-that meaans you can guess how the app works specifically.

### Steps
0. Rooting/Jail-breaking
1. ipa extraction without decryption
2. ipa unbundling
3. ipa extraction with decryption
4. disassembling

## Rooting/Jail-breaking

### Why do I need 'Rooting'
1. Basically all actions bellow require root previledges on iPhone (Install apps that not allowed in normal process, run a process on terminal, or etc.) 
2. Rooting a device is varying by iOS version / device model, and it is not compatible with EULA thus we do not talk details in here.
    * Anyway, if you have a rooted device then can go on!

## ipa extraction without decryption

### Few ways to extract an ipa file
1. Getting pulbic ipa
    * In general iPhone is not allowed to circulate apps while avoiding appstore, thus almost of commercial apps have no public ipa file.
    * But you may have some ipa with some luck!
2. Using Apple configurator
    * Previously it was able with iTunes but nowadays we can extract apps with Apple configurator.
    * With short words, we get intermediately created ipa file on temporary directory when updating/installing apps using iPhone configurator.
    * Bellow route that Configurator is installed the file will be created and be removed shortly. Don't miss timing.
    * `/Lybrary/Group\ Containers/K36BKF7T3D.group.com.apple.configurator/Library/Caches/Assets/TemporaryItems/MobileApps`
    * You can refer this article for this step.
    * [Download IPA Files for the iOS Apps on Your iPhone, Justin Meyers, 2018.04.11](https://ios.gadgethacks.com/how-to/download-ipa-files-for-ios-apps-your-iphone-0184056/)
3. Get from Rooted iPhone device 
    * You should access rooted device via ssh/sftp. Maybe it is not realistic that you have a rooted device during this step, but you can read bellow article.
    * [Extracting the IPA File and Local Data Storage of an iOS Application, Lucideus, 2018.12.26](https://medium.com/@lucideus/extracting-the-ipa-file-and-local-data-storage-of-an-ios-application-be637745624d)

## ipa unbundling

### How?
1. Ipa files are zipped with [LZFSE](https://github.com/lzfse/lzfse)`Lempel-Ziv` style data algorithm.
2. First you need to install [unzip-lzfse](https://github.com/sskaje/unzip-lzfse)
3. And then run `unzip \{some app bundle}.ipa`  .
4. As result you may have a package, and when open it, we will see many assets and configure files in readable format.
5. But we cannot read binary still.

## ipa extraction with decryption

### Reasons why we need `Decryption`
1. iOS Apps are encrypted with Apple's DRM `FairPlay`.
    * That means we cannot read binaries even we disassemble an ipa.
2. The `FairPlay` DRM is not decrypted before it is ran on a CPU: There is no way to decrypting binary just by reading that.
3. We need to read a program that is on-loaded on memory, so we cannot go further without rooting.

### How?
1. Check whether a binary is encrypted with `otool`
    * https://stackoverflow.com/questions/7038143/executable-encryption-check-anti-piracy-measure
    * `otool -arch armv7 -l YourAppName | grep crypt`
    * Should set architecture correctly.
2. Extracting + Decrypting
    * Using gdb (https://reverseengineering.stackexchange.com/a/1601)
        - Install `otool`, `gdb`, `Idid` on device.
        - Install ipa and run and suspend it in few seconds.
        - Dump process using gdb (`gdb -p {process id}`, `dump output.bin 0x2000 0xNNNN`)
        - Pick off `0x1000` bytes from original binary, and join dumped binary after that.
        - Sign newly this binary with `Idid`
        - (It burden lot thus I couldn't have real experiment.)
    * Saving un-encrypted app while a Mac is connected 
        - There are tools which helps this process. (`Clutch`, `dumpdecrypted`, `bfinject`, `Frida`)
        - You can follow detail method with bellow article. FYI I couldn't succeed any of it.
        - [Removing Apple DRM via CLI, The Mobile Security Guys, 2020.05.15](https://medium.com/@mobsecguys/removing-apple-drm-via-cli-f5c0d75ba6eb)
    * CrackerXI
        - The only successful method of mine.
        - Install CrackerXI on rooted device. Official repo: http://cydia.iphonecake.com/ Unoffficial(Chinese) repo: https://apt.cydia.love/
          - I installed it with Cydia.
          - Why I have an unofficial repository? because in sometimes/on someplaces official repo is not accessible.
        - After that, you may run again the app you want to unpack and then will see on list of CrackerXI.
        - Then you can choose to extract in Full IPA or Binary only.
        - The app will be saved on iPhone and you may use sftp to pick it out.

## disassembling

### Reason why we need disassembling
  We do not need diassembling if you can read bytecode on your raw eyes..
    
### How?
1. iPhone apps are structured with a compiler architecture named LLVM, so we need a disassembler which is support it.
2. Tools
    * [Official LLVM disassembler](https://llvm.org/docs/CommandGuide/llvm-dis.html): It may be a good choice if you ar familiar with assembly..
    * [The only way, de facto: Hopper Disassembler](https://www.hopperapp.com/)
    * You can just use hopper to disassemble iPhone app. It draws call trees and pseudo codes.
3. Open decrypted ipa in hopper. There is enough functions for free version except time limiting 30 minutes.
4. You will see assembly codes in front and symbol list on left side now! Let's find some informations what you need.

## Code Obfuscation
1. Surely there are lot of studies to prevent such hacking.
2. If you interested in- read this. [ORK – Code obfuscation/compiling tool , Sangmin Chung, 2020.03.06](https://engineering.linecorp.com/ko/blog/code-obfuscation-compiler-tool-ork-1/)
    
    
