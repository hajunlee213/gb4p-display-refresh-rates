# gb4p-display-refresh-rates

삼성 갤럭시 북4 프로 16인치(Samsung ATNA60CL07-0, 2880×1800 AMOLED)에서 리눅스의 디스플레이 설정에 **60 / 75 / 80 / 100 / 120Hz**를 모두 나타나게 하는 커스텀 EDID 패치입니다.

Custom EDID that exposes 60 / 75 / 80 / 100 / 120 Hz modes on the Galaxy Book 4 Pro 16" (Samsung ATNA60CL07-0 AMOLED) under Linux, where the factory EDID lists 120 Hz only.

> **경고 / Warning**
> 이 EDID는 위 패널 전용입니다. 다른 기기·패널에 적용하면 화면이 나오지 않을 수 있습니다. 설치 스크립트는 패널 이름(`ATNA60CL07-0`)을 확인하고 다르면 중단합니다 (`--force`로 우회 가능, 권장하지 않음).
> 설치는 `/etc/default/grub`(백업 생성)과 부팅 램디스크를 수정합니다. 아래 [복구](#복구)를 먼저 읽어 두세요.
> 보증 없이 제공됩니다 (MIT).

## 문제

삼성 출고 EDID에는 `2880x1800 @ 120Hz` 상세 타이밍(DTD) 하나뿐이라, 커널 DRM이 120Hz 모드 하나만 등록합니다. 윈도우 드라이버는 패널의 VRR 범위(48~120Hz)를 보고 낮은 주사율을 만들어 내지만 리눅스는 그렇게 하지 않습니다. 그래서 배터리용 60Hz를 고를 수 없습니다.

일반 LCD처럼 픽셀 클럭을 낮추면 이 OLED의 TCON이 링크 동기를 잃고 화면이 꺼집니다.

## 해결

도트클럭(655.13 MHz)과 수평 라인 주파수(219.84 kHz)를 120Hz와 똑같이 두고, **수직 블랭킹(V-Blank) 라인만 늘려** 주사율을 낮추는 DTD를 EDID에 추가합니다.

| 주사율 | V-Total | V-Blank |
| :--- | :---: | :---: |
| 120Hz (출고) | 1832 | 32 |
| 100Hz | 2198 | 398 |
| 80Hz | 2748 | 948 |
| 75Hz | 2931 | 1131 |
| 60Hz | 3664 | 1864 |

BT.2020 색역, HDR Static Metadata, FreeSync/VRR(48~120Hz) 블록은 그대로 둡니다.

## 설치

```bash
git clone https://github.com/hajunlee213/gb4p-display-refresh-rates.git
cd gb4p-display-refresh-rates
sudo ./install.sh
sudo reboot
```

재부팅 뒤 **설정 → 디스플레이 → 주사율**에서 60 / 75 / 80 / 100 / 120Hz를 고를 수 있습니다.

`install.sh`가 하는 일 (여러 번 실행해도 안전합니다):

| 경로 | 내용 |
| :--- | :--- |
| `/lib/firmware/edid/gb4p_custom_edid.bin`, `/usr/lib/firmware/edid/gb4p_custom_edid.bin` | 커스텀 EDID 바이너리 (256B) |
| `/etc/dracut.conf.d/edid.conf` | dracut 램디스크에 EDID 포함 (dracut이 있을 때) |
| `/etc/initramfs-tools/hooks/edid` | initramfs-tools 램디스크에 EDID 포함 (있을 때) |
| `/etc/default/grub` | 커널 파라미터 `drm.edid_firmware=eDP-1:edid/gb4p_custom_edid.bin` 추가 (`.bak.<시각>` 백업 생성), `update-grub` 실행 |

요구 사항: `grub2`, `dracut` 또는 `initramfs-tools`, root 권한. Ubuntu 22.04 / 24.04 / 26.04에서 쓰던 스크립트이며 Wayland·X11 모두 상관없습니다. 확인 환경은 Ubuntu 26.04 (kernel 7.0), i915입니다.

## 확인

```bash
grep -o 'drm.edid_firmware=[^ ]*' /proc/cmdline
```

## 제거

```bash
sudo ./uninstall.sh
sudo reboot
```

`~/.config/monitors.xml`(GNOME 디스플레이 캐시), dracut/initramfs 설정, 펌웨어 바이너리, GRUB 파라미터를 지우고 램디스크와 GRUB을 다시 만듭니다.

## 복구

화면이 검게 나오거나 부팅이 멈추면:

1. **부팅 중**: 로고가 뜰 때 `Shift` 또는 `Esc`를 눌러 GRUB 메뉴로 들어갑니다. 우분투 항목에서 `e`를 누르고 `linux` 줄의 `drm.edid_firmware=...`를 지운 뒤 `Ctrl+X`로 부팅합니다. 순정 120Hz로 뜹니다.
2. **TTY**: `Ctrl+Alt+F3`로 콘솔에 로그인해 `sudo ./uninstall.sh` 후 재부팅합니다.

## 다른 패널 / EDID 다시 만들기

`generate-edid.py`는 현재 패널의 EDID(`/sys/class/drm/card*-eDP-*/edid`)를 읽어 DTD만 바꾼 256B 바이너리를 만듭니다. 새 EDID가 필요하면:

```bash
python3 generate-edid.py [입력_edid] [출력_bin]
```

다른 패널은 픽셀 클럭과 H/V 타이밍이 다르므로 스크립트 안의 `pclk_fixed`, 블랭킹 값, 해상도를 직접 맞춰야 합니다. 검증하지 않은 값을 쓰면 블랙아웃이 날 수 있습니다.

## 관련

* 전원(AC/배터리)에 따라 주사율을 자동 전환하고 `sdr-native`를 유지하는 데몬은 이 레포가 아니라 [GB4P_ubuntu_my_preferences](https://github.com/hajunlee213/GB4P_ubuntu_my_preferences)의 `power_refresh_sdr.sh`에 있습니다.
