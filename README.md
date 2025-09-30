# Digilent Zybo Z7-20 PetaLinux 완벽 가이드

## 빌드 성공 및 Warning 해결 포함

---

## 목차
1. [VirtualBox Ubuntu 22.04.5 설치](#1-virtualbox-ubuntu-22045-설치)
2. [Ubuntu 시스템 준비](#2-ubuntu-시스템-준비)
3. [PetaLinux 2022.2 설치](#3-petalinux-20222-설치)
4. [Zybo Z7-20 프로젝트 생성](#4-zybo-z7-20-프로젝트-생성)
5. [PetaLinux 빌드](#5-petalinux-빌드)
6. [빌드 Warning 해결](#6-빌드-warning-해결)
7. [SD 카드 이미지 생성](#7-sd-카드-이미지-생성)
8. [Windows에서 SD 카드 굽기](#8-windows에서-sd-카드-굽기)
9. [Zybo Z7-20 부팅](#9-zybo-z7-20-부팅)
10. [트러블슈팅](#10-트러블슈팅)

---

## 1. VirtualBox Ubuntu 22.04.5 설치

### 1.1 VirtualBox 가상머신 생성

**시스템 사양 (권장)**
```
이름: Zybo-PetaLinux
타입: Linux
버전: Ubuntu (64-bit)

메모리: 16384 MB (16GB) - 최소 8GB
프로세서: 8 CPU - 최소 4 CPU
디스크: 200 GB (VDI, 동적 할당) - 최소 150GB
```

**고급 설정**
- 설정 → 시스템 → 프로세서
  - ✅ PAE/NX 사용
  - ✅ 하드웨어 가상화 (VT-x/AMD-V) 활성화
  
- 설정 → 디스플레이
  - 비디오 메모리: 128 MB
  - ✅ 3D 가속 사용

- 설정 → 공유 폴더
  - 새 공유 폴더 추가
  - 이름: `SharedFolder`
  - 경로: `C:\SharedFolder` (Windows에 먼저 생성)
  - ✅ 자동 마운트
  - ✅ 영구적으로 만들기

### 1.2 Ubuntu 22.04.5 설치

1. **ISO 마운트 및 부팅**
   - `ubuntu-22.04.5-desktop-amd64.iso` 선택
   - 가상머신 시작

2. **설치 옵션**
   - Install Ubuntu
   - 언어: English
   - 키보드: English (US)
   - Normal installation
   - ✅ Download updates while installing Ubuntu
   - ✅ Install third-party software

3. **디스크 설정**
   - Erase disk and install Ubuntu
   - Install Now

4. **사용자 계정**
   ```
   Your name: Zybo User
   Computer name: zybo-petalinux
   Username: zybo (또는 원하는 이름)
   Password: [원하는 비밀번호]
   ```

5. **설치 완료 후 재부팅**

### 1.3 VirtualBox Guest Additions 설치

```bash
# 터미널 열기 (Ctrl+Alt+T)
sudo apt update
sudo apt install -y build-essential dkms linux-headers-$(uname -r)

# VirtualBox 메뉴: Devices → Insert Guest Additions CD image
# 자동 실행 또는 수동 실행:
cd /media/$USER/VBox*
sudo ./VBoxLinuxAdditions.run

# 재부팅
sudo reboot
```

### 1.4 공유 폴더 권한 설정

```bash
# 사용자를 vboxsf 그룹에 추가
sudo usermod -aG vboxsf gotree94

# 재부팅
sudo reboot

# 공유 폴더 확인
ls -la /mnt/share
```

---

## 2. Ubuntu 시스템 준비

### 2.1 시스템 업데이트

```bash
sudo apt update
sudo apt upgrade -y
```

### 2.2 32비트 라이브러리 지원 추가

```bash
sudo dpkg --add-architecture i386
sudo apt update
```

### 2.3 필수 패키지 설치

```bash
sudo apt install -y \
    build-essential \
    gcc-multilib \
    g++-multilib \
    gawk \
    wget \
    git \
    diffstat \
    unzip \
    texinfo \
    chrpath \
    socat \
    cpio \
    python3 \
    python3-pip \
    python3-pexpect \
    xz-utils \
    debianutils \
    iputils-ping \
    python3-git \
    python3-jinja2 \
    libegl1-mesa \
    libsdl1.2-dev \
    pylint \
    xterm \
    rsync \
    curl \
    libncurses5-dev \
    libncursesw5-dev \
    libssl-dev \
    flex \
    bison \
    libselinux1 \
    gnupg \
    zlib1g-dev \
    libtool \
    autoconf \
    automake \
    net-tools \
    screen \
    pax \
    gzip \
    vim \
    iproute2 \
    locales \
    libncurses5 \
    libtinfo5
```

### 2.4 32비트 라이브러리 설치

```bash
sudo apt install -y \
    libncurses5:i386 \
    libc6:i386 \
    libstdc++6:i386 \
    lib32z1 \
    zlib1g:i386
```

### 2.5 Locale 설정

```bash
sudo locale-gen en_US.UTF-8
sudo update-locale LANG=en_US.UTF-8
```

### 2.6 Dash를 Bash로 변경

```bash
sudo dpkg-reconfigure dash
```
- 메뉴가 나타나면 **"No"** 선택

### 2.7 TFTP 서버 설치 (선택사항)

```bash
# TFTP 서버 설치
sudo apt install -y tftpd-hpa

# TFTP 디렉토리 생성 및 권한 설정
sudo mkdir -p /tftpboot
sudo chmod 777 /tftpboot
sudo chown nobody:nogroup /tftpboot

# TFTP 설정 편집
sudo vi /etc/default/tftpd-hpa
```

**TFTP 설정 내용:**
```
# /etc/default/tftpd-hpa
TFTP_USERNAME="tftp"
TFTP_DIRECTORY="/tftpboot"
TFTP_ADDRESS=":69"
TFTP_OPTIONS="--secure"
```

```bash
# TFTP 서비스 재시작
sudo systemctl restart tftpd-hpa
sudo systemctl enable tftpd-hpa

# 상태 확인
sudo systemctl status tftpd-hpa
```

---

## 3. PetaLinux 2022.2 설치

### 3.1 작업 디렉토리 생성

```bash
mkdir -p ~/petalinux_work
cd ~/petalinux_work
```

### 3.2 인스톨러 준비

Windows에서 Ubuntu로 파일 복사:
```bash
# petalinux-v2022.2-10141622-installer.run을 
# C:\SharedFolder에 복사한 후

# Ubuntu에서:
cp /mnt/share/petalinux-v2022.2-10141622-installer.run ~/petalinux_work/
chmod +x ~/petalinux_work/petalinux-v2022.2-10141622-installer.run
```

### 3.3 PetaLinux 설치

```bash
# 설치 디렉토리 생성
mkdir -p ~/petalinux/2022.2

# 인스톨러 실행
cd ~/petalinux_work
./petalinux-v2022.2-10141622-installer.run -d ~/petalinux/2022.2
```

**설치 진행:**
- 라이센스 동의: `y` 입력하고 Enter
- 설치 시간: 약 10-30분 소요
- 디스크 사용량: 약 8GB

### 3.4 PetaLinux 환경 설정

```bash
# PetaLinux 환경 활성화
source ~/petalinux/2022.2/settings.sh

# 확인
echo $PETALINUX
# 출력: /home/사용자명/petalinux/2022.2
```

**영구 설정 (선택사항):**
```bash
echo "source ~/petalinux/2022.2/settings.sh" >> ~/.bashrc
```

---

## 4. Zybo Z7-20 프로젝트 생성

### 4.1 프로젝트 디렉토리 생성

```bash
mkdir -p ~/projects
cd ~/projects
```

### 4.2 XSA 파일 준비

```bash
# design_1_wrapper.xsa를 Windows의 SharedFolder로 복사한 후
cp /mnt/share/design_1_wrapper.xsa ~/projects/

# XSA 파일 내용 확인
unzip -l design_1_wrapper.xsa
```

**예상 출력:**
```
Archive:  design_1_wrapper.xsa
  Length      Date    Time    Name
---------  ---------- -----   ----
      306  2025-09-22 21:05   aie_primitive.json
     3168  2025-09-22 21:05   design_1.bda
   155208  2025-09-22 21:05   design_1.hwh
   618914  2025-09-22 21:05   design_1_wrapper.bit
   542756  2025-09-22 21:05   ps7_init.c
     3794  2025-09-22 21:05   ps7_init.h
  2951774  2025-09-22 21:05   ps7_init.html
    35774  2025-09-22 21:05   ps7_init.tcl
   543373  2025-09-22 21:05   ps7_init_gpl.c
     4412  2025-09-22 21:05   ps7_init_gpl.h
     1441  2025-09-22 21:05   sysdef.xml
     2432  2025-09-22 21:05   xsa.json
     1221  2025-09-22 21:05   xsa.xml
```

### 4.3 Zynq-7000 프로젝트 생성

```bash
cd ~/projects

# PetaLinux 환경이 활성화되어 있는지 확인
source ~/petalinux/2022.2/settings.sh

# Zybo Z7-20용 프로젝트 생성
petalinux-create --type project --template zynq --name myproject

# 프로젝트 디렉토리로 이동
cd myproject
```

### 4.4 하드웨어 설정 가져오기

```bash
# XSA 파일로 하드웨어 설정
petalinux-config --get-hw-description=~/projects/
```

**설정 메뉴가 나타남**

### 4.5 시스템 설정 (중요!)

#### **Image Packaging Configuration**
```
Image Packaging Configuration  --->
    Root filesystem type (SD card)  --->
        (X) SD card
        ( ) INITRAMFS
        ( ) INITRD
        ( ) NFS
    
    Copy final images to tftpboot  --->
        [ ] Copy final images to tftpboot  (비활성화 권장)
```

**확인내용**
```
misc/config System Configuration
	Arrow keys navigate the menu.  
	<Enter> selects submenus ---> (or empty submenus ----).  
	Highlighted letters are hotkeys.  
	Pressing <Y> includes, <N> excludes, <M> modularizes features.  
	Press <Esc><Esc> to exit, <?> for Help, </> for Search.  
	Legend: [*] built-in  [ ] excluded  <M> module  < > module capable 
  
-*- ZYNQ Configuration
  Linux Components Selection  --->
  Auto Config Settings  --->
-*- Subsystem AUTO Hardware Settings  --->
  DTG Settings  --->
  FSBL Configuration  --->
  FPGA Manager  --->
  u-boot Configuration  --->
  Linux Configuration  --->
  Image Packaging Configuration  --->
  Firmware Version Configuration  --->
  Yocto Settings  --->
```

```
Image Packaging Configuration
	Arrow keys navigate the menu.  
	<Enter> selects submenus ---> (or empty submenus ----).  
	Highlighted letters are hotkeys.  
	Pressing <Y> includes, <N> excludes, <M> modularizes features.  
	Press <Esc><Esc> to exit, <?> for Help, </> for Search.  
	Legend: [*] built-in  [ ] excluded  <M> module  < > module capable  
  
Root filesystem type (INITRD)  --->
  (0x0) RAMDISK loadaddr
  (petalinux-image-minimal) INITRAMFS/INITRD Image name
  (image.ub) name for bootable kernel image
  (cpio cpio.gz cpio.gz.u-boot ext4 tar.gz jffs2) Root filesystem formats
  (0x1000) DTB padding size
  [*] Copy final images to tftpboot
  (/tftpboot) tftpboot directory
```

```
┌─────────────────────Root filesystem type ─────────────────────┐
│  Use the arrow keys to navigate this window or press the      │  
│  hotkey of the item you wish to select followed by the <SPACE │  
│  BAR>. Press <?> for additional information about this        │  
│ ┌───────────────────────────────────────────────────────────┐ │  
│ │                ( ) INITRAMFS                              │ │  
│ │                (X) INITRD                                 │ │  
│ │                ( ) JFFS2                                  │ │  
│ │                ( ) UBI/UBIFS                              │ │  
│ │                ( ) NFS                                    │ │  
│ │                ( ) EXT4 (SD/eMMC/SATA/USB)                │ │  
│ │               ( ) other                                   │ │  
│ └───────────────────────────────────────────────────────────┘ │  
└───────────────────────────────────────────────────────────────┘ 
```

#### **Yocto Settings**

```
Yocto Settings  --->
    YOCTO_MACHINE_NAME (zynq-generic)  --->
    [*] Enable auto resize SD card root filesystem
```

**확인내용**
```
  ┌──────────────────────────── Yocto Settings ──────────────────────────────────────────┐
  │  Arrow keys navigate the menu.                                                       │  
  │  <Enter> selects submenus ---> (or empty submenus ----).                             │  
  │  Highlighted letters are hotkeys.                                                    │  
  │  Pressing <Y> includes, <N> excludes, <M> modularizes features.                      │  
  │  Press <Esc><Esc> to exit, <?> for Help, </> for Search.                             │  
  │  Legend: [*] built-in  [ ] excluded  <M> module  < > module capable                  │  
  │                                                                                      │  
  │ ┌──────────────────────────────────────────────────────────────────────────────────┐ │  
  │ │                  (zynq-generic) YOCTO_MACHINE_NAME                               │ │  
  │ │                       TMPDIR Location  --->                                      │ │  
  │ │                       Devtool Workspace Location  --->                           │ │  
  │ │                       Parallel thread execution  --->                            │ │  
  │ │                       Add pre-mirror url   --->                                  │ │  
  │ │                       Local sstate feeds settings  --->                          │ │  
  │ │                  [*] Enable Network sstate feeds                                 │ │  
  │ │                       Network sstate feeds URL  --->                             │ │  
  │ │                  [ ] Enable BB NO NETWORK                                        │ │  
  │ │                  [ ] Enable Buildtools Extended                                  │ │  
  │ │                      User Layers  --->                                           │ │ 
  │ └──────────────────────────────────────────────────────────────────────────────────┘ │ 
  └──────────────────────────────────────────────────────────────────────────────────────┘
```


#### **Subsystem AUTO Hardware Settings**
```
Subsystem AUTO Hardware Settings  --->
    Serial Settings  --->
        Primary stdin/stdout (ps7_uart_1)  --->
            (X) ps7_uart_1
    
    Ethernet Settings  --->
        Primary Ethernet (ps7_ethernet_0)  --->
            (X) ps7_ethernet_0
    
    SD/SDIO Settings  --->
        Primary SD/SDIO (ps7_sd_0)  --->
            (X) ps7_sd_0
```
**설정 저장:**
- `Save` 선택
- 기본 파일명 `.config` 그대로 저장
- `Exit` 선택
  
**확인내용**
  ---
```
  ┌───────────────────Subsystem AUTO Hardware Settings ────────────────────────────┐
  │  Arrow keys navigate the menu.                                                 │  
  │    <Enter> selects submenus ---> (or empty submenus ----).                     │  
  │  Highlighted letters are hotkeys.                                              │  
  │  Pressing <Y> includes, <N> excludes, <M>                                      │  
  │  modularizes features.                                                         │   
  │  Press <Esc><Esc> to exit, <?> for Help, </> for Search.                       │   
  │  Legend: [*] built-in  [ ] excluded  <M> module  < > module capable            │  
  │                                                                                │  
  │ ┌────────────────────────────────────────────────────────────────────────────┐ │  
  │ │   --- Subsystem AUTO Hardware Settings                                     │ │  
  │ │         System Processor (ps7_cortexa9_0)  --->                            │ │  
  │ │         Memory Settings  --->                                              │ │  
  │ │         Serial Settings  --->                                              │ │  
  │ │         Ethernet Settings  --->                                            │ │  
  │ │         Flash Settings  --->                                               │ │  
  │ │         SD/SDIO Settings  --->                                             │ │  
  │ │         RTC Settings  --->                                                 │ │  
  │ │                                                                            │ │  
  │ └────────────────────────────────────────────────────────────────────────────┘ │  
  └────────────────────────────────────────────────────────────────────────────────┘
```
  ---  
```  
  ┌────────────────────── Serial Settings ─────────────────────────────────────────┐
  │  Arrow keys navigate the menu.                                                 │  
  │ <Enter> selects submenus ---> (or empty submenus ----).                        │  
  │ Highlighted letters are hotkeys.                                               │  
  │ Pressing <Y> includes, <N> excludes, <M>                                       │  
  │  modularizes features.                                                         │  
  │ Press <Esc><Esc> to exit, <?> for Help, </> for Search.                        │  
  │ Legend: [*] built-in  [ ] excluded  <M> module  < > module capable             │  
  │                                                                                │ 
  │                                                                                │ 
  │ ┌────────────────────────────────────────────────────────────────────────────┐ │  
  │ │  FSBL Serial stdin/stdout (ps7_uart_1)  --->                               │ │  
  │ │  DTG Serial stdin/stdout (ps7_uart_1)  --->                                │ │  
  │ │  System stdin/stdout baudrate for ps7_uart_1 (115200)  --->                │ │
  │ └────────────────────────────────────────────────────────────────────────────┘ │  
  └────────────────────────────────────────────────────────────────────────────────┘ 

  ┌──────────────────────────FSBL Serial stdin/stdout ────────────────────────────┐
  │  Use the arrow keys to navigate this window or press the                      │  
  │  hotkey of the item you wish to select followed by the <SPACE BAR>.           │
  │  Press <?> for additional information about this                              │  
  │ ┌───────────────────────────────────────────────────────────────────────────┐ │  
  │ │                      (X) ps7_uart_1                                       │ │  
  │ │                      ( ) manual                                           │ │  
  │ │                                                                           │ │  
  │ └───────────────────────────────────────────────────────────────────────────┘ │  
  └───────────────────────────────────────────────────────────────────────────────┘
```
  ---
```  
  ┌───────────────────────────── Ethernet Settings ───────────────────────────────┐
  │  Arrow keys navigate the menu.                                                │
  │  <Enter> selects submenus ---> (or empty submenus ----).                      │
  │  Highlighted letters are hotkeys.                                             │
  │  Pressing <Y> includes, <N> excludes, <M>                                     │  
  │  modularizes features.                                                        │
  │  Press <Esc><Esc> to exit, <?> for Help, </> for Search.                      │
  │  Legend: [*] built-in  [ ] excluded  <M> module  < > module capable           │  
  │                                                                               │ 
  │ ┌───────────────────────────────────────────────────────────────────────────┐ │  
  │ │            Primary Ethernet (ps7_ethernet_0)  --->                        │ │  
  │ │        [ ] Randomise MAC address                                          │ │  
  │ │        (00:0a:35:00:1e:53) Ethernet MAC address                           │ │  
  │ │        [*] Obtain IP address automatically                                │ │  
  │ │                                                                           │ │  
  │ └───────────────────────────────────────────────────────────────────────────┘ │  
  └───────────────────────────────────────────────────────────────────────────────┘ 

  ┌──────────────────────────── Primary Ethernet ───────────────────────┐
  │  Use the arrow keys to navigate this window or press the            │  
  │  hotkey of the item you wish to select followed by the <SPACE       │  
  │  BAR>. Press <?> for additional information about this              │  
  │ ┌─────────────────────────────────────────────────────────────────┐ │  
  │ │                    (X) ps7_ethernet_0                           │ │  
  │ │                    ( ) manual                                   │ │  
  │ │                                                                 │ │  
  │ └─────────────────────────────────────────────────────────────────┘ │  
  └─────────────────────────────────────────────────────────────────────┘
```
  ---
```  
  ┌──────────────────── SD/SDIO Settings ──────────────────────────────────┐
  │  Arrow keys navigate the menu.                                         │  
  │  <Enter> selects submenus ---> (or empty submenus ----).               │  
  │  Highlighted letters are hotkeys.                                      │  
  │  Pressing <Y> includes, <N> excludes, <M> modularizes features.        │  
  │  Press <Esc><Esc> to exit, <?> for Help, </> for Search.               │  
  │  Legend: [*] built-in  [ ] excluded  <M> module  < > module capable    │  
  │                                                                        │ 
  │                                                                        │ 
  │ ┌────────────────────────────────────────────────────────────────────┐ │  
  │ │                   Primary SD/SDIO (ps7_sd_0)  --->                 │ │  
  │ │                                                                    │ │  
  │ └────────────────────────────────────────────────────────────────────┘ │  
  └────────────────────────────────────────────────────────────────────────┘ 
  
 ┌─────────────────────── Primary SD/SDIO ───────────────────────┐
 │  Use the arrow keys to navigate this window or press the      │  
 │  hotkey of the item you wish to select followed by the <SPACE │  
 │  BAR>. Press <?> for additional information about this        │  
 │ ┌───────────────────────────────────────────────────────────┐ │  
 │ │                       (X) ps7_sd_0                        │ │  
 │ │                       ( ) manual                          │ │  
 │ │                                                           │ │  
 │ └───────────────────────────────────────────────────────────┘ │  
 └───────────────────────────────────────────────────────────────┘
```
  ---


### 4.6 Root Filesystem 설정

```bash
petalinux-config -c rootfs
```

**유용한 패키지:**
```
Filesystem Packages  --->
    admin  --->
        [*] sudo
    
    console/utils  --->
        [*] vim
        [*] nano
    
    devel  --->
        [*] gcc
        [*] g++
        [*] make
    
    network  --->
        [*] openssh
        [*] openssh-sshd
```

**설정 저장:**
- `Save` → `Exit`

---
### 4.7 Root 로그인 설정 (중요!)

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

#### 4.7.1 해결 방법 - Rootfs 설정 (필수!)

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

#### 4.7.2 설정 확인

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

1. GCC 컴파일러 찾기
검색 결과 gcc-sanitizers만 나왔다면, 실제 gcc 패키지는 다른 위치에 있거나 다른 이름일 수 있습니다.

다시 검색해보세요:
```
/packagegroup-core-buildessential
```
또는 직접 메뉴에서:
```
Filesystem Packages → 
  packagegroup → 
    packagegroup-core-buildessential
```   
이 패키지를 활성화하면 gcc, g++, make가 모두 포함됩니다.
```
Filesystem Packages → devel → 
  [*] packagegroup-core-buildessential
```

3. Image Features 옵션들이 안 보이는 이유
PetaLinux 버전에 따라 이런 옵션들이 다른 방식으로 관리됩니다.
해결 방법: 설정 파일 직접 수정
```bash
cd ~/projects/myproject
vi project-spec/configs/rootfs_config
```
다음 라인들을 추가:
```
CONFIG_debug-tweaks=y
CONFIG_empty-root-password=y
CONFIG_auto-login=y
```

그리고 project-spec/meta-user/conf/petalinuxbsp.conf 파일도 확인:

```bash
vi project-spec/meta-user/conf/petalinuxbsp.conf
```
다음 라인 추가:
```
EXTRA_IMAGE_FEATURES += "debug-tweaks empty-root-password allow-empty-password"
EXTRA_IMAGE_FEATURES += "serial-autologin-root"
```
또는 local.conf 수정:

```bash
vi project-spec/meta-user/conf/local.conf
```
추가:
```
EXTRA_IMAGE_FEATURES += "debug-tweaks"
EXTRA_IMAGE_FEATURES += "empty-root-password"
EXTRA_IMAGE_FEATURES += "allow-empty-password"
```

---





## 5. PetaLinux 빌드

### 5.1 전체 시스템 빌드

```bash
cd ~/projects/myproject

# PetaLinux 환경 확인
source ~/petalinux/2022.2/settings.sh

# 빌드 시작
petalinux-build
```

**빌드 시간:**
- 첫 빌드: 1-3시간 (시스템 사양에 따라)
- 이후 빌드: 10-30분

**빌드 성공 메시지:**
```
NOTE: Tasks Summary: Attempted 5162 tasks of which 1350 didn't need to be rerun and all succeeded.
Summary: There were 2 WARNING messages shown.
INFO: Failed to copy built images to tftp dir: /tftpboot
[INFO] Successfully built project
```

### 5.2 부트 이미지 생성 (BOOT.BIN)

```bash
cd ~/projects/myproject

petalinux-package --boot \
    --fsbl images/linux/zynq_fsbl.elf \
    --fpga images/linux/design_1_wrapper.bit \
    --u-boot images/linux/u-boot.elf \
    --force
```

**생성된 파일:**
```
images/linux/BOOT.BIN
```

---

## 6. 빌드 Warning 해결

빌드 중 발생한 2개의 Warning에 대한 분석과 해결 방법입니다.

### 6.1 Warning 1: Host Distribution 검증 안됨

**Warning 메시지:**
```
WARNING: Host distribution "ubuntu-22.04" has not been validated with this version of the build system; 
you may possibly experience unexpected failures. It is recommended that you use a tested distribution.
```

**원인:**
- PetaLinux 2022.2는 공식적으로 Ubuntu 20.04를 지원
- Ubuntu 22.04는 검증되지 않은 배포판

**영향:**
- ⚠️ 경고성 메시지이며, 실제로는 빌드가 정상적으로 완료됨
- 대부분의 경우 문제없이 작동

**해결 방법 (선택사항):**

#### 방법 1: 경고 무시 (권장)
```bash
# 빌드가 성공했다면 무시해도 됨
# 실제 문제가 발생할 때만 조치
```

#### 방법 2: 배포판 검증 우회
```bash
# 환경 변수로 검증 비활성화
export SKIP_DISTRO_CHECK=1

# 또는 .bashrc에 추가
echo "export SKIP_DISTRO_CHECK=1" >> ~/.bashrc
```

#### 방법 3: 공식 지원 배포판 사용
- Ubuntu 20.04 LTS 사용 (권장하지 않음 - 재설치 필요)

### 6.2 Warning 2: Uninative glibc 버전 불일치

**Warning 메시지:**
```
WARNING: Your host glibc version (2.35) is newer than that in uninative (2.34). 
Disabling uninative so that sstate is not corrupted.
```

**원인:**
- Ubuntu 22.04의 glibc 버전 (2.35)이 PetaLinux uninative (2.34)보다 최신
- Yocto는 자동으로 uninative를 비활성화하여 sstate 손상 방지

**영향:**
- ✅ 자동으로 처리되므로 문제없음
- 빌드 시간이 약간 증가할 수 있음 (sstate 캐시 미사용)

**해결 방법:**

#### 방법 1: 경고 무시 (권장)
```bash
# Yocto가 자동으로 처리하므로 조치 불필요
# 빌드는 정상적으로 완료됨
```

#### 방법 2: Uninative 비활성화 (명시적)
```bash
# project-spec/meta-user/conf/petalinuxbsp.conf 편집
vi ~/projects/myproject/project-spec/meta-user/conf/petalinuxbsp.conf

# 다음 줄 추가:
INHERIT_remove = "uninative"
```

### 6.3 Info: TFTP 복사 실패

**Info 메시지:**
```
INFO: Failed to copy built images to tftp dir: /tftpboot
```

**원인:**
- `/tftpboot` 디렉토리가 없거나 권한 부족
- TFTP 서버가 설치되지 않음

**영향:**
- ⚠️ SD 카드 부팅에는 영향 없음
- TFTP 네트워크 부팅을 사용하지 않는다면 무시 가능

**해결 방법:**

#### 방법 1: TFTP 복사 비활성화 (권장)
```bash
petalinux-config

# Image Packaging Configuration --->
#     [ ] Copy final images to tftpboot  (비활성화)
```

#### 방법 2: TFTP 디렉토리 생성
```bash
# TFTP 디렉토리 생성 및 권한 설정
sudo mkdir -p /tftpboot
sudo chmod 777 /tftpboot
sudo chown $USER:$USER /tftpboot

# 재빌드 (또는 이미지만 복사)
cp ~/projects/myproject/images/linux/BOOT.BIN /tftpboot/
cp ~/projects/myproject/images/linux/image.ub /tftpboot/
cp ~/projects/myproject/images/linux/boot.scr /tftpboot/
```

#### 방법 3: TFTP 서버 완전 설치 (네트워크 부팅용)
```bash
# TFTP 서버 설치
sudo apt install -y tftpd-hpa

# 설정
sudo vi /etc/default/tftpd-hpa
```

**TFTP 설정:**
```
TFTP_USERNAME="tftp"
TFTP_DIRECTORY="/tftpboot"
TFTP_ADDRESS=":69"
TFTP_OPTIONS="--secure"
```

```bash
# 권한 설정
sudo mkdir -p /tftpboot
sudo chmod 777 /tftpboot

# 서비스 재시작
sudo systemctl restart tftpd-hpa
sudo systemctl enable tftpd-hpa
```

### 6.4 Warning 요약

| Warning | 심각도 | 조치 필요 | 권장 사항 |
|---------|--------|-----------|-----------|
| Ubuntu 22.04 미검증 | 낮음 | 선택적 | 무시 가능 |
| glibc 버전 불일치 | 낮음 | 불필요 | 자동 처리됨 |
| TFTP 복사 실패 | 매우 낮음 | 선택적 | SD 카드 부팅 시 무시 |

**결론:**
- ✅ 빌드는 성공적으로 완료됨
- ✅ SD 카드 부팅에 문제 없음
- ⚠️ Warning은 참고용이며 필수 조치 아님

---

## 7. SD 카드 이미지 생성

### 7.1 생성된 이미지 파일 확인

```bash
cd ~/projects/myproject/images/linux/

ls -lh
```

**주요 파일:**
```
BOOT.BIN           - 부트 이미지 (FSBL + Bitstream + U-Boot)
image.ub           - 커널 + Device Tree (FIT 이미지)
boot.scr           - U-Boot 부팅 스크립트
rootfs.tar.gz      - 루트 파일시스템
rootfs.ext4        - EXT4 형식 루트 파일시스템
```

### 7.2 WIC 이미지 생성 (권장)

```bash
cd ~/projects/myproject

# WIC 이미지 생성
petalinux-package --wic \
    --bootfiles "BOOT.BIN image.ub boot.scr" \
    --images-dir images/linux/
```

**생성된 파일:**
```
images/linux/petalinux-sdimage.wic
```

### 7.3 이미지 압축 (선택사항)

```bash
cd ~/projects/myproject/images/linux/

# gzip 압축
gzip -k petalinux-sdimage.wic

# 압축 파일 확인
ls -lh petalinux-sdimage.wic.gz
```

### 7.4 Windows로 파일 복사

```bash
# WIC 이미지 복사
cp petalinux-sdimage.wic /media/sf_SharedFolder/

# 또는 압축 파일
cp petalinux-sdimage.wic.gz /media/sf_SharedFolder/

# 개별 부트 파일도 백업
mkdir -p /media/sf_SharedFolder/zybo_boot_files/
cp BOOT.BIN image.ub boot.scr /media/sf_SharedFolder/zybo_boot_files/
cp rootfs.tar.gz /media/sf_SharedFolder/zybo_boot_files/
```

---

## 8. Windows에서 SD 카드 굽기

### 8.1 준비물

- **SD 카드**: 최소 4GB (8GB 이상 권장)
- **SD 카드 리더기**
- **balenaEtcher 2.1.2** (또는 최신 버전)

### 8.2 balenaEtcher로 SD 카드 굽기

#### Step 1: balenaEtcher 실행

Windows에서 `balenaEtcher-2.1.2.Setup.exe` 실행 및 설치

#### Step 2: 이미지 파일 선택

1. **Flash from file** 클릭
2. 파일 선택:
   - `C:\SharedFolder\petalinux-sdimage.wic` 또는
   - `petalinux-sdimage.wic.gz` (압축 파일, 자동 해제)

#### Step 3: SD 카드 선택

1. **Select target** 클릭
2. SD 카드 드라이브 선택
   - ⚠️ **주의**: 올바른 드라이브인지 확인!
   - 모든 데이터가 삭제됩니다

#### Step 4: 굽기 시작

1. **Flash!** 클릭
2. 진행 상황 표시 (약 5-10분)
3. "Flash Complete!" 메시지 확인

#### Step 5: 안전하게 제거

- Windows에서 "하드웨어 안전하게 제거"
- SD 카드 제거

### 8.3 SD 카드 파티션 확인

**디스크 관리 (diskmgmt.msc):**
```
파티션 1: ~500MB, FAT32, BOOT (활성)
파티션 2: ~나머지, EXT4, rootfs
```

---

## 9. Zybo Z7-20 부팅

### 9.1 하드웨어 준비

#### Zybo Z7-20 점퍼 설정

**JP5 (Boot Mode) 점퍼:**
```
SD 카드 부팅 모드:
JP5: [  ] [  ]
     [SD] [  ]
```

#### 연결

1. **SD 카드 삽입**
   - Zybo Z7-20의 SD 카드 슬롯에 삽입

2. **UART 연결**
   - USB-UART 케이블을 J14 포트에 연결
   - Windows PC와 연결

3. **이더넷 연결** (선택사항)
   - RJ45 케이블로 네트워크 연결

4. **전원**
   - USB 전원 또는 DC 12V 어댑터
   - 전원 스위치 OFF 상태

### 9.2 Windows에서 시리얼 콘솔 연결

#### FTDI 드라이버 설치

- [FTDI 드라이버 다운로드](https://ftdichip.com/drivers/vcp-drivers/)
- 설치 후 재부팅

#### 장치 관리자에서 COM 포트 확인

1. `Win + X` → 장치 관리자
2. "포트 (COM & LPT)" 확인
3. "USB Serial Port (COMx)" 찾기 (예: COM3)

#### PuTTY 설정

**설정:**
```
Connection type: Serial
Serial line: COM3
Speed: 115200

Category: Connection → Serial
  - Speed: 115200
  - Data bits: 8
  - Stop bits: 1
  - Parity: None
  - Flow control: None
```

### 9.3 부팅

1. **시리얼 콘솔 열기** (PuTTY 또는 Tera Term)
2. **전원 켜기** (SW0 스위치 ON)
3. **부팅 메시지 확인**

```
Xilinx Zynq First Stage Boot Loader
Release 2022.2

U-Boot 2022.01 (Sep 22 2025 - 21:05:30 +0000)

Starting kernel ...

[    0.000000] Booting Linux on physical CPU 0x0
[    0.000000] Linux version 5.15.36-xilinx-v2022.2

PetaLinux 2022.2 myproject ttyPS0

myproject login:
```

### 9.4 로그인

**기본 계정:**
```
Username: root
Password: root
```

**처음 로그인 후:**
```bash
# 호스트명 확인
hostname

# 네트워크 확인
ifconfig eth0

# 커널 버전 확인
uname -a

# PetaLinux 버전 확인
cat /etc/os-release
```

### 9.5 네트워크 설정

#### DHCP (자동)
```bash
# DHCP 클라이언트 실행
udhcpc -i eth0

# IP 확인
ifconfig eth0
```

#### 고정 IP (수동)
```bash
# 임시 설정
ifconfig eth0 192.168.1.100 netmask 255.255.255.0
route add default gw 192.168.1.1

# ping 테스트
ping 192.168.1.1
```

---

## 10. 트러블슈팅

### 10.1 빌드 관련 문제

#### 메모리 부족
```bash
# 스왑 파일 생성
sudo fallocate -l 8G /swapfile
sudo chmod 600 /swapfile
sudo mkswap /swapfile
sudo swapon /swapfile

# 영구 설정
echo '/swapfile none swap sw 0 0' | sudo tee -a /etc/fstab
```

#### 디스크 공간 부족
```bash
# 디스크 사용량 확인
df -h

# 빌드 캐시 정리
cd ~/projects/myproject
petalinux-build -x clean
```

### 10.2 부팅 문제

#### 부팅 멈춤
```bash
# Ubuntu에서 BOOT.BIN 재생성
cd ~/projects/myproject

petalinux-package --boot \
    --fsbl images/linux/zynq_fsbl.elf \
    --fpga images/linux/design_1_wrapper.bit \
    --u-boot images/linux/u-boot.elf \
    --force

# SD 카드에 다시 복사
```

#### UART 출력 없음
```bash
# COM 포트 번호 확인
# Baud Rate 115200 확인
# FTDI 드라이버 재설치
```

#### SD 카드 인식 안됨
```bash
# 다른 SD 카드로 테스트
# Class 10 이상 사용
# 32GB 이하 권장
```

### 10.3 네트워크 문제

#### 이더넷 연결 안됨
```bash
# 링크 상태 확인
ethtool eth0

# DHCP 수동 실행
killall udhcpc
udhcpc -i eth0 -v

# 수동 IP 설정
ifconfig eth0 192.168.1.100 netmask 255.255.255.0 up
route add default gw 192.168.1.1
```

---

## 11. 체크리스트

### 11.1 설치 전 체크리스트

- [ ] VirtualBox 설치 완료
- [ ] Ubuntu 22.04.5 ISO 다운로드
- [ ] 충분한 디스크 공간 (200GB+)
- [ ] 충분한 RAM (16GB+)
- [ ] petalinux-v2022.2-10141622-installer.run 다운로드
- [ ] design_1_wrapper.xsa 준비
- [ ] 공유 폴더 설정 완료

### 11.2 빌드 전 체크리스트

- [ ] PetaLinux 환경 활성화 (`source settings.sh`)
- [ ] 모든 필수 패키지 설치 완료
- [ ] XSA 파일 복사 완료
- [ ] 충분한 빌드 시간 확보 (1-3시간)
- [ ] 안정적인 인터넷 연결

### 11.3 SD 카드 굽기 전 체크리스트

- [ ] WIC 이미지 생성 완료
- [ ] balenaEtcher 설치
- [ ] SD 카드 준비 (4GB+, Class 10+)
- [ ] SD 카드 리더기 연결
- [ ] 올바른 드라이브 선택 확인

### 11.4 부팅 전 체크리스트

- [ ] JP5 점퍼 SD 모드로 설정
- [ ] SD 카드 삽입
- [ ] UART 케이블 연결
- [ ] FTDI 드라이버 설치
- [ ] 시리얼 콘솔 설정 (115200 8N1)
- [ ] 전원 준비

---

## 12. 빌드 출력 분석

### 12.1 정상 빌드 출력

```
[INFO] Sourcing buildtools
[INFO] Building project
[INFO] Sourcing build environment
[INFO] Generating workspace directory
INFO: bitbake petalinux-image-minimal
NOTE: Started PRServer
WARNING: Host distribution "ubuntu-22.04" has not been validated
WARNING: Your host glibc version (2.35) is newer than that in uninative (2.34)
Loading cache: 100%
Parsing recipes: 100%
Parsing of 4461 .bb files complete
NOTE: Resolving any missing task queue dependencies
Initialising tasks: 100%
Checking sstate mirror object availability: 100%
Sstate summary: Wanted 1945 Local 0 Network 1328 Missed 617 Current 0
NOTE: Executing Tasks
NOTE: Tasks Summary: Attempted 5162 tasks of which 1350 didn't need to be rerun and all succeeded.
Summary: There were 2 WARNING messages shown.
INFO: Failed to copy built images to tftp dir: /tftpboot
[INFO] Successfully built project
```

### 12.2 빌드 통계

**Task 통계:**
- 총 시도: 5162 tasks
- 재실행 불필요: 1350 tasks
- 모두 성공: 5162 tasks

**Sstate 캐시:**
- 필요: 1945
- 로컬: 0
- 네트워크: 1328 (68% match)
- 누락: 617

**Warning:**
- Ubuntu 22.04 미검증 (무시 가능)
- glibc 버전 불일치 (자동 처리됨)

---

## 13. 고급 활용

### 13.1 커스텀 Device Tree 수정

```bash
cd ~/projects/myproject/project-spec/meta-user/

# Device Tree 파일 생성
mkdir -p recipes-bsp/device-tree/files
vi recipes-bsp/device-tree/files/system-user.dtsi
```

**예제 - GPIO LED 추가:**
```dts
/include/ "system-conf.dtsi"
/ {
    gpio-leds {
        compatible = "gpio-leds";
        led0 {
            label = "led0";
            gpios = <&gpio0 7 0>;
            default-state = "off";
        };
    };
};
```

**재빌드:**
```bash
petalinux-build -c device-tree -x cleansstate
petalinux-build
```

### 13.2 커스텀 애플리케이션 추가

```bash
cd ~/projects/myproject

# 애플리케이션 생성
petalinux-create -t apps --name myapp --enable

# 소스 편집
vi project-spec/meta-user/recipes-apps/myapp/files/myapp.c
```

**간단한 Hello World:**
```c
#include <stdio.h>

int main(void) {
    printf("Hello from Zybo Z7-20!\n");
    return 0;
}
```

**빌드:**
```bash
petalinux-build -c myapp
petalinux-build
```

### 13.3 성능 최적화

```bash
# ~/.bashrc에 추가
export BB_NUMBER_THREADS="8"
export PARALLEL_MAKE="-j 8"

# 또는 프로젝트별 설정
vi ~/projects/myproject/project-spec/meta-user/conf/petalinuxbsp.conf

# 추가:
BB_NUMBER_THREADS = "8"
PARALLEL_MAKE = "-j 8"
```

---

## 14. 백업 및 복구

### 14.1 프로젝트 백업

```bash
# 전체 프로젝트 백업
cd ~/projects
tar czf myproject_backup_$(date +%Y%m%d).tar.gz myproject/

# Windows로 복사
cp myproject_backup_*.tar.gz /media/sf_SharedFolder/
```

### 14.2 이미지 백업

```bash
cd ~/projects/myproject/images/linux/

# 부트 파일 백업
mkdir -p ~/backups/zybo_boot_$(date +%Y%m%d)
cp BOOT.BIN image.ub boot.scr rootfs.tar.gz \
    ~/backups/zybo_boot_$(date +%Y%m%d)/

# WIC 이미지 백업
cp petalinux-sdimage.wic ~/backups/
```

### 14.3 복구

```bash
# 프로젝트 복구
cd ~/projects
tar xzf myproject_backup_YYYYMMDD.tar.gz

# 환경 설정
cd myproject
source ~/petalinux/2022.2/settings.sh

# 필요시 재빌드
petalinux-build
```

---

## 15. 자주 사용하는 명령어

### 15.1 PetaLinux 명령어

```bash
# 환경 설정
source ~/petalinux/2022.2/settings.sh

# 프로젝트 생성
petalinux-create -t project -n <이름> --template zynq

# 하드웨어 가져오기
petalinux-config --get-hw-description=<XSA 경로>

# 설정
petalinux-config                  # 시스템 설정
petalinux-config -c kernel       # 커널 설정
petalinux-config -c rootfs       # rootfs 설정
petalinux-config -c u-boot       # U-Boot 설정

# 빌드
petalinux-build                   # 전체 빌드
petalinux-build -c <컴포넌트>    # 특정 컴포넌트
petalinux-build -x clean          # 클린
petalinux-build -x mrproper       # 완전 클린

# 패키징
petalinux-package --boot          # BOOT.BIN 생성
petalinux-package --wic           # WIC 이미지 생성

# 부팅
petalinux-boot --qemu --kernel    # QEMU 에뮬레이션
```

### 15.2 Zybo Z7-20 시스템 명령어

```bash
# 시스템 정보
uname -a
cat /etc/os-release
cat /proc/cpuinfo

# 네트워크
ifconfig
ip addr
route -n
ping <IP>

# GPIO 제어
echo <번호> > /sys/class/gpio/export
echo out > /sys/class/gpio/gpio<번호>/direction
echo 1 > /sys/class/gpio/gpio<번호>/value

# 커널 모듈
lsmod
modprobe <모듈>
dmesg
```

---

## 16. 참고 자료

### 16.1 공식 문서

- **AMD/Xilinx PetaLinux**: https://docs.amd.com/
- **Zybo Z7 Reference**: https://digilent.com/reference/programmable-logic/zybo-z7/
- **Zynq-7000 TRM**: https://docs.amd.com/v/u/en-US/ug585-zynq-7000-trm

### 16.2 커뮤니티

- **Xilinx Forums**: https://support.xilinx.com/
- **Digilent Forums**: https://forum.digilent.com/
- **Stack Overflow**: Tag [petalinux], [zynq]

---

## 17. 최종 요약

### 전체 프로세스

```
1. VirtualBox + Ubuntu 22.04.5 설치
   ↓
2. 필수 패키지 설치
   ↓
3. PetaLinux 2022.2 설치
   ↓
4. Zybo Z7-20 프로젝트 생성
   ↓
5. XSA 하드웨어 설정
   ↓
6. 시스템/Rootfs 설정
   ↓
7. PetaLinux 빌드 (1-3시간)
   ↓
8. BOOT.BIN 생성
   ↓
9. WIC SD 이미지 생성
   ↓
10. balenaEtcher로 SD 카드 굽기
   ↓
11. Zybo Z7-20 부팅
   ↓
12. 로그인 (root/root)
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

# 빌드
petalinux-build

# BOOT.BIN 생성
petalinux-package --boot --fsbl images/linux/zynq_fsbl.elf \
    --fpga images/linux/design_1_wrapper.bit \
    --u-boot images/linux/u-boot.elf --force

# WIC 이미지 생성
petalinux-package --wic --bootfiles "BOOT.BIN image.ub boot.scr"

# Windows로 복사
cp images/linux/petalinux-sdimage.wic /media/sf_SharedFolder/
```

### 예상 소요 시간

| 작업 | 소요 시간 |
|------|-----------|
| VirtualBox + Ubuntu 설치 | 30-60분 |
| 패키지 설치 | 10-20분 |
| PetaLinux 설치 | 10-30분 |
| 프로젝트 생성 및 설정 | 10-20분 |
| 빌드 (첫 빌드) | 1-3시간 |
| 이미지 생성 및 SD 카드 | 10-20분 |
| **총 소요 시간** | **약 2-5시간** |

### Warning 요약

| Warning | 영향 | 조치 |
|---------|------|------|
| Ubuntu 22.04 미검증 | 없음 | 무시 |
| glibc 버전 불일치 | 없음 | 자동 처리 |
| TFTP 복사 실패 | 없음 | 선택적 |

---

## 18. 추가 리소스

### 18.1 유용한 링크

**PetaLinux 도구:**
- PetaLinux Tools Documentation: UG1144
- Embedded Design Tutorial: UG1165
- PetaLinux Command Line Reference: UG1157

**Zybo Z7-20 자료:**
- Schematic: Digilent 공식 사이트
- Constraint File (XDC): Digilent GitHub
- Example Projects: Digilent Reference

**Yocto/OpenEmbedded:**
- Yocto Project: https://www.yoctoproject.org/
- OpenEmbedded: https://www.openembedded.org/

### 18.2 지원 연락처

**Digilent 지원:**
- 이메일: support@digilentinc.com
- 포럼: https://forum.digilent.com/

**AMD/Xilinx 지원:**
- 지원 포털: https://support.amd.com/
- 커뮤니티: https://support.xilinx.com/

---

## 19. FAQ (자주 묻는 질문)

### Q1: 빌드에 얼마나 시간이 걸리나요?
**A:** 첫 빌드는 1-3시간, 이후 증분 빌드는 10-30분 소요됩니다.

### Q2: Warning 메시지가 나와도 괜찮나요?
**A:** 네, Ubuntu 22.04와 glibc 관련 Warning은 무시해도 됩니다. 빌드는 정상적으로 완료됩니다.

### Q3: TFTP 서버가 꼭 필요한가요?
**A:** 아니요, SD 카드 부팅만 사용한다면 필요 없습니다.

### Q4: SD 카드 크기는 얼마나 필요한가요?
**A:** 최소 4GB, 권장 8GB 이상입니다.

### Q5: 다른 Ubuntu 버전을 사용할 수 있나요?
**A:** 공식 지원은 Ubuntu 20.04지만, 22.04에서도 정상 작동합니다.

### Q6: 빌드 실패 시 어떻게 하나요?
**A:** 로그 확인 (`build/build.log`), 클린 빌드 시도 (`petalinux-build -x clean`), 디스크 공간 및 메모리 확인

### Q7: rootfs를 커스터마이징할 수 있나요?
**A:** 네, `petalinux-config -c rootfs`로 패키지 추가/제거 가능합니다.

### Q8: QEMU로 테스트할 수 있나요?
**A:** 네, `petalinux-boot --qemu --kernel` 명령어로 에뮬레이션 가능합니다.

---

## 20. 문서 정보

**작성일:** 2025년 9월 29일  
**버전:** 1.0  
**대상 하드웨어:** Digilent Zybo Z7-20 (Zynq-7020)  
**PetaLinux 버전:** 2022.2  
**호스트 OS:** Ubuntu 22.04.5 LTS (VirtualBox)  

**변경 이력:**
- v1.0 (2025-09-29): 초기 작성, Warning 해결 포함

---

## 부록 A: 전체 명령어 스크립트

### A.1 Ubuntu 준비 스크립트

```bash
#!/bin/bash
# Ubuntu 22.04 준비 스크립트

# 시스템 업데이트
sudo apt update
sudo apt upgrade -y

# 32비트 지원 추가
sudo dpkg --add-architecture i386
sudo apt update

# 필수 패키지 설치
sudo apt install -y \
    build-essential gcc-multilib g++-multilib gawk wget git \
    diffstat unzip texinfo chrpath socat cpio python3 \
    python3-pip python3-pexpect xz-utils debianutils \
    iputils-ping python3-git python3-jinja2 libegl1-mesa \
    libsdl1.2-dev pylint xterm rsync curl libncurses5-dev \
    libncursesw5-dev libssl-dev flex bison libselinux1 \
    gnupg zlib1g-dev libtool autoconf automake net-tools \
    screen pax gzip vim iproute2 locales libncurses5 libtinfo5

# 32비트 라이브러리
sudo apt install -y \
    libncurses5:i386 libc6:i386 libstdc++6:i386 \
    lib32z1 zlib1g:i386

# Locale 설정
sudo locale-gen en_US.UTF-8

# Dash를 Bash로 변경
echo "dash dash/sh boolean false" | sudo debconf-set-selections
sudo dpkg-reconfigure -f noninteractive dash

echo "Ubuntu 준비 완료!"
```

### A.2 PetaLinux 빌드 스크립트

```bash
#!/bin/bash
# PetaLinux 빌드 자동화 스크립트

# 환경 설정
source ~/petalinux/2022.2/settings.sh

# 프로젝트 디렉토리
PROJECT_DIR=~/projects/myproject

cd $PROJECT_DIR

# 빌드
echo "빌드 시작..."
petalinux-build

# BOOT.BIN 생성
echo "BOOT.BIN 생성..."
petalinux-package --boot \
    --fsbl images/linux/zynq_fsbl.elf \
    --fpga images/linux/design_1_wrapper.bit \
    --u-boot images/linux/u-boot.elf \
    --force

# WIC 이미지 생성
echo "WIC 이미지 생성..."
petalinux-package --wic \
    --bootfiles "BOOT.BIN image.ub boot.scr" \
    --images-dir images/linux/

# 완료
echo "빌드 완료!"
echo "이미지 위치: $PROJECT_DIR/images/linux/petalinux-sdimage.wic"
```

---

이 가이드를 따라하시면 Digilent Zybo Z7-20 보드용 PetaLinux 시스템을 성공적으로 구축할 수 있습니다. 빌드 과정에서 발생하는 Warning은 정상적이며 무시해도 됩니다. 

문제가 발생하면 트러블슈팅 섹션을 참고하시고, 추가 도움이 필요하면 Digilent 또는 Xilinx 포럼을 이용하시기 바랍니다.

**성공적인 개발을 기원합니다! 🚀**
