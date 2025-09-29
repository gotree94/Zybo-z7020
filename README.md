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
  - 이름: `share`
  - 경로: `C:\share` (Windows에 먼저 생성)
  - 마운트 지점: `/mnt/share`
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

### 1.4 공유 폴더 설정 및 권한

```bash
# 공유 폴더 마운트 포인트 생성
sudo mkdir -p /mnt/share

# 사용자를 vboxsf 그룹에 추가
sudo usermod -aG vboxsf $USER

# fstab에 자동 마운트 추가 (선택사항)
echo "share /mnt/share vboxsf defaults,uid=$(id -u),gid=$(id -g) 0 0" | sudo tee -a /etc/fstab

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
# TFTP 서버 설치 (네트워크 부팅용)
sudo apt install -y tftpd-hpa

# TFTP 디렉토리 생성 및 권한 설정
sudo mkdir -p /tftpboot
sudo chmod 777 /tftpboot
sudo chown nobody:nogroup /tftpboot

# TFTP 설정
sudo vi /etc/default/tftpd-hpa
```

**TFTP 설정 내용:**
```
TFTP_USERNAME="tftp"
TFTP_DIRECTORY="/tftpboot"
TFTP_ADDRESS=":69"
TFTP_OPTIONS="--secure"
```

```bash
# TFTP 서비스 시작
sudo systemctl restart tftpd-hpa
sudo systemctl enable tftpd-hpa
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
# C:\share에 복사한 후

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

**영구 설정 (권장):**
```bash
echo "source ~/petalinux/2022.2/settings.sh" >> ~/.bashrc
source ~/.bashrc
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
# design_1_wrapper.xsa를 Windows의 C:\share로 복사한 후
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

### 4.5 시스템 설정

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

#### **Yocto Settings**
```
Yocto Settings  --->
    YOCTO_MACHINE_NAME (zynq-generic)  --->
    [*] Enable auto resize SD card root filesystem
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

---

## 5. Root 로그인 설정 (중요!)

### 5.1 문제 이해

**기본 PetaLinux 설정의 문제점:**
```
myproject login: root
Password: (무엇을 입력해도)
Login incorrect
```

**원인:**
- PetaLinux 2022.2는 보안을 위해 기본적으로 빈 패스워드 로그인을 차단
- 하지만 실제 root 패스워드는 설정되지 않음
- 결과: 로그인 불가능

### 5.2 해결 방법 - Rootfs 설정 (필수!)

```bash
cd ~/projects/myproject

# Rootfs 설정 메뉴 열기
petalinux-config -c rootfs
```

**중요: 다음 항목들을 반드시 활성화해야 합니다!**

#### 방법 1: 자동 로그인 (가장 편함 - 개발용 권장)

```
Image Features  --->
    [*] debug-tweaks                    # ← 반드시 체크!
    [*] allow-empty-password            # ← 반드시 체크!
    [*] allow-root-login                # ← 반드시 체크!
    [*] empty-root-password             # ← 반드시 체크!
    [*] serial-autologin-root           # ← 자동 로그인 (권장)
```

**자동 로그인 활성화 시:**
- 부팅 후 패스워드 입력 없이 자동으로 root 로그인
- 개발 단계에서 가장 편리함

#### 방법 2: 수동 로그인 (패스워드 입력 없이 Enter만)

```
Image Features  --->
    [*] debug-tweaks                    # ← 반드시 체크!
    [*] allow-empty-password            # ← 반드시 체크!
    [*] allow-root-login                # ← 반드시 체크!
    [*] empty-root-password             # ← 반드시 체크!
    [ ] serial-autologin-root           # ← 비활성화
```

**수동 로그인 시:**
```
myproject login: root
Password: (그냥 Enter)
```

### 5.3 추가 패키지 설정 (선택사항)

로그인 설정을 하는 김에 유용한 패키지도 추가:

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
        [*] openssh-sftp-server
    
    misc  --->
        python3  --->
            [*] python3
            [*] python3-pip
```

### 5.4 설정 저장 및 확인

**저장:**
- `Save` 선택
- 기본 파일명 그대로 저장
- `Exit` 선택

**설정 확인:**
```bash
# 설정 파일 확인
cat ~/projects/myproject/project-spec/configs/rootfs_config | grep -i "debug\|empty\|autologin"

# 다음과 같은 출력이 있어야 함:
# CONFIG_debug-tweaks=y
# CONFIG_allow-empty-password=y
# CONFIG_empty-root-password=y
# CONFIG_serial-autologin-root=y  (활성화한 경우)
```

---

## 6. PetaLinux 빌드

### 6.1 전체 시스템 빌드

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

**참고: Warning 메시지에 대하여**
```
WARNING: Host distribution "ubuntu-22.04" has not been validated...
WARNING: Your host glibc version (2.35) is newer than that in uninative (2.34)...
```
- ✅ 이 경고들은 무시해도 됩니다
- ✅ Ubuntu 22.04에서 정상적으로 작동합니다
- ✅ 빌드가 성공했으면 문제 없습니다

### 6.2 부트 이미지 생성 (BOOT.BIN)

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
cp petalinux-sdimage.wic /mnt/share/

# 또는 압축 파일
cp petalinux-sdimage.wic.gz /mnt/share/

# 개별 부트 파일도 백업
mkdir -p /mnt/share/zybo_boot_files/
cp BOOT.BIN image.ub boot.scr /mnt/share/zybo_boot_files/
cp rootfs.tar.gz /mnt/share/zybo_boot_files/

# 동기화
sync
```

**Windows에서 확인:**
```
C:\share\petalinux-sdimage.wic
C:\share\zybo_boot_files\BOOT.BIN
C:\share\zybo_boot_files\image.ub
C:\share\zybo_boot_files\boot.scr
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
   - `C:\share\petalinux-sdimage.wic` 또는
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

## 9. Zybo Z7-20 부팅 및 로그인

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
Release 2022.2   Sep 22 2025 - 21:05:30
...

U-Boot 2022.01 (Sep 22 2025 - 21:05:30 +0000)
...

Starting kernel ...

[    0.000000] Booting Linux on physical CPU 0x0
[    0.000000] Linux version 5.15.36-xilinx-v2022.2
...

PetaLinux 2022.2_release_S10071807 myproject /dev/ttyPS0
```

### 9.4 로그인

#### 경우 1: 자동 로그인 활성화 (serial-autologin-root)

```
PetaLinux 2022.2_release_S10071807 myproject /dev/ttyPS0

myproject login: root (automatic login)

root@myproject:~#
```

- **자동으로 root 로그인됨**
- 패스워드 입력 불필요
- 바로 사용 가능!

#### 경우 2: 수동 로그인 (자동 로그인 비활성화)

```
PetaLinux 2022.2_release_S10071807 myproject /dev/ttyPS0

myproject login: root
Password: (그냥 Enter 키만 누름)

root@myproject:~#
```

- `root` 입력
- 패스워드는 **Enter만 누르기**
- 로그인 성공!

### 9.5 로그인 후 확인

```bash
# 호스트명 확인
hostname
# 출력: myproject

# 시스템 정보
uname -a
# 출력: Linux myproject 5.15.36-xilinx-v2022.2 #1 SMP PREEMPT...

# PetaLinux 버전
cat /etc/os-release
# 출력:
# NAME="PetaLinux"
# VERSION="2022.2"
# ID=petalinux
# PRETTY_NAME="PetaLinux 2022.2"

# 현재 사용자
whoami
# 출력: root

# 네트워크 인터페이스
ifconfig
# eth0, lo 확인

# 디스크 사용량
df -h

# 메모리 사용량
free -h
```

### 9.6 네트워크 설정

#### DHCP (자동)
```bash
# DHCP 클라이언트 실행
udhcpc -i eth0

# IP 확인
ifconfig eth0
# inet addr:192.168.1.xxx 확인

# ping 테스트
ping -c 3 8.8.8.8
```

#### 고정 IP (수동)
```bash
# 임시 설정
ifconfig eth0 192.168.1.100 netmask 255.255.255.0 up
route add default gw 192.168.1.1

# DNS 설정
echo "nameserver 8.8.8.8" > /etc/resolv.conf

# ping 테스트
ping -c 3 google.com
```

### 9.7 SSH 접속 (openssh 설치한 경우)

```bash
# Zybo Z7-20에서 SSH 서비스 시작
systemctl start sshd
systemctl enable sshd

# IP 주소 확인
ifconfig eth0

# Windows에서 접속
ssh root@192.168.1.xxx
# 패스워드: (Enter만 누르기)
```

---

## 10. 로그인 문제 해결

### 10.1 증상별 해결 방법

#### 문제 1: "Login incorrect" 오류

**증상:**
```
myproject login: root
Password: 
Login incorrect
```

**원인:**
- Rootfs 설정에서 `allow-empty-password`가 활성화되지 않음

**해결:**

**방법 A: 재빌드 (권장)**

```bash
cd ~/projects/myproject
source ~/petalinux/2022.2/settings.sh

# Rootfs 재설정
petalinux-config -c rootfs

# Image Features --->
#     [*] debug-tweaks
#     [*] allow-empty-password
#     [*] allow-root-login
#     [*] empty-root-password
#     [*] serial-autologin-root

# 저장 후 재빌드
petalinux-build -c rootfs -x cleansstate
petalinux-build

# 이미지 재생성
petalinux-package --boot --fsbl images/linux/zynq_fsbl.elf \
    --fpga images/linux/design_1_wrapper.bit \
    --u-boot images/linux/u-boot.elf --force

petalinux-package --wic --bootfiles "BOOT.BIN image.ub boot.scr"

# Windows로 복사
cp images/linux/petalinux-sdimage.wic /mnt/share/

# 새 이미지로 SD 카드 다시 굽기
```

**방법 B: SD 카드 직접 수정 (빠름)**

```bash
# Ubuntu에서 SD 카드 삽입
lsblk
# 예: /dev/sdb1 (BOOT), /dev/sdb2 (rootfs)

# rootfs 파티션 마운트
sudo mkdir -p /mnt/sd_rootfs
sudo mount /dev/sdb2 /mnt/sd_rootfs

# /etc/shadow 백업
sudo cp /mnt/sd_rootfs/etc/shadow /mnt/sd_rootfs/etc/shadow.backup

# root 패스워드 제거
sudo sed -i 's/^root:[^:]*:/root::/' /mnt/sd_rootfs/etc/shadow

# 확인
sudo cat /mnt/sd_rootfs/etc/shadow | grep root
# 출력: root::10933:0:99999:7:::
#             ↑ 여기가 비어있어야 함

# 동기화 및 언마운트
sync
sudo umount /mnt/sd_rootfs

# SD 카드를 Zybo Z7-20에 삽입하고 부팅
# Password: (Enter만 누르기)
```

#### 문제 2: 자동 로그인이 안됨

**증상:**
- `serial-autologin-root` 활성화했는데도 로그인 프롬프트 표시

**원인:**
- systemd 설정이 제대로 적용되지 않음

**해결:**

```bash
cd ~/projects/myproject

# 설정 확인
cat project-spec/configs/rootfs_config | grep serial-autologin

# 없다면 다시 설정
petalinux-config -c rootfs

# Image Features --->
#     [*] serial-autologin-root

# 완전히 클린 빌드
petalinux-build -x mrproper
petalinux-config --get-hw-description=~/projects/
petalinux-config -c rootfs
# (설정 다시 확인)

petalinux-build
```

#### 문제 3: 시리얼 콘솔에 아무것도 안 나옴

**증상:**
- PuTTY/Tera Term에 아무 출력이 없음

**해결:**

1. **COM 포트 확인**
   ```
   - Windows 장치 관리자
   - 다른 COM 포트 번호 시도
   - USB 케이블 재연결
   ```

2. **시리얼 설정 확인**
   ```
   Baud Rate: 115200
   Data bits: 8
   Parity: None
   Stop bits: 1
   Flow control: None
   ```

3. **FTDI 드라이버 재설치**
   ```
   - FTDI 공식 사이트에서 최신 드라이버 다운로드
   - 기존 드라이버 제거 후 재설치
   ```

4. **Zybo Z7-20 UART 연결 확인**
   ```
   - J14 포트에 올바르게 연결
   - USB 케이블이 데이터 전송 지원하는지 확인
   ```

#### 문제 4: 패스워드 입력이 안됨

**증상:**
- Password: 프롬프트에서 키보드 입력이 안됨

**원인:**
- 시리얼 콘솔 설정 문제

**해결:**

```bash
# PuTTY 설정 확인
Terminal --->
    Keyboard --->
        The Backspace key: Control-H
        The Home and End keys: Standard
    
Connection --->
    Serial --->
        Flow control: None (중요!)
```

#### 문제 5: SSH 로그인 실패

**증상:**
- 네트워크는 연결되었으나 SSH 접속 안됨

**해결:**

```bash
# Zybo Z7-20에서 (시리얼 콘솔)

# SSH 서비스 상태 확인
systemctl status sshd

# SSH 서비스 시작
systemctl start sshd
systemctl enable sshd

# SSH 설정 확인
vi /etc/ssh/sshd_config

# 다음 설정 확인:
PermitRootLogin yes
PasswordAuthentication yes
PermitEmptyPasswords yes

# SSH 재시작
systemctl restart sshd

# 방화벽 확인 (있다면)
iptables -L -n
```

**빌드 시 SSH 설정:**

```bash
petalinux-config -c rootfs

# Filesystem Packages --->
#     network --->
#         [*] openssh
#         [*] openssh-sshd
#         [*] openssh-sftp-server
#
# Image Features --->
#     [*] ssh-server-openssh
```

---

## 11. 트러블슈팅

### 11.1 빌드 관련 문제

#### 메모리 부족

**증상:**
```
ERROR: Task xxx failed
Virtual memory exhausted
```

**해결:**
```bash
# 스왑 파일 생성
sudo fallocate -l 8G /swapfile
sudo chmod 600 /swapfile
sudo mkswap /swapfile
sudo swapon /swapfile

# 영구 설정
echo '/swapfile none swap sw 0 0' | sudo tee -a /etc/fstab

# 확인
free -h
```

#### 디스크 공간 부족

**증상:**
```
No space left on device
```

**해결:**
```bash
# 디스크 사용량 확인
df -h

# 빌드 캐시 정리
cd ~/projects/myproject
petalinux-build -x clean

# 다운로드 캐시 정리 (조심!)
rm -rf build/downloads/*

# tmp 디렉토리 정리
rm -rf build/tmp/

# 완전 클린 (주의: 재빌드 시간 증가)
petalinux-build -x mrproper
```

#### 특정 패키지 빌드 실패

**증상:**
```
ERROR: Task do_compile failed for xxx
```

**해결:**
```bash
# 로그 확인
cd ~/projects/myproject
find build/tmp/work -name "log.do_compile*" | xargs tail

# 해당 패키지만 재빌드
petalinux-build -c <패키지명> -x cleansstate
petalinux-build -c <패키지명>

# 예: U-Boot 재빌드
petalinux-build -c u-boot -x cleansstate
petalinux-build -c u-boot
```

### 11.2 부팅 관련 문제

#### U-Boot까지만 부팅되고 멈춤

**증상:**
```
U-Boot 2022.01
...
ZynqMP>
```

**해결:**
```bash
# U-Boot 콘솔에서 수동 부팅
ZynqMP> fatload mmc 0 0x2000000 image.ub
ZynqMP> bootm 0x2000000

# boot.scr 확인
# Ubuntu에서:
cd ~/projects/myproject/images/linux/
file boot.scr
# "boot.scr: u-boot legacy uImage" 출력 확인

# boot.scr 재생성
mkimage -A arm -O linux -T script -C none \
    -a 0 -e 0 -n "Boot Script" \
    -d boot.cmd boot.scr

# SD 카드에 다시 복사
```

#### Kernel panic 발생

**증상:**
```
Kernel panic - not syncing: VFS: Unable to mount root fs
```

**해결:**
```bash
# rootfs 파티션 확인
# SD 카드를 Ubuntu에 삽입
sudo fdisk -l /dev/sdb

# 파티션 2가 존재하는지 확인
# 파티션 2를 마운트
sudo mount /dev/sdb2 /mnt/sd_rootfs
ls /mnt/sd_rootfs/
# bin, etc, lib 등이 있어야 함

# rootfs가 비어있다면 다시 압축 해제
sudo rm -rf /mnt/sd_rootfs/*
sudo tar xzf ~/projects/myproject/images/linux/rootfs.tar.gz \
    -C /mnt/sd_rootfs/

sync
sudo umount /mnt/sd_rootfs
```

#### SD 카드 인식 안됨

**증상:**
```
mmc0: error -110 whilst initialising SD card
```

**해결:**
1. **다른 SD 카드 시도**
   - Class 10 이상 권장
   - 32GB 이하 권장 (SDHC)
   - 신뢰할 수 있는 브랜드 사용

2. **SD 카드 포맷**
   ```bash
   # Ubuntu에서
   sudo fdisk -l  # SD 카드 장치 확인 (/dev/sdb)
   
   # 완전 포맷
   sudo dd if=/dev/zero of=/dev/sdb bs=1M count=100
   
   # 이미지 다시 굽기
   ```

3. **부트 모드 점퍼 확인**
   ```
   JP5가 SD 위치에 있는지 확인
   ```

### 11.3 네트워크 관련 문제

#### 이더넷 링크 안됨

**증상:**
```bash
ifconfig eth0
# eth0: flags=4098<BROADCAST,MULTICAST>
# UP 플래그가 없음
```

**해결:**
```bash
# 인터페이스 활성화
ifconfig eth0 up

# 링크 상태 확인
ethtool eth0
# Link detected: yes 확인

# 케이블 연결 확인
# LED 확인 (Zybo Z7-20의 이더넷 포트)
```

#### DHCP 실패

**증상:**
```bash
udhcpc -i eth0
# udhcpc: no lease, failing
```

**해결:**
```bash
# DHCP 서버 확인 (공유기 등)
# 고정 IP로 수동 설정
ifconfig eth0 192.168.1.100 netmask 255.255.255.0 up
route add default gw 192.168.1.1

# ping 테스트
ping -c 3 192.168.1.1
ping -c 3 8.8.8.8
```

### 11.4 공유 폴더 문제

#### /mnt/share 마운트 안됨

**증상:**
```bash
ls /mnt/share
# ls: cannot access '/mnt/share': No such file or directory
```

**해결:**
```bash
# 디렉토리 생성
sudo mkdir -p /mnt/share

# VirtualBox Guest Additions 확인
lsmod | grep vbox

# 수동 마운트
sudo mount -t vboxsf share /mnt/share

# 권한 설정
sudo mount -t vboxsf -o uid=$(id -u),gid=$(id -g) share /mnt/share

# fstab 확인
cat /etc/fstab | grep share

# 없다면 추가
echo "share /mnt/share vboxsf defaults,uid=$(id -u),gid=$(id -g) 0 0" | \
    sudo tee -a /etc/fstab

# 재부팅
sudo reboot
```

---

## 12. 체크리스트

### 12.1 설치 전 체크리스트

- [ ] VirtualBox 설치 완료
- [ ] Ubuntu 22.04.5 ISO 다운로드
- [ ] 충분한 디스크 공간 (200GB+)
- [ ] 충분한 RAM (16GB+)
- [ ] Windows에 C:\share 폴더 생성
- [ ] petalinux-v2022.2-10141622-installer.run 다운로드
- [ ] design_1_wrapper.xsa 준비

### 12.2 설정 전 체크리스트

- [ ] VirtualBox 공유 폴더 설정 완료 (/mnt/share)
- [ ] Guest Additions 설치 완료
- [ ] 모든 필수 패키지 설치 완료
- [ ] PetaLinux 환경 활성화 확인

### 12.3 빌드 전 체크리스트 (중요!)

- [ ] XSA 파일 복사 완료
- [ ] 프로젝트 생성 완료
- [ ] 하드웨어 설정 가져오기 완료
- [ ] **Rootfs 설정 완료 (로그인 설정!)**
  - [ ] debug-tweaks 활성화
  - [ ] allow-empty-password 활성화
  - [ ] allow-root-login 활성화
  - [ ] empty-root-password 활성화
  - [ ] serial-autologin-root 활성화 (권장)
- [ ] 충분한 빌드 시간 확보 (1-3시간)
- [ ] 안정적인 인터넷 연결

### 12.4 SD 카드 굽기 전 체크리스트

- [ ] 빌드 성공 확인
- [ ] WIC 이미지 생성 완료
- [ ] 이미지 파일 Windows로 복사 완료
- [ ] balenaEtcher 설치
- [ ] SD 카드 준비 (4GB+, Class 10+)
- [ ] SD 카드 리더기 연결

### 12.5 부팅 전 체크리스트

- [ ] SD 카드 굽기 완료
- [ ] JP5 점퍼 SD 모드로 설정
- [ ] SD 카드 Zybo Z7-20에 삽입
- [ ] UART 케이블 연결 (J14)
- [ ] FTDI 드라이버 설치
- [ ] 시리얼 콘솔 설정 (115200 8N1)
- [ ] COM 포트 번호 확인
- [ ] 전원 준비

---

## 13. 빌드 출력 분석

### 13.1 정상 빌드 출력

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

### 13.2 Warning 메시지 설명

#### Warning 1: Ubuntu 22.04 미검증
```
WARNING: Host distribution "ubuntu-22.04" has not been validated...
```
- ✅ **무시해도 됨**
- PetaLinux 2022.2는 공식적으로 Ubuntu 20.04 지원
- Ubuntu 22.04에서도 정상 작동
- 빌드 성공했으면 문제 없음

#### Warning 2: glibc 버전 불일치
```
WARNING: Your host glibc version (2.35) is newer than that in uninative (2.34)
```
- ✅ **무시해도 됨**
- Yocto가 자동으로 처리
- 빌드 성공에 영향 없음

#### Info: TFTP 복사 실패
```
INFO: Failed to copy built images to tftp dir: /tftpboot
```
- ✅ **무시해도 됨**
- SD 카드 부팅에는 영향 없음
- TFTP 네트워크 부팅을 사용하지 않으면 필요 없음

---

## 14. 고급 활용

### 14.1 커스텀 애플리케이션 추가

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
    printf("PetaLinux 2022.2\n");
    return 0;
}
```

**빌드:**
```bash
petalinux-build -c myapp
petalinux-build

# Zybo Z7-20에서 실행:
# /usr/bin/myapp
```

### 14.2 GPIO LED 제어

```bash
# Zybo Z7-20에서 (부팅 후)

# LED0 제어 (MIO7 예시)
echo 7 > /sys/class/gpio/export
echo out > /sys/class/gpio/gpio7/direction
echo 1 > /sys/class/gpio/gpio7/value  # LED ON
sleep 1
echo 0 > /sys/class/gpio/gpio7/value  # LED OFF
echo 7 > /sys/class/gpio/unexport
```

### 14.3 성능 최적화

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

## 15. 백업 및 복구

### 15.1 프로젝트 백업

```bash
# 전체 프로젝트 백업
cd ~/projects
tar czf myproject_backup_$(date +%Y%m%d).tar.gz myproject/

# /mnt/share로 복사
cp myproject_backup_*.tar.gz /mnt/share/
```

### 15.2 이미지 백업

```bash
cd ~/projects/myproject/images/linux/

# 부트 파일 백업
mkdir -p /mnt/share/zybo_backup_$(date +%Y%m%d)
cp BOOT.BIN image.ub boot.scr rootfs.tar.gz \
    /mnt/share/zybo_backup_$(date +%Y%m%d)/

# WIC 이미지 백업
cp petalinux-sdimage.wic /mnt/share/

# 동기화
sync
```

### 15.3 복구

```bash
# 프로젝트 복구
cd ~/projects
tar xzf /mnt/share/myproject_backup_YYYYMMDD.tar.gz

# 환경 설정
cd myproject
source ~/petalinux/2022.2/settings.sh

# 필요시 재빌드
petalinux-build
```

---

## 16. 자주 사용하는 명령어

### 16.1 PetaLinux 명령어

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
petalinux-config -c rootfs       # rootfs 설정 (로그인!)
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

### 16.2 Zybo Z7-20 시스템 명령어

```bash
# 시스템 정보
uname -a
cat /etc/os-release
hostname
whoami

# 네트워크
ifconfig
ip addr
route -n
ping <IP>

# 파일시스템
df -h
mount
lsblk

# 프로세스
ps aux
top

# GPIO
echo <번호> > /sys/class/gpio/export
echo out > /sys/class/gpio/gpio<번호>/direction
echo 1 > /sys/class/gpio/gpio<번호>/value

# 로그
dmesg
journalctl
```

---

## 17. FAQ (자주 묻는 질문)

### Q1: Root 패스워드가 무엇인가요?
**A:** PetaLinux 2022.2는 기본적으로 패스워드가 설정되지 않습니다. 
- `allow-empty-password` 활성화 시: Enter만 누르기
- `serial-autologin-root` 활성화 시: 자동 로그인

### Q2: "Login incorrect" 오류가 나옵니다.
**A:** Rootfs 설정에서 다음을 활성화하세요:
- debug-tweaks
- allow-empty-password
- allow-root-login
- empty-root-password

### Q3: 빌드에 얼마나 시간이 걸리나요?
**A:** 첫 빌드는 1-3시간, 이후 증분 빌드는 10-30분 소요됩니다.

### Q4: Ubuntu 22.04 Warning이 걱정됩니다.
**A:** 무시해도 됩니다. Ubuntu 22.04에서 정상적으로 작동합니다.

### Q5: SD 카드 크기는 얼마나 필요한가요?
**A:** 최소 4GB, 권장 8GB 이상입니다.

### Q6: SSH로 접속할 수 있나요?
**A:** 네, openssh 패키지를 rootfs에 추가하고 빌드하면 됩니다.

### Q7: 자동 로그인이 보안상 걱정됩니다.
**A:** 개발 완료 후 다음을 비활성화하세요:
- debug-tweaks
- serial-autologin-root
- allow-empty-password
그리고 `passwd root`로 강력한 패스워드 설정

### Q8: 공유 폴더가 마운트되지 않습니다.
**A:** 
1. Guest Additions 설치 확인
2. vboxsf 그룹에 사용자 추가
3. `sudo mount -t vboxsf share /mnt/share`

---

## 18. 참고 자료

### 18.1 공식 문서

- **AMD/Xilinx PetaLinux**: https://docs.amd.com/
- **Zybo Z7 Reference**: https://digilent.com/reference/programmable-logic/zybo-z7/
- **Zynq-7000 TRM**: https://docs.amd.com/v/u/en-US/ug585-zynq-7000-trm

### 18.2 커뮤니티

- **Xilinx Forums**: https://support.xilinx.com/
- **Digilent Forums**: https://forum.digilent.com/
- **Stack Overflow**: Tag [petalinux], [zynq]

---

## 19. 최종 요약

### 전체 프로세스

```
1. VirtualBox + Ubuntu 22.04.5 설치
2. 공유 폴더 설정 (/mnt/share)
3. 필수 패키지 설치
4. PetaLinux 2022.2 설치
5. Zybo Z7-20 프로젝트 생성
6. XSA 하드웨어 설정
7. ⭐ Root 로그인 설정 (중요!)
8. PetaLinux 빌드 (1-3시간)
9. BOOT.BIN 생성
10. WIC SD 이미지 생성
11. balenaEtcher로 SD 카드 굽기
12. Zybo Z7-20 부팅
13. Root 로그인 (자동 또는 Enter)
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

| 작업 | 소요 시간 |
|------|-----------|
| VirtualBox + Ubuntu 설치 | 30-60분 |
| 패키지 설치 | 10-20분 |
| PetaLinux 설치 | 10-30분 |
| 프로젝트 생성 및 설정 | 10-20분 |
| 빌드 (첫 빌드) | 1-3시간 |
| 이미지 생성 및 SD 카드 | 10-20분 |
| **총 소요 시간** | **약 2-5시간** |

### 중요 사항 정리

✅ **반드시 해야 할 것**
1. Rootfs 설정에서 로그인 옵션 활성화
   - debug-tweaks
   - allow-empty-password
   - serial-autologin-root (권장)
2. 빌드 전 설정 확인
3. Ubuntu 22.04 Warning은 무시

❌ **하지 말아야 할 것**
1. Rootfs 로그인 설정 빠뜨리기
2. Ubuntu 버전 다운그레이드
3. 빌드 중 강제 종료

---

## 20. 문서 정보

**작성일:** 2025년 9월 29일  
**버전:** 2.0 (최종판)  
**대상 하드웨어:** Digilent Zybo Z7-20 (Zynq-7020)  
**PetaLinux 버전:** 2022.2  
**호스트 OS:** Ubuntu 22.04.5 LTS (VirtualBox)  
**공유 폴더:** /mnt/share

**변경 이력:**
- v2.0 (2025-09-29): Root 로그인 문제 완전 해결, 공유 폴더 /mnt/share로 통일
- v1.0 (2025-09-29): 초기 작성

---

## 부록 A: 전체 자동화 스크립트

### A.1 Ubuntu 준비 스크립트

**파일: setup_ubuntu.sh**

```bash
#!/bin/bash
# Ubuntu 22.04 PetaLinux 준비 스크립트

echo "====================================="
echo "Ubuntu 22.04 PetaLinux 환경 준비"
echo "====================================="

# 시스템 업데이트
echo "[1/7] 시스템 업데이트..."
sudo apt update
sudo apt upgrade -y

# 32비트 지원 추가
echo "[2/7] 32비트 라이브러리 지원 추가..."
sudo dpkg --add-architecture i386
sudo apt update

# 필수 패키지 설치
echo "[3/7] 필수 패키지 설치..."
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
echo "[4/7] 32비트 라이브러리 설치..."
sudo apt install -y \
    libncurses5:i386 libc6:i386 libstdc++6:i386 \
    lib32z1 zlib1g:i386

# Locale 설정
echo "[5/7] Locale 설정..."
sudo locale-gen en_US.UTF-8
sudo update-locale LANG=en_US.UTF-8

# Dash를 Bash로 변경
echo "[6/7] Dash를 Bash로 변경..."
echo "dash dash/sh boolean false" | sudo debconf-set-selections
sudo dpkg-reconfigure -f noninteractive dash

# 공유 폴더 설정
echo "[7/7] 공유 폴더 설정..."
sudo mkdir -p /mnt/share
sudo usermod -aG vboxsf $USER

echo ""
echo "====================================="
echo "설치 완료!"
echo "====================================="
echo "재부팅 후 다음 명령어로 확인하세요:"
echo "  ls -la /mnt/share"
echo ""
echo "재부팅하려면: sudo reboot"
```

**사용 방법:**
```bash
chmod +x setup_ubuntu.sh
./setup_ubuntu.sh
sudo reboot
```

### A.2 PetaLinux 프로젝트 생성 스크립트

**파일: create_petalinux_project.sh**

```bash
#!/bin/bash
# PetaLinux 프로젝트 생성 및 설정 스크립트

# 설정
PROJECT_NAME="myproject"
PROJECT_DIR="$HOME/projects"
XSA_FILE="/mnt/share/design_1_wrapper.xsa"

echo "====================================="
echo "PetaLinux 프로젝트 생성"
echo "====================================="

# PetaLinux 환경 활성화
echo "[1/4] PetaLinux 환경 활성화..."
source ~/petalinux/2022.2/settings.sh

# 프로젝트 디렉토리 생성
echo "[2/4] 프로젝트 디렉토리 생성..."
mkdir -p $PROJECT_DIR
cd $PROJECT_DIR

# XSA 파일 복사
echo "[3/4] XSA 파일 복사..."
if [ -f "$XSA_FILE" ]; then
    cp $XSA_FILE $PROJECT_DIR/
    echo "XSA 파일 복사 완료"
else
    echo "오류: $XSA_FILE 파일을 찾을 수 없습니다!"
    echo "C:\\share\\design_1_wrapper.xsa 파일이 있는지 확인하세요."
    exit 1
fi

# 프로젝트 생성
echo "[4/4] Zynq 프로젝트 생성..."
petalinux-create --type project --template zynq --name $PROJECT_NAME

echo ""
echo "====================================="
echo "프로젝트 생성 완료!"
echo "====================================="
echo "다음 단계:"
echo "  1. cd $PROJECT_DIR/$PROJECT_NAME"
echo "  2. petalinux-config --get-hw-description=$PROJECT_DIR/"
echo "  3. petalinux-config -c rootfs"
echo "     (로그인 설정 활성화!)"
echo "  4. petalinux-build"
```

**사용 방법:**
```bash
chmod +x create_petalinux_project.sh
./create_petalinux_project.sh
```

### A.3 PetaLinux 빌드 스크립트

**파일: build_petalinux.sh**

```bash
#!/bin/bash
# PetaLinux 빌드 자동화 스크립트

PROJECT_DIR="$HOME/projects/myproject"

echo "====================================="
echo "PetaLinux 빌드 시작"
echo "====================================="

# 환경 설정
echo "[1/5] PetaLinux 환경 활성화..."
source ~/petalinux/2022.2/settings.sh

# 프로젝트 디렉토리로 이동
if [ ! -d "$PROJECT_DIR" ]; then
    echo "오류: $PROJECT_DIR 디렉토리를 찾을 수 없습니다!"
    exit 1
fi

cd $PROJECT_DIR

# 빌드 시작
echo "[2/5] 전체 빌드 시작..."
echo "빌드 시간: 약 1-3시간 소요"
START_TIME=$(date +%s)

petalinux-build

if [ $? -ne 0 ]; then
    echo "오류: 빌드 실패!"
    exit 1
fi

END_TIME=$(date +%s)
DURATION=$((END_TIME - START_TIME))
MINUTES=$((DURATION / 60))

echo "빌드 완료! (소요 시간: ${MINUTES}분)"

# BOOT.BIN 생성
echo "[3/5] BOOT.BIN 생성..."
petalinux-package --boot \
    --fsbl images/linux/zynq_fsbl.elf \
    --fpga images/linux/design_1_wrapper.bit \
    --u-boot images/linux/u-boot.elf \
    --force

# WIC 이미지 생성
echo "[4/5] WIC 이미지 생성..."
petalinux-package --wic \
    --bootfiles "BOOT.BIN image.ub boot.scr" \
    --images-dir images/linux/

# Windows로 복사
echo "[5/5] 이미지 파일 공유 폴더로 복사..."
cp images/linux/petalinux-sdimage.wic /mnt/share/
cp images/linux/BOOT.BIN /mnt/share/
cp images/linux/image.ub /mnt/share/
cp images/linux/boot.scr /mnt/share/

echo ""
echo "====================================="
echo "빌드 완료!"
echo "====================================="
echo "생성된 파일:"
echo "  - /mnt/share/petalinux-sdimage.wic"
echo "  - /mnt/share/BOOT.BIN"
echo "  - /mnt/share/image.ub"
echo "  - /mnt/share/boot.scr"
echo ""
echo "다음 단계:"
echo "  1. Windows에서 C:\\share\\petalinux-sdimage.wic 확인"
echo "  2. balenaEtcher로 SD 카드 굽기"
echo "  3. Zybo Z7-20 부팅"
```

**사용 방법:**
```bash
chmod +x build_petalinux.sh
./build_petalinux.sh
```

---

## 부록 B: 설정 파일 템플릿

### B.1 .bashrc 추가 설정

**파일: ~/.bashrc (끝에 추가)**

```bash
# PetaLinux 환경 설정
export PETALINUX_HOME=~/petalinux/2022.2
source $PETALINUX_HOME/settings.sh 2>/dev/null

# 빌드 성능 최적화
export BB_NUMBER_THREADS="8"
export PARALLEL_MAKE="-j 8"

# PetaLinux 단축 명령어
alias plconfig='petalinux-config'
alias plbuild='petalinux-build'
alias plclean='petalinux-build -x clean'
alias plrootfs='petalinux-config -c rootfs'
alias plkernel='petalinux-config -c kernel'

# 프로젝트 이동 단축키
alias cdpl='cd ~/projects/myproject'

# 공유 폴더 단축키
alias cdshare='cd /mnt/share'

echo "PetaLinux 환경이 활성화되었습니다."
```

### B.2 Rootfs 설정 요약

**필수 설정 항목 (petalinux-config -c rootfs):**

```
Image Features --->
    [*] debug-tweaks                    # 개발 편의 기능
    [*] allow-empty-password            # 빈 패스워드 허용
    [*] allow-root-login                # Root 로그인 허용
    [*] empty-root-password             # Root 빈 패스워드
    [*] serial-autologin-root           # 자동 로그인 (권장)
    [*] ssh-server-openssh              # SSH 서버 (선택)

Filesystem Packages --->
    admin --->
        [*] sudo
    
    console/utils --->
        [*] vim
        [*] nano
    
    network --->
        [*] openssh
        [*] openssh-sshd
        [*] openssh-sftp-server
```

---

## 부록 C: 빠른 참조 카드

### C.1 PetaLinux 주요 명령어

| 명령어 | 설명 |
|--------|------|
| `source ~/petalinux/2022.2/settings.sh` | 환경 활성화 |
| `petalinux-create -t project --template zynq -n <이름>` | 프로젝트 생성 |
| `petalinux-config --get-hw-description=<경로>` | 하드웨어 설정 |
| `petalinux-config` | 시스템 설정 |
| `petalinux-config -c rootfs` | Rootfs 설정 |
| `petalinux-config -c kernel` | 커널 설정 |
| `petalinux-build` | 전체 빌드 |
| `petalinux-build -c <컴포넌트>` | 특정 컴포넌트 빌드 |
| `petalinux-build -x clean` | 클린 빌드 |
| `petalinux-package --boot ...` | BOOT.BIN 생성 |
| `petalinux-package --wic ...` | WIC 이미지 생성 |

### C.2 Zybo Z7-20 하드웨어 정보

| 항목 | 사양 |
|------|------|
| FPGA | Zynq-7020 (XC7Z020-1CLG400C) |
| ARM CPU | Dual-core ARM Cortex-A9 @ 650MHz |
| 메모리 | 1GB DDR3 |
| 저장소 | SD 카드 슬롯 |
| 이더넷 | 10/100/1000 Mbps |
| USB | USB 2.0 OTG, USB-UART |
| HDMI | 입력/출력 |
| 오디오 | 헤드폰, 마이크 잭 |
| GPIO | Pmod 포트, Arduino 호환 |

### C.3 시리얼 콘솔 설정

| 설정 | 값 |
|------|-----|
| Baud Rate | 115200 |
| Data Bits | 8 |
| Parity | None |
| Stop Bits | 1 |
| Flow Control | None |

### C.4 부트 모드 점퍼

| 모드 | JP5 설정 |
|------|----------|
| SD 카드 | [SD] [  ] |
| JTAG | [  ] [JT] |
| QSPI | [QS] [  ] |

---

## 부록 D: 에러 코드 및 해결

### D.1 일반적인 빌드 에러

| 에러 메시지 | 원인 | 해결 방법 |
|------------|------|-----------|
| `Virtual memory exhausted` | 메모리 부족 | 스왑 파일 생성 |
| `No space left on device` | 디스크 부족 | 빌드 캐시 정리 |
| `Task do_compile failed` | 패키지 빌드 실패 | 로그 확인 후 해당 패키지 재빌드 |
| `Unable to find XSA` | XSA 파일 없음 | 경로 확인 |
| `License error` | 라이센스 동의 안함 | 인스톨러 재실행 |

### D.2 부팅 에러

| 증상 | 원인 | 해결 방법 |
|------|------|-----------|
| U-Boot에서 멈춤 | boot.scr 문제 | boot.scr 재생성 |
| Kernel panic | rootfs 마운트 실패 | rootfs 파티션 확인 |
| SD 카드 인식 안됨 | SD 카드 문제 | 다른 SD 카드 시도 |
| 시리얼 출력 없음 | COM 포트 설정 | COM 포트 및 설정 확인 |

### D.3 로그인 에러

| 증상 | 원인 | 해결 방법 |
|------|------|-----------|
| Login incorrect | 로그인 설정 누락 | rootfs 재설정 |
| 자동 로그인 안됨 | systemd 설정 | 완전 클린 빌드 |
| SSH 접속 안됨 | openssh 미설치 | rootfs에 openssh 추가 |

---

## 부록 E: 추가 리소스

### E.1 온라인 도구

- **balenaEtcher**: https://www.balena.io/etcher/
- **PuTTY**: https://www.putty.org/
- **Tera Term**: https://ttssh2.osdn.jp/
- **FTDI 드라이버**: https://ftdichip.com/drivers/vcp-drivers/

### E.2 유용한 링크

**AMD/Xilinx 문서:**
- PetaLinux Tools Documentation (UG1144)
- Embedded Design Tutorial (UG1165)
- Zynq-7000 TRM (UG585)

**Digilent 리소스:**
- Zybo Z7 Reference Manual
- Zybo Z7 Schematic
- Zybo Z7 Master XDC
- Example Projects (GitHub)

**커뮤니티:**
- r/FPGA (Reddit)
- Stack Overflow - [petalinux], [zynq] 태그
- Xilinx Community Forums
- Digilent Forums

### E.3 비디오 튜토리얼

검색 키워드:
- "PetaLinux Tutorial Zynq"
- "Zybo Z7 Getting Started"
- "Xilinx Embedded Linux"
- "PetaLinux SD Card Boot"

---

## 마무리

이 가이드를 따라하시면 Digilent Zybo Z7-20 보드에서 PetaLinux 2022.2를 성공적으로 실행할 수 있습니다.

### 핵심 포인트

1. ✅ **Ubuntu 22.04에서 정상 작동** - Warning 무시 가능
2. ⭐ **Root 로그인 설정 필수** - `petalinux-config -c rootfs`에서 설정
3. 📁 **공유 폴더 활용** - /mnt/share로 파일 전송
4. ⏱️ **충분한 시간 확보** - 첫 빌드는 1-3시간 소요
5. 💾 **정기적인 백업** - 프로젝트 및 이미지 백업

### 성공을 위한 팁

- 각 단계마다 확인하며 진행
- 로그인 설정을 절대 빠뜨리지 말 것
- 빌드 전에 충분한 디스크 공간 확보
- 문제 발생 시 로그 파일 확인
- 커뮤니티 활용

### 다음 단계

1. **기본 동작 확인** - LED, 버튼 테스트
2. **네트워크 설정** - 이더넷, SSH
3. **커스텀 애플리케이션 개발** - petalinux-create
4. **하드웨어 가속** - FPGA 로직 활용
5. **최종 제품 준비** - 보안 설정, 패스워드 설정

---

**행운을 빕니다! Happy Embedded Linux Development! 🚀**

---

*이 문서에 대한 질문이나 제안사항이 있으시면 언제든지 문의해주세요.*

*문서 버전: 2.0 (최종판)*  
*최종 업데이트: 2025년 9월 29일*
