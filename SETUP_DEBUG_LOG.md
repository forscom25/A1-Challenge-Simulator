# FSDS 시뮬레이터 + ROS2 Humble 브릿지 — 설치·디버깅 기록

날짜: 2026-09-02
머신: Ubuntu 22.04.5, RTX A5000, 초기 설치 상태 (git/gcc/ROS2/NVIDIA 드라이버 모두 없음)

## 목표

FS-Driverless의 Formula Student Driverless Simulator를 `fsds_ros2_bridge`
(ament_cmake 패키지)와 함께, 실제 ROS2 Humble 설치 위에서
`/home/ailab/git/A1 Challenge Simulator/`에서 실행. 이후 실제 ROS2 Humble
자율주행 스택과 연동하는 것이 최종 목표.

## 최종 폴더 구조

- `Formula-Student-Driverless-Simulator/` — `FS-Driverless/Formula-Student-Driverless-Simulator`
  (`master`) 클론. 서브모듈 `AirSim/external/rpclib`, `ros2/src/fs_msgs`
  (ros2 브랜치) 초기화됨. 여기 `ros2/`가 빌드된 colcon 워크스페이스
  (`fs_msgs`, `fsds_ros2_bridge`)를 담고 있음.
- `simulator/` — 미리 빌드된 Linux 바이너리 릴리즈 `v2.2.0`
  (`fsds-v2.2.0-linux.zip`) 압축 해제한 것. Unreal Engine을 소스에서
  빌드하는 대신 사용.

## 설치 과정

1. 기본 도구: `git`, `git-lfs`, `build-essential`, `cmake`,
   `libcurl4-openssl-dev`, `python3-pip` 등
2. NVIDIA 드라이버는 `ubuntu-drivers autoinstall`로 설치 (아래 버그 1 참고 —
   나중에 다시 손봐야 했음)
3. ROS2 Humble desktop을 공식 apt 저장소로 설치, `ros-dev-tools`,
   `python3-colcon-common-extensions`, `python3-rosdep` 포함
4. `--recurse-submodules`로 저장소 클론
5. `v2.2.0` Linux 시뮬레이터 바이너리 다운로드/압축해제
6. `AirSim/setup.sh` 사전 요구사항 수동 설치 (Ubuntu 22.04 분기는
   `clang-12`/`libc++-12-dev`/`libc++abi-12-dev` 필요; Eigen 3.3.7을
   `AirLib/deps/eigen3`에 다운로드)
7. `ros2/`에서 `rosdep install --from-paths src --ignore-src -r -y` —
   `ros-humble-desktop` + `libcurl4-openssl-dev`로 이미 다 충족되어 있었음
8. `ros2/`에서 `colcon build` — 시스템 기본 `g++`(11.4)로 성공;
   clang-12는 `setup.sh`를 만족시키기 위해서만 필요했고 실제 브릿지 빌드에는
   필요 없었음

여기까지는 순조로웠음. 시뮬레이터를 실제로 실행하면서 진짜 버그 두 개를 만남.

## 버그 1: NVIDIA 드라이버 595가 이 UE4.27 기반 시뮬레이터에게 너무 최신임

**증상:** 메인 메뉴 레벨에 도달한 지 몇 초 만에 시뮬레이터 프로세스가 크래시:
```
LogVulkanRHI: Error: VulkanRHI::vkDeviceWaitIdle(Device) failed, VkResult=-4
... VK_ERROR_DEVICE_LOST
Signal=11 (segfault)
```

**조사:**
- `ubuntu-drivers autoinstall`이 당시 기준 매우 최신/최전선 드라이버인
  `595.84`를 설치했었음
- 이 시뮬레이터는 Unreal Engine 4.27(~2021년) 기반이고, 이 엔진의 Vulkan
  RHI는 그 드라이버보다 몇 년이나 앞서 있음
- 저렴한 우회책으로 `-opengl4`를 시도: 좀 더 진행되긴 했음(메뉴에서
  크래시하는 대신 실제 트랙 레벨까지 로드됨)했지만 `__cxa_throw` 안에서
  `SIGABRT`로 중단됨 — 패키지된 바이너리의 셰이더가 Vulkan용으로만
  쿠킹되어 있어서, 에디터가 아닌 패키지 빌드에서 OpenGL을 강제하는 건
  실제로 가능한 선택지가 아님
- 웹/GitHub 검색으로 Linux에서의 `VK_ERROR_DEVICE_LOST`가 Unreal Engine +
  매우 최신 NVIDIA 드라이버 조합에서 흔히 발생하는 문제군임을 확인함
  (UE4 AirSim, UE5.7/5.8, CARLA 등에서 다수 보고됨), 보통 더 오래된 드라이버
  브랜치로 바꾸면 해결됨

**해결:** 595 드라이버를 제거하고 `nvidia-driver-470-server`를 설치
(UE4.27과 비슷한 시기의 롱텀 브랜치이면서 Ampere 기반 RTX A5000도 완전히
지원함), 재부팅:
```bash
sudo apt purge -y '^nvidia-.*'
sudo apt autoremove -y
sudo apt install -y nvidia-driver-470-server
sudo reboot
```
이후로는 기본 Vulkan RHI에서 시뮬레이터가 안정적으로 실행됨.

## 버그 2: 프로젝트 경로에 공백이 있어서 settings.json 파싱 실패

**증상:** 버그 1을 고친 뒤 시뮬레이터가 메인 메뉴까지는 잘 도달했지만,
실제 트랙 레벨로 전환하는 순간 바로 크래시:
```
libc++abi: terminating with uncaught exception of type std::invalid_argument:
Error while parsing settings.json: parse error - unexpected '"'
```
`settings.json` 파일 자체는 유효한 JSON인데도 이런 에러가 남
(`python3 -c "import json; json.load(...)"`로 검증 완료).

**근본 원인:** 프로젝트 디렉토리 이름이 `A1 Challenge Simulator`로 공백을
포함하고 있음. 시뮬레이터를 명시적으로
`-settings "/home/ailab/git/A1 Challenge Simulator/.../settings.json"`
인자와 함께 실행했었음. Unreal Engine의 Linux 커맨드라인 처리는 argv를 하나의
문자열로 다시 합친 뒤 내부적으로 재토큰화하는데, 이스케이프되지 않은 공백이
있는 경로는 이 단계에서 여러 토큰으로 쪼개짐 — 그 결과 `-settings` 플래그가
실제 파일이 아니라 깨지고 잘린 값을 가리키게 되고, 경로로도 인라인 JSON
문자열로도 파싱에 실패함.

**해결 (최초, 2026-09-02):** `-settings`를 아예 넘기지 않기로 함. 대신
`settings.json`을 `FSDS.sh`와 같은 디렉토리(`simulator/settings.json`)에
복사해두고, 그 디렉토리에서 바이너리를 실행 — 커맨드라인 인자 대신
시뮬레이터의 현재 작업 디렉토리(CWD) 탐색 기능(공식 문서에 나온
settings.json 탐색 경로 중 하나)에 의존함. 이렇게 하면 argv 공백 분리
버그를 완전히 피할 수 있음 — 이 시점에는 `$HOME` 심볼릭 링크나 위치 이동은
쓰지 않았음.

**대체됨 (2026-09-03):** 공식 문서가 실제로 권장하는 `$HOME` 심볼릭 링크
방식으로 전환함 (아래 "2026-09-03 업데이트" 참고) — 위의 CWD 복사 방식도
여전히 작동하고 근거도 유효하지만, 더 이상 실제로 쓰이는 방식은 아님.

## 버그 3: 기본값(풀스크린)으로 실행하면 VK_ERROR_INITIALIZATION_FAILED로 크래시

**증상:** `./FSDS.sh`를 인자 없이 실행하면(기본값은 풀스크린) 몇 초 만에 크래시:
```
LogCore: Fatal error: [File:.../VulkanSwapChain.cpp] [Line: 538]
Result failed, VkResult=-3 ... with error VK_ERROR_INITIALIZATION_FAILED
Signal=11 (segfault)
```
2026-09-03에 재부팅 후에도 계속 재현됨 — 일시적 현상이 아님. 드라이버는
여전히 정상 동작하는 470.256.02 브랜치 그대로(버그 1 수정 유지), 디스플레이/X
세션도 정상(`xrandr`/`nvidia-smi` 모두 정상, GPU가 실제로 데스크탑을
렌더링하고 있음).

**해결 (최초, 2026-09-03):** 항상 `-windowed`를 (`-ResX`/`-ResY`와 함께)
넘김:
```bash
./FSDS.sh -windowed -ResX=1280 -ResY=720
```
오늘 그리고 최초 설치 세션 둘 다에서 이 방식이 안정적이었음 — 인자 없는
(풀스크린) 경로만 크래시함. 근본 원인은 완전히 규명하지 못함 (이 단일
모니터 Vulkan 환경에서 풀스크린-exclusive/컴포지터 상호작용 문제일
가능성이 높음); 창모드로 실행하면 아예 문제를 피해가므로 추가 조사는
하지 않음.

**대체됨 (같은 날, 2026-09-03):** `simulator/FSDS.sh` 자체를 수정해서 인자
없는 `./FSDS.sh`도 항상 창모드로 실행되도록 함 (아래 "2026-09-03 업데이트"
참고). 위 플래그들을 명시적으로 넘겨도 여전히 동일하게 작동함 (스크립트
자체 기본값과 충돌하지 않고 중복될 뿐).

## 2026-09-03 업데이트: `$HOME` 심볼릭 링크 + `FSDS.sh` 창모드 기본값

두 가지 워크플로 변경 — 둘 다 FSDS 공식 문서가 가정하는 동작에 더 가깝게
맞추고 평소 실행을 더 간단하게 하기 위함:

**1. `$HOME` 심볼릭 링크** (버그 2의 최초 CWD 복사 방식을 대체함): FSDS
공식 문서는 저장소가 `~/Formula-Student-Driverless-Simulator`에 클론되어
있다고 가정하고 거기서 `settings.json`을 먼저 찾음. 이 프로젝트는 대신
`/home/ailab/git/A1 Challenge Simulator/Formula-Student-Driverless-Simulator`
(공백이 포함된 경로 — 이게 바로 버그 2가 발생한 이유)에 있으므로, 두 번째
복사본을 만드는 대신 심볼릭 링크를 생성함:
```bash
ln -s "/home/ailab/git/A1 Challenge Simulator/Formula-Student-Driverless-Simulator" \
      ~/Formula-Student-Driverless-Simulator
```
이것만으로도 정상 작동하는 것을 확인함 (`simulator/settings.json`을 완전히
삭제하고, ROS2 브릿지를 통해 센서 설정이 심볼릭 링크만으로도 정상 로드되는
것을 확인) — 그래서 `simulator/settings.json`은 더 이상 필요 없고 유지하지도
않음; `Formula-Student-Driverless-Simulator/settings.json`(git으로 추적되는
파일)이 이제 유일한 원본이며 `$HOME`을 통해 자동으로 찾아짐. 이는 또한
`ros2/src/fsds_ros2_bridge/launch/fsds_ros2_bridge.launch.py` 런치 파일
(설정된 카메라마다 노드를 하나씩 띄우기 위해
`~/Formula-Student-Driverless-Simulator/settings.json`을 하드코딩해서
읽음)도 수동 파일 복사 없이 작동하게 됨을 의미함.

참고: 이 심볼릭 링크는 *settings.json* 탐색 문제만 해결함.
`-CustomMapPath`는 커맨드라인으로 넘길 때 여전히 공백 없는 경로가 필요함
(버그 2의 argv 분리 문제는 settings 파일 탐색과는 무관한 별개의 문제) —
커스텀 맵 CSV는 이전처럼 `/home/ailab/track_*.csv` 같은 곳에 계속 복사해서
써야 함.

**2. `FSDS.sh` 창모드 기본값**: `simulator/FSDS.sh`(다운로드한 릴리즈 zip
안의 런처 스크립트, git으로 추적되지 않음)를 수정해서 `"$@"` 앞에 항상
`-windowed -ResX=1280 -ResY=720`을 넣도록 함:
```sh
"$UE4_PROJECT_ROOT/FSOnline/Binaries/Linux/Blocks" FSOnline -windowed -ResX=1280 -ResY=720 "$@"
```
이제 `./FSDS.sh`만 실행해도 정상 작동함 (버그 3은 플래그를 기억할 필요가
없어져서 아예 문제가 되지 않음), `./FSDS.sh "-CustomMapPath=..."` 같은
추가 인자도 그대로 전달됨. **주의**: 다운로드 릴리즈 zip 안에 들어있는
파일을 수정한 것이므로, 나중에 새 `fsds-vX.Y.Z-linux.zip`을 다시
압축해제하면 이 수정이 덮어써짐 — 그럴 경우 위의 한 줄 수정을 다시
적용할 것.

## 검증된 정상 작동 상태

- 시뮬레이터: `simulator/FSDS.sh "-CustomMapPath=/home/ailab/track_yongin.csv"`
  (창모드는 이제 자동), 메인 메뉴를 수동으로 클릭해서 트랙 로드
  (또는 `-CustomMapPath`를 쓰면 메뉴를 아예 건너뜀)
- 브릿지: `ros2 run fsds_ros2_bridge fsds_ros2_bridge`가 즉시 연결되고
  `/gps`, `/imu`(~249Hz), `/gss`, `/lidar/Lidar1`, `/lidar/Lidar2`,
  `/testing_only/odom`(~249Hz), `/testing_only/track`, `/signal/go` 등을
  발행함 (`ros2 launch fsds_ros2_bridge fsds_ros2_bridge.launch.py`는
  추가로 설정된 카메라마다 카메라 노드를 하나씩 `/fsds/...` 아래에 띄움)
- 참고: `ros2 run`은 네임스페이스 없는 토픽(예: `/gps`)을 주고,
  `ros2 launch`는 (ROS1 기준으로 작성된) 공식 문서가 설명하는
  `/fsds/...` 네임스페이스를 줌

## 참고: `fsds_ros2_bridge`는 자체적으로 settings.json을 디스크에서 읽지 않음

`fsds_ros2_bridge`의 소스
(`ros2/src/fsds_ros2_bridge/src/airsim_ros_wrapper.cpp:17`)를 확인해서,
브릿지가 센서/토픽을 결정할 때 자체 로컬 settings.json을 읽는 것이
**아님**을 확인함. 대신 실행 중인 시뮬레이터에서 RPC 연결로 실시간으로
settings 텍스트를 가져옴 (`airsim_client_.getSettingsString()`); 그 파일
안에 로컬 파일 에러 메시지를 포함한 `readTextFromFile()` 헬퍼 함수가
있긴 하지만 죽은 코드로, 실제로 호출되지 않음. 그래서 브릿지는 항상
연결된 시뮬레이터 인스턴스가 실제로 로드한 것을 그대로 반영함 — 브릿지
자체를 위해 동기화해야 할 두 번째 settings.json은 없음 (카메라 *런치
파일*은 예외 — 위 `$HOME` 심볼릭 링크 항목 참고, 이건 어떤 카메라 노드를
띄울지 알기 위해 settings.json을 직접 읽음).
