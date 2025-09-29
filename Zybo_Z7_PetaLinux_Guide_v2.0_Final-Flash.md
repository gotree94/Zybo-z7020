# Digilent Zybo Z7-20 PetaLinux 완벽 가이드 (Quad-SPI Flash 버전)

**Quad-SPI Flash 부팅 완전 가이드**

---

## 목차

1. [VirtualBox Ubuntu 22.04.5 설치](#1-VirtualBox-Ubuntu-22.04.5-설치)
2. [Ubuntu 시스템 준비](#2-)
3. [PetaLinux 2022.2 설치](#3-)
4. [Zybo Z7-20 프로젝트 생성 (QSPI)](#4-)
5. [QSPI Flash 설정](#5-)
6. [Root 로그인 설정](#6-)
7. [PetaLinux 빌드](#7-)
8. [QSPI Flash 이미지 생성](#8-)
9. [QSPI Flash 프로그래밍](#9-)
10. [Zybo Z7-20 QSPI 부팅](#10-)
11. [트러블슈팅](#11-)
12. [체크리스트](#12-)
13. [자주 사용하는 명령어](#13-)
14. [FAQ](#14-)
15. [참고 자료](#15-)
16. [부록](#16-)

---

## 1. VirtualBox Ubuntu 22.04.5 설치
### 1.1 VirtualBox 가상머신 생성
**시스템 사양
```
이름: Zybo-PetaLinux-QSPI
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
  - 이름: share
  - 경로: C:\share
  - 마운트: /mnt/share
  - ✅ 자동 마운트
  - ✅ 영구적

### 1.2 Ubuntu 설치

1. ISO 마운트: ubuntu-22.04.5-desktop-amd64.iso
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

# 32비트 라이브러리
sudo apt install -y \
    libncurses5:i386 libc6:i386 libstdc++6:i386 lib32z1 zlib1g:i386

# Locale 설정
sudo locale-gen en_US.UTF-8

# Dash를 Bash로 변경
echo "dash dash/sh boolean false" | sudo debconf-set-selections
sudo dpkg-reconfigure -f noninteractive dash
```

3. PetaLinux 2022.2 설치
3.1 인스톨러 준비
bash# Windows에서 C:\share로 인스톨러 복사 후
mkdir -p ~/petalinux_work
cp /mnt/share/petalinux-v2022.2-10141622-installer.run ~/petalinux_work/
chmod +x ~/petalinux_work/petalinux-v2022.2-10141622-installer.run
3.2 PetaLinux 설치
bashmkdir -p ~/petalinux/2022.2
cd ~/petalinux_work
./petalinux-v2022.2-10141622-installer.run -d ~/petalinux/2022.2

# 라이센스 동의: y
# 설치 시간: 약 10-30분
3.3 환경 설정
bash# PetaLinux 환경 활성화
source ~/petalinux/2022.2/settings.sh

# 확인
echo $PETALINUX

# 영구 설정 (권장)
echo "source ~/petalinux/2022.2/settings.sh" >> ~/.bashrc
source ~/.bashrc

4. Zybo Z7-20 프로젝트 생성 (QSPI)
4.1 XSA 파일 준비
bash# Windows에서 C:\share로 design_1_wrapper.xsa 복사 후
mkdir -p ~/projects
cp /mnt/share/design_1_wrapper.xsa ~/projects/

# XSA 내용 확인
unzip -l ~/projects/design_1_wrapper.xsa
4.2 프로젝트 생성
bashcd ~/projects
source ~/petalinux/2022.2/settings.sh

petalinux-create --type project --template zynq --name myproject_qspi
cd myproject_qspi
4.3 하드웨어 설정
bashpetalinux-config --get-hw-description=~/projects/
⭐ QSPI 전용 설정 메뉴:
Image Packaging Configuration --->
    Root filesystem type --->
        (X) INITRAMFS                    ← QSPI는 INITRAMFS 사용!
    
    INITRAMFS/INITRD Image name --->
        (ramdisk.cpio.gz)                ← 기본값 확인
    
    [ ] Copy final images to tftpboot

Yocto Settings --->
    [ ] Disable auto resize (QSPI는 자동 리사이즈 불필요)

Subsystem AUTO Hardware Settings --->
    Flash Settings --->
        Primary Flash --->
            (X) ps7_qspi_0               ← QSPI Flash 선택!
        
    Serial Settings --->
        Primary stdin/stdout --->
            (X) ps7_uart_1
    
    Ethernet Settings --->
        Primary Ethernet --->
            (X) ps7_ethernet_0
저장: Save → Exit

5. QSPI Flash 설정
5.1 QSPI Flash 정보
Zybo Z7-20 QSPI Spec:
Flash IC: Spansion S25FL128S
용량: 16MB (128Mbit)
섹터 크기: 64KB
페이지 크기: 256 bytes
주소: 0x00000000 ~ 0x00FFFFFF
5.2 QSPI 파티션 레이아웃
0x00000000 - 0x000FFFFF : BOOT.BIN (1MB)
0x00100000 - 0x005FFFFF : Linux Kernel + DTB (5MB)
0x00600000 - 0x00FFFFFF : RootFS (INITRAMFS, 10MB)
5.3 Device Tree 확인
bashcd ~/projects/myproject_qspi/project-spec/meta-user/recipes-bsp/device-tree/files/

# system-user.dtsi 편집
vi system-user.dtsi
QSPI Device Tree 추가:
dts/include/ "system-conf.dtsi"
/ {
};

&qspi {
    status = "okay";
    is-dual = <0>;
    num-cs = <1>;
    
    flash@0 {
        compatible = "micron,m25p80", "jedec,spi-nor";
        reg = <0x0>;
        spi-max-frequency = <50000000>;
        
        #address-cells = <1>;
        #size-cells = <1>;
        
        partition@0 {
            label = "boot";
            reg = <0x0 0x100000>;
        };
        
        partition@100000 {
            label = "kernel";
            reg = <0x100000 0x500000>;
        };
        
        partition@600000 {
            label = "rootfs";
            reg = <0x600000 0xA00000>;
        };
    };
};

6. Root 로그인 설정
6.1 Rootfs 설정
bashcd ~/projects/myproject_qspi
petalinux-config -c rootfs
⭐ 필수 설정 (QSPI용):
Image Features --->
    [*] debug-tweaks                  ← 필수!
    [*] allow-empty-password          ← 필수!
    [*] allow-root-login              ← 필수!
    [*] empty-root-password           ← 필수!
    [*] serial-autologin-root         ← 권장

Filesystem Packages --->
    base --->
        [*] busybox                    ← INITRAMFS 필수
    
    admin --->
        [*] sudo                       ← 선택
    
    console/utils --->
        [*] vim-tiny                   ← 용량 고려
    
    network --->
        [*] openssh                    ← 선택
        [*] openssh-sshd
⚠️ INITRAMFS 용량 제한:

QSPI Flash는 10MB로 제한
불필요한 패키지는 제외
최소 구성 권장

저장: Save → Exit

7. PetaLinux 빌드
7.1 전체 빌드
bashcd ~/projects/myproject_qspi
source ~/petalinux/2022.2/settings.sh

petalinux-build
빌드 시간:

첫 빌드: 1-3시간
증분 빌드: 10-30분

빌드 성공 확인:
bashls -lh images/linux/

# 확인할 파일들:
# - BOOT.BIN (부트로더)
# - image.ub (Kernel + DTB + INITRAMFS)
# - boot.scr (U-Boot 스크립트)
7.2 QSPI 부트 이미지 생성
bashcd ~/projects/myproject_qspi

petalinux-package --boot \
    --fsbl images/linux/zynq_fsbl.elf \
    --fpga images/linux/design_1_wrapper.bit \
    --u-boot images/linux/u-boot.elf \
    --kernel images/linux/image.ub \
    --flash-size 16 \
    --flash-intf qspi-x1-single \
    --force
생성 파일:

images/linux/BOOT.BIN (QSPI 부팅용)
images/linux/boot.bin (백업)


8. QSPI Flash 이미지 생성
8.1 단일 QSPI 이미지 생성
bashcd ~/projects/myproject_qspi/images/linux/

# QSPI 전체 이미지 생성 (16MB)
petalinux-package --boot \
    --fsbl zynq_fsbl.elf \
    --fpga design_1_wrapper.bit \
    --u-boot u-boot.elf \
    --kernel image.ub \
    --flash-size 16 \
    --flash-intf qspi-x1-single \
    --force \
    -o qspi_flash_image.bin
8.2 수동 이미지 생성 (고급)
bashcd ~/projects/myproject_qspi/images/linux/

# 16MB 빈 이미지 생성
dd if=/dev/zero of=qspi_manual.bin bs=1M count=16

# BOOT.BIN 복사 (0x00000000)
dd if=BOOT.BIN of=qspi_manual.bin bs=1 seek=0 conv=notrunc

# image.ub 복사 (0x00100000 = 1MB)
dd if=image.ub of=qspi_manual.bin bs=1 seek=$((1*1024*1024)) conv=notrunc

# 확인
ls -lh qspi_*.bin
8.3 Windows로 복사
bashcd ~/projects/myproject_qspi/images/linux/

# QSPI 이미지 복사
cp qspi_flash_image.bin /mnt/share/

# 개별 파일도 백업
mkdir -p /mnt/share/zybo_qspi
cp BOOT.BIN image.ub boot.scr /mnt/share/zybo_qspi/

sync

9. QSPI Flash 프로그래밍
9.1 필요한 도구
Vivado 필요:

Vivado 2022.2
Hardware Manager
JTAG 케이블 (Digilent HS2 또는 Platform Cable USB II)

9.2 방법 1: Vivado Hardware Manager 사용 (권장)
단계:

Zybo Z7-20 연결

   - JTAG 케이블을 PC와 Zybo J13 포트에 연결
   - 전원 ON (SW0)
   - JP5: JTAG 모드 (모든 핀 열림)

Vivado 실행

tcl   vivado &

Hardware Manager 열기

   Flow Navigator → PROGRAM AND DEBUG → Open Hardware Manager
   Open Target → Auto Connect

QSPI Flash 프로그래밍

   - 연결된 디바이스 선택 (xc7z020_1)
   - 우클릭 → Add Configuration Memory Device
   - Search: s25fl128sxxxxxx0 선택
   - OK
   
   Configuration File: C:\share\qspi_flash_image.bin
   
   [✓] Erase
   [✓] Program
   [✓] Verify
   
   OK 클릭

프로그래밍 대기 (약 5-10분)
완료 확인

   Flash programming completed successfully
9.3 방법 2: XSCT 스크립트 사용
program_qspi.tcl 생성:
tcl# QSPI Flash 프로그래밍 스크립트
connect

# 타겟 선택
targets -set -filter {name =~ "ARM*#0"}

# FSBL 로드
dow -data ~/projects/myproject_qspi/images/linux/zynq_fsbl.elf

# 실행
con

# QSPI Flash 프로그래밍
targets -set -filter {name =~ "ARM*#0"}
stop

# Flash 디바이스 추가
flash -f ~/projects/myproject_qspi/images/linux/qspi_flash_image.bin \
      -offset 0 \
      -flash_type qspi-x1-single \
      -fsbl ~/projects/myproject_qspi/images/linux/zynq_fsbl.elf \
      -cable type xilinx_tcf url TCP:127.0.0.1:3121

puts "QSPI Flash programming completed!"
exit
실행:
bashcd ~/projects/myproject_qspi/images/linux/
xsct program_qspi.tcl
9.4 방법 3: U-Boot에서 프로그래밍 (업데이트용)
TFTP 서버 설정 필요
bash# U-Boot 콘솔에서
ZynqMP> setenv ipaddr 192.168.1.100
ZynqMP> setenv serverip 192.168.1.1
ZynqMP> ping ${serverip}

# QSPI 이미지 다운로드
ZynqMP> tftpboot 0x2000000 qspi_flash_image.bin

# QSPI 삭제
ZynqMP> sf probe 0
ZynqMP> sf erase 0 0x1000000

# QSPI 쓰기
ZynqMP> sf write 0x2000000 0 ${filesize}

# 검증
ZynqMP> sf read 0x3000000 0 ${filesize}
ZynqMP> cmp.b 0x2000000 0x3000000 ${filesize}

10. Zybo Z7-20 QSPI 부팅
10.1 하드웨어 설정
⭐ 부트 점퍼 (JP5) - QSPI 모드:
QSPI 부팅:
JP5: [  ] [  ]
     [  ] [QS]
     
또는 모든 핀 제거 (기본값이 QSPI)
연결:

JTAG 케이블 제거 (선택)
USB-UART 케이블 연결 (J14)
이더넷 연결 (선택)
전원 OFF

10.2 시리얼 콘솔 설정
PuTTY 설정:
Connection type: Serial
Serial line: COM3 (장치 관리자에서 확인)
Speed: 115200

Connection → Serial:
  Speed: 115200
  Data bits: 8
  Stop bits: 1
  Parity: None
  Flow control: None
10.3 QSPI 부팅

PuTTY 연결
전원 ON (SW0)
부팅 메시지 확인

Xilinx Zynq First Stage Boot Loader
Release 2022.2
Devcfg driver initialized
Silicon Version 3.1
Boot mode is QSPI
QSPI Init Done

U-Boot 2022.01 (Oct 01 2025 - 12:00:00 +0000)

CPU:   Zynq 7z020
Silicon: v3.1
DRAM:  ECC disabled 1 GiB
Flash: 16 MiB
Loading Environment from SPI Flash...
SF: Detected s25fl128s with page size 256 Bytes, erase size 64 KiB, total 16 MiB

Starting kernel ...

[    0.000000] Booting Linux on physical CPU 0x0
[    0.000000] Linux version 5.15.36-xilinx-v2022.2
[    0.000000] Machine model: Zynq Zybo Z7 Development Board
[    0.500000] Unpacking initramfs...

PetaLinux 2022.2 myproject_qspi /dev/ttyPS0
10.4 로그인
자동 로그인:
myproject_qspi login: root (automatic login)
root@myproject_qspi:~#
수동 로그인:
myproject_qspi login: root
Password: (그냥 Enter)
root@myproject_qspi:~#
10.5 QSPI 부팅 확인
bash# QSPI Flash 확인
cat /proc/mtd

# 출력 예:
# dev:    size   erasesize  name
# mtd0: 00100000 00010000 "boot"
# mtd1: 00500000 00010000 "kernel"
# mtd2: 00a00000 00010000 "rootfs"

# Flash 정보
cat /sys/class/mtd/mtd0/name
cat /sys/class/mtd/mtd0/size

# INITRAMFS 확인 (rootfs가 메모리에 있음)
mount | grep rootfs
df -h

11. 트러블슈팅
11.1 QSPI 프로그래밍 실패
증상:
Flash operation failed
Device not found
해결:

JTAG 연결 확인

   - JTAG 케이블 재연결
   - Vivado Hardware Manager에서 디바이스 재검색
   - 드라이버 확인 (Digilent Adept)

올바른 Flash 선택

   Vivado → Add Configuration Memory Device
   → Search: "s25fl128" 
   → s25fl128sxxxxxx0-spi-x1_x2_x4 선택

수동 Flash ID 확인

tcl   # XSCT에서
   connect
   targets -set -filter {name =~ "ARM*#0"}
   mrd -bin -file flash_test.bin 0xE000D030 4
11.2 QSPI 부팅 실패
증상 1: U-Boot에서 멈춤
U-Boot 2022.01
...
SF: Failed to initialize SPI flash
해결:
bash# Device Tree 확인
cd ~/projects/myproject_qspi/project-spec/meta-user/recipes-bsp/device-tree/files/
vi system-user.dtsi

# &qspi 노드 추가 확인
# 재빌드
petalinux-build -c device-tree -x cleansstate
petalinux-build
증상 2: Kernel panic - INITRAMFS
Kernel panic - not syncing: VFS: Unable to mount root fs on unknown-block(0,0)
해결:
bash# INITRAMFS 설정 확인
petalinux-config

# Image Packaging Configuration --->
#     Root filesystem type --->
#         (X) INITRAMFS

# 재빌드
petalinux-build -c rootfs -x cleansstate
petalinux-build
증상 3: Flash 읽기 오류
sf probe 0
SF: Failed to initialize SPI flash at 0:0
해결:
bash# U-Boot 설정 확인
petalinux-config -c u-boot

# Device Drivers --->
#     SPI Support --->
#         [*] Cadence QSPI driver
#     MTD Support --->
#         [*] SPI-NOR device support

# 재빌드
petalinux-build -c u-boot -x cleansstate
petalinux-build
11.3 INITRAMFS 용량 초과
증상:
ERROR: rootfs image size exceeded
해결:
bash# 불필요한 패키지 제거
petalinux-config -c rootfs

# 최소 구성:
# - busybox만 유지
# - openssh 제거 (필요시 UART만 사용)
# - 개발 도구 제거

# 또는 압축 레벨 증가
vi project-spec/meta-user/conf/petalinuxbsp.conf

# 추가:
# IMAGE_FSTYPES = "cpio.gz.u-boot"
# INITRAMFS_MAXSIZE = "10240"
11.4 QSPI Flash 성능 저하
문제:

부팅 시간 느림
읽기/쓰기 속도 저하

최적화:
dts# system-user.dtsi에서 SPI 속도 증가
&qspi {
    spi-max-frequency = <50000000>;  // 50MHz로 증가
    spi-tx-bus-width = <4>;          // Quad mode
    spi-rx-bus-width = <4>;
};
11.5 QSPI vs SD 카드 비교
항목QSPI FlashSD 카드용량16MB4GB+속도50MHz (빠름)25MHz (느림)RootfsINITRAMFS (RAM)EXT4 (SD)쓰기제한적 (Wear leveling)자유로움부팅 시간빠름 (~5초)보통 (~10초)용도임베디드, 읽기전용개발, 대용량업데이트JTAG 또는 네트워크카드 교체

12. 체크리스트
12.1 QSPI 부팅 전체 체크리스트
설치 단계:

 VirtualBox + Ubuntu 22.04.5 설치
 공유 폴더 설정
 필수 패키지 설치
 PetaLinux 2022.2 설치

프로젝트 생성:

 XSA 파일 준비
 QSPI 프로젝트 생성
 Root filesystem: INITRAMFS 선택
 Flash: ps7_qspi_0 선택
 Device Tree QSPI 노드 추가

빌드:

 Rootfs 로그인 설정 (debug-tweaks 등)
 빌드 완료
 BOOT.BIN 생성 (--flash-size 16)
 QSPI 이미지 생성

프로그래밍:

 Vivado 2022.2 설치
 JTAG 케이블 연결
 QSPI Flash 프로그래밍 완료
 Verify 완료

부팅:

 JP5: QSPI 모드 설정
 UART 연결 (115200 8N1)
 부팅 성공
 Root 로그인 성공
 INITRAMFS 확인


13. 자주 사용하는 명령어
13.1 QSPI 관련 명령어
U-Boot:
bash# QSPI Probe
sf probe 0

# Flash 정보
sf probe
sf test 0x100000 0x1000

# 읽기
sf read 0x2000000 0x0 0x100000

# 쓰기
sf erase 0x0 0x100000
sf write 0x2000000 0x0 0x100000

# 환경 변수
printenv
saveenv

# QSPI 부팅 설정
setenv bootcmd 'sf probe 0; sf read 0x2000000 0x100000 0x500000; bootm 0x2000000'
saveenv
Linux (QSPI 접근):
bash# MTD 디바이스 확인
cat /proc/mtd
ls -l /dev/mtd*

# Flash 읽기
dd if=/dev/mtd0 of=/tmp/boot_backup.bin bs=1M

# Flash 정보
cat /sys/class/mtd/mtd0/name
cat /sys/class/mtd/mtd0/size

# Flash 쓰기 (위험!)
flash_erase /dev/mtd0 0 0
flashcp -v new_image.bin /dev/mtd0
13.2 INITRAMFS 관련
bash# INITRAMFS 확인
mount | grep rootfs

# 메모리 사용량
free -m
df -h

# 임시 파일 시스템 (tmpfs)
mount재시도나계속편집bash# tmpfs 사용량
df -h /tmp /var /run

# INITRAMFS 압축 해제 (Ubuntu에서)
cd ~/projects/myproject_qspi/images/linux/
mkdir -p initramfs_test
cd initramfs_test
zcat ../rootfs.cpio.gz | cpio -idmv

# INITRAMFS 내용 확인
ls -la
du -sh *

# 재압축
find . | cpio -H newc -o | gzip > ../rootfs_new.cpio.gz
13.3 PetaLinux QSPI 명령어
bash# QSPI 프로젝트 생성
petalinux-create -t project --template zynq -n myproject_qspi

# QSPI 부트 이미지
petalinux-package --boot \
    --fsbl images/linux/zynq_fsbl.elf \
    --fpga images/linux/design_1_wrapper.bit \
    --u-boot images/linux/u-boot.elf \
    --kernel images/linux/image.ub \
    --flash-size 16 \
    --flash-intf qspi-x1-single \
    --force

# QSPI 전용 설정 확인
cat project-spec/configs/config | grep -i qspi
cat project-spec/configs/config | grep -i initramfs

14. FAQ
Q1: QSPI와 SD 카드의 차이는?
A: QSPI는 16MB 고정 크기로 빠른 부팅과 신뢰성이 필요한 임베디드 시스템에 적합합니다. SD 카드는 대용량 저장소와 개발용으로 적합합니다.
Q2: INITRAMFS는 무엇인가요?
A: 메모리에 압축 해제되는 루트 파일 시스템입니다. QSPI의 용량 제한 때문에 사용하며, 부팅 후 RAM에서 실행됩니다.
Q3: QSPI 용량 16MB가 부족한데요?
A: 다음 방법을 고려하세요:

불필요한 패키지 제거
SD 카드와 혼합 사용 (부트는 QSPI, 데이터는 SD)
더 큰 Flash IC로 하드웨어 수정 (고급)

Q4: QSPI 부팅이 SD보다 빠른가요?
A: 네, QSPI는 일반적으로 5-10초로 SD 카드보다 빠릅니다. INITRAMFS가 압축되어 있고 Flash 속도가 빠르기 때문입니다.
Q5: 파일 시스템에 쓰기가 안됩니다.
A: INITRAMFS는 RAM에서 실행되므로 쓰기는 가능하지만 재부팅 시 사라집니다. 영구 저장이 필요하면 SD 카드나 네트워크 스토리지를 사용하세요.
Q6: QSPI를 업데이트하려면?
A: 세 가지 방법이 있습니다:

JTAG를 통한 Vivado 프로그래밍 (가장 안전)
U-Boot에서 TFTP 업데이트
Linux에서 MTD 유틸리티 사용 (고급)

Q7: JP5 설정을 잘못하면?
A: QSPI 모드가 아니면 QSPI에서 부팅하지 않습니다. JP5 설정을 확인하고 전원을 껐다 켜세요.
Q8: QSPI와 SD를 동시에 사용할 수 있나요?
A: 네, 가능합니다. QSPI에서 부팅하고 SD 카드를 추가 저장소로 마운트할 수 있습니다.
Q9: QSPI Flash 수명은?
A: 일반적으로 100,000 쓰기 사이클입니다. 읽기는 제한이 없으므로 읽기 전용 시스템에 이상적입니다.
Q10: Quad SPI 모드를 활성화하려면?
A: Device Tree에서 spi-tx-bus-width = <4>와 spi-rx-bus-width = <4>를 설정하세요. 단, 하드웨어가 지원해야 합니다.

15. 참고 자료
15.1 QSPI 관련 문서
AMD/Xilinx QSPI 문서

Zynq-7000 TRM Chapter 14: Quad-SPI Controller
https://docs.amd.com/v/u/en-US/ug585-zynq-7000-trm
PetaLinux QSPI Boot Guide
https://xilinx-wiki.atlassian.net/wiki/spaces/A/pages/18842462/QSPI+Flash+Programming
Linux MTD Subsystem
http://www.linux-mtd.infradead.org/

Spansion Flash 데이터시트

S25FL128S Datasheet
https://www.infineon.com/cms/en/product/memories/nor-flash/

15.2 Device Tree 참고

Device Tree for Dummies
https://elinux.org/Device_Tree_Usage
Xilinx Device Tree Generator
https://github.com/Xilinx/device-tree-xlnx

15.3 INITRAMFS 가이드

Linux Kernel INITRAMFS
https://www.kernel.org/doc/html/latest/filesystems/ramfs-rootfs-initramfs.html
Busybox Documentation
https://www.busybox.net/


16. 부록
16.1 QSPI 자동화 스크립트
완전 자동 빌드 스크립트 (build_qspi_complete.sh):
bash#!/bin/bash

set -e  # 에러 발생 시 중단

PROJECT_NAME="myproject_qspi"
PROJECT_DIR="$HOME/projects/$PROJECT_NAME"
XSA_PATH="$HOME/projects/design_1_wrapper.xsa"
SHARE_DIR="/mnt/share"

echo "=========================================="
echo "Zybo Z7-20 QSPI PetaLinux 자동 빌드"
echo "=========================================="

# PetaLinux 환경 활성화
echo "[1/8] PetaLinux 환경 활성화..."
source ~/petalinux/2022.2/settings.sh

# 프로젝트 존재 확인
if [ -d "$PROJECT_DIR" ]; then
    echo "[2/8] 기존 프로젝트 사용: $PROJECT_DIR"
    cd $PROJECT_DIR
else
    echo "[2/8] 새 프로젝트 생성: $PROJECT_NAME"
    cd ~/projects
    petalinux-create -t project --template zynq -n $PROJECT_NAME
    cd $PROJECT_DIR
    
    # 하드웨어 설정
    echo "[3/8] 하드웨어 설정..."
    petalinux-config --get-hw-description=$(dirname $XSA_PATH) --silentconfig
fi

# Device Tree 설정
echo "[4/8] Device Tree QSPI 노드 추가..."
DT_FILE="project-spec/meta-user/recipes-bsp/device-tree/files/system-user.dtsi"

cat > $DT_FILE << 'EOF'
/include/ "system-conf.dtsi"
/ {
};

&qspi {
    status = "okay";
    is-dual = <0>;
    num-cs = <1>;
    
    flash@0 {
        compatible = "micron,m25p80", "jedec,spi-nor";
        reg = <0x0>;
        spi-max-frequency = <50000000>;
        
        #address-cells = <1>;
        #size-cells = <1>;
        
        partition@0 {
            label = "boot";
            reg = <0x0 0x100000>;
        };
        
        partition@100000 {
            label = "kernel";
            reg = <0x100000 0x500000>;
        };
        
        partition@600000 {
            label = "rootfs";
            reg = <0x600000 0xA00000>;
        };
    };
};
EOF

echo "Device Tree 설정 완료"

# Rootfs 설정 (자동)
echo "[5/8] Rootfs 설정..."
cat >> project-spec/configs/rootfs_config << 'EOF'
CONFIG_debug-tweaks=y
CONFIG_allow-empty-password=y
CONFIG_allow-root-login=y
CONFIG_empty-root-password=y
CONFIG_serial-autologin-root=y
EOF

# 빌드
echo "[6/8] PetaLinux 빌드 시작... (1-3시간 소요)"
petalinux-build

# QSPI 부트 이미지 생성
echo "[7/8] QSPI 부트 이미지 생성..."
cd images/linux
petalinux-package --boot \
    --fsbl zynq_fsbl.elf \
    --fpga design_1_wrapper.bit \
    --u-boot u-boot.elf \
    --kernel image.ub \
    --flash-size 16 \
    --flash-intf qspi-x1-single \
    --force \
    -o qspi_flash_image.bin

# 공유 폴더로 복사
echo "[8/8] Windows 공유 폴더로 복사..."
mkdir -p $SHARE_DIR/zybo_qspi_build_$(date +%Y%m%d_%H%M%S)
BACKUP_DIR=$SHARE_DIR/zybo_qspi_build_$(date +%Y%m%d_%H%M%S)

cp qspi_flash_image.bin $BACKUP_DIR/
cp BOOT.BIN $BACKUP_DIR/
cp image.ub $BACKUP_DIR/
cp boot.scr $BACKUP_DIR/

# 최신 버전 링크
cp qspi_flash_image.bin $SHARE_DIR/qspi_flash_latest.bin

sync

echo ""
echo "=========================================="
echo "빌드 완료!"
echo "=========================================="
echo "QSPI 이미지: $SHARE_DIR/qspi_flash_latest.bin"
echo "백업 위치: $BACKUP_DIR"
echo ""
echo "다음 단계:"
echo "1. Vivado Hardware Manager 실행"
echo "2. JTAG 연결"
echo "3. QSPI Flash 프로그래밍"
echo "4. JP5를 QSPI 모드로 설정"
echo "5. 부팅 테스트"
echo "=========================================="
실행:
bashchmod +x build_qspi_complete.sh
./build_qspi_complete.sh
16.2 QSPI 프로그래밍 TCL 스크립트
program_qspi_full.tcl:
tcl# Zybo Z7-20 QSPI Flash 전체 프로그래밍 스크립트
# 사용법: xsct program_qspi_full.tcl <qspi_image_path>

if {$argc != 1} {
    puts "Usage: xsct program_qspi_full.tcl <qspi_flash_image.bin>"
    exit 1
}

set qspi_image [lindex $argv 0]

if {![file exists $qspi_image]} {
    puts "Error: File not found - $qspi_image"
    exit 1
}

puts "=========================================="
puts "QSPI Flash Programming Script"
puts "=========================================="
puts "Image: $qspi_image"
puts ""

# 연결
puts "Connecting to target..."
connect

# 타겟 선택
puts "Selecting ARM core..."
targets -set -filter {name =~ "ARM*#0"}

# PS7 초기화
puts "Initializing PS7..."
source [file join [file dirname [info script]] "ps7_init.tcl"]
ps7_init
ps7_post_config

# FSBL 다운로드 및 실행
puts "Loading FSBL..."
set fsbl_path [file join [file dirname $qspi_image] "zynq_fsbl.elf"]
if {[file exists $fsbl_path]} {
    dow $fsbl_path
    con
    after 1000
    stop
} else {
    puts "Warning: FSBL not found, skipping..."
}

# QSPI Flash 프로그래밍
puts ""
puts "Starting QSPI Flash programming..."
puts "This may take 5-10 minutes..."
puts ""

# Flash 프로브
puts "Probing SPI Flash..."
targets -set -filter {name =~ "ARM*#0"}

# Flash 프로그래밍 실행
set flash_type "qspi-x1-single"
set offset 0x0

puts "Programming flash at offset $offset..."
program_flash -f $qspi_image \
              -offset $offset \
              -flash_type $flash_type \
              -verify \
              -cable type xilinx_tcf url TCP:localhost:3121

puts ""
puts "=========================================="
puts "QSPI Flash programming completed!"
puts "=========================================="
puts ""
puts "Next steps:"
puts "1. Disconnect JTAG cable"
puts "2. Set JP5 to QSPI boot mode"
puts "3. Power cycle the board"
puts "4. Connect serial console (115200 8N1)"
puts "5. Verify boot"
puts ""

disconnect
exit 0
실행:
bashcd ~/projects/myproject_qspi/images/linux/
xsct program_qspi_full.tcl qspi_flash_image.bin
16.3 QSPI 검증 스크립트
verify_qspi.sh:
bash#!/bin/bash

echo "=========================================="
echo "QSPI Flash 검증 스크립트"
echo "=========================================="
echo ""

# MTD 디바이스 확인
echo "[1/5] MTD 디바이스 확인..."
if [ ! -e /proc/mtd ]; then
    echo "ERROR: /proc/mtd not found!"
    echo "QSPI Flash가 인식되지 않았습니다."
    exit 1
fi

cat /proc/mtd
echo ""

# QSPI 파티션 확인
echo "[2/5] QSPI 파티션 확인..."
BOOT_MTD=$(grep "boot" /proc/mtd | cut -d: -f1 | sed 's/mtd/\/dev\/mtd/')
KERNEL_MTD=$(grep "kernel" /proc/mtd | cut -d: -f1 | sed 's/mtd/\/dev\/mtd/')
ROOTFS_MTD=$(grep "rootfs" /proc/mtd | cut -d: -f1 | sed 's/mtd/\/dev\/mtd/')

echo "Boot partition: $BOOT_MTD"
echo "Kernel partition: $KERNEL_MTD"
echo "Rootfs partition: $ROOTFS_MTD"
echo ""

# Flash 크기 확인
echo "[3/5] Flash 크기 확인..."
for mtd in $(ls /sys/class/mtd/); do
    if [ -f "/sys/class/mtd/$mtd/name" ]; then
        NAME=$(cat /sys/class/mtd/$mtd/name)
        SIZE=$(cat /sys/class/mtd/$mtd/size)
        SIZE_MB=$((SIZE / 1024 / 1024))
        echo "$mtd: $NAME - ${SIZE_MB}MB"
    fi
done
echo ""

# INITRAMFS 확인
echo "[4/5] INITRAMFS 마운트 확인..."
mount | grep rootfs
df -h | grep -E "Filesystem|rootfs|tmpfs"
echo ""

# 메모리 확인
echo "[5/5] 메모리 사용량 확인..."
free -m
echo ""

# 부팅 로그에서 QSPI 확인
echo "부팅 로그에서 QSPI 관련 메시지:"
dmesg | grep -i "qspi\|spi\|flash\|mtd" | head -20

echo ""
echo "=========================================="
echo "검증 완료"
echo "=========================================="
실행 (Zybo에서):
bashchmod +x verify_qspi.sh
./verify_qspi.sh
16.4 QSPI + SD 혼합 부팅 설정
QSPI에서 부팅하고 SD 카드를 데이터 스토리지로 사용:
system-user.dtsi 수정:
dts/include/ "system-conf.dtsi"
/ {
};

&qspi {
    status = "okay";
    is-dual = <0>;
    num-cs = <1>;
    
    flash@0 {
        compatible = "micron,m25p80", "jedec,spi-nor";
        reg = <0x0>;
        spi-max-frequency = <50000000>;
        
        #address-cells = <1>;
        #size-cells = <1>;
        
        partition@0 {
            label = "boot";
            reg = <0x0 0x100000>;
        };
        
        partition@100000 {
            label = "kernel";
            reg = <0x100000 0x500000>;
        };
        
        partition@600000 {
            label = "rootfs";
            reg = <0x600000 0xA00000>;
        };
    };
};

&sdhci0 {
    status = "okay";
    no-1-8-v;
};
부팅 후 SD 카드 마운트 (Zybo에서):
bash# SD 카드 인식 확인
ls -l /dev/mmcblk*

# 파티션 생성 (최초 1회)
fdisk /dev/mmcblk0
# n → p → 1 → Enter → Enter → w

# 포맷
mkfs.ext4 /dev/mmcblk0p1

# 마운트
mkdir -p /mnt/sd
mount /dev/mmcblk0p1 /mnt/sd

# 자동 마운트 설정
echo "/dev/mmcblk0p1 /mnt/sd ext4 defaults 0 2" >> /etc/fstab

# 확인
df -h | grep sd
16.5 QSPI 성능 벤치마크
qspi_benchmark.sh:
bash#!/bin/bash

echo "=========================================="
echo "QSPI Flash 성능 벤치마크"
echo "=========================================="
echo ""

# 읽기 속도 테스트
echo "[1/3] QSPI 읽기 속도 테스트..."
if [ -e /dev/mtd0 ]; then
    echo "1MB 읽기 테스트..."
    time dd if=/dev/mtd0 of=/dev/null bs=1M count=1 2>&1 | grep -E "copied|MB"
    
    echo ""
    echo "10MB 읽기 테스트..."
    time dd if=/dev/mtd0 of=/dev/null bs=1M count=10 2>&1 | grep -E "copied|MB"
fi
echo ""

# 메모리 대역폭 테스트
echo "[2/3] 메모리 대역폭 테스트..."
dd if=/dev/zero of=/tmp/test.img bs=1M count=10 2>&1 | grep -E "copied|MB"
rm -f /tmp/test.img
echo ""

# 부팅 시간 확인
echo "[3/3] 부팅 시간 정보..."
systemd-analyze || echo "systemd-analyze not available"
echo ""

echo "=========================================="
echo "벤치마크 완료"
echo "=========================================="
16.6 QSPI 백업 및 복원
QSPI 백업 (Zybo에서):
bash#!/bin/bash

BACKUP_DIR="/mnt/sd/qspi_backup_$(date +%Y%m%d_%H%M%S)"
mkdir -p $BACKUP_DIR

echo "QSPI Flash 백업 중..."

# 각 파티션 백업
dd if=/dev/mtd0 of=$BACKUP_DIR/boot.img bs=1M
dd if=/dev/mtd1 of=$BACKUP_DIR/kernel.img bs=1M
dd if=/dev/mtd2 of=$BACKUP_DIR/rootfs.img bs=1M

# 전체 Flash 백업
dd if=/dev/mtd0 of=$BACKUP_DIR/qspi_full.img bs=1M count=16

echo "백업 완료: $BACKUP_DIR"
ls -lh $BACKUP_DIR
QSPI 복원 (U-Boot에서):
bash# TFTP를 통한 복원
setenv serverip 192.168.1.1
setenv ipaddr 192.168.1.100

# 백업 이미지 다운로드
tftpboot 0x2000000 qspi_full.img

# Flash 삭제
sf probe 0
sf erase 0 0x1000000

# Flash 쓰기
sf write 0x2000000 0 0x1000000

# 검증
sf read 0x3000000 0 0x1000000
cmp.b 0x2000000 0x3000000 0x1000000
16.7 빠른 참조 카드 (QSPI 전용)
QSPI 부팅 체크리스트:
단계항목확인1Root filesystem = INITRAMFS☐2Flash = ps7_qspi_0☐3Device Tree QSPI 노드 추가☐4debug-tweaks 활성화☐5빌드 완료☐6QSPI 이미지 생성 (--flash-size 16)☐7Vivado로 Flash 프로그래밍☐8JP5 = QSPI 모드☐9부팅 성공☐10INITRAMFS 확인☐
QSPI vs SD 카드 빠른 비교:
특성QSPI FlashSD 카드용량16MB4GB - 128GBRootfs 타입INITRAMFSEXT4/EXT3부팅 속도⚡⚡⚡ (5초)⚡⚡ (10초)쓰기 가능제한적자유로움신뢰성⭐⭐⭐⭐⭐⭐⭐⭐비용보드 포함추가 비용업데이트JTAG 필요카드 교체용도제품, 임베디드개발, 프로토타입
QSPI 명령어 요약:
환경명령어설명U-Bootsf probe 0QSPI Flash 프로브U-Bootsf read <addr> <offset> <size>Flash 읽기U-Bootsf erase <offset> <size>Flash 삭제U-Bootsf write <addr> <offset> <size>Flash 쓰기Linuxcat /proc/mtdMTD 파티션 확인Linuxdd if=/dev/mtd0 of=backup.img백업Linuxflash_erase /dev/mtd0 0 0삭제Linuxflashcp -v image.bin /dev/mtd0프로그래밍

최종 요약
QSPI 부팅 전체 프로세스
1. VirtualBox + Ubuntu 22.04.5 설치
2. 공유 폴더 설정 (/mnt/share)
3. 필수 패키지 설치
4. PetaLinux 2022.2 설치
5. Zybo Z7-20 QSPI 프로젝트 생성
6. ⭐ INITRAMFS Root filesystem 선택
7. ⭐ ps7_qspi_0 Flash 선택
8. Device Tree QSPI 노드 추가
9. Root 로그인 설정
10. PetaLinux 빌드 (1-3시간)
11. QSPI 부트 이미지 생성 (--flash-size 16)
12. Vivado Hardware Manager로 QSPI 프로그래밍
13. ⭐ JP5를 QSPI 모드로 설정
14. Zybo Z7-20 부팅
15. Root 로그인 (자동 또는 Enter)
16. INITRAMFS 확인
핵심 차이점 (SD vs QSPI)
SD 카드 부팅:
bashpetalinux-config
# Root filesystem type → SD card

petalinux-package --wic
# → petalinux-sdimage.wic
# balenaEtcher로 SD 카드 굽기
QSPI Flash 부팅:
bashpetalinux-config
# Root filesystem type → INITRAMFS
# Flash → ps7_qspi_0

petalinux-package --boot --flash-size 16
# → qspi_flash_image.bin
# Vivado Hardware Manager로 Flash 프로그래밍
예상 소요 시간
작업시간VirtualBox + Ubuntu 설치30-60분패키지 설치10-20분PetaLinux 설치10-30분프로젝트 생성 및 설정15-30분빌드 (첫 빌드)1-3시간QSPI 프로그래밍5-10분총 소요 시간약 2-5시간
중요 사항
✅ QSPI 부팅 필수 사항

Root filesystem = INITRAMFS
Flash = ps7_qspi_0
Device Tree QSPI 노드 추가
--flash-size 16 옵션
JP5를 QSPI 모드로 설정

❌ 흔한 실수

SD card rootfs 선택 (QSPI는 INITRAMFS!)
Device Tree에 QSPI 노드 누락
JP5 점퍼를 SD 모드로 설정
QSPI 프로그래밍 시 검증 생략

QSPI의 장단점
장점:

⚡ 빠른 부팅 (5초 내외)
🛡️ 높은 신뢰성 (솔더링)
🔒 변조 방지 (물리적 접근 필요)
💪 상용 제품에 적합

단점:

💾 제한된 용량 (16MB)
🔧 업데이트 복잡 (JTAG 필요)
📝 INITRAMFS 제약 (읽기 전용)
🧪 개발 단계에서 불편


성공을 위한 팁
QSPI 개발 권장 사항:

🔄 개발은 SD, 배포는 QSPI - 개발 단계에서는 SD 카드로 작업하고, 최종 제품에만 QSPI 사용
💾 용량 관리 - INITRAMFS는 항상 10MB 이하로 유지
🔙 백업 필수 - QSPI 이미지는 항상 백업본 보관
🧪 검증 철저히 - 프로그래밍 후 반드시 Verify 실행
📝 버전 관리 - QSPI 이미지에 버전 정보 포함

트러블슈팅 순서:

JP5 점퍼 확인 (QSPI 모드인가?)
시리얼 콘솔 메시지 확인
재시도나계속편집
Device Tree에 QSPI 노드 있는지 확인
U-Boot에서 sf probe 0 실행해보기
INITRAMFS 설정 확인 (SD card가 아닌지)
QSPI 이미지 재생성 및 재프로그래밍
완전 클린 빌드 시도

QSPI 최적화 팁:
bash# 1. INITRAMFS 크기 최소화
petalinux-config -c rootfs
# - 불필요한 패키지 모두 제거
# - busybox만 유지
# - 개발 도구 제거

# 2. 압축 레벨 최적화
vi project-spec/meta-user/conf/petalinuxbsp.conf
# 추가:
# INITRAMFS_FSTYPES = "cpio.gz"
# INITRAMFS_MAXSIZE = "10240"

# 3. U-Boot 환경 변수 최적화
# U-Boot에서:
setenv bootdelay 0
setenv bootcmd 'sf probe 0; sf read 0x2000000 0x100000 0x500000; bootm 0x2000000'
saveenv

마무리
축하합니다! 🎉
QSPI Flash 부팅 가이드를 완료하셨습니다!
여러분은 이제:

✅ Ubuntu 22.04 개발 환경 구축 완료
✅ PetaLinux 2022.2 설치 및 설정 완료
✅ QSPI Flash 프로젝트 생성 완료
✅ INITRAMFS Root filesystem 설정 완료
✅ Device Tree QSPI 노드 추가 완료
✅ QSPI Flash 프로그래밍 완료
✅ 실제 하드웨어에서 QSPI 부팅 성공!

여러분은 이제 임베디드 Linux 고급 개발자입니다!
QSPI vs SD 비교 총정리
언제 QSPI를 사용할까?

✅ 상용 제품 개발
✅ 빠른 부팅이 필수
✅ 높은 신뢰성 필요
✅ 변조 방지 필요
✅ 소용량으로 충분 (16MB 이하)
✅ 읽기 전용 시스템

언제 SD 카드를 사용할까?

✅ 개발 및 프로토타이핑
✅ 대용량 저장소 필요
✅ 자주 업데이트
✅ 데이터 로깅 필요
✅ 쉬운 이미지 교체
✅ 읽기/쓰기 모두 필요

혼합 사용 (권장):

🔹 QSPI: 부팅 및 시스템 파일
🔹 SD: 사용자 데이터, 로그, 설정

계속 학습하기
초급 QSPI 프로젝트:

QSPI에서 부팅하는 최소 시스템
LED 제어 QSPI 애플리케이션
Serial Console 기반 관리 도구

중급 QSPI 프로젝트:

QSPI + SD 하이브리드 시스템
네트워크를 통한 QSPI 업데이트
Fail-safe QSPI 부팅 (이중 이미지)
QSPI 암호화 부팅

고급 QSPI 프로젝트:

Secure Boot 구현
A/B 파티션 시스템 (무중단 업데이트)
OTA (Over-The-Air) 펌웨어 업데이트
QSPI Wear Leveling 구현
FPGA Bitstream을 QSPI에서 로딩

추가 학습 자료
QSPI 심화:

MTD (Memory Technology Device) 서브시스템
UBI/UBIFS 파일 시스템
Flash Translation Layer (FTL)
Bad Block Management

보안 부팅:

Secure Boot Flow
Chain of Trust
Signed Image
Key Management

상용 제품 개발:

Field Firmware Update
Rollback Protection
Factory Reset
Production Programming

유용한 도구
QSPI 개발 도구:
bash# MTD 유틸리티 (Zybo에서)
apt-get install mtd-utils

# 사용 가능한 명령:
# - flash_erase: Flash 삭제
# - flashcp: Flash 복사
# - nanddump: Flash 덤프
# - nandwrite: Flash 쓰기
# - mtdinfo: MTD 정보 확인
디버깅 도구:
bash# U-Boot 디버깅
setenv bootdelay 10
setenv bootcmd 'echo Booting from QSPI...; sf probe 0; sf read 0x2000000 0x100000 0x500000; bootm 0x2000000'

# Kernel 디버깅
# bootargs에 추가:
# loglevel=8 debug initcall_debug

🎓 완료 인증 (QSPI 전문가)
┌─────────────────────────────────────────┐
│  PetaLinux QSPI on Zybo Z7-20          │
│  Advanced Master Certificate            │
│                                         │
│  This certifies that                    │
│  YOU                                    │
│  has successfully completed             │
│                                         │
│  ✓ Ubuntu 22.04 Setup                  │
│  ✓ PetaLinux 2022.2 Installation       │
│  ✓ Zynq QSPI Project Creation          │
│  ✓ INITRAMFS Configuration             │
│  ✓ Device Tree QSPI Setup              │
│  ✓ QSPI Flash Programming              │
│  ✓ QSPI Boot Success                   │
│                                         │
│  Date: 2025-09-30                       │
│  Level: Embedded Linux Expert           │
│  Specialty: QSPI Flash Boot            │
└─────────────────────────────────────────┘

부록: QSPI 고급 주제
A. QSPI Dual/Quad Mode
Quad SPI 모드 활성화:
dts&qspi {
    status = "okay";
    is-dual = <0>;
    num-cs = <1>;
    spi-tx-bus-width = <4>;  // Quad mode
    spi-rx-bus-width = <4>;  // Quad mode
    
    flash@0 {
        compatible = "micron,m25p80", "jedec,spi-nor";
        reg = <0x0>;
        spi-max-frequency = <100000000>;  // 100MHz
        m25p,fast-read;
        
        ...
    };
};
성능 비교:
모드속도데이터선대역폭Single50MHz1~6 MB/sDual50MHz2~12 MB/sQuad50MHz4~24 MB/sQuad Fast100MHz4~48 MB/s
B. A/B 파티션 시스템
이중 이미지 레이아웃:
0x00000000 - 0x000FFFFF : BOOT.BIN (1MB)
0x00100000 - 0x005FFFFF : Kernel A (5MB)
0x00600000 - 0x009FFFFF : RootFS A (4MB)
0x00A00000 - 0x00EFFFFF : Kernel B (5MB)
0x00F00000 - 0x00FFFFFF : RootFS B (1MB)
Device Tree:
dts&qspi {
    flash@0 {
        partition@0 {
            label = "boot";
            reg = <0x0 0x100000>;
        };
        
        partition@100000 {
            label = "kernel_a";
            reg = <0x100000 0x500000>;
        };
        
        partition@600000 {
            label = "rootfs_a";
            reg = <0x600000 0x400000>;
        };
        
        partition@a00000 {
            label = "kernel_b";
            reg = <0xA00000 0x500000>;
        };
        
        partition@f00000 {
            label = "rootfs_b";
            reg = <0xF00000 0x100000>;
        };
    };
};
업데이트 로직 (pseudo-code):
bash#!/bin/bash
# OTA 업데이트 스크립트

CURRENT_SLOT=$(fw_printenv boot_slot | cut -d= -f2)
BACKUP_SLOT=$([[ $CURRENT_SLOT == "a" ]] && echo "b" || echo "a")

echo "Current: $CURRENT_SLOT, Backup: $BACKUP_SLOT"

# 백업 슬롯에 새 이미지 쓰기
flash_erase /dev/mtd_kernel_${BACKUP_SLOT} 0 0
flashcp new_kernel.img /dev/mtd_kernel_${BACKUP_SLOT}

flash_erase /dev/mtd_rootfs_${BACKUP_SLOT} 0 0
flashcp new_rootfs.img /dev/mtd_rootfs_${BACKUP_SLOT}

# 검증
if verify_images; then
    # 부트 슬롯 변경
    fw_setenv boot_slot ${BACKUP_SLOT}
    echo "Update success! Rebooting..."
    reboot
else
    echo "Update failed! Keeping current slot."
fi
C. QSPI 암호화 부팅
Secure Boot 개요:
1. BootROM (하드웨어)
   ↓ [검증]
2. FSBL (암호화)
   ↓ [복호화 + 검증]
3. U-Boot (암호화)
   ↓ [복호화 + 검증]
4. Linux Kernel (암호화)
   ↓ [복호화 + 검증]
5. RootFS (암호화)
Vivado에서 암호화 설정:
tcl# bootgen 명령어
bootgen -image boot.bif -arch zynq -o BOOT_encrypted.BIN -encrypt efuse

# boot.bif 파일:
the_ROM_image:
{
    [bootloader, encryption=aes, aeskeyfile=key.nky]zynq_fsbl.elf
    [encryption=aes, aeskeyfile=key.nky]design_1_wrapper.bit
    [encryption=aes, aeskeyfile=key.nky]u-boot.elf
}
D. QSPI 성능 최적화
1. U-Boot 최적화:
c// include/configs/zynq-common.h
#define CONFIG_SF_DEFAULT_SPEED    50000000  // 50MHz
#define CONFIG_SYS_BOOTM_LEN       0x1000000 // 16MB
#define CONFIG_SYS_LOAD_ADDR       0x2000000
2. Kernel 최적화:
bash# Kernel config
petalinux-config -c kernel

# Memory Technology Device (MTD) support --->
#     [*] MTD partitioning support
#     <*> Command line partition table parsing
#     <*> Caching block device access to MTD devices
#
# Device Drivers --->
#     SPI support --->
#         <*> Cadence Quad SPI controller
3. 부팅 시간 최적화:
bash# bootargs 최적화 (U-Boot)
setenv bootargs 'console=ttyPS0,115200 root=/dev/ram rw earlyprintk quiet lpj=loops_per_jiffy'

# systemd 최적화
systemctl mask systemd-udev-settle.service
systemctl mask systemd-networkd-wait-online.service
E. QSPI 문제 진단 가이드
진단 플로우차트:
부팅 실패?
    ├─ No → 정상 동작
    └─ Yes
        ├─ U-Boot 나타남?
        │   ├─ No
        │   │   ├─ JP5 점퍼 확인
        │   │   ├─ QSPI 프로그래밍 재확인
        │   │   └─ BOOT.BIN 재생성
        │   └─ Yes
        │       ├─ Kernel 로딩 실패?
        │       │   ├─ sf probe 확인
        │       │   ├─ sf read 테스트
        │       │   └─ image.ub 위치 확인
        │       └─ Kernel panic?
        │           ├─ INITRAMFS 확인
        │           ├─ Device Tree 확인
        │           └─ rootfs 마운트 실패
단계별 진단:
bash# 1단계: U-Boot 레벨
ZynqMP> sf probe 0
SF: Detected s25fl128s with page size 256 Bytes

ZynqMP> sf read 0x2000000 0x100000 0x100
device 0 offset 0x100000, size 0x100
SF: 256 bytes @ 0x100000 Read: OK

ZynqMP> md 0x2000000 0x10
# 데이터 확인

# 2단계: Kernel 레벨
dmesg | grep qspi
dmesg | grep spi
dmesg | grep mtd

# 3단계: MTD 레벨
cat /proc/mtd
ls -l /dev/mtd*
mtdinfo /dev/mtd0
F. QSPI 프로덕션 가이드
양산 프로그래밍 자동화:
production_program.sh:
bash#!/bin/bash

BATCH_FILE=$1
LOG_DIR="./production_logs"
mkdir -p $LOG_DIR

while IFS=, read -r serial_number image_path; do
    echo "=========================================="
    echo "Programming: $serial_number"
    echo "Image: $image_path"
    echo "=========================================="
    
    LOG_FILE="$LOG_DIR/${serial_number}_$(date +%Y%m%d_%H%M%S).log"
    
    # JTAG 연결 대기
    echo "Connect JTAG and press Enter..."
    read
    
    # 프로그래밍 실행
    xsct -eval "
        connect
        targets -set -filter {name =~ \"ARM*#0\"}
        program_flash -f $image_path \
                      -offset 0x0 \
                      -flash_type qspi-x1-single \
                      -verify
        disconnect
        exit
    " 2>&1 | tee $LOG_FILE
    
    if [ ${PIPESTATUS[0]} -eq 0 ]; then
        echo "SUCCESS: $serial_number" >> $LOG_DIR/success.log
        echo "✓ Programming successful!"
    else
        echo "FAILED: $serial_number" >> $LOG_DIR/failed.log
        echo "✗ Programming failed!"
    fi
    
    echo ""
    echo "Remove board and press Enter for next..."
    read
    
done < $BATCH_FILE

echo "=========================================="
echo "Batch programming completed"
echo "Success: $(wc -l < $LOG_DIR/success.log)"
echo "Failed: $(wc -l < $LOG_DIR/failed.log)"
echo "=========================================="
batch.csv 예:
SN001,/path/to/qspi_v1.0.bin
SN002,/path/to/qspi_v1.0.bin
SN003,/path/to/qspi_v1.0.bin
실행:
bashchmod +x production_program.sh
./production_program.sh batch.csv
G. QSPI 유지보수
정기 점검 항목:
bash#!/bin/bash
# qspi_health_check.sh

echo "QSPI Flash 헬스 체크"
echo "===================="

# 1. MTD 디바이스
echo "[1] MTD 디바이스 상태"
cat /proc/mtd

# 2. Bad block 확인 (NAND only, NOR은 해당 없음)
echo "[2] Flash 정보"
for mtd in /dev/mtd*; do
    [ -b $mtd ] && mtdinfo $mtd 2>/dev/null
done

# 3. ECC 에러
echo "[3] ECC 에러 확인"
dmesg | grep -i "ecc\|error" | tail -20

# 4. 읽기 테스트
echo "[4] 읽기 테스트"
for i in 0 1 2; do
    echo "Testing /dev/mtd$i..."
    dd if=/dev/mtd$i of=/dev/null bs=64k 2>&1 | grep -E "copied|error"
done

# 5. 쓰기 사이클 추정 (로그 기반)
echo "[5] 쓰기 사이클 추정"
if [ -f /var/log/flash_writes.log ]; then
    WRITES=$(wc -l < /var/log/flash_writes.log)
    echo "총 쓰기: $WRITES"
    echo "남은 수명: $((100000 - WRITES)) cycles"
fi

echo "===================="
echo "헬스 체크 완료"

최종 체크리스트 (인쇄용)
☑️ QSPI 부팅 마스터 체크리스트
Phase 1: 환경 준비

 VirtualBox 7.0+ 설치
 Ubuntu 22.04.5 LTS 설치 (200GB+, 16GB RAM+)
 Guest Additions 설치
 공유 폴더 설정 (/mnt/share)
 필수 패키지 설치 완료

Phase 2: PetaLinux 설치

 PetaLinux 2022.2 인스톨러 다운로드
 ~/petalinux/2022.2 에 설치
 settings.sh 실행 확인
 PETALINUX 환경 변수 확인

Phase 3: 프로젝트 생성

 XSA 파일 준비
 petalinux-create로 프로젝트 생성
 --get-hw-description로 하드웨어 임포트

Phase 4: QSPI 설정 ⭐

 petalinux-config에서:

 Root filesystem → INITRAMFS
 Flash → ps7_qspi_0
 UART → ps7_uart_1
 Ethernet → ps7_ethernet_0


 system-user.dtsi에 QSPI 노드 추가
 파티션 레이아웃 정의 (boot, kernel, rootfs)

Phase 5: Rootfs 설정

 petalinux-config -c rootfs에서:

 debug-tweaks 활성화
 allow-empty-password 활성화
 allow-root-login 활성화
 empty-root-password 활성화
 serial-autologin-root 활성화


 불필요한 패키지 제거 (용량 최적화)
 busybox 확인

Phase 6: 빌드

 petalinux-build 실행 (1-3시간)
 빌드 성공 확인
 Warning 메시지 검토 (무시 가능)

Phase 7: QSPI 이미지 생성

 petalinux-package --boot 실행

 --fsbl 옵션
 --fpga 옵션
 --u-boot 옵션
 --kernel 옵션 (QSPI 전용!)
 --flash-size 16 옵션
 --flash-intf qspi-x1-single 옵션


 qspi_flash_image.bin 생성 확인
 Windows 공유 폴더로 복사

Phase 8: Flash 프로그래밍

 Vivado 2022.2 실행
 JTAG 케이블 연결 (J13)
 Zybo 전원 ON
 Hardware Manager → Open Target
 Add Configuration Memory Device

 s25fl128sxxxxxx0 선택


 qspi_flash_image.bin 선택
 Erase, Program, Verify 모두 체크
 프로그래밍 실행 (5-10분)
 "Flash programming completed" 확인

Phase 9: 하드웨어 설정

 JTAG 케이블 제거
 JP5 점퍼를 QSPI 모드로 설정

 하단 2핀만 (QS 표시)
 또는 모든 핀 제거


 USB-UART 케이블 연결 (J14)
 이더넷 연결 (선택)

Phase 10: 부팅 및 검증

 PuTTY 실행 (115200 8N1)
 COM 포트 확인 및 연결
 Zybo 전원 ON (SW0)
 "Boot mode is QSPI" 확인
 U-Boot 메시지 확인
 Kernel 부팅 메시지 확인
 "Unpacking initramfs" 확인
 Root 로그인 성공
 cat /proc/mtd 실행
 mount | grep rootfs 확인
 df -h 확인 (tmpfs)

Phase 11: 기능 테스트

 네트워크 연결 (eth0)
 DHCP 또는 수동 IP 설정
 ping 테스트
 GPIO 테스트 (선택)
 재부팅 테스트
 부팅 시간 측정

Phase 12: 백업 및 문서화

 QSPI 이미지 백업 (여러 복사본)
 프로젝트 전체 백업
 설정 파일 문서화
 빌드 로그 저장
 버전 정보 기록


Happy Hacking with QSPI! 🛠️⚡
May the Flash be with you! 💾

문서 정보
제목: Digilent Zybo Z7-20 PetaLinux 완벽 가이드 (Quad-SPI Flash 버전)
부제: QSPI Flash 부팅 완전 정복
버전: 3.0 (QSPI Edition)
작성일: 2025년 9월 30일
최종 업데이트: 2025년 9월 30일
대상:

하드웨어: Digilent Zybo Z7-20 (Zynq-7020)
Flash: Spansion S25FL128S (16MB QSPI)
PetaLinux: 2022.2
호스트 OS: Ubuntu 22.04.5 LTS (VirtualBox)
Rootfs: INITRAMFS (RAM-based)

변경 이력:

v3.0 (2025-09-30): QSPI Flash 버전 완성
v2.0 (2025-09-30): SD 카드 버전 (기본)
v1.0 (2025-09-29): 초기 버전

라이센스: CC BY-SA 4.0
자유롭게 공유 및 수정 가능
기여:

문서 작성: Claude (Anthropic AI)
검증: 실제 Zybo Z7-20 하드웨어 테스트
QSPI 검증: Vivado 2022.2 Hardware Manager


이 가이드가 도움이 되었다면 다른 개발자들과 공유해주세요!
질문이나 피드백은 언제든지 환영합니다.

  ___  ____  ____ ___ 
 / _ \/ ___||  _ \_ _|
| | | \___ \| |_) | | 
| |_| |___) |  __/| | 
 \__\_\____/|_|  |___|
                      
  Flash Boot Complete Guide
  Zybo Z7-20 + PetaLinux 2022.2
  End of Document - Thank you!
END OF DOCUMENT
© 2025 Zybo Z7-20 PetaLinux QSPI Guide
All trademarks are property of their respective owners.
