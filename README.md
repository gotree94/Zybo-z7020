# Zybo-z7020


# Zybo Z7-20 PetaLinux 설치 완전 가이드

## 개요
Digilent Zybo Z7-20 보드에 PetaLinux를 설치하여 GPIO 4버튼을 사용할 수 있도록 하는 전체 과정을 다룹니다.

## 사전 요구사항
- Vivado 2022.1 이상
- PetaLinux Tools 2022.1 이상
- Ubuntu 22.04 (WSL2 또는 네이티브)
- 최소 100GB의 저장공간
- 16GB 이상의 RAM 권장

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
sudo apt install -y pylint3 xterm rsync
```

### 1.2 Bash 쉘 설정 (Ubuntu 22.04에서 dash 사용하는 경우)
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

## 2단계: PetaLinux Tools 설치 및 문제 해결

### 2.1 현재 상태 확인
먼저 PetaLinux가 설치되어 있는지 확인합니다:
```bash
# PetaLinux 명령어 확인
which petalinux-create
echo $PETALINUX

# 설치 경로 확인
ls -la /opt/pkg/petalinux/ 2>/dev/null || echo "PetaLinux not found in /opt/pkg/petalinux/"
ls -la ~/petalinux/ 2>/dev/null || echo "PetaLinux not found in ~/petalinux/"

# 환경변수 확인
grep -i petalinux ~/.bashrc
```

### 2.2 PetaLinux Tools 다운로드
Xilinx 다운로드 페이지에서 PetaLinux Tools 설치 파일을 다운로드합니다.

**다운로드 방법:**
1. https://www.xilinx.com/support/download/index.html/content/xilinx/en/downloadNav/embedded-design-tools.html 접속
2. AMD 계정으로 로그인 (무료 계정 생성 가능)
3. PetaLinux Tools 2022.1 (또는 최신 버전) 다운로드
4. `petalinux-v2022.1-final-installer.run` 파일을 다운로드 폴더에 저장

### 2.3 PetaLinux 설치 (Option 1: 시스템 전역 설치)
```bash
cd ~/Downloads
chmod +x petalinux-v2022.1-final-installer.run

# 설치 디렉토리 생성
sudo mkdir -p /opt/pkg/petalinux/2022.1
sudo chown $USER:$USER /opt/pkg/petalinux/2022.1

# PetaLinux 설치 (약 10-30분 소요)
./petalinux-v2022.1-final-installer.run -d /opt/pkg/petalinux/2022.1

# 환경변수 설정
echo 'source /opt/pkg/petalinux/2022.1/settings.sh' >> ~/.bashrc
source ~/.bashrc
```

### 2.4 PetaLinux 설치 (Option 2: 홈 디렉토리 설치 - 권한 문제 시)
```bash
cd ~/Downloads
chmod +x petalinux-v2022.1-final-installer.run

# 홈 디렉토리에 설치
mkdir -p ~/petalinux/2022.1
./petalinux-v2022.1-final-installer.run -d ~/petalinux/2022.1

# 환경변수 설정
echo 'source ~/petalinux/2022.1/settings.sh' >> ~/.bashrc
source ~/.bashrc
```

### 2.5 설치 후 확인
```bash
# 새 터미널을 열거나 bashrc 재로드
source ~/.bashrc

# PetaLinux 명령어 확인
which petalinux-create
petalinux-create --help

# 버전 확인
echo $PETALINUX
```

### 2.6 문제 해결
만약 여전히 `petalinux-create: command not found` 오류가 발생하면:

**A. 수동으로 환경변수 설정:**
```bash
# 설치 경로 확인 (Option 1이나 2 중 설치한 경로)
ls -la /opt/pkg/petalinux/2022.1/settings.sh
# 또는
ls -la ~/petalinux/2022.1/settings.sh

# 수동으로 환경변수 로드
source /opt/pkg/petalinux/2022.1/settings.sh
# 또는
source ~/petalinux/2022.1/settings.sh
```

**B. 설치 파일 재확인:**
```bash
cd ~/Downloads
ls -la petalinux*
file petalinux-v2022.1-final-installer.run
```

**C. 터미널 재시작:**
현재 터미널을 닫고 새 터미널을 열어 환경변수가 제대로 로드되는지 확인하세요.

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
```bash
# design_1_wrapper.xsa를 현재 디렉토리로 복사 후
petalinux-config --get-hw-description=./
```

---

## 4단계: PetaLinux 구성

### 4.1 시스템 구성
위 명령어 실행 후 menuconfig가 열리면 다음 설정을 확인/변경합니다:

**DTG Settings**
- System Tree Information → bootargs
  - `earlycon console=ttyPS0,115200 clk_ignore_unused root=/dev/mmcblk0p2 rw rootwait`

**Image Packaging Configuration**
- Root filesystem type → `EXT4 (SD/eMMC/SATA/USB)`
- Device node of SD device → `/dev/mmcblk0p2`

**Yocto Settings**
- YOCTO_MACHINE_NAME → `zynq-generic`

설정 완료 후 저장하고 종료합니다.

### 4.2 루트 파일시스템 구성
```bash
petalinux-config -c rootfs
```

필요한 패키지들을 선택합니다:
- `user packages` → 필요한 패키지 선택
- `Filesystem Packages` → `base` → `busybox` 등 기본 패키지들

### 4.3 커널 구성 (필요시)
```bash
petalinux-config -c kernel
```

GPIO 관련 드라이버가 활성화되어 있는지 확인:
- `Device Drivers` → `GPIO Support`
- `Device Drivers` → `GPIO Support` → `Memory mapped GPIO drivers`

---

## 5단계: 디바이스 트리 수정 (GPIO 버튼 설정)

### 5.1 시스템 사용자 디바이스 트리 편집
```bash
vim project-spec/meta-user/recipes-bsp/device-tree/files/system-user.dtsi
```

다음 내용을 추가합니다:
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
            gpios = <&gpio0 50 0>; /* GPIO 50 */
            linux,code = <0x100>; /* BTN_0 */
            gpio-key,wakeup;
        };
        
        btn1 {
            label = "btn1";
            gpios = <&gpio0 51 0>; /* GPIO 51 */
            linux,code = <0x101>; /* BTN_1 */
            gpio-key,wakeup;
        };
        
        btn2 {
            label = "btn2";
            gpios = <&gpio0 52 0>; /* GPIO 52 */
            linux,code = <0x102>; /* BTN_2 */
            gpio-key,wakeup;
        };
        
        btn3 {
            label = "btn3";
            gpios = <&gpio0 53 0>; /* GPIO 53 */
            linux,code = <0x103>; /* BTN_3 */
            gpio-key,wakeup;
        };
    };
};
```

**주의**: GPIO 번호는 Vivado 설계에서 사용한 실제 핀 번호에 맞게 수정해야 합니다.

---

## 6단계: PetaLinux 빌드

### 6.1 전체 시스템 빌드
```bash
petalinux-build
```
빌드 시간은 시스템 사양에 따라 30분~2시간 정도 소요됩니다.

### 6.2 부트 이미지 생성
```bash
petalinux-package --boot --fsbl images/linux/zynq_fsbl.elf --fpga images/linux/design_1_wrapper.bit --u-boot
```

---

## 7단계: SD 카드 이미지 생성

### 7.1 WIC 이미지 생성
```bash
petalinux-package --wic --images-dir images/linux/ --bootfiles "BOOT.BIN boot.scr Image system.dtb"
```

### 7.2 이미지 파일 확인
생성된 이미지 파일들:
```bash
ls images/linux/
# BOOT.BIN - 부트로더 + FPGA 비트스트림
# Image - 리눅스 커널
# system.dtb - 디바이스 트리
# rootfs.ext4 - 루트 파일시스템
# petalinux-sdimage.wic - 완전한 SD 카드 이미지
```

---

## 8단계: Windows로 이미지 전송

### 8.1 WSL에서 Windows로 파일 복사
```bash
# WSL 사용시
cp images/linux/petalinux-sdimage.wic /mnt/c/Users/[사용자명]/Desktop/
```

### 8.2 네이티브 Ubuntu 사용시
- USB 드라이브나 네트워크를 통해 Windows로 전송
- 또는 scp를 사용하여 전송

---

## 9단계: SD 카드에 이미지 쓰기 (Windows)

### 9.1 도구 설치
Windows에서 다음 중 하나의 도구를 설치합니다:
- **Rufus** (권장): https://rufus.ie/
- **Win32 Disk Imager**: https://sourceforge.net/projects/win32diskimager/
- **Raspberry Pi Imager**: https://rpi.org/imager/

### 9.2 Rufus를 사용한 이미지 쓰기
1. Rufus 실행
2. **Device**: SD 카드 선택 (보통 8GB 이상 권장)
3. **Boot selection**: `petalinux-sdimage.wic` 파일 선택
4. **Partition scheme**: `MBR`
5. **Target system**: `BIOS or UEFI`
6. **START** 버튼 클릭
7. 모든 데이터가 삭제된다는 경고에서 **OK** 클릭

### 9.3 Win32 Disk Imager 사용법
1. Win32 Disk Imager 실행
2. **Image File**: `petalinux-sdimage.wic` 파일 선택
3. **Device**: SD 카드 드라이브 선택
4. **Write** 버튼 클릭

---

## 10단계: 보드 부팅 및 테스트

### 10.1 하드웨어 설정
1. SD 카드를 Zybo Z7-20 보드에 삽입
2. UART-USB 케이블 연결 (115200 baud)
3. 점퍼 설정을 SD 부팅 모드로 변경
4. 전원 공급

### 10.2 부팅 확인
시리얼 터미널(PuTTY, TeraTerm 등)에서 부팅 과정 확인:
```
U-Boot 2022.01 (날짜) Xilinx Zynq
...
Starting kernel ...
...
Welcome to PetaLinux
zynq-generic login: root
```

### 10.3 GPIO 버튼 테스트
```bash
# 부팅 후 루트로 로그인
root@zynq-generic:~# 

# GPIO 버튼 이벤트 확인
cat /proc/bus/input/devices
# gpio-keys 디바이스 확인

# 버튼 입력 테스트
evtest /dev/input/event0
# 버튼을 누르면 이벤트가 표시됨
```

---

## 트러블슈팅

### 일반적인 문제들

**1. 빌드 오류**
```bash
# 클린 빌드
petalinux-build -c cleanall
petalinux-build
```

**2. 디스크 공간 부족**
```bash
# 임시 파일 정리
petalinux-build -c cleanall
rm -rf build/tmp/
```

**3. 부팅이 안 되는 경우**
- SD 카드 파티션 테이블 확인
- UART 연결 및 점퍼 설정 재확인
- BOOT.BIN 파일 재생성

**4. GPIO 버튼이 인식되지 않는 경우**
- 디바이스 트리의 GPIO 번호 확인
- Vivado 설계에서 실제 핀 할당 재확인
- 커널 로그에서 gpio-keys 드라이버 로딩 확인

### 유용한 디버깅 명령어
```bash
# 커널 메시지 확인
dmesg | grep gpio

# GPIO 상태 확인
cat /sys/kernel/debug/gpio

# 입력 디바이스 목록
ls /dev/input/

# 시스템 정보
cat /proc/version
cat /proc/cpuinfo
```

---

## 참고 자료
- [Xilinx PetaLinux Tools Documentation](https://docs.xilinx.com/r/en-US/ug1144-petalinux-tools-reference-guide)
- [Digilent Zybo Z7 Reference Manual](https://digilent.com/reference/programmable-logic/zybo-z7/reference-manual)
- [Linux GPIO Subsystem Documentation](https://www.kernel.org/doc/html/latest/driver-api/gpio/)

이 가이드를 통해 Zybo Z7-20에서 GPIO 4버튼이 포함된 PetaLinux 시스템을 성공적으로 구축할 수 있습니다.
