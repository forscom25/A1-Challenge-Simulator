# 컨트롤러 스켈레톤 (Pure Pursuit, Python) — 진행상황

2026-09-07 기준 상태. 목표: 실제 스케일 용인 서킷(`YonginCircuitMap`)에서
ROS2 제어 루프가 실제로 동작하는지 — 기본적인 차량 기구학 + 실제 스케일
맵이 맞는지 — 검증하는 최소 컨트롤러 스켈레톤. Pure Pursuit는 스켈레톤
검증용이고, 최종 컨트롤러는 이후 MPC(C++)로 교체 예정
([PROGRESS_UE4_BUILD.md](PROGRESS_UE4_BUILD.md) §13 참고).

## 완료

1. **`fsds_controller` ROS2 패키지 스캐폴딩** (`ros2/src/fsds_controller/`,
   `ament_python`): `package.xml`, `setup.py`, `setup.cfg`,
   `resource/fsds_controller`, `fsds_controller/pure_pursuit.py`
   (순수 알고리즘), `fsds_controller/controller_node.py`,
   `fsds_controller/path_publisher_node.py`, `launch/controller.launch.py`.
2. **Pure Pursuit 알고리즘**: 목표점 탐색(lookahead) + 곡률 기반 조향각
   계산 + 목표 속도(3 m/s) 비례 스로틀. `fs_msgs/ControlCommand` 퍼블리시.
3. **아키텍처**: `path_publisher_node`(참조 경로 퍼블리시)와
   `controller_node`(제어 로직) 완전히 분리, 토픽으로만 통신 — 나중에
   MPC(C++)로 교체할 때 `controller_node`만 갈아끼우면 되도록 의도적으로
   설계.
4. **최종 검증 결과**: 실시간 RPC ground truth(`/testing_only/track`)
   기준으로 차량이 트랙 중심선에서 **0.17m 오차**로 안정적으로 추종 —
   트랙 폭(3.5m)에 비해 매우 작은 오차. 사용자가 전체 랩 주행 확인 +
   화면 녹화 진행 중.

## 겪은 문제와 해결 (시간순)

여러 층위의 버그가 겹쳐 있어서 하나씩 벗겨내야 했음. 매 단계 "컨트롤러가
스스로의 참조 경로를 얼마나 잘 따라가는가"만 보면 속기 쉬웠고, 실제
문제를 잡은 건 항상 **시뮬레이터가 실시간으로 제공하는 ground truth
토픽과 직접 대조**했을 때였음.

### 1. 좌표축 방향 불일치 (Y축 미러링)

센터라인 CSV(`yongin_centerline.csv`)의 첫 구간 방향(bearing)이 차량의
실제 스폰 방향(odom 쿼터니언에서 역산, 약 -40°)과 맞지 않음 — Y를
뒤집으면(-y) 거의 정확히 일치(-38.02° vs -39.97°, 2° 이내). 원인:
콘 스플라인은 Unreal 월드(왼손 좌표계)에 직접 배치됐는데, ROS ENU는
오른손 좌표계라서 축이 하나 뒤집힘. `path_publisher_node`에서 Y를
`-y`로 발행하도록 수정.

### 2. 조향 부호 반전

`pure_pursuit.py`는 표준 로보틱스 관례(양수 = 좌회전)로 조향각을
계산하는데, `CarPawnSimApi.cpp`가 `ControlCommand.steering`을 그대로
UE4의 `movement_->SetSteeringInput()`에 넘기고, **UE4의 관례는
양수 = 우회전**으로 정반대. 증상: 초반에 반대로 꺾다가 목표점이 우연히
차량 정면(혹은 정후면)과 일직선이 되는 "가짜 평형 상태"에 빠져서 그
이후로는 똑바로 직진해버림 — steering 값이 계속 거의 0에 수렴하는
패턴으로 나타나서 알아챔. `controller_node.py`에서 최종 `cmd.steering`
계산 시 부호 반전.

### 3. 목표점 탐색이 경로 이탈 시 복구 못 함

`find_target_point`가 `last_target_index`부터 **배열 순서로만** 앞으로
스캔하는 구조라서, 배열 순서와 실제 공간적 위치가 벌어지면(코너를
크게 자르거나 살짝 밀려나면) 엉뚱한 목표를 계속 고르게 됨 — 특히
헤어핀류에서 취약. 매 콜백마다 **가장 가까운 점을 먼저 찾고** 거기서부터
lookahead 스캔을 시작하도록 재작성(`find_nearest_index` 추가)해서
경로 이탈에서도 스스로 복구되도록 함. Lookahead도 6.0m → 2.5m로 축소
(저속 주행에는 과도하게 길었음).

### 4. 참조 경로 해상도가 너무 낮음

원본 센터라인이 10m 간격 256개 점뿐이라, 두 점 사이는 실제로는 직선
보간(chord)이라 진짜 곡선을 못 따라감 — 특히 코너에서 눈에 띄게 잘림.
`path_publisher_node`에 선형 보간 기반 `densify()`를 추가해서 1m
간격으로 재샘플링.

### 5. 근본 원인 — 참조 경로 자체가 stale한 데이터에서 나옴

위 네 가지를 다 고치고도 시각적으로 트랙 밖을 벗어나는 문제가
계속됐음. 실제 RPC ground truth(`/testing_only/track`)와 직접
대조해보니, 그동안 참조 경로의 소스로 썼던 모든 데이터
(`yongin_centerline.csv`, 그리고 Python으로 직접 덤프한
`blue_cone_locations`/`yellow_cone_locations` 배열 둘 다)가 **실제
라이브 RPC 데이터와 평균 5m, 최대 9m 어긋나 있었음**. 즉 지금까지
고친 좌표계/조향/탐색 로직은 전부 맞았는데, 애초에 목표로 삼던
"트랙"이 진짜 트랙이 아니었던 것.

추정 원인: 레벨 제작 과정에서 Construction Script가 자동 재실행 안
되는 문제를 뷰포트 수동 이동으로 우회했는데(`UE4_소스빌드_가이드.md`
참고), 그때 시각적 콘 메시는 갱신됐지만 `blue_cone_locations`/
`yellow_cone_locations` 배열 프로퍼티 자체는 갱신이 안 된 것으로
보임. 이 배열이 stale하다는 사실은 [PROGRESS_UE4_BUILD.md](PROGRESS_UE4_BUILD.md)의
"남은 작업"에 경고로 남겨둠 — 근본 원인 자체는 미조사.

**최종 해결**: 참조 경로를 어떤 파일이나 캐시된 배열에서도 만들지 않고,
`path_publisher_node`가 **`/testing_only/track`을 직접 subscribe**해서
그 자리에서 실시간으로 구성하도록 변경. 이 토픽은 차량의 `odom`과
**같은 파이프라인**(같은 `ros_bridge` 노드)에서 나오기 때문에 서로
어긋날 수가 없음 — 구조적으로 정합성이 보장됨.

### 부록: "약 128개 콘 상한"은 애초에 없었음

이 조사 과정에서 [PROGRESS_UE4_BUILD.md](PROGRESS_UE4_BUILD.md)에
"미해결"로 남겨뒀던 그 이슈도 같이 풀림 — 실제 캡이 아니라
`TRANSIENT_LOCAL` QoS를 안 맞추고 조회했던 문제 + `ros2 topic echo`의
CLI 출력 자체가 긴 메시지를 잘라 보여주는 문제였음. 자세한 내용은
그 문서의 "해결됨" 절 참고.

## 기술 메모

- **QoS 주의**: `/testing_only/track`은 `TRANSIENT_LOCAL` +
  `RELIABLE`로 subscribe해야 함 (기본 volatile QoS로는 아예 못 받음).
- **네임스페이스**: `fsds_ros2_bridge`를 카메라 포함 풀 런치
  (`fsds_ros2_bridge.launch.py`)로 띄우면 모든 토픽이 `/fsds/` 아래로
  들어감 — `fsds_controller`의 두 노드는 remap으로 대응:
  ```bash
  ros2 run fsds_controller path_publisher_node --ros-args \
    -r testing_only/track:=/fsds/testing_only/track
  ros2 run fsds_controller controller_node --ros-args \
    -r testing_only/odom:=/fsds/testing_only/odom \
    -r control_command:=/fsds/control_command
  ```
- **디버그 로깅**: `controller_node.py`에 15콜백마다 한 번씩 위치/목표/
  조향각을 찍는 로그가 남아있음 (`self.get_logger().info(...)`) —
  문제 재발 시 바로 켜서 볼 수 있음, 필요 없으면 지워도 됨.

## MGeo → FSDS 변환 파이프라인 독립 검증 (MATLAB)

컨트롤러가 왜 그렇게 오래 안 맞았는지 완전히 이해하기 위해, 원본 MGeo
데이터부터 실시간 시뮬레이터 ground truth까지 데이터가 거치는 각 단계를
Python 파이프라인과 완전히 독립적인 MATLAB 스크립트로 재검증함
(`Formula-Student-Driverless-Simulator/maps/YonginCircuit/validate_yongin_centerline.m`
— 원격 PC에서도 실행 가능하도록 필요한 데이터 전부를 같은 폴더에 패키징:
`mgeo_data/`, `yongin_centerline.csv`, `rpc_track_cones.csv`).

데이터 3종류를 구분해서 이해하는 게 핵심이었음:

| 데이터 | 무엇인가 |
|---|---|
| (a) `yongin_centerline.csv` | 레벨 제작 **입력** (원본 MGeo → 변환된 256점 센터라인) |
| (b) `cone_positions.csv` | 스플라인 Blueprint의 `blue/yellow_cone_locations` 배열을 Python으로 덤프한 것 — **stale함이 밝혀짐** (§겪은 문제 5번 참고) |
| (c) `rpc_track_cones.csv` | 시뮬레이터가 **지금 실제로** 쓰는 라이브 ground truth |

**1단계 (점 개수/닫힘/길이)**: 원본 MGeo(5128점, 완전 폐곡선, 길이
2563.46m) vs (a)(256점, 10m 간격, 길이 2560.83m) — 차이 0.1% 이내,
스케일/단위 문제 없음.

**2단계 (방향/오버레이/코너 시퀀스/국소 곡률)**: 시작 방향 일치, 오버레이
거의 완전히 겹침, 코너 좌우 시퀀스 정상, 최대 곡률 지점 위치 차이
2.7m·크기 차이 0.9% — 전부 10m 리샘플링에서 예상되는 chord 근사 오차
수준. **(a)는 원본 MGeo를 충실히 반영함.**

**3단계 ((a) vs (c) 직접 비교, 사용자 제안으로 추가)**: 여기서 예상 밖의
결과가 나옴 — **(a)와 (c)의 점 대 점 최근접 거리가 평균 6.66m, 최대
9.51m**로, (b)와 (c)의 차이(평균 5.10m, 최대 9.33m)와 비슷한 크기.
즉 드리프트가 (a)→(b) 단계(stale 배열)에서만 생긴 게 아니라, **(a)
자체가 처음부터 (c)와 이 정도 차이가 있었음.** 전체 256점의 88.7%가
트랙 폭(3.5m)보다 큰 편차를 보였고, 이 편차는 특정 헤어핀 하나가 아니라
**트랙 전체에 걸쳐 광범위하게 나타남** — 가장 유력한 설명은 Unreal
스플라인이 256개의 성긴 제어점을 매끄러운 곡선으로 보간해서 콘을
배치하기 때문에(직선 chord가 아니라 곡선 기반), 실제 도로가 거의
끊임없이 굽어 있는 이 트랙 특성상 이 차이가 국지적이지 않고 전반적으로
나타난 것으로 추정.

**추가 확인 (코너 급함 정도)**: 원본 MGeo의 가장 급한 코너(반경 약
13.6m) 대비 라이브 시뮬레이터의 가장 급한 코너는 반경 약 16.2m —
**약 16~19% 더 완만함**. 전체 코너별 정밀 대조는 두 데이터셋의 진행
방향이 서로 반대(원본은 시계방향, 라이브 콘 리스트는 반시계방향으로
등록됨 — signed area는 거의 일치, 108076 vs 108068, 방향만 반대)라서
위상을 맞추려다 정렬 오차가 섞여 신뢰할 만한 코너별 수치는 못 얻음 —
전체 최대 곡률끼리의 비교만 신뢰할 수 있는 결과로 채택.

**결론 및 권고**:
- 컨트롤러의 참조 경로를 (a)/(b)가 아니라 (c)(라이브 RPC)로 이미
  바꿔서 우회했으므로 이 발견이 지금 당장 뭔가를 깨뜨리지는 않음.
- **향후 레이싱 라인/MPC 참조 궤적 설계는 반드시 (c) 기준으로 할 것**
  — (a)는 레벨을 처음 만들 때 입력으로만 유효했고, 정밀도가 필요한
  후속 작업의 기준으로 쓰기엔 트랙 폭보다 큰 오차가 있음.
- **MPC 검증 시 유의할 점**: 이 시뮬레이터의 트랙은 실제 용인 서킷보다
  코너가 약 15~20% 완만함. "이 컨트롤러가 실제 서킷 수준의 요구를
  처리한다"는 주장을 할 때는 이 한계를 명시할 것. 순수 제어 루프
  검증(수렴하는지, 안정적으로 추종하는지) 목적에는 무관함.
- 더 높은 정밀도가 필요해지면(예: 실제 대회 전 코너 반경 여유를
  정밀하게 검토해야 할 때) 데이터 후처리가 아니라 **레벨을 더 촘촘한
  스플라인 제어점으로 다시 제작**하는 게 근본적 해결책 — 지금은 급하지
  않아 보류.

## 남은 작업

- [ ] 사용자가 진행 중인 전체 랩 안정성 확인 + 화면 녹화
- [ ] 스로틀/속도 제어가 매우 단순함 (고정 목표 속도 3 m/s, 코너 감속
      없음, 브레이크 로직 없음) — 스켈레톤 목적상 지금은 충분하지만
      MPC로 넘어갈 때 개선 필요
- [ ] `blue_cone_locations`/`yellow_cone_locations` 배열이 stale한
      근본 원인 조사 (지금은 컨트롤러가 이 배열을 안 쓰도록 우회했을
      뿐, 레벨 자체의 데이터 정합성 문제는 미해결 —
      [PROGRESS_UE4_BUILD.md](PROGRESS_UE4_BUILD.md) 참고)
- [x] MGeo→FSDS 변환 파이프라인 MATLAB 독립 검증 — 완료 (위 절 참고),
      `validate_yongin_centerline.m` 이 검증 근거로 저장소에 남아있음
- [ ] 향후 레이싱 라인/MPC 참조 궤적을 (c) 라이브 ground truth 기준으로
      구성 — (a) `yongin_centerline.csv`는 더 이상 정밀도 기준으로
      쓰지 않기로 함
- [ ] MPC 결과 보고/발표 시 "시뮬레이터 트랙이 실제 용인 서킷보다 코너가
      약 15~20% 완만함" 한계를 명시하기로 함 — 반영 필요
- [ ] (급하지 않음, 보류) 레벨을 더 촘촘한 스플라인 제어점으로 재제작해서
      (a)/(c) 간 편차와 코너 완만화 자체를 줄이는 근본 개선
- [ ] 발표 후: MPC(C++) 컨트롤러 설계 시작 — `controller_node`만
      교체, 인터페이스(토픽/메시지)는 유지
