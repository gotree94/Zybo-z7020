# Zybo-z7020

---

* PetaLinux Tools Documentation: Reference Guide (UG1144) - AMD
   * https://docs.amd.com/r/en-US/ug1144-petalinux-tools-reference-guide/Navigating-Content-by-Design-Process

# 1. Installation Requirements

## The PetaLinux tools installation requirements are:

   * Minimum workstation requirements:
      * 8 GB RAM (recommended minimum for AMD tools)
      * 2 GHz CPU clock or equivalent (minimum of eight cores)
      * 100 GB free HDD space
      * Supported OS:
         * Completely removed CENTOS and RHEL to align with upstream Yocto.
         * Ubuntu Desktop/Server 22.04.2 LTS,22.04.3 LTS, 22.04.4 LTS and 22.04.5 LTS
         * OpenSuse Leap 15.4
         * AlmaLinux 8.10,9.4 and 9.5

---
# 2. VirtualBox & Share folder setting

## 1단계: 사용자를 vboxsf 그룹에 추가
```
sudo usermod -aG vboxsf gotree94
```

## 2단계: 로그아웃 후 재로그인
GUI에서 로그아웃하거나
터미널에서: logout 또는 exit
또는 시스템 재부팅: sudo reboot

## 3단계: 재로그인 후 확인
```
groups
```

   * 이제 출력에 vboxsf가 포함되어야 합니다.

---

   * 만약 위 방법이 안 되면, 공유 폴더가 제대로 마운트되었는지 확인해보세요:

```
bashmount | grep vboxsf
```

   * 출력이 없다면 Guest Additions 설치나 공유 폴더 설정에 문제가 있을 수 있습니다.
   * Guest Additions 재설치:
```
sudo apt update
sudo apt install virtualbox-guest-utils virtualbox-guest-x11 virtualbox-guest-dkms
sudo reboot
```

   * 재부팅 후 VirtualBox 설정에서 공유 폴더가 올바르게 설정되어 있는지 확인하고, 다시 시도해보세요.

---

# 3. Zybo Z7-20 PetaLinux 설치 가이드 (VirtualBox Ubuntu)

## 개요
VirtualBox에 설치된 Ubuntu 22.04에서 Digilent Zybo Z7-20 보드용 PetaLinux를 빌드하고 SD 카드 이미지를 생성하는 전체 과정입니다.

## VirtualBox 환경 사전 요구사항
- **디스크 공간**: 최소 150GB 이상 (200GB 권장)
- **메모리(RAM)**: 최소 8GB, 16GB 이상 강력 권장
- **CPU 코어**: 4코어 이상 할당 권장
- **네트워크**: 인터넷 연결 필수
- **Guest Additions**: 설치 권장 (파일 공유용)

---

## VirtualBox 설정

### 디스크 공간 확인
```bash
df -h
# /home 파티션에 최소 100GB 이상의 여유 공간 확인
```

### VirtualBox Guest Additions 설치 (선택사항)
SD 카드 이미지를 호스트 Windows로 쉽게 전송하기 위해 설치합니다.
```bash
sudo apt update
sudo apt install -y virtualbox-guest-utils virtualbox-guest-x11
```

### 공유 폴더 설정 (VirtualBox)
1. VirtualBox 메뉴: **장치** → **공유 폴더** → **공유 폴더 설정**
2. **추가** 버튼 클릭
3. **폴더 경로**: Windows의 공유할 폴더 선택 (예: `C:\SharedVM`)
4. **폴더 이름**: `shared` (원하는 이름)
5. **자동 마운트** 체크
6. **영구적으로 만들기** 체크

Ubuntu에서 마운트:
```bash
sudo mkdir -p /mnt/shared
sudo mount -t vboxsf shared /mnt/shared
# 또는 자동 마운트된 경로 사용: /media/sf_shared

# 현재 사용자를 vboxsf 그룹에 추가 (권한 문제 해결)
sudo usermod -aG vboxsf $USER
# 로그아웃 후 재로그인 필요
```

---

## 1단계: Ubuntu 22.04 환경 설정

### 1.1 필수 패키지 설치
```bash
sudo apt update
sudo apt install -y build-essential git wget curl
sudo apt install -y python3 python3-pip python3-venv
sudo apt install -y gawk chrpath socat cpio python3-pexpect
sudo apt install -y xz-utils debianutils iputils-ping python3-git
sudo apt install -y python3-jinja2 libegl1-mesa libsdl1.2-dev
sudo apt install -y pylint3 xterm rsync vim net-tools
sudo apt install -y gcc-multilib libc6-dev-i386 libncurses5-dev
sudo apt install -y zlib1g:i386 libtool texinfo
```

```bash
# 패키지 목록 업데이트
sudo apt update

# 누락된 시스템 도구들 설치
sudo apt install -y net-tools xterm autoconf libtool texinfo gcc-multilib

# 누락된 개발 라이브러리 설치
sudo apt install -y libncurses5-dev libncursesw5-dev

# 32비트 라이브러리 지원 활성화
sudo dpkg --add-architecture i386
sudo apt update

# 32비트 zlib 설치
sudo apt install -y zlib1g:i386

# 추가로 필요할 수 있는 패키지들
sudo apt install -y libc6:i386 libstdc++6:i386 libgcc1:i386
sudo apt install -y lib32z1 lib32ncurses6

# 설치 확인
which netstat xterm autoconf libtool
dpkg -l | grep libncurses
dpkg -l | grep zlib1g:i386
```

### 1.2 Bash 쉘 설정
```bash
sudo dpkg-reconfigure dash
# "No" 선택하여 bash를 기본 쉘로 설정
```

### 1.3 로케일 설정
```bash
sudo locale-gen en_US.UTF-8
echo 'export LC_ALL=en_US.UTF-8' >> ~/.bashrc
echo 'export LANG=en_US.UTF-8' >> ~/.bashrc
source ~/.bashrc
```

---

## 2단계: PetaLinux Tools 설치

### 2.1 현재 상태 확인
```bash
# PetaLinux가 이미 설치되어 있는지 확인
which petalinux-create
echo $PETALINUX

# 설치 경로 확인
ls -la /opt/pkg/petalinux/ 2>/dev/null || echo "Not found in /opt/pkg/"
ls -la ~/petalinux/ 2>/dev/null || echo "Not found in home directory"

# bashrc 확인
grep -i petalinux ~/.bashrc
```

### 2.2 PetaLinux Tools 다운로드
AMD/Xilinx 웹사이트에서 다운로드해야 합니다:

1. 웹브라우저에서 https://www.xilinx.com/support/download.html 접속
2. **Embedded Design Tools** 선택
3. AMD 계정으로 로그인 (무료 계정 생성 가능)
4. **PetaLinux Tools** 선택
5. 버전 선택 (예: 2022.1 또는 2022.2 권장)
6. `petalinux-v2022.1-final-installer.run` 다운로드
7. ~/Downloads 폴더에 저장

### 2.3 PetaLinux 설치 (홈 디렉토리 방식 권장)
```bash
cd ~/Downloads

# 실행 권한 부여
chmod +x petalinux-v2022.2-10141622-installer.run

# 설치 디렉토리 생성
mkdir -p ~/petalinux/2022.2

# PetaLinux 설치 (약 10-30분 소요, 프롬프트에서 엔터 입력)
./petalinux-v2022.2-10141622-installer.run -d ~/petalinux/2022.2
```

```
INFO: Checking installation environment requirements...
WARNING: This is not a supported OS
INFO: Checking free disk space
INFO: Checking installed tools
INFO: Checking installed development libraries
INFO: Checking network and other services
WARNING: No tftp server found - please refer to "UG1144  PetaLinux Tools Documentation Reference Guide" for its impact and solution
INFO: Checking installer checksum...
INFO: Extracting PetaLinux installer...

LICENSE AGREEMENTS

PetaLinux SDK contains software from a number of sources.  Please review
the following licenses and indicate your acceptance of each to continue.

You do not have to accept the licenses, however if you do not then you may 
not use PetaLinux SDK.

Use PgUp/PgDn to navigate the license viewer, and press 'q' to close

Press Enter to display the license agreements
Do you accept Xilinx End User License Agreement? [y/N] > y
Do you accept Third Party End User License Agreement? [y/N] > y
INFO: Installing PetaLinux...
INFO: Checking PetaLinux installer integrity...
INFO: Installing PetaLinux SDK to "/home/gotree94/petalinux/2022.2/."
INFO: Installing buildtools in /home/gotree94/petalinux/2022.2/./components/yocto/buildtools
INFO: Installing buildtools-extended in /home/gotree94/petalinux/2022.2/./components/yocto/buildtools_extended
INFO: PetaLinux SDK has been installed to /home/gotree94/petalinux/2022.2/.
gotree94@gotree94-VirtualBox:~/Downloads$ 

```

설치 중 라이센스 동의를 요구하면:
- **q**를 눌러 라이센스 끝으로 이동
- **y**를 입력하여 동의

### 2.4 환경변수 설정
```bash
# bashrc에 추가
echo '' >> ~/.bashrc
echo '# PetaLinux Settings' >> ~/.bashrc
echo 'source ~/petalinux/2022.2/settings.sh' >> ~/.bashrc

# 현재 세션에 적용
source ~/.bashrc
```

### 2.5 설치 확인
```bash
# 새 터미널 열기 또는
source ~/.bashrc

# PetaLinux 명령어 확인
which petalinux-create
petalinux-create --help
echo $PETALINUX

# 정상 출력 예시:
# /home/gotree94/petalinux/2022.1/tools/common/petalinux/bin/petalinux-create
```

### 2.6 문제 해결

**명령어를 찾을 수 없는 경우:**
```bash
# 수동으로 환경변수 로드
source ~/petalinux/2022.2/settings.sh

# settings.sh 파일 존재 확인
ls -la ~/petalinux/2022.2/settings.sh

# 설치 디렉토리 내용 확인
ls -la ~/petalinux/2022.2/
```

---

## 3단계: PetaLinux 프로젝트 생성

### 3.1 프로젝트 디렉토리 생성
```bash
mkdir -p ~/petalinux_projects
cd ~/petalinux_projects
```

### 3.2 Zybo Z7-20용 PetaLinux 프로젝트 생성
```bash
petalinux-create --type project --template zynq --name zybo_z7_20_project
cd zybo_z7_20_project
```

### 3.3 하드웨어 정의 파일 가져오기
Vivado에서 생성한 design_1_wrapper.xsa 파일을 프로젝트로 복사합니다.

**공유 폴더를 통한 방법:**
```bash
# Windows에서 xsa 파일을 공유 폴더에 복사 후
cp /media/share/design_1_wrapper.xsa ~/petalinux_projects/zybo_z7_20_project/
cd ~/petalinux_projects/zybo_z7_20_project
```

**또는 USB를 통한 방법:**
VirtualBox 메뉴: **장치** → **USB** → USB 디바이스 선택
```bash
# USB가 자동 마운트되면
cp /media/gotree94/USB드라이브이름/design_1_wrapper.xsa ~/petalinux_projects/zybo_z7_20_project/
cd ~/petalinux_projects/zybo_z7_20_project
```

### 3.4 하드웨어 정의 가져오기 및 구성
```bash
petalinux-config --get-hw-description=./
```

---

## 4단계: PetaLinux 구성

### 4.1 시스템 구성
위 명령어 실행 후 menuconfig가 열리면:

**DTG Settings**
- `Kernel Bootargs` 항목으로 이동
- `generate boot args automatically` → **비활성화**
- `User Set Kernel Bootargs` 입력:
  ```
  earlycon console=ttyPS0,115200 clk_ignore_unused root=/dev/mmcblk0p2 rw rootwait
  ```

**Image Packaging Configuration**
- `Root filesystem type` → **EXT4 (SD/eMMC/SATA/USB)** 선택
- `Device node of SD device` → `/dev/mmcblk0p2` 입력

**Yocto Settings** (필요시)
- `YOCTO_MACHINE_NAME` → `zynq-generic` 확인

저장: **Exit** → **Yes** (저장)

### 4.2 루트 파일시스템 구성
```bash
petalinux-config -c rootfs
```

필요한 패키지 선택:
- `Filesystem Packages` → 필요한 도구들 선택
- `user packages` → 커스텀 패키지 (필요시)

저장 후 종료

### 4.3 커널 구성 (GPIO 확인)
```bash
petalinux-config -c kernel
```

다음 항목들이 활성화되어 있는지 확인:
- `Device Drivers` → `GPIO Support` → **활성화**
- `Device Drivers` → `GPIO Support` → `Memory mapped GPIO drivers` → **활성화**
- `Device Drivers` → `Input device support` → `Keyboards` → `GPIO Buttons` → **활성화**

저장 후 종료

---

## 5단계: 디바이스 트리 수정 (GPIO 버튼)

### 5.1 시스템 사용자 디바이스 트리 편집
```bash
vim project-spec/meta-user/recipes-bsp/device-tree/files/system-user.dtsi
```

**중요**: GPIO 번호는 Vivado 설계에 따라 다릅니다. 아래는 예시입니다.

```dts
/include/ "system-conf.dtsi"
/ {
    gpio-keys {
        compatible = "gpio-keys";
        #address-cells = <1>;
        #size-cells = <0>;
        autorepeat;
        
        btn0 {
            label = "btn0";
            gpios = <&gpio0 50 0>;
            linux,code = <0x100>;
            gpio-key,wakeup;
        };
        
        btn1 {
            label = "btn1";
            gpios = <&gpio0 51 0>;
            linux,code = <0x101>;
            gpio-key,wakeup;
        };
        
        btn2 {
            label = "btn2";
            gpios = <&gpio0 52 0>;
            linux,code = <0x102>;
            gpio-key,wakeup;
        };
        
        btn3 {
            label = "btn3";
            gpios = <&gpio0 53 0>;
            linux,code = <0x103>;
            gpio-key,wakeup;
        };
    };
};
```

### 5.2 GPIO 번호 확인 방법
Vivado에서:
1. Block Design에서 GPIO 블록 확인
2. AXI GPIO를 사용하는 경우: EMIO 핀 번호 확인
3. EMIO GPIO는 54번부터 시작 (0-53은 MIO)

---

## 6단계: PetaLinux 빌드

### 6.1 전체 시스템 빌드
```bash
cd ~/petalinux_projects/zybo_z7_20_project
petalinux-build
```

**주의사항:**
- 첫 빌드는 1-3시간 소요될 수 있습니다
- VirtualBox 환경에서는 더 오래 걸릴 수 있습니다
- 충분한 디스크 공간 확인 (최소 50GB 여유 필요)

**빌드 진행 상황 모니터링:**
```bash
# 다른 터미널에서
cd ~/petalinux_projects/zybo_z7_20_project
tail -f build/build.log
```

### 6.2 부트 이미지 생성
빌드가 완료되면:
```bash
cd ~/petalinux_projects/zybo_z7_20_project
petalinux-package --boot --fsbl images/linux/zynq_fsbl.elf \
    --fpga images/linux/design_1_wrapper.bit --u-boot
```

---

## 7단계: SD 카드 이미지 생성

### 7.1 단일 WIC 이미지 생성
```bash
cd ~/petalinux_projects/zybo_z7_20_project
petalinux-package --wic --images-dir images/linux/ \
    --bootfiles "BOOT.BIN boot.scr Image system.dtb"
```

### 7.2 생성된 이미지 확인
```bash
ls -lh images/linux/
```

주요 파일들:
- **BOOT.BIN** - 부트로더 + FPGA 비트스트림 + U-Boot
- **Image** - 리눅스 커널
- **system.dtb** - 디바이스 트리
- **rootfs.ext4** - 루트 파일시스템
- **petalinux-sdimage.wic** - 완전한 SD 카드 이미지 (이것을 사용!)

---

## 8단계: SD 카드에 이미지 쓰기

### 8.1 VirtualBox에서 SD 카드 액세스

**방법 1: Ubuntu에서 직접 쓰기**

1. SD 카드를 PC에 삽입
2. VirtualBox 메뉴: **장치** → **USB** → SD 카드 리더기 선택
3. Ubuntu에서 SD 카드 인식 확인:
```bash
lsblk
# SD 카드가 /dev/sdb 또는 /dev/sdc로 표시됨
```

4. SD 카드 언마운트 (마운트되어 있는 경우):
```bash
sudo umount /dev/sdb*
# 또는 해당 장치명
```

5. 이미지 쓰기:
```bash
cd ~/petalinux_projects/zybo_z7_20_project
sudo dd if=images/linux/petalinux-sdimage.wic of=/dev/sdb bs=4M status=progress
sudo sync
```

**주의**: `/dev/sdb`는 예시이며, 실제 SD 카드 장치명을 확인해야 합니다!

### 8.2 공유 폴더를 통한 방법

Windows에서 SD 카드에 쓰려면:

1. 이미지를 공유 폴더로 복사:
```bash
cp ~/petalinux_projects/zybo_z7_20_project/images/linux/petalinux-sdimage.wic /media/sf_shared/
```

2. Windows에서 Rufus, Win32 Disk Imager, 또는 balenaEtcher 사용하여 SD 카드에 쓰기

### 8.3 etcher 설치 (Ubuntu에서 GUI로 쓰기)
```bash
# balenaEtcher 다운로드 및 설치
wget https://github.com/balena-io/etcher/releases/download/v1.18.11/balenaEtcher-1.18.11-x64.AppImage
chmod +x balenaEtcher-*.AppImage
./balenaEtcher-*.AppImage
```

Etcher에서:
1. **Flash from file** → `petalinux-sdimage.wic` 선택
2. **Select target** → SD 카드 선택
3. **Flash!** 클릭

---

## 9단계: 보드 부팅 및 테스트

### 9.1 하드웨어 설정
1. SD 카드를 Zybo Z7-20 보드에 삽입
2. 점퍼 설정: **SD 부팅 모드**로 변경
   - JP5: SD로 설정
3. UART-USB 케이블 연결
4. 전원 공급

### 9.2 시리얼 콘솔 연결 (Ubuntu)

**minicom 설치 및 설정:**
```bash
sudo apt install -y minicom
sudo minicom -s
```

설정:
- **Serial port setup** 선택
- Serial Device: `/dev/ttyUSB0` (또는 `/dev/ttyUSB1`)
- Bps/Par/Bits: `115200 8N1`
- Hardware Flow Control: **No**
- Software Flow Control: **No**
- **Save setup as dfl** 선택
- **Exit**

연결:
```bash
sudo minicom
```

**또는 screen 사용:**
```bash
sudo apt install -y screen
sudo screen /dev/ttyUSB0 115200
```

### 9.3 부팅 확인
시리얼 콘솔에서 부팅 과정 확인:
```
U-Boot 2022.01 (날짜)
...
Starting kernel ...
...
PetaLinux 2022.1 zynq-generic /dev/ttyPS0

zynq-generic login: root
Password: root
```

### 9.4 GPIO 버튼 테스트
```bash
# 루트로 로그인
root@zynq-generic:~# 

# 입력 디바이스 확인
cat /proc/bus/input/devices | grep -A 5 gpio-keys

# GPIO 상태 확인
cat /sys/kernel/debug/gpio

# 버튼 이벤트 테스트
evtest /dev/input/event0
# 버튼을 누르면 이벤트가 화면에 표시됨
```

---

## 트러블슈팅

### VirtualBox 관련 문제

**1. 디스크 공간 부족**
```bash
# 디스크 사용량 확인
df -h
du -sh ~/petalinux_projects/zybo_z7_20_project

# 빌드 캐시 정리
cd ~/petalinux_projects/zybo_z7_20_project
petalinux-build -c cleanall
```

**2. USB 장치가 인식되지 않음**
- VirtualBox 확장 팩 설치 필요
- **파일** → **환경설정** → **확장** → Oracle VM VirtualBox Extension Pack 설치
- VM을 종료하고 USB 3.0 컨트롤러 활성화

**3. 공유 폴더 접근 권한 오류**
```bash
sudo usermod -aG vboxsf $USER
# 로그아웃 후 재로그인
```

### PetaLinux 빌드 문제

**1. 빌드 오류 발생**
```bash
# 로그 확인
cat build/build.log | grep -i error

# 클린 빌드
petalinux-build -c cleanall
petalinux-build
```

**2. 메모리 부족**
- VirtualBox에서 VM에 더 많은 RAM 할당 (최소 8GB, 권장 16GB)
- 스왑 공간 증가:
```bash
sudo fallocate -l 8G /swapfile
sudo chmod 600 /swapfile
sudo mkswap /swapfile
sudo swapon /swapfile
echo '/swapfile none swap sw 0 0' | sudo tee -a /etc/fstab
```

**3. 네트워크 타임아웃**
```bash
# DNS 확인
cat /etc/resolv.conf

# 네트워크 연결 확인
ping -c 4 google.com

# VirtualBox 네트워크 어댑터를 NAT로 설정
```

### 부팅 문제

**1. SD 카드가 부팅되지 않음**
- 점퍼 설정 재확인 (SD 모드)
- SD 카드 포맷 후 재시도
- BOOT.BIN 파일 확인

**2. 커널 패닉**
- bootargs 확인 (rootfs 경로 `/dev/mmcblk0p2`)
- 디바이스 트리 확인

**3. GPIO 버튼 미작동**
```bash
# 커널 메시지 확인
dmesg | grep gpio
dmesg | grep input

# GPIO 드라이버 확인
lsmod | grep gpio
```

---

## 유용한 명령어 모음

```bash
# PetaLinux 환경 재로드
source ~/petalinux/2022.1/settings.sh

# 빌드 로그 실시간 확인
tail -f build/build.log

# 특정 컴포넌트만 재빌드
petalinux-build -c u-boot -x cleanall
petalinux-build -c u-boot

# 커널만 재빌드
petalinux-build -c kernel

# 루트 파일시스템만 재빌드
petalinux-build -c rootfs

# 디바이스 트리만 재빌드
petalinux-build -c device-tree

# 부팅 이미지만 재생성
cd ~/petalinux_projects/zybo_z7_20_project
petalinux-package --boot --force --fsbl images/linux/zynq_fsbl.elf \
    --fpga images/linux/design_1_wrapper.bit --u-boot
```

---

## 참고 자료
- [Xilinx PetaLinux Tools Documentation](https://docs.xilinx.com/r/en-US/ug1144-petalinux-tools-reference-guide)
- [Digilent Zybo Z7 Reference Manual](https://digilent.com/reference/programmable-logic/zybo-z7/reference-manual)
- [VirtualBox Documentation](https://www.virtualbox.org/manual/)

이 가이드를 통해 VirtualBox Ubuntu 환경에서 Zybo Z7-20용 PetaLinux 시스템을 성공적으로 구축할 수 있습니다.
