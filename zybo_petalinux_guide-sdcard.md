# Digilent Zybo 7020 PetaLinux SD Card Boot Guide

## 2. Ubuntu 시스템 준비

### 2.1 시스템 업데이트

``` bash
sudo apt update
sudo apt upgrade -y
```

### 2.2 32비트 라이브러리 지원 추가

``` bash
sudo dpkg --add-architecture i386
sudo apt update
```

### 2.3 필수 패키지 설치

``` bash
sudo apt install -y     build-essential     gcc-multilib     g++-multilib     gawk     wget     git     diffstat     unzip     texinfo     chrpath     socat     cpio     python3     python3-pip     python3-pexpect     xz-utils     debianutils     iputils-ping     python3-git     python3-jinja2     libegl1-mesa     libsdl1.2-dev     pylint     xterm     rsync     curl     libncurses5-dev     libncursesw5-dev     libssl-dev     flex     bison     libselinux1     gnupg     zlib1g-dev     libtool     autoconf     automake     net-tools     screen     pax     gzip     vim     iproute2     locales     libncurses5     libtinfo5
```

### 2.4 32비트 라이브러리 설치

``` bash
sudo apt install -y     libncurses5:i386     libc6:i386     libstdc++6:i386     lib32z1     zlib1g:i386
```

### 2.5 Locale 설정

``` bash
sudo locale-gen en_US.UTF-8
sudo update-locale LANG=en_US.UTF-8
```

### 2.6 Dash를 Bash로 변경

``` bash
sudo dpkg-reconfigure dash
```

👉 메뉴가 나타나면 **No** 선택

### 2.7 TFTP 서버 설치 (선택사항)

``` bash
sudo apt install -y tftpd-hpa
sudo mkdir -p /tftpboot
sudo chmod 777 /tftpboot
sudo chown nobody:nogroup /tftpboot
sudo vi /etc/default/tftpd-hpa
```

`/etc/default/tftpd-hpa` 설정:

    TFTP_USERNAME="tftp"
    TFTP_DIRECTORY="/tftpboot"
    TFTP_ADDRESS=":69"
    TFTP_OPTIONS="--secure"

``` bash
sudo systemctl restart tftpd-hpa
sudo systemctl enable tftpd-hpa
sudo systemctl status tftpd-hpa
```

------------------------------------------------------------------------

## 3. PetaLinux 2022.2 설치

### 3.1 작업 디렉토리 생성

``` bash
mkdir -p ~/petalinux_work
cd ~/petalinux_work
```

### 3.2 인스톨러 준비

Windows → Ubuntu 복사:

``` bash
cp /mnt/share/petalinux-v2022.2-10141622-installer.run ~/petalinux_work/
chmod +x ~/petalinux_work/petalinux-v2022.2-10141622-installer.run
```

### 3.3 PetaLinux 설치

``` bash
mkdir -p ~/petalinux/2022.2
cd ~/petalinux_work
./petalinux-v2022.2-10141622-installer.run -d ~/petalinux/2022.2
```

설치 시: - 라이센스 동의: y - 소요 시간: 10\~30분 - 디스크 사용량: 약
8GB

### 3.4 환경 설정

``` bash
source ~/petalinux/2022.2/settings.sh
echo $PETALINUX
```

👉 출력: `/home/사용자명/petalinux/2022.2`

영구 적용:

``` bash
echo "source ~/petalinux/2022.2/settings.sh" >> ~/.bashrc
```

------------------------------------------------------------------------

## 4. Zybo Z7-20 프로젝트 생성

### 4.1 프로젝트 디렉토리 준비

``` bash
mkdir -p ~/projects
cd ~/projects
```

### 4.2 XSA 파일 준비

``` bash
cp /mnt/share/design_1_wrapper.xsa ~/projects/
unzip -l design_1_wrapper.xsa
```

### 4.3 Zynq 프로젝트 생성

``` bash
source ~/petalinux/2022.2/settings.sh
petalinux-create --type project --template zynq --name myproject
cd myproject
```

### 4.4 하드웨어 설정 가져오기

``` bash
petalinux-config --get-hw-description=~/projects/
```

### 4.5 시스템 설정 (중요)

-   **Root FS**: SD card
-   **UART**: ps7_uart_1
-   **Ethernet**: ps7_ethernet_0
-   **SD/SDIO**: ps7_sd_0

### 4.6 Root Filesystem 패키지

``` bash
petalinux-config -c rootfs
```

권장 패키지: sudo, vim, nano, gcc, g++, make, openssh

### 4.7 Root 로그인 설정

-   Username: root
-   Password: root

------------------------------------------------------------------------

## 5. PetaLinux 빌드

### 5.1 전체 빌드

``` bash
cd ~/projects/myproject
source ~/petalinux/2022.2/settings.sh
petalinux-build
```

### 5.2 부트 이미지 생성

``` bash
petalinux-package --boot     --fsbl images/linux/zynq_fsbl.elf     --fpga images/linux/design_1_wrapper.bit     --u-boot images/linux/u-boot.elf     --force
```

👉 생성: `images/linux/BOOT.BIN`

------------------------------------------------------------------------

## 7. SD 카드 이미지 생성

### 7.1 WIC 이미지 생성

``` bash
petalinux-package --wic     --bootfiles "BOOT.BIN image.ub boot.scr"     --images-dir images/linux/
```

👉 생성: `images/linux/petalinux-sdimage.wic`

### 7.2 Windows로 복사

``` bash
cp petalinux-sdimage.wic /mnt/share/
```

------------------------------------------------------------------------

## 8. Windows에서 SD 카드 굽기

-   Tool: **balenaEtcher**
-   파일: `petalinux-sdimage.wic`

절차: 1. balenaEtcher 실행 2. Flash from file → `petalinux-sdimage.wic`
3. SD 카드 선택 4. Flash! 클릭

------------------------------------------------------------------------

## 9. Zybo Z7-20 부팅

-   Boot Mode: SD 선택 (JP5 점퍼)
-   UART 연결: 115200 baud
-   전원 ON

부팅 로그:

    Xilinx Zynq First Stage Boot Loader
    U-Boot 2022.01
    Starting kernel ...
    Linux version 5.15.36-xilinx-v2022.2

로그인:

``` bash
Username: root
Password: root
```

------------------------------------------------------------------------
