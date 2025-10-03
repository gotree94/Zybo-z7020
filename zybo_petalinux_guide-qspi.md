# Digilent Zybo 7020 PetaLinux SD Card + Quad-SPI Flash Boot Guide

## 2. Ubuntu 시스템 준비

👉 SD 카드 버전과 동일 (생략)

------------------------------------------------------------------------

## 3. PetaLinux 2022.2 설치

👉 동일 (생략)

------------------------------------------------------------------------

## 4. Zybo Z7-20 프로젝트 생성

👉 동일 (생략)

------------------------------------------------------------------------

## 5. PetaLinux 빌드

### 5.1 전체 빌드

``` bash
cd ~/projects/myproject
source ~/petalinux/2022.2/settings.sh
petalinux-build
```

### 5.2 부트 이미지 생성 (Quad-SPI Flash 용)

``` bash
petalinux-package --boot \
    --fsbl images/linux/zynq_fsbl.elf \
    --fpga images/linux/design_1_wrapper.bit \
    --u-boot images/linux/u-boot.elf \
    --force
```

👉 생성: `images/linux/BOOT.BIN`

### 5.3 Flash 이미지 생성

``` bash
petalinux-package --boot \
  --fsbl images/linux/zynq_fsbl.elf \
  --fpga images/linux/design_1_wrapper.bit \
  --u-boot images/linux/u-boot.elf \
  --dtb images/linux/system.dtb \
  --kernel images/linux/image.ub \
  --offset 0xF40000 \
  --format MCS \
  --force
```

👉 Quad-SPI Flash 용 MCS 파일 생성됨:\
`images/linux/download.mcs`

------------------------------------------------------------------------

## 6. QSPI Flash에 프로그래밍

### 6.1 Vivado Hardware Manager 사용

1.  보드 JTAG 연결
2.  Vivado 실행 → Hardware Manager → Open target
3.  "Add Configuration Memory Device" 선택
4.  16 MB SPI Flash 선택 (`s25fl128sxxxxxx0-spi-x1_x2_x4`)
5.  `download.mcs` 파일 선택 후 Program

### 6.2 XSDB 사용

``` bash
xsct
connect
targets -set -filter {name =~ "APU*"} 
fpga -file images/linux/design_1_wrapper.bit
source images/linux/zynq_fsbl.elf
program_flash -f images/linux/download.mcs -offset 0 -flash_type qspi_single -verify
```

------------------------------------------------------------------------

## 7. SD 카드 + QSPI 부트 구성

-   **QSPI Flash**: FSBL + Bitstream + U-Boot
-   **SD 카드**: `image.ub`, `boot.scr`, `rootfs.ext4`

### SD 카드 준비

👉 SD 카드 버전과 동일하게 `petalinux-package --wic`로 이미지 생성 후 SD
카드에 굽기

------------------------------------------------------------------------

## 8. Zybo Z7-20 부팅 (QSPI + SD 조합)

-   Boot Mode: **QSPI** 선택 (JP5 점퍼)
-   QSPI에서 FSBL → Bitstream → U-Boot 로딩
-   이후 SD 카드에서 커널/루트파일시스템 로드

부팅 로그:

    Xilinx Zynq First Stage Boot Loader
    QSPI: Booting from Flash
    U-Boot 2022.01
    Loading image.ub from SD...
    Starting kernel ...
    Linux version 5.15.36-xilinx-v2022.2

로그인:

``` bash
Username: root
Password: root
```

------------------------------------------------------------------------
