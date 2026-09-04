# FSDS 시뮬레이터 + ROS2 브릿지 — 팀 가이드

이 문서는 이 컴퓨터에서 시뮬레이터와 ROS2 브릿지를 실행하는 방법을 빠르게
참고하기 위한 문서입니다. 최초 1회만 필요한 환경 설정(NVIDIA 드라이버, ROS2
Humble 설치, colcon 빌드)은 이미 끝나있다고 가정합니다 — **새 컴퓨터**에서
처음부터 설치하거나, 낯선 크래시를 만났다면 먼저 `SETUP_DEBUG_LOG.md`를
참고하세요 (지금까지 만난 모든 버그와 해결 방법의 전체 기록).

(영문 원본: `SIMULATOR_GUIDE.md`)

## 폴더 구조

```
A1 Challenge Simulator/
├── Formula-Student-Driverless-Simulator/   # 클론한 저장소 (원본/기준 파일)
│   ├── settings.json                       # 차량/센서 설정 — 여기를 수정
│   ├── ros2/                                # 빌드된 colcon 워크스페이스
│   └── maps/YonginCircuit/                  # 커스텀 트랙 (maps/YonginCircuit/README.md 참고)
├── simulator/                               # 미리 빌드된 v2.2.0 바이너리
│   └── FSDS.sh                              # 이걸 실행
└── SETUP_DEBUG_LOG.md / SIMULATOR_GUIDE.md  # 문서 (이 파일)
```

`~/Formula-Student-Driverless-Simulator`는 위 저장소 폴더로 연결된
**심볼릭 링크(symlink)** 입니다 (FSDS 공식 문서와 ROS2 launch 파일이
기대하는 경로와 일치시키기 위함). 실제 디렉토리로 다시 만들지 마세요 —
혹시 없어졌다면 아래 명령으로 복구:
```bash
ln -s "/home/ailab/git/A1 Challenge Simulator/Formula-Student-Driverless-Simulator" \
      ~/Formula-Student-Driverless-Simulator
```

## 1. 시뮬레이터 실행

```bash
cd "/home/ailab/git/A1 Challenge Simulator/simulator"
./FSDS.sh
```
이게 전부입니다 — 창모드(windowed)가 이제 기본값으로 `FSDS.sh`에
내장되어 있고, `settings.json`도 위의 `$HOME` 심볼릭 링크를 통해 자동으로
찾습니다.

실행하면 메인 메뉴가 뜨는데, 기본 훈련용 트랙을 쓰려면 **Play**를
클릭하면 되고, 특정 트랙을 쓰려면 **CustomMap**을 선택한 뒤 트랙 CSV
경로를 입력창에 붙여넣으면 됩니다.

### 커맨드라인으로 커스텀 트랙(예: 용인 서킷) 불러오기

```bash
./FSDS.sh -CustomMapPath=/home/ailab/track_yongin.csv
```
이 경우 메인 메뉴를 건너뛰고 바로 해당 트랙으로 진입합니다.

**중요**: `-CustomMapPath=` 뒤의 경로에는 공백이 있으면 안 됩니다 (이
프로젝트 폴더 이름 자체에 공백이 있음) — 커맨드라인 인자에 공백이 있으면
Unreal 엔진 내부에서 인자를 다시 토큰화하는 과정에서 깨집니다. 공백 없는
경로에 복사본을 하나 둬서 사용하세요:
```bash
cp "/home/ailab/git/A1 Challenge Simulator/Formula-Student-Driverless-Simulator/maps/YonginCircuit/track_yongin.csv" \
   /home/ailab/track_yongin.csv
```
(GUI의 CustomMap 텍스트 입력창으로 불러올 때는 이 제약이 없습니다 — 그건
커맨드라인 인자가 아니라 텍스트 입력창이기 때문입니다.)

## 2. ROS2 브릿지 실행

필요에 따라 아래 두 가지 방법 중 하나를 선택하세요:

### A. 센서만 (IMU/GPS/GSS/라이다, 카메라 없음)
```bash
source /opt/ros/humble/setup.bash
source "/home/ailab/git/A1 Challenge Simulator/Formula-Student-Driverless-Simulator/ros2/install/setup.bash"
ros2 run fsds_ros2_bridge fsds_ros2_bridge
```
토픽이 **네임스페이스 없이** 나옵니다: `/gps`, `/imu`, `/gss`,
`/lidar/Lidar1`, `/lidar/Lidar2`, `/control_command`,
`/testing_only/odom`, `/testing_only/track`, `/signal/go`,
`/signal/finished`, `/tf_static`, `/wheel_states`.

### B. 카메라 포함 전체 구성 (권장)
```bash
source /opt/ros/humble/setup.bash
source "/home/ailab/git/A1 Challenge Simulator/Formula-Student-Driverless-Simulator/ros2/install/setup.bash"
ros2 launch fsds_ros2_bridge fsds_ros2_bridge.launch.py
```
`settings.json`에 설정된 카메라마다 노드를 하나씩 추가로 띄우고 (현재는
`cam1`, `cam2`), 모든 토픽이 `/fsds/...` 네임스페이스로 제대로 나옵니다:
`/fsds/gps`, `/fsds/imu`, `/fsds/cam1/image_color`,
`/fsds/cam1/camera_info`, `/fsds/lidar/Lidar1` 등. 이 방식은 위에서 만든
`$HOME` 심볼릭 링크를 이용해 `settings.json`을 찾습니다.

어느 방법이든 터미널에 아래처럼 뜨면 정상입니다:
```
[INFO] [fsds_ros2_bridge]: Connected to the simulator!
...
GPS enabled
GSS enabled
IMU enabled
```
이건 브릿지가 실행 중인 시뮬레이터를 찾아서 연결에 성공했다는 뜻입니다
(반드시 시뮬레이터를 **먼저** 켜고, 브릿지를 나중에 실행하세요).

## 3. 정상 작동 확인

```bash
ros2 topic list
ros2 topic hz /imu          # B 방법이면 /fsds/imu — 약 250Hz가 나와야 정상
ros2 topic echo /gps --once # B 방법이면 /fsds/gps
```
커스텀 맵을 쓸 때는 화면에 보이는 것만 믿지 말고, 시뮬레이터가 실제로
가지고 있는 콘 목록을 직접 확인하세요 (이유는 `maps/YonginCircuit/README.md`
참고):
```bash
ros2 topic echo /testing_only/track --once
```

## 4. 차량/센서 설정 바꾸기

**`Formula-Student-Driverless-Simulator/settings.json`**을 직접
수정하세요 — 이게 유일한 원본 파일입니다 (시뮬레이터와 `ros2 launch`의
카메라 노드 생성 로직 둘 다 `$HOME` 심볼릭 링크를 통해 이 파일을 읽고,
브릿지 자체의 센서 토픽은 파일이 아니라 RPC로 시뮬레이터에서 실시간으로
받아오므로 항상 시뮬레이터가 실제로 로드한 설정과 일치합니다). 수정 후
반영하려면 시뮬레이터와 브릿지를 둘 다 재시작해야 합니다.

### 4.1 지원되는 센서 종류

`Vehicles.FSCar.Sensors` 아래에 추가하는 일반 센서 (`SensorBase.hpp`에서
직접 확인한 값):

| 센서 | `SensorType` 값 |
|---|---|
| `Imu` | 2 |
| `Gps` | 3 |
| `Distance` | 5 |
| `Lidar` | 6 |
| `GSS` | 7 |

카메라는 이 목록과 별개로, `Vehicles.FSCar.Cameras` 아래에 따로 추가합니다
(자체 `SensorType` 값이 없음).

### 4.2 새 라이다 추가 예시

`Sensors` 항목 안에 원하는 이름(키)으로 블록을 하나 추가하면 됩니다.
예를 들어 `Lidar3`이라는 이름으로 추가하려면:

```json
"Lidar3": {
  "SensorType": 6,
  "Enabled": true,
  "X": -1.0, "Y": 0, "Z": 1.0,
  "Roll": 0, "Pitch": 0, "Yaw": 180,
  "NumberOfLasers": 4,
  "PointsPerScan": 4096,
  "VerticalFOVUpper": 5,
  "VerticalFOVLower": -5,
  "HorizontalFOVStart": -90,
  "HorizontalFOVEnd": 90,
  "RotationsPerSecond": 10,
  "DrawDebugPoints": false
}
```
`X`/`Y`/`Z`는 차량 기준 위치(미터), `Roll`/`Pitch`/`Yaw`는 장착 각도(도)
입니다. 이렇게 추가하면 시뮬레이터·브릿지 재시작 후 `/lidar/Lidar3`
(또는 `ros2 launch`로 실행했다면 `/fsds/lidar/Lidar3`) 토픽이 자동으로
생깁니다.

### 4.3 새 카메라 추가 예시

`Cameras` 항목 안에 원하는 이름으로 블록을 추가합니다. 예를 들어 `cam3`:

```json
"cam3": {
  "CaptureSettings": [
    {
      "ImageType": 0,
      "Width": 640,
      "Height": 480,
      "FOV_Degrees": 120
    }
  ],
  "X": 1.0,
  "Y": 0.0,
  "Z": 0.6,
  "Pitch": 0.0,
  "Roll": 0.0,
  "Yaw": 180.0
}
```
**주의**: 카메라는 `ros2 run fsds_ros2_bridge fsds_ros2_bridge`로는 절대
나타나지 않습니다 — 반드시 **2번 섹션의 B번 방법**(`ros2 launch
fsds_ros2_bridge fsds_ros2_bridge.launch.py`)으로 실행해야 카메라별로
별도 노드(`/fsds/cam3/image_color`, `/fsds/cam3/camera_info`)가 뜹니다.

### 4.4 추가 후 확인 방법

코드 수정이나 재빌드는 전혀 필요 없습니다 — `settings.json`만 고치고
시뮬레이터·브릿지를 재시작하면 됩니다. 실제로 데이터가 나오는지까지
확인하세요 (토픽 이름만 있고 데이터가 안 나올 수도 있으므로):
```bash
ros2 topic list | grep -i lidar3    # 또는 cam3
ros2 topic hz /lidar/Lidar3         # 설정한 RotationsPerSecond와 비슷하게 나와야 함
ros2 topic hz /fsds/cam3/image_color   # cam3 예시 (B 방법으로 실행했을 때)
```
새 라이다(`Lidar3`)와 새 카메라(`cam3`)를 실제로 추가해서 각각 설정한
속도(10Hz)와 해상도(640×480)로 데이터가 정상적으로 나오는 것까지 직접
검증했습니다.

## 알아두면 좋은 함정들 (자세한 내용은 `SETUP_DEBUG_LOG.md` 참고)

- **NVIDIA 드라이버를 새로 설치할 때**: 이 UE4.27 기반 시뮬레이터에서
  안정적으로 확인된 것은 `nvidia-driver-470-server`뿐입니다 — 더 최신
  드라이버(595 이상)는 `VK_ERROR_DEVICE_LOST`로 크래시납니다.
- **커스텀 맵 콘 개수 상한**: `CustomMap`은 (화면상이 아니라
  `/testing_only/track`으로 확인했을 때) 전체 콘 개수가 약 **128개**를
  넘으면 조용히 잘라버리며, 렌더링/월드 범위 자체도 실제 서킷 규모가
  아니라 소형 FS 트랙 기준으로 만들어져 있습니다. 새 커스텀 트랙을 만들
  때는 `maps/YonginCircuit/README.md` 참고.
- **`-settings=`/`-CustomMapPath=`와 공백은 같이 못 씀** — 커맨드라인
  인자로 넘기는 경로는 항상 공백이 없어야 합니다 (`settings.json`은
  `$HOME` 심볼릭 링크가 자동으로 처리해주므로 해당 없음; `-CustomMapPath`
  에만 해당).
