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

## 남은 작업

- [ ] 사용자가 진행 중인 전체 랩 안정성 확인 + 화면 녹화
- [ ] 스로틀/속도 제어가 매우 단순함 (고정 목표 속도 3 m/s, 코너 감속
      없음, 브레이크 로직 없음) — 스켈레톤 목적상 지금은 충분하지만
      MPC로 넘어갈 때 개선 필요
- [ ] `blue_cone_locations`/`yellow_cone_locations` 배열이 stale한
      근본 원인 조사 (지금은 컨트롤러가 이 배열을 안 쓰도록 우회했을
      뿐, 레벨 자체의 데이터 정합성 문제는 미해결 —
      [PROGRESS_UE4_BUILD.md](PROGRESS_UE4_BUILD.md) 참고)
- [ ] 발표 후: MPC(C++) 컨트롤러 설계 시작 — `controller_node`만
      교체, 인터페이스(토픽/메시지)는 유지
