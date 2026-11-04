# holybro-flash-image

[holybro-build-image](https://github.com/JunghanShin/holybro-build-image)로 만든 플래시 이미지를 **Jetson Orin NX에 쓰는** 스크립트입니다. 이미지를 만들지는 않습니다.

보드를 USB로 연결하는 단계는 이것 하나입니다. 이미지는 미리 만들어 두었으므로, 연결 시간이 짧아 USB write 에러가 나지 않습니다.

```text
holybro-build-image.sh   (보드 없이, 오래 걸림)
        │ 작업 폴더/Linux_for_Tegra/tools/kernel_flash/images/
        ▼
holybro-flash-image.sh   (보드 REC 모드, USB 연결)   ← 이 저장소
  NVMe SSD(nvme0n1p1)와 QSPI 부트로더에 쓰기
```

| 항목 | 값 |
|---|---|
| 모듈 | Jetson Orin NX |
| 캐리어 보드 | Holybro Pixhawk Jetson Baseboard |
| 소프트웨어 | JetPack 6.2.2 (L4T 36.5.0) |
| 설치 위치 | NVMe SSD, 부트로더는 QSPI |

## 준비

| 항목 | 내용 |
|---|---|
| Linux | x86_64 Ubuntu 22.04 PC 또는 가상 머신 (VirtualBox, VMware). **이미지를 만든 환경과 같은 곳** (작업 폴더가 있어야 함) |
| 권한 | `sudo` (스크립트가 스스로 `sudo`로 다시 실행) |
| 이미지 | `holybro-build-image.sh`로 만든 작업 폴더 |
| 케이블 | PC와 캐리어 보드 USB-C 포트를 잇는 데이터 케이블 |

가상 머신에서 플래시하려면 USB 설정이 필요합니다. 플래시 중에 보드가 USB 장치로 두 번 나타나기 때문에, NVIDIA 장치(Vendor ID `0955`)가 항상 가상 머신으로 자동 연결되게 해야 합니다. 설정 방법은 [holybro-build-image README의 가상 머신 절](https://github.com/JunghanShin/holybro-build-image#가상-머신-virtualbox-vmware)에 있습니다.

WSL2는 USB 장치가 재연결될 때 자동으로 넘겨지지 않아 쓰지 않습니다.

## 설치

```bash
git clone https://github.com/JunghanShin/holybro-flash-image.git
```

이미지를 만드는 [holybro-build-image](https://github.com/JunghanShin/holybro-build-image)도 같이 받아 둡니다.

## 사용법

1. 이미지를 먼저 만듭니다. 방법은 holybro-build-image README를 참고합니다.
   ```bash
   ./holybro-build-image/holybro-build-image.sh -w ~/holybro-work
   ```
2. 보드 전원을 끄고 캐리어 보드의 **REC 스위치를 켭니다**.
3. PC와 보드를 USB-C로 연결하고 전원을 켭니다.
4. 복구 장치가 **480Mbps**로 잡혔는지 확인합니다. NVIDIA 장치(`0955:7323` 등)가 `480M`으로 보여야 합니다.
   ```bash
   lsusb -t
   ```
5. 이미지를 만들 때와 **같은 작업 폴더**를 지정해 씁니다.
   ```bash
   ./holybro-flash-image/holybro-flash-image.sh -w ~/holybro-work
   ```
6. `Flash finished`가 나오면 전원을 끄고, REC 스위치를 원래 위치로 돌린 뒤 전원을 켭니다.

초기 설정 화면 없이 부팅되고 자동 로그인됩니다. 이미지에 사용자 `dev`(비밀번호 `roqkf`)와 JetPack 6.2.2 런타임이 들어 있어 바로 SSH로 접속할 수 있습니다. 플래시할 때 쓴 USB-C 케이블을 그대로 두면 USB 원격으로 접속할 수 있고, 보드 주소는 `192.168.55.1`입니다. 부팅 후 1분쯤 기다립니다. USB 원격은 보드에서 `sudo usb-role on|off`로 켜고 끕니다(기본 켜짐).

```bash
ssh dev@192.168.55.1
```

## 작업 폴더

- `-w 폴더`로 지정합니다.
- 지정하지 않으면 환경 변수 `WORK_DIR`, 그것도 없으면 **명령을 실행한 현재 폴더**를 씁니다.
- 작업 폴더에 `Linux_for_Tegra/tools/kernel_flash/images/external/system.img`가 없으면 아무것도 쓰지 않고 끝납니다.

## 옵션

| 옵션 | 설명 |
|---|---|
| `-w 폴더` | `holybro-build-image.sh`가 쓴 작업 폴더 (기본: `WORK_DIR`, 없으면 현재 폴더) |
| `-h` | 사용법 출력 |

## 스크립트가 하는 일

1. 작업 폴더에 이미지가 있는지 확인합니다.
2. USB 버퍼를 키우고(`usbfs_memory_mb=2048`) USB 절전을 끕니다. 쓰는 도중 연결이 끊기지 않게 하기 위해서입니다.
3. NVIDIA 복구 장치(`0955:7323`/`7423`/`7523`/`7623`)를 찾고, 속도가 480Mbps인지 확인합니다.
4. `l4t_initrd_flash.sh --flash-only`로 이미지를 씁니다.

## 문제 해결

| 증상 | 원인과 조치 |
|---|---|
| `Flash images are missing` | 작업 폴더가 다르거나 이미지를 아직 만들지 않았습니다. `-w`로 이미지를 만든 폴더를 지정합니다. |
| `Need a 480 Mbps recovery device` (`speed=missing`) | 보드가 REC 모드가 아닙니다. 전원을 끄고 REC 스위치를 켠 뒤 다시 켭니다. |
| `Need a 480 Mbps recovery device` (`speed=5000` 등) | USB 3.x 속도로 잡혔습니다. PC의 다른 USB 포트(USB 2.0)나 USB 2.0 허브를 씁니다. |
| 쓰는 도중 USB write 에러 | 케이블을 짧은 데이터 케이블로 바꾸고, PC 본체 뒤쪽 포트에 직접 연결합니다. |

## 상태

- Ubuntu 22.04 PC에서 플래시를 확인했습니다(`Flash successful`).
- 가상 머신(VirtualBox, VMware)에서는 아직 시험하지 않았습니다.
