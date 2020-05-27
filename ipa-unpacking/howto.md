---
title: "ipa 언패킹 가이드"
date: 2020-05-27 15:26:00 +0900
categories: ipa-unpacking
---

## Disclaimer:
1. 아래 소개하는 모든 내용은 iPhone app의 bundle/unbundle, encrypt/decrypt 에 대하여 학술적인 목적에서 다루는 글입니다.
2. 독자의 책임하에 활용해야 하며 이를 활용하는 어떤 동작도 기능을 보장하거나 결과를 책임지지 않습니다.

## 개요

### ipa unpacking
1. 아이폰 앱 번들을 읽을 수 있는 형태로 분해하는 것.
2. disassembled 된 코드의 flow와 keyword, Asset을 추출할 수 있음

### 절차
0. Rooting/Jail-breaking
1. ipa extraction without decryption
2. ipa unbundling
3. ipa extraction with decryption
4. disassembling

## Rooting/Jail-breaking

### Rooting이 필요한 이유
1. 기본적으로 아래 언급되는 거의 모든 동작이 root 권한을 필요로 한다. (허용되지 않는 앱 설치, 터미널에서 프로세스 실행 등..)
2. Rooting에 대해서는 iOS 버전, 기기 버전에 따라 다르고, 기본적으로 계약 위반이므로 여기에서는 다루지 않는다.
    * 어떻게든 탈옥하고 나면 아래를 진행할 수 있다.

## ipa extraction without decription

### ipa 파일을 추출하는 여러가지 방법
1. 공개된 ipa를 얻는 방법
    * 일반적으로 아이폰은 앱스토어를 벗어난 경로로 앱을 유통할 수 없으므로 (jail break 제외) 마찬가지로 상용앱은 공개된 ipa가 존재하지 않음.
2. Apple configurator를 이용하는 방법
    * 과거에는 iTunes를 통해서도 가능했다고 하는데, 지금은 Apple configurator를 통해 추출할 수 있다.
    * 간략히 적으면 iPhone configurator를 통해 앱을 업데이트(또는 설치)하는 중간에 생성되는 파일을 가져오는 것.
    * Configurator가 설치된 기기의 아래경로에 파일이 순간적으로 생겼다 사라진다. 이때를 놓치지 말자.
    * `/Lybrary/Group\ Containers/K36BKF7T3D.group.com.apple.configurator/Library/Caches/Assets/TemporaryItems/MobileApps`
    * 자세한 설명은 아래 블로그를 참조하자.
    * [Download IPA Files for the iOS Apps on Your iPhone, Justin Meyers, 2018.04.11](https://ios.gadgethacks.com/how-to/download-ipa-files-for-ios-apps-your-iphone-0184056/)
3. 루팅된 iPhone 단말의 설치위치에서 가져오는 법
    * 루팅된 기기에 ssh/sftp로 접근하여 가져오는 방법. 이 작업을 하는 시점에 rooting이 되어있다는 전제가 조금 비현실적이지만, 아래 블로그에 자세히 소개되어 있음.
    * [Extracting the IPA File and Local Data Storage of an iOS Application, Lucideus, 2018.12.26](https://medium.com/@lucideus/extracting-the-ipa-file-and-local-data-storage-of-an-ios-application-be637745624d)

## ipa unbundling

### 어떻게?
1. ipa 파일은 [LZFSE](https://github.com/lzfse/lzfse)`Lempel-Ziv` style data 압축 알고리즘으로 압축되어 있다.
2. 먼저 [unzip-lzfse](https://github.com/sskaje/unzip-lzfse)를 설치하고
3. `unzip {some app bundle}.ipa` 명령어를 실행하자.
4. 압축결과 패키지가 생성되고 이 패키지를 열어보면 각종 asset들과 설정파일들이 읽을수 있는 형태로 저장된 것을 볼 수 있다.
5. 다만 여기에서 나오는 Binary는 아직 읽을 수 없다.

## ipa extraction with decryption

### Decryption이 필요한 이유
1. iOS 앱은 Apple의 DRM인 `FairPlay`로 암호화되어 있다.
    * 따라서 그냥 번들을 풀어 바이너리를 disassemble해도 읽을 수 없다.
2. 이 `FairPlay` DRM은 기기 CPU에서 실행 되기 전까지는 복호화되지 않으므로 바이너리를 읽어서는 해독하는 것은 불가능하다.
3. 메모리에 on-load 된 프로그램을 읽어야 하므로 아래부터는 단말기가 루팅되어 있다고 전제한다.

### 어떻게?
1. `otool`로 암호화 여부 확인하기
    * https://stackoverflow.com/questions/7038143/executable-encryption-check-anti-piracy-measure
    * `otool -arch armv7 -l YourAppName | grep crypt`
    * 아키텍쳐를 잘못적으면 작동하지 않음
2. 추출+복호화
    * gdb를 이용하는 방법 (https://reverseengineering.stackexchange.com/a/1601)
        - 단말에 `otool`, `gdb`, `Idid`를 설치한다.
        - ipa를 설치하고 실행 직후 suspend한다.
        - gdb를 이용해 프로세스를 덤프 (`gdb -p {process id}`, `dump output.bin 0x2000 0xNNNN`)
        - 원본에서 `0x1000` 바이트를 긁어 새파일을 만들고, 덤프된 파일을 뒤에 이어붙인다.
        - `Idid`로 새롭게 sign하면 끝
        - (바이트 편집이 부담스러워 실험하지 못했음)
    * 맥이 연결된 상태에서 복호화된 상태의 앱을 그대로 저장하는 방법 
        - 이를 도와주는 각종 툴이 있다. (`Clutch`, `dumpdecrypted`, `bfinject`, `Frida`)
        - 아래 링크를 참조하자. 참고로 다 시도해봤지만 안됐다.
        - [Removing Apple DRM via CLI, The Mobile Security Guys, 2020.05.15](https://medium.com/@mobsecguys/removing-apple-drm-via-cli-f5c0d75ba6eb)
    * CrackerXI
        - 유일하게 성공한 방법이다.
        - 루팅된 단말기에 CrackerXI를 설치하자. 공식 repo: http://cydia.iphonecake.com/ 비공식 중국 repo: https://apt.cydia.love/
        - 설치된 상태에서 앱을 재설치하면 목록에 뜬다.
        - 앱을 선택하고 Full IPA 또는 Binary를 뽑아내면 된다.
        - 앱은 iPhone에 저장되므로 이를 sftp 등의 방법으로 꺼내면 된다.

## disassembling

### disassembling이 필요한 이유
  bytecode를 눈으로 읽을 수 있다면 굳이 안해도 된다.
    
### 어떻게?
1. 아이폰 앱은 LLVM 이라는 컴파일러 아키텍처로 구성되어 있으므로, 이를 지원하는 disassembler를 써야 한다.
2. 도구
    * [공식 LLVM disassembler](https://llvm.org/docs/CommandGuide/llvm-dis.html): assembly에 익숙하면 이정도로 괜찮을지도..
    * [사실상 유일한 대안 Hopper Disassembler](https://www.hopperapp.com/)
    * 아이폰 앱 disassemble 시 그냥 hopper를 쓰면 된다. 호출관계와 pseudo code까지 그려주는 훌륭한 프로그램이다.
3. 위에서 얻어온 decrypt 된 ipa를 hopper에서 열어주자. free 버전이라도 30분 시간제한을 제외하면 쓰는데 거의 지장 없다.
4. 여기까지 했다면 이제 정면에 assembly 코드와 좌측에 symbol 리스트를 보고 있을 것이다. 필요한 정보를 찾아보자.
    
    
    
