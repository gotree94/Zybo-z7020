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

## Ubuntu 22.04.5 및 PetaLinux 2022.2 설치 가이드
### 1. VirtualBox에 Ubuntu 22.04.5 설치
#### 1.1 VirtualBox 설정

* 새 가상머신 생성
  * 이름: PetaLinux-Ubuntu
  * 타입: Linux
  * 버전: Ubuntu (64-bit)

* 리소스 할당
  * 메모리: 최소 8GB (권장 16GB)
  * CPU: 최소 4코어 (권장 8코어)
  * 디스크: 최소 100GB (권장 200GB 이상)
  * VT-x/AMD-V 가상화 활성화

* 네트워크 설정
  * NAT 또는 Bridge 모드 설정

#### 1.2 Ubuntu 설치
* ubuntu-22.04.5-desktop-amd64.iso 마운트
* 일반 설치 진행
* 사용자 계정 생성 (예: petalinux)

### 2. Ubuntu 시스템 준비
#### 2.1 시스템 업데이트

   * 전체 통합 명령어
```bash
sudo apt update
sudo apt upgrade -y

# 시스템 업데이트
sudo apt update
sudo apt upgrade -y

# 32비트 지원 추가
sudo dpkg --add-architecture i386
sudo apt update

# 메인 패키지 설치
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

# 32비트 라이브러리
sudo apt install -y \
    libncurses5:i386 \
    libc6:i386 \
    libstdc++6:i386 \
    lib32z1 \
    zlib1g:i386

# Locale 설정
sudo locale-gen en_US.UTF-8

# Dash를 Bash로 변경
sudo dpkg-reconfigure dash
# "No" 선택
```





## 참고 자료
- [Xilinx PetaLinux Tools Documentation](https://docs.xilinx.com/r/en-US/ug1144-petalinux-tools-reference-guide)
- [Digilent Zybo Z7 Reference Manual](https://digilent.com/reference/programmable-logic/zybo-z7/reference-manual)
- [VirtualBox Documentation](https://www.virtualbox.org/manual/)

이 가이드를 통해 VirtualBox Ubuntu 환경에서 Zybo Z7-20용 PetaLinux 시스템을 성공적으로 구축할 수 있습니다.
