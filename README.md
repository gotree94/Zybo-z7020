# Digilent Zybo Z7-20 PetaLinux 완벽 가이드 (최종판)

**Root 로그인 문제 완전 해결 포함**

---

## 목차

1. [VirtualBox Ubuntu 22.04.5 설치](#1-virtualbox-ubuntu-22045-설치)
2. [Ubuntu 시스템 준비](#2-ubuntu-시스템-준비)
3. [PetaLinux 2022.2 설치](#3-petalinux-20222-설치)
4. [Zybo Z7-20 프로젝트 생성](#4-zybo-z7-20-프로젝트-생성)
5. [Root 로그인 설정 (중요!)](#5-root-로그인-설정-중요)
6. [PetaLinux 빌드](#6-petalinux-빌드)
7. [SD 카드 이미지 생성](#7-sd-카드-이미지-생성)
8. [Windows에서 SD 카드 굽기](#8-windows에서-sd-카드-굽기)
9. [Zybo Z7-20 부팅 및 로그인](#9-zybo-z7-20-부팅-및-로그인)
10. [로그인 문제 해결](#10-로그인-문제-해결)
11. [트러블슈팅](#11-트러블슈팅)
12. [체크리스트](#12-체크리스트)
13. [자주 사용하는 명령어](#13-자주-사용하는-명령어)
14. [FAQ](#14-faq)
15. [참고 자료](#15-참고-자료)
16. [부록](#16-부록)

---

## 1. VirtualBox Ubuntu 22.04.5 설치

### 1.1 VirtualBox 가상머신 생성

**시스템 사양**
```
이름: Zybo-PetaLinux
타입: Linux
버전: Ubuntu (64-bit)

메모리: 16GB (최소 8GB)
CPU: 8 코어 (최소 4 코어)
디스크: 200GB (최소 150GB)
```

**설정**
- 시스템 → 프로세서
  - ✅ PAE/NX 활성화
  - ✅ VT-x/AMD-V 활성화
  
- 공유 폴더
  - 이름: `share`
  - 경로: `C:\share`
  - 마운트: `/mnt/share`
  - ✅ 자동 마운트
  - ✅ 영구적

### 1.2 Ubuntu 설치

1. ISO 마운트: `ubuntu-22.04.5-desktop-amd64.iso`
2. 설치 타입: Normal installation
3. 사용자 생성: 원하는 이름/패스워드
4. 설치 완료 후 재부팅

### 1.3 Guest Additions 설치

```bash
sudo apt update
sudo apt install -y build-essential dkms linux-headers-$(uname -r)

# VirtualBox 메뉴: Devices → Insert Guest Additions CD
cd /media/$USER/VBox*
sudo ./VBoxLinuxAdditions.run

sudo reboot
```

### 1.4 공유 폴더 설정

```bash
sudo mkdir -p /mnt/share
sudo usermod -aG vboxsf $USER
echo "share /mnt/share vboxsf defaults,uid=$(id -u),gid=$(id -g) 0 0" | sudo tee -a /etc/fstab

sudo reboot

# 확인
ls -la /mnt/share
```

---

## 2. Ubuntu 시스템 준비

### 2.1 시스템 업데이트

```bash
sudo apt update
sudo apt upgrade -y
```

### 2.2 필수 패키지 설치

```bash
# 32비트 지원
sudo dpkg --add-architecture i386
sudo apt update

# PetaLinux 필수 패키지
sudo apt install -y \
    build-essential gcc-multilib g++-multilib gawk wget git \
    diffstat unzip texinfo chrpath socat cpio python3 \
    python3-pip python3-pexpect xz-utils debianutils \
    iputils-ping python3-git python3-jinja2 libegl1-mesa \
    libsdl1.2-dev pylint xterm rsync curl \
    libncurses5-dev libncursesw5-dev libssl-dev \
    flex bison libselinux1 gnupg zlib1g-dev \
    libtool autoconf automake net-tools screen pax gzip vim \
    iproute2 locales libncurses5 libtinfo5

sudo apt install -y \
    libncurses5:i386 libc6:i386 libstdc++6:i386 lib32z1 zlib1g:i386

sudo locale-gen en_US.UTF-8
echo "dash dash/sh boolean false" | sudo debconf-set-selections
sudo dpkg-reconfigure -f noninteractive dash

sudo mkdir -p /mnt/share
sudo usermod -aG vboxsf $USER

echo "설치 완료! 재부팅하세요: sudo reboot"
```

**PetaLinux 빌드 스크립트 (build_petalinux.sh):**

```bash
#!/bin/bash
PROJECT_DIR="$HOME/projects/myproject"

source ~/petalinux/2022.2/settings.sh
cd $PROJECT_DIR

echo "빌드 시작..."
petalinux-build

echo "BOOT.BIN 생성..."
petalinux-package --boot \
    --fsbl images/linux/zynq_fsbl.elf \
    --fpga images/linux/design_1_wrapper.bit \
    --u-boot images/linux/u-boot.elf \
    --force

echo "WIC 이미지 생성..."
petalinux-package --wic \
    --bootfiles "BOOT.BIN image.ub boot.scr" \
    --images-dir images/linux/

echo "공유 폴더로 복사..."
cp images/linux/petalinux-sdimage.wic /mnt/share/
cp images/linux/BOOT.BIN /mnt/share/
cp images/linux/image.ub /mnt/share/
cp images/linux/boot.scr /mnt/share/

sync
echo "완료! C:\share에서 파일 확인하세요."
```

### 16.2 빠른 참조 카드

**PetaLinux 주요 명령어:**

| 명령어 | 설명 |
|--------|------|
| `source ~/petalinux/2022.2/settings.sh` | 환경 활성화 |
| `petalinux-create -t project --template zynq -n <이름>` | 프로젝트 생성 |
| `petalinux-config --get-hw-description=<경로>` | 하드웨어 설정 |
| `petalinux-config -c rootfs` | Rootfs 설정 |
| `petalinux-build` | 전체 빌드 |
| `petalinux-build -x clean` | 클린 빌드 |
| `petalinux-package --boot` | BOOT.BIN 생성 |
| `petalinux-package --wic` | WIC 이미지 생성 |

**Zybo Z7-20 하드웨어 정보:**

| 항목 | 사양 |
|------|------|
| FPGA | Zynq-7020 (XC7Z020-1CLG400C) |
| CPU | Dual-core ARM Cortex-A9 @ 650MHz |
| 메모리 | 1GB DDR3 |
| 저장소 | SD 카드 슬롯 |
| 이더넷 | 10/100/1000 Mbps |
| USB | USB 2.0 OTG, USB-UART |
| UART | 115200 8N1 |

**시리얼 콘솔 설정:**

| 설정 | 값 |
|------|-----|
| Baud Rate | 115200 |
| Data Bits | 8 |
| Parity | None |
| Stop Bits | 1 |
| Flow Control | None |

### 16.3 에러 코드 및 해결

**일반적인 빌드 에러:**

| 에러 | 원인 | 해결 |
|------|------|------|
| Virtual memory exhausted | 메모리 부족 | 스왑 파일 생성 |
| No space left on device | 디스크 부족 | 빌드 캐시 정리 |
| Task do_compile failed | 패키지 빌드 실패 | 로그 확인 후 재빌드 |
| Unable to find XSA | XSA 파일 없음 | 경로 확인 |
| License error | 라이센스 미동의 | 인스톨러 재실행 |

**부팅 에러:**

| 증상 | 원인 | 해결 |
|------|------|------|
| U-Boot에서 멈춤 | boot.scr 문제 | boot.scr 재생성 |
| Kernel panic | rootfs 마운트 실패 | rootfs 파티션 확인 |
| SD 카드 인식 안됨 | SD 카드 문제 | 다른 SD 카드 시도 |
| 시리얼 출력 없음 | COM 포트 설정 | COM 포트 및 설정 확인 |

**로그인 에러:**

| 증상 | 원인 | 해결 |
|------|------|------|
| Login incorrect | 로그인 설정 누락 | rootfs 재설정 |
| 자동 로그인 안됨 | systemd 설정 | 완전 클린 빌드 |
| SSH 접속 안됨 | openssh 미설치 | rootfs에 openssh 추가 |

### 16.4 추가 리소스

**온라인 도구:**
- SD Card Formatter: https://www.sdcard.org/downloads/formatter/
- Win32 Disk Imager: https://sourceforge.net/projects/win32diskimager/
- WinSCP: https://winscp.net/
- Visual Studio Code: https://code.visualstudio.com/

**Yocto/OpenEmbedded:**
- Yocto Project: https://www.yoctoproject.org/
- OpenEmbedded: https://www.openembedded.org/
- BitBake User Manual: https://docs.yoctoproject.org/bitbake/

**유용한 GitHub:**
- meta-xilinx: https://github.com/Xilinx/meta-xilinx
- Digilent Zybo-Z7: https://github.com/Digilent/Zybo-Z7
- PetaLinux Examples: https://github.com/topics/petalinux

---

## 최종 요약

### 전체 프로세스

```
1. VirtualBox + Ubuntu 22.04.5 설치
2. 공유 폴더 설정 (/mnt/share)
3. 필수 패키지 설치
4. PetaLinux 2022.2 설치
5. Zybo Z7-20 프로젝트 생성
6. ⭐ Root 로그인 설정 (필수!)
7. PetaLinux 빌드 (1-3시간)
8. BOOT.BIN 생성
9. WIC SD 이미지 생성
10. balenaEtcher로 SD 카드 굽기
11. Zybo Z7-20 부팅
12. Root 로그인 (자동 또는 Enter)
```

### 핵심 명령어

```bash
# PetaLinux 환경
source ~/petalinux/2022.2/settings.sh

# 프로젝트 생성
petalinux-create -t project --template zynq -n myproject
cd myproject

# 하드웨어 설정
petalinux-config --get-hw-description=~/projects/

# ⭐ Root 로그인 설정 (반드시!)
petalinux-config -c rootfs
# Image Features --->
#     [*] debug-tweaks
#     [*] allow-empty-password
#     [*] allow-root-login
#     [*] empty-root-password
#     [*] serial-autologin-root

# 빌드
petalinux-build

# BOOT.BIN 생성
petalinux-package --boot --fsbl images/linux/zynq_fsbl.elf \
    --fpga images/linux/design_1_wrapper.bit \
    --u-boot images/linux/u-boot.elf --force

# WIC 이미지 생성
petalinux-package --wic --bootfiles "BOOT.BIN image.ub boot.scr"

# Windows로 복사
cp images/linux/petalinux-sdimage.wic /mnt/share/
```

### 예상 소요 시간

| 작업 | 시간 |
|------|------|
| VirtualBox + Ubuntu 설치 | 30-60분 |
| 패키지 설치 | 10-20분 |
| PetaLinux 설치 | 10-30분 |
| 프로젝트 생성 및 설정 | 10-20분 |
| 빌드 (첫 빌드) | 1-3시간 |
| 이미지 생성 및 SD 카드 | 10-20분 |
| **총 소요 시간** | **약 2-5시간** |

### 중요 사항

✅ **반드시 해야 할 것**
1. Rootfs 로그인 설정 활성화
   - debug-tweaks
   - allow-empty-password
   - serial-autologin-root
2. 빌드 전 설정 확인
3. 정기적인 백업

❌ **하지 말아야 할 것**
1. Rootfs 로그인 설정 누락
2. Ubuntu 버전 다운그레이드
3. 빌드 중 강제 종료

### Warning 메시지 요약

| Warning | 영향 | 조치 |
|---------|------|------|
| Ubuntu 22.04 미검증 | 없음 | 무시 |
| glibc 버전 불일치 | 없음 | 자동 처리 |
| TFTP 복사 실패 | 없음 (SD 부팅 시) | 선택적 |

---

## 성공을 위한 팁

**개발자의 습관**
1. 📖 문서를 읽는다 - RTFM
2. 🐣 작게 시작한다 - Hello World부터
3. 💾 자주 백업한다 - Git 사용
4. 📝 로그를 확인한다 - 추측하지 말고 확인
5. 👥 커뮤니티를 활용한다 - 혼자 고민하지 말 것
6. ⏰ 인내심을 갖는다 - 임베디드는 시간이 걸림
7. 🔬 실험을 즐긴다 - 실패는 학습의 기회

**막힐 때 시도할 것**
1. 로그 파일 확인 (`dmesg`, `build/build.log`)
2. 커뮤니티 검색 (Xilinx Forums, Stack Overflow)
3. 클린 빌드 (`petalinux-build -x clean`)
4. 이 가이드의 트러블슈팅 섹션 참고

**다음 단계**

**초급 프로젝트**
- LED 제어 애플리케이션
- 버튼 입력 처리
- GPIO 인터페이스
- UART 통신

**중급 프로젝트**
- 이더넷 네트워크 서버
- 웹 서버 구축
- 카메라 인터페이스
- 커스텀 디바이스 드라이버

**고급 프로젝트**
- FPGA 가속기 연동
- 실시간 비디오 처리
- 머신러닝 추론
- 상용 제품 개발

---

## 마무리

### 축하합니다! 🎉

이 가이드를 완료하신 분들께 축하드립니다!

여러분은 이제:
- ✅ Ubuntu 22.04 개발 환경 구축 완료
- ✅ PetaLinux 2022.2 설치 및 설정 완료
- ✅ Zynq-7000 프로젝트 생성 완료
- ✅ Root 로그인 문제 해결 완료
- ✅ SD 카드 부팅 이미지 생성 완료
- ✅ 실제 하드웨어에서 Linux 부팅 성공!

**여러분은 이제 Zynq 개발자입니다!**

### 계속 학습하기

**추천 학습 순서:**
1. 기본 Linux 명령어 마스터 (1주)
2. Device Tree 이해 (2주)
3. 커널 모듈 개발 (3주)
4. Yocto/BitBake 심화 (4주)
5. FPGA-ARM 통합 (진행 중)

**유용한 자료:**
- The Zynq Book (무료): http://www.zynqbook.com/
- AMD 공식 문서: https://docs.amd.com/
- Digilent 리소스: https://digilent.com/reference/
- 커뮤니티 포럼: Stack Overflow, Xilinx Forums

### 커뮤니티 기여

**여러분의 경험을 공유하세요:**
- GitHub에 프로젝트 공개
- 블로그에 학습 내용 정리
- 포럼에서 다른 사람 돕기
- 한국어 자료 만들기

### 지원

**문제 해결이 필요하다면:**
1. 이 가이드의 트러블슈팅 섹션
2. AMD Support: https://support.amd.com/
3. Digilent Forum: https://forum.digilent.com/
4. Stack Overflow: [petalinux], [zynq] 태그

### 격려의 메시지

> "The journey of a thousand miles begins with a single step."
> 
> 천 리 길도 한 걸음부터 시작됩니다.

**포기하지 마세요!**
- 에러는 정상입니다
- 실패는 배움의 과정입니다
- 모든 전문가도 초보자였습니다
- 한 걸음씩 나아가면 됩니다

**다음에 만들 수 있는 것들:**
- 🤖 로봇 제어 시스템
- 📹 실시간 비디오 처리
- 🌐 IoT 게이트웨이
- 🎮 임베디드 게임 콘솔
- 📡 SDR (Software Defined Radio)
- 🔬 과학 측정 장비
- 🏭 산업용 제어 시스템

**여러분의 상상력이 한계입니다!**

---

## 🎓 완료 인증

```
┌─────────────────────────────────────────┐
│  PetaLinux on Zybo Z7-20               │
│  Complete Master Certificate            │
│                                         │
│  This certifies that                    │
│  YOU                                    │
│  has successfully completed             │
│                                         │
│  ✓ Ubuntu 22.04 Setup                  │
│  ✓ PetaLinux 2022.2 Installation       │
│  ✓ Zynq Project Creation               │
│  ✓ Root Login Configuration            │
│  ✓ System Build & Deployment           │
│  ✓ SD Card Boot Success                │
│                                         │
│  Date: 2025-09-30                       │
│  Level: Embedded Linux Developer        │
└─────────────────────────────────────────┘
```

---

**Happy Hacking! 🛠️**

**May the Source be with you! 💻**

---

## 문서 정보

**제목:** Digilent Zybo Z7-20 PetaLinux 완벽 가이드  
**부제:** Root 로그인 문제 완전 해결 포함  
**버전:** 2.0 (최종판)  
**작성일:** 2025년 9월 30일  
**최종 업데이트:** 2025년 9월 30일  

**대상:**
- 하드웨어: Digilent Zybo Z7-20 (Zynq-7020)
- PetaLinux: 2022.2
- 호스트 OS: Ubuntu 22.04.5 LTS (VirtualBox)
- 공유 폴더: /mnt/share

**변경 이력:**
- v2.0 (2025-09-30): 최종 완결판, 깔끔하게 재정리
- v1.0 (2025-09-29): 초기 버전

**라이센스:** CC BY-SA 4.0  
*자유롭게 공유 및 수정 가능*

**기여:**
- 문서 작성: Claude (Anthropic AI)
- 검증: 실제 Zybo Z7-20 하드웨어 테스트

---

**이 가이드가 도움이 되었다면 다른 개발자들과 공유해주세요!**

**질문이나 피드백은 언제든지 환영합니다.**

---

```
 ____       _        _     _                  
|  _ \ ___| |_ __ _| |   (_)_ __  _   ___  __
| |_) / _ \ __/ _` | |   | | '_ \| | | \ \/ /
|  __/  __/ || (_| | |___| | | | | |_| |>  < 
|_|   \___|\__\__,_|_____|_|_| |_|\__,_/_/\_\

  Zybo Z7-20 Complete Guide
  End of Document - Thank you!
```

**END OF DOCUMENT**

© 2025 Zybo Z7-20 PetaLinux Guide  
All trademarks are property of their respective owners.\
    python3-pip python3-pexpect xz-utils debianutils \
    iputils-ping python3-git python3-jinja2 libegl1-mesa \
    libsdl1.2-dev pylint xterm rsync curl \
    libncurses5-dev libncursesw5-dev libssl-dev \
    flex bison libselinux1 gnupg zlib1g-dev \
    libtool autoconf automake net-tools screen pax gzip vim \
    iproute2 locales libncurses5 libtinfo5

# 32비트 라이브러리
sudo apt install -y \
    libncurses5:i386 libc6:i386 libstdc++6:i386 lib32z1 zlib1g:i386

# Locale 설정
sudo locale-gen en_US.UTF-8

# Dash를 Bash로 변경
sudo dpkg-reconfigure dash  # "No" 선택
```

---

## 3. PetaLinux 2022.2 설치

### 3.1 인스톨러 준비

```bash
# Windows에서 C:\share로 인스톨러 복사 후
mkdir -p ~/petalinux_work
cp /mnt/share/petalinux-v2022.2-10141622-installer.run ~/petalinux_work/
chmod +x ~/petalinux_work/petalinux-v2022.2-10141622-installer.run
```

### 3.2 PetaLinux 설치

```bash
mkdir -p ~/petalinux/2022.2
cd ~/petalinux_work
./petalinux-v2022.2-10141622-installer.run -d ~/petalinux/2022.2

# 라이센스 동의: y
# 설치 시간: 약 10-30분
```

### 3.3 환경 설정

```bash
# PetaLinux 환경 활성화
source ~/petalinux/2022.2/settings.sh

# 확인
echo $PETALINUX

# 영구 설정 (권장)
echo "source ~/petalinux/2022.2/settings.sh" >> ~/.bashrc
source ~/.bashrc
```

---

## 4. Zybo Z7-20 프로젝트 생성

### 4.1 XSA 파일 준비

```bash
# Windows에서 C:\share로 design_1_wrapper.xsa 복사 후
mkdir -p ~/projects
cp /mnt/share/design_1_wrapper.xsa ~/projects/

# XSA 내용 확인
unzip -l ~/projects/design_1_wrapper.xsa
```

### 4.2 프로젝트 생성

```bash
cd ~/projects
source ~/petalinux/2022.2/settings.sh

petalinux-create --type project --template zynq --name myproject
cd myproject
```

### 4.3 하드웨어 설정

```bash
petalinux-config --get-hw-description=~/projects/
```

**설정 메뉴:**

```
Image Packaging Configuration --->
    Root filesystem type --->
        (X) SD card
    [ ] Copy final images to tftpboot

Yocto Settings --->
    [*] Enable auto resize SD card root filesystem

Subsystem AUTO Hardware Settings --->
    Serial Settings --->
        Primary stdin/stdout --->
            (X) ps7_uart_1
    Ethernet Settings --->
        Primary Ethernet --->
            (X) ps7_ethernet_0
    SD/SDIO Settings --->
        Primary SD/SDIO --->
            (X) ps7_sd_0
```

저장: `Save` → `Exit`

---

## 5. Root 로그인 설정 (중요!)

### 5.1 문제 이해

**기본 상태의 문제:**
```
myproject login: root
Password: (무엇을 입력해도)
Login incorrect
```

**원인:**
- PetaLinux는 보안상 빈 패스워드 로그인 차단
- 하지만 root 패스워드가 설정되지 않음
- 결과: 로그인 불가능

### 5.2 해결 방법 - Rootfs 설정 (필수!)

```bash
cd ~/projects/myproject
petalinux-config -c rootfs
```

**⭐ 반드시 다음 항목들을 활성화:**

```
Image Features --->
    [*] debug-tweaks                  ← 필수!
    [*] allow-empty-password          ← 필수!
    [*] allow-root-login              ← 필수!
    [*] empty-root-password           ← 필수!
    [*] serial-autologin-root         ← 권장 (자동 로그인)
```

**추가 패키지 (선택사항):**

```
Filesystem Packages --->
    admin --->
        [*] sudo
    console/utils --->
        [*] vim
        [*] nano
    network --->
        [*] openssh
        [*] openssh-sshd
```

저장: `Save` → `Exit`

### 5.3 설정 확인

```bash
# 설정이 제대로 되었는지 확인
cat ~/projects/myproject/project-spec/configs/rootfs_config | grep -i "debug\|empty\|autologin"

# 다음 항목들이 있어야 함:
# CONFIG_debug-tweaks=y
# CONFIG_allow-empty-password=y
# CONFIG_empty-root-password=y
# CONFIG_serial-autologin-root=y
```

---

## 6. PetaLinux 빌드

### 6.1 전체 빌드

```bash
cd ~/projects/myproject
source ~/petalinux/2022.2/settings.sh

petalinux-build
```

**빌드 시간:**
- 첫 빌드: 1-3시간
- 증분 빌드: 10-30분

**빌드 성공 메시지:**
```
NOTE: Tasks Summary: Attempted 5162 tasks of which 1350 didn't need to be rerun and all succeeded.
Summary: There were 2 WARNING messages shown.
[INFO] Successfully built project
```

**Warning 메시지 (무시 가능):**
```
WARNING: Host distribution "ubuntu-22.04" has not been validated...
WARNING: Your host glibc version (2.35) is newer than that in uninative (2.34)...
INFO: Failed to copy built images to tftp dir: /tftpboot
```
- ✅ 이 경고들은 무시해도 됩니다
- ✅ 빌드가 성공했으면 문제 없습니다

### 6.2 부트 이미지 생성

```bash
petalinux-package --boot \
    --fsbl images/linux/zynq_fsbl.elf \
    --fpga images/linux/design_1_wrapper.bit \
    --u-boot images/linux/u-boot.elf \
    --force
```

**생성 파일:** `images/linux/BOOT.BIN`

---

## 7. SD 카드 이미지 생성

### 7.1 WIC 이미지 생성

```bash
cd ~/projects/myproject

petalinux-package --wic \
    --bootfiles "BOOT.BIN image.ub boot.scr" \
    --images-dir images/linux/
```

**생성 파일:** `images/linux/petalinux-sdimage.wic`

### 7.2 Windows로 복사

```bash
cd ~/projects/myproject/images/linux/

# WIC 이미지 복사
cp petalinux-sdimage.wic /mnt/share/

# 개별 파일도 백업
mkdir -p /mnt/share/zybo_boot
cp BOOT.BIN image.ub boot.scr rootfs.tar.gz /mnt/share/zybo_boot/

sync
```

**Windows에서 확인:**
- `C:\share\petalinux-sdimage.wic`
- `C:\share\zybo_boot\*`

---

## 8. Windows에서 SD 카드 굽기

### 8.1 준비물

- SD 카드: 최소 4GB (권장 8GB+)
- SD 카드 리더기
- balenaEtcher 2.1.2

### 8.2 balenaEtcher 사용

1. **balenaEtcher 실행**

2. **Flash from file** 클릭
   - `C:\share\petalinux-sdimage.wic` 선택

3. **Select target** 클릭
   - SD 카드 선택 (⚠️ 올바른 드라이브 확인!)

4. **Flash!** 클릭
   - 진행 (약 5-10분)

5. **완료 후 안전하게 제거**

---

## 9. Zybo Z7-20 부팅 및 로그인

### 9.1 하드웨어 설정

**부트 점퍼 (JP5):**
```
SD 카드 부팅:
JP5: [  ] [  ]
     [SD] [  ]
```

**연결:**
1. SD 카드 삽입
2. USB-UART 케이블 연결 (J14)
3. 이더넷 연결 (선택)
4. 전원 OFF

### 9.2 시리얼 콘솔 설정 (Windows)

**FTDI 드라이버 설치:**
- https://ftdichip.com/drivers/vcp-drivers/

**PuTTY 설정:**
```
Connection type: Serial
Serial line: COM3 (장치 관리자에서 확인)
Speed: 115200

Connection → Serial:
  Speed: 115200
  Data bits: 8
  Stop bits: 1
  Parity: None
  Flow control: None
```

### 9.3 부팅

1. PuTTY 연결
2. 전원 ON (SW0)
3. 부팅 메시지 확인

```
Xilinx Zynq First Stage Boot Loader
Release 2022.2

U-Boot 2022.01

Starting kernel ...

[    0.000000] Booting Linux on physical CPU 0x0
[    0.000000] Linux version 5.15.36-xilinx-v2022.2

PetaLinux 2022.2 myproject /dev/ttyPS0
```

### 9.4 로그인

**자동 로그인 (serial-autologin-root 활성화 시):**
```
myproject login: root (automatic login)
root@myproject:~#
```

**수동 로그인 (자동 로그인 비활성화 시):**
```
myproject login: root
Password: (그냥 Enter)
root@myproject:~#
```

### 9.5 시스템 확인

```bash
# 호스트명
hostname

# 시스템 정보
uname -a

# PetaLinux 버전
cat /etc/os-release

# 네트워크
ifconfig

# DHCP
udhcpc -i eth0
```

---

## 10. 로그인 문제 해결

### 10.1 "Login incorrect" 오류

**증상:**
```
myproject login: root
Password:
Login incorrect
```

**해결 방법 A - 재빌드 (권장):**

```bash
cd ~/projects/myproject
source ~/petalinux/2022.2/settings.sh

petalinux-config -c rootfs
# Image Features --->
#     [*] debug-tweaks
#     [*] allow-empty-password
#     [*] allow-root-login
#     [*] empty-root-password
#     [*] serial-autologin-root

petalinux-build -c rootfs -x cleansstate
petalinux-build

petalinux-package --boot --fsbl images/linux/zynq_fsbl.elf \
    --fpga images/linux/design_1_wrapper.bit \
    --u-boot images/linux/u-boot.elf --force

petalinux-package --wic --bootfiles "BOOT.BIN image.ub boot.scr"

cp images/linux/petalinux-sdimage.wic /mnt/share/
```

**해결 방법 B - SD 카드 직접 수정 (빠름):**

```bash
# Ubuntu에서 SD 카드 삽입
lsblk

# rootfs 파티션 마운트
sudo mkdir -p /mnt/sd_rootfs
sudo mount /dev/sdb2 /mnt/sd_rootfs

# /etc/shadow 백업
sudo cp /mnt/sd_rootfs/etc/shadow /mnt/sd_rootfs/etc/shadow.backup

# root 패스워드 제거
sudo sed -i 's/^root:[^:]*:/root::/' /mnt/sd_rootfs/etc/shadow

# 확인 (두 번째 필드가 비어있어야 함)
sudo cat /mnt/sd_rootfs/etc/shadow | grep root

# 언마운트
sync
sudo umount /mnt/sd_rootfs
```

### 10.2 자동 로그인 안됨

```bash
cd ~/projects/myproject

# 설정 확인
cat project-spec/configs/rootfs_config | grep serial-autologin

# 없으면 완전 클린 빌드
petalinux-build -x mrproper
petalinux-config --get-hw-description=~/projects/
petalinux-config -c rootfs
# (로그인 설정 다시 확인)

petalinux-build
```

### 10.3 시리얼 콘솔 출력 없음

**확인 사항:**
1. COM 포트 번호 (장치 관리자)
2. Baud Rate: 115200
3. Flow control: None
4. FTDI 드라이버 재설치
5. USB 케이블 교체

### 10.4 SSH 접속 안됨

```bash
# Zybo에서
systemctl status sshd
systemctl start sshd
systemctl enable sshd

# SSH 설정 확인
vi /etc/ssh/sshd_config
# PermitRootLogin yes
# PasswordAuthentication yes
# PermitEmptyPasswords yes

systemctl restart sshd
```

---

## 11. 트러블슈팅

### 11.1 빌드 문제

**메모리 부족:**
```bash
# 스왑 파일 생성
sudo fallocate -l 8G /swapfile
sudo chmod 600 /swapfile
sudo mkswap /swapfile
sudo swapon /swapfile
echo '/swapfile none swap sw 0 0' | sudo tee -a /etc/fstab
```

**디스크 부족:**
```bash
df -h
cd ~/projects/myproject
petalinux-build -x clean
```

**특정 패키지 실패:**
```bash
# 로그 확인
find build/tmp/work -name "log.do_compile*"

# 재빌드
petalinux-build -c <패키지명> -x cleansstate
petalinux-build -c <패키지명>
```

### 11.2 부팅 문제

**U-Boot에서 멈춤:**
```bash
# U-Boot 콘솔에서 수동 부팅
ZynqMP> fatload mmc 0 0x2000000 image.ub
ZynqMP> bootm 0x2000000

# Ubuntu에서 boot.scr 재생성
cd ~/projects/myproject/images/linux/
mkimage -A arm -O linux -T script -C none \
    -a 0 -e 0 -n "Boot Script" -d boot.cmd boot.scr
```

**Kernel panic:**
```bash
# rootfs 파티션 확인
sudo mount /dev/sdb2 /mnt/sd_rootfs
ls /mnt/sd_rootfs/

# rootfs 재압축 해제
sudo rm -rf /mnt/sd_rootfs/*
sudo tar xzf ~/projects/myproject/images/linux/rootfs.tar.gz \
    -C /mnt/sd_rootfs/

sync
sudo umount /mnt/sd_rootfs
```

**SD 카드 인식 안됨:**
- 다른 SD 카드 시도 (Class 10+)
- 32GB 이하 권장
- 포맷 후 재시도

### 11.3 네트워크 문제

**이더넷 링크 안됨:**
```bash
ifconfig eth0 up
ethtool eth0  # Link detected: yes 확인
```

**DHCP 실패:**
```bash
# 수동 IP 설정
ifconfig eth0 192.168.1.100 netmask 255.255.255.0 up
route add default gw 192.168.1.1
echo "nameserver 8.8.8.8" > /etc/resolv.conf

ping 8.8.8.8
```

### 11.4 공유 폴더 문제

**/mnt/share 마운트 안됨:**
```bash
sudo mkdir -p /mnt/share
lsmod | grep vbox

# 수동 마운트
sudo mount -t vboxsf -o uid=$(id -u),gid=$(id -g) share /mnt/share

# fstab 확인
cat /etc/fstab | grep share

# 재부팅
sudo reboot
```

```
sudo usermod -aG vboxsf gotree94
```


---

## 12. 체크리스트

### 12.1 설치 전

- [ ] VirtualBox 설치
- [ ] Ubuntu 22.04.5 ISO
- [ ] 디스크 공간 200GB+
- [ ] RAM 16GB+
- [ ] C:\share 폴더 생성
- [ ] PetaLinux 인스톨러
- [ ] design_1_wrapper.xsa

### 12.2 빌드 전

- [ ] PetaLinux 환경 활성화
- [ ] 필수 패키지 설치
- [ ] XSA 파일 복사
- [ ] **Rootfs 로그인 설정 완료**
  - [ ] debug-tweaks
  - [ ] allow-empty-password
  - [ ] serial-autologin-root
- [ ] 빌드 시간 확보 (1-3시간)

### 12.3 SD 카드 굽기 전

- [ ] 빌드 성공 확인
- [ ] WIC 이미지 생성
- [ ] Windows로 복사
- [ ] balenaEtcher 설치
- [ ] SD 카드 준비 (4GB+)

### 12.4 부팅 전

- [ ] SD 카드 굽기 완료
- [ ] JP5 점퍼 SD 모드
- [ ] SD 카드 삽입
- [ ] UART 케이블 연결
- [ ] FTDI 드라이버 설치
- [ ] COM 포트 확인
- [ ] PuTTY 설정 (115200 8N1)

---

## 13. 자주 사용하는 명령어

### 13.1 PetaLinux 명령어

```bash
# 환경 활성화
source ~/petalinux/2022.2/settings.sh

# 프로젝트 생성
petalinux-create -t project --template zynq -n <이름>

# 설정
petalinux-config                      # 시스템
petalinux-config -c rootfs           # Rootfs (로그인!)
petalinux-config -c kernel           # 커널
petalinux-config --get-hw-description=<경로>

# 빌드
petalinux-build                       # 전체
petalinux-build -c <컴포넌트>        # 특정
petalinux-build -x clean              # 클린
petalinux-build -x mrproper           # 완전 클린

# 패키징
petalinux-package --boot --fsbl ... --fpga ... --u-boot ...
petalinux-package --wic --bootfiles "BOOT.BIN image.ub boot.scr"
```

### 13.2 Zybo 시스템 명령어

```bash
# 시스템
uname -a
hostname
cat /etc/os-release

# 네트워크
ifconfig
ip addr
udhcpc -i eth0
ping 8.8.8.8

# GPIO
echo <번호> > /sys/class/gpio/export
echo out > /sys/class/gpio/gpio<번호>/direction
echo 1 > /sys/class/gpio/gpio<번호>/value

# 로그
dmesg
journalctl
```

---

## 14. FAQ

**Q1: Root 패스워드가 무엇인가요?**  
A: 기본적으로 설정되지 않습니다. `allow-empty-password` 활성화 시 Enter만 누르면 됩니다.

**Q2: "Login incorrect" 오류가 나옵니다.**  
A: `petalinux-config -c rootfs`에서 로그인 관련 설정 활성화 후 재빌드하세요.

**Q3: Ubuntu 22.04 Warning이 걱정됩니다.**  
A: 무시해도 됩니다. 빌드가 성공하면 정상입니다.

**Q4: 빌드에 얼마나 걸리나요?**  
A: 첫 빌드는 1-3시간, 이후는 10-30분입니다.

**Q5: SD 카드 크기는?**  
A: 최소 4GB, 권장 8GB 이상입니다.

**Q6: SSH 접속이 안됩니다.**  
A: rootfs에 openssh 패키지 추가 후 재빌드하세요.

**Q7: 자동 로그인 보안이 걱정됩니다.**  
A: 개발 완료 후 debug-tweaks와 serial-autologin-root를 비활성화하고 passwd로 패스워드 설정하세요.

**Q8: 공유 폴더가 안 보입니다.**  
A: Guest Additions 설치 확인, vboxsf 그룹 추가, 재부팅 후 확인하세요.

---

## 15. 참고 자료

### 15.1 공식 문서

**AMD/Xilinx**
- PetaLinux Tools Reference (UG1144)  
  https://docs.amd.com/r/en-US/ug1144-petalinux-tools-reference-guide
- PetaLinux Command Line Guide (UG1157)  
  https://docs.amd.com/r/en-US/ug1157-petalinux-tools-command-line-guide
- Embedded Design Tutorial (UG1165)  
  https://docs.amd.com/r/en-US/ug1165-embedded-design-tutorial
- Zynq-7000 TRM (UG585)  
  https://docs.amd.com/v/u/en-US/ug585-zynq-7000-trm

**Digilent**
- Zybo Z7 Reference Manual  
  https://digilent.com/reference/programmable-logic/zybo-z7/reference-manual
- Zybo Z7 GitHub  
  https://github.com/Digilent/Zybo-Z7
- Digilent XDC Files  
  https://github.com/Digilent/digilent-xdc

### 15.2 개발 도구

**필수 다운로드**
- VirtualBox  
  https://www.virtualbox.org/wiki/Downloads
- Ubuntu 22.04.5 LTS  
  https://ubuntu.com/download/desktop
- balenaEtcher  
  https://www.balena.io/etcher/
- PuTTY  
  https://www.putty.org/
- FTDI 드라이버  
  https://ftdichip.com/drivers/vcp-drivers/

### 15.3 커뮤니티

- Xilinx Community Forums  
  https://support.xilinx.com/
- Digilent Forum  
  https://forum.digilent.com/
- Stack Overflow  
  https://stackoverflow.com/questions/tagged/petalinux
- Reddit r/FPGA  
  https://www.reddit.com/r/FPGA/

### 15.4 학습 자료

- The Zynq Book (무료 PDF)  
  http://www.zynqbook.com/
- FPGA Developer  
  https://www.fpgadeveloper.com/
- Embedded Linux Wiki  
  https://elinux.org/

---

## 16. 부록

### 16.1 자동화 스크립트

**Ubuntu 준비 스크립트 (setup_ubuntu.sh):**

```bash
#!/bin/bash
sudo apt update && sudo apt upgrade -y
sudo dpkg --add-architecture i386
sudo apt update

sudo apt install -y \
    build-essential gcc-multilib g++-multilib gawk wget git \
    diffstat unzip texinfo chrpath socat cpio python3
