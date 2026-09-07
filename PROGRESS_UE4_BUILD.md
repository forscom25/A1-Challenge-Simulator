# 실차 스케일 용인 서킷 — Unreal Engine 소스 빌드 진행상황

2026-09-07 기준 상태. 목표: FSDS를 Unreal Engine 소스에서 직접 빌드해서
실제 스케일(축소 없는) 용인 서킷을 (`CustomMap` 축소판 대신) 실제 레벨로
쓸 수 있게 만드는 것 — 패키지된 바이너리 `CustomMap`의 약 128개 콘 상한과
좁은 월드 범위 한계를 우회.

**요약**: 레벨 제작 자체는 끝났고 Play-in-Editor로 전체 랩 주행까지
확인함. `fsds_ros2_bridge` 검증 과정에서 처음엔 `CustomMap`과 동일한 약
128개 콘 상한이 새 레벨에서도 재현되는 것처럼 보였으나, 이후 컨트롤러
작업 중에 **실제 캡이 아니라 QoS/CLI 조회 방식의 문제였음이 밝혀짐**
(아래 "해결됨" 절 참고) — 실제로는 1282개 콘 전부 정상 반환됨. 이후
컨트롤러 스켈레톤 작업으로 전환함 (아래 13, 14 항목, 자세한 진행상황은
[PROGRESS_CONTROLLER.md](PROGRESS_CONTROLLER.md)).

## 완료

1. **Unreal Engine 4.27을 `~/UnrealEngine`에 소스 빌드** (다른 대용량
   프로젝트 디렉토리와 달리 홈 디렉토리 바로 밑에 그대로 둠 — 옮기면 UBT
   빌드 캐시의 절대경로가 깨져서 불필요한 재컴파일이 생길 수 있음, 자세한
   이유는 [UE4_소스빌드_가이드.md](UE4_소스빌드_가이드.md) 참고) (Editor,
   ShaderCompileWorker, UnrealLightmass). 타겟을 하나씩 빌드해야 했음
   (`Build.sh <Target> Linux Development`) — 기본 7개 타겟을 인자 없는
   `make`로 한번에 병렬 빌드하면 Epic의 Linux 진행률 표시 래퍼 스크립트에서
   레이스 컨디션이 발생해서 에러 메시지도 없이 7개 중 4개 타겟이 조용히
   누락됨.
2. **FSDS 자체 프로젝트 컴파일 완료** (`BlocksEditor` = `Blocks` 게임 모듈 +
   `AirSim` 플러그인). 진행 중 진짜 버그 두 개를 만나서 해결:
   - `AirSim/build.sh`를 이 프로젝트에서 한 번도 실행한 적이 없었음
     (ROS2 브릿지용 별도 빌드 경로만 썼었음), 그래서 `AirLib`이
     `UE4Project/Plugins/AirSim/Source/AirLib`에 스테이징된 적이 없었음 —
     실행해서 해결.
   - **ABI 불일치**: `librpc.a`(시스템 `clang++-12`로 빌드됨)가
     `pthread_cond_clockwait` 심볼을 참조하는데, 이 심볼은 UE4가 번들로
     갖고 있는 Linux 툴체인의 sysroot
     (`Engine/Extras/.../v19_clang-11.0.1-centos7`, 오래된 CentOS7 기반
     glibc)에는 없음. `AirLib`/`rpclib`을 UE4 자체 번들 `clang`/`clang++`로,
     **그리고** 그것과 일치하는 `--sysroot=`를 지정해서 다시 빌드함으로써
     해결 — 오래된 sysroot의 헤더를 보고 컴파일하면 libc++의 기능 감지
     로직이 그 코드 경로를 올바르게 피해가서, UE4Editor 자체 링커가 링크하는
     대상과 맞아떨어짐.
3. **`UE4Editor`가 `FSOnline.uproject`를 로드해서 정상 실행됨**,
   `TrainingMap` 열림. Python 에디터 스크립팅
   (`unreal.EditorLevelLibrary` 등)이 동작하도록
   `EditorScriptingUtilities` 플러그인 활성화 (`FSOnline.uproject`에 추가) —
   적용하려면 재시작 필요했음.
4. **Python으로 `TrainingMap`의 바닥 조사**: `floor`라는 이름의 평범한
   `StaticMeshActor`(Landscape 아님), 기본 타일 1m×1m, 현재 200m×250m로
   스케일됨 — `Scale`만 바꾸면 손쉽게 리사이즈 가능.
5. **`spline_cones_best_200` 액터 조사**: 이건 (`docs/map-tutorial.md`에
   문서화된) 표준 맵 제작용 콘 스플라인 시스템이지, 패키지된 바이너리
   `CustomMap` 레벨 안의 속을 알 수 없는 캡 걸린 Blueprint가 *아님*.
   컴포넌트 개수로 확인 (스플라인 포인트 30개에 `StaticMeshComponent` 192개)
   — 일정한 내부 간격으로 스플라인을 재샘플링해서 자동으로 콘을 생성하는
   구조이며, `CustomMap`의 약 128개 같은 하드캡이 없음.
6. **실제 스케일 용인 서킷 센터라인 데이터로 채움** (10m 간격 256개 포인트,
   2563m 루프, 시작점이 월드 원점이 되도록 재중심화). Python의
   `SplineComponent.set_spline_points()`는 Blueprint의 Construction
   Script를 트리거하지 않음 (이건 실제 UI 기반 트랜스폼 편집에만 연결되어
   있고, 이 엔진 버전에서는 Python에 노출되어 있지 않음) — 액터를 선택한
   뒤 뷰포트 이동 기즈모(**W**, 드래그)로 살짝 움직여서 우회함. 결과:
   **콘 메시 컴포넌트 1278개**, 에디터 뷰포트에서 육안으로 확인 — 실제
   트랙 형태가 정확함 (헤어핀, 에스자 구간, 넓은 코너까지 전부 인식
   가능한 형태로 존재).
7. 이전 UI 실수 클릭으로 생긴 중복 액터 정리 완료.
8. **새 맵으로 저장 완료**: `unreal.EditorLevelLibrary.save_current_level()`로
   현재 편집 상태를 `TrainingMap.umap`에 임시로 저장 → 셸에서 그 파일을
   `YonginCircuitMap.umap`으로 복사 → `git checkout`으로 원본
   `TrainingMap.umap`을 깨끗한 상태로 복원. 이제
   `UE4Project/Content/YonginCircuitMap.umap`에 우리 작업이 안전하게
   보관되어 있고, 원본 `TrainingMap.umap`은 git에 커밋된 그대로 손상 없음.

9. **`floor` 리사이즈 및 재배치** — 실제 트랙 범위(bbox ≈ 822m × 476m,
   중심 ≈ (224, -78)m)에 맞게 `Scale`/`Location` 조정 완료.
10. **`PlayerStart`/`StartFinishLine`/`Referee` 재배치** — 몇 차례 시행착오
    끝에 완료:
    - 1차 시도: 색상별 콘 배열(blue/yellow)을 인덱스로 짝지어(`groupA[0]`
      + `groupB[0]`) 그 중점을 시작선으로 삼음 — **실패**. 두 배열이 트랙을
      따라 위상적으로(phase) 정렬되어 있지 않아서 시작선이 트랙 진행
      방향으로 밀려남 (사용자 피드백: "가운데가 아니라 앞으로 갔다").
    - 최종 해결: 인덱스가 아니라 **3D 거리 기준 최근접 쌍**을 탐색해서
      진짜 마주보는 콘 쌍을 찾음 (`ue_fix_start_nearest_pair.py`) — index
      637에서 3.50m 간격의 정확한 쌍을 발견, 이 중점을 시작선으로 사용.
    - **`PlayerStart` 자체의 Rotation은 차량 스폰 방향에 영향을 주지
      않음**을 확인 — `AirSimGameMode`가 UE4 기본 폰 스폰 시스템을 꺼버리고
      (`DefaultPawnClass = nullptr`) `SpectatorClass`를 쓰기 때문. 실제
      차량의 초기 heading은 **`settings.json`의 `Vehicles.FSCar.Yaw`**
      값(도 단위, 라디안 변환 없음)에서 옴 — 이걸 눈으로 확인한
      `StartFinishLine`의 회전각(40°)에 맞춰서 해결.
    - `StartFinishLine`의 콜리전 박스 X축 `Scale`이 트랜스폼 편집/PIE
      재시작마다 `0`으로 리셋되는 현상 발견 — 손대지 않은 원본
      `TrainingMap`에서도 동일하게 재현되는 것으로 봐서 `StartFinishLine_C`
      자체 Construction Script의 기존 버그로 추정. 주행에는 영향 없어 보여
      **미수정 상태로 둠**.
11. **키보드 수동 조작으로 전체 랩(lap) 주행 테스트 완료** — 헤어핀, 에스자
    구간을 포함한 실제 트랙 형태를 차량이 물리적으로 완주 가능함을 확인.
12. **`fsds_ros2_bridge`로 실제 연결 검증** — 여기서 중요한 발견이 나옴:
    `/testing_only/track`(RPC ground truth) 토픽이 실제로는 **128개 콘만
    (그것도 전부 한 가지 색)** 반환함. 소스 스플라인 액터의 콘 배열 자체는
    완전한 상태(639+639 = 1278개, `ue_check_array_lengths.py`로 확인)인데도.

## 해결됨: "약 128개 콘 상한"은 실제 캡이 아니라 QoS/조회 방식 문제였음

**정정 (컨트롤러 작업 중 발견)**: 아래에서 "약 128개로 상한"이라고 적었던 건
실제 RPC 데이터 캡이 아니라 **조회 방법 자체의 문제**였음이 밝혀짐. 두 가지가
겹쳐 있었음:

1. `/testing_only/track`은 `TRANSIENT_LOCAL` durability로 퍼블리시되는데,
   `ros2 topic echo`나 매칭 안 된 QoS로 그냥 subscribe하면 메시지를 아예 못
   받거나 불완전하게 받음 — QoS를 `TRANSIENT_LOCAL` + `RELIABLE`로 맞춰서
   subscribe해야 정상 수신됨.
2. `ros2 topic echo`의 CLI 출력 자체가 긴 메시지를 표시 줄 수 기준으로
   잘라버리는 것으로 보임 — `reference_path` 토픽에서도 실제로는 2000개가
   넘는 포즈가 있는데 `echo`로는 "128"이라는 동일한 숫자가 찍히는 걸 발견하고
   나서 이 가능성을 의심하게 됨.

QoS를 맞춰서 Python으로 직접 subscribe한 결과, `/testing_only/track`은
**1282개 콘 전부**(blue 639 + yellow 639 + big-orange 시작선 4개) 정상
반환함 — `spline_cones_best_200`의 소스 배열 개수(1278)와 정확히 일치.
즉 C++ 코드 계층에도, Blueprint 계층에도 캡은 애초에 없었음. 컨트롤러
작업(`PROGRESS_CONTROLLER.md` 참고)에서 이 제대로 된 RPC 데이터를 참조
경로로 직접 활용 중.

원래 기록(참고용, 아래 내용은 위 정정으로 무효화됨):

> 패키지 바이너리 `CustomMap`에서 원래 겪었던 것과 같은 숫자(약 128개)의
> 콘 상한이, **완전히 다른 방식으로 만든** 이 프로젝트의 새 레벨
> (`YonginCircuitMap`, `spline_cones_best_200` 기반 Editor 제작 레벨)에서도
> `/testing_only/track` RPC 응답에서 그대로 나타남. C++ 코드 전 계층
> (`airsim_ros_wrapper.cpp::publish_track()`, `WorldSimApi::getRefereeState()`,
> `Referee::getState()`/`AppendXCone()`, `CarRpcLibAdapators.hpp`)과 Blueprint
> 등록 로직을 전부 조사했으나 캡을 거는 코드를 찾지 못했음 — 당시엔 원인
> 불명으로 보류.

## 다음 단계로 전환

13. **컨트롤러 스켈레톤 작업 우선순위로 전환** — Pure Pursuit(Python)으로
    시작, 실제 최종 컨트롤러는 이후 MPC(C++)로 교체 예정. ROS2 패키지
    `fsds_controller` 스캐폴딩 시작 (`ros2/src/fsds_controller/`,
    `package.xml` 작성 완료, 나머지 진행 중).
14. **차량 동역학(vehicle dynamics) 파라미터 조사 시작** — `TechnionCarPawn`
    Blueprint가 쓰는 PhysX Vehicle 파라미터를 Python Editor Scripting으로
    추출 중. PhysX Vehicle 시스템 자체의 설명과 확인된 파라미터는
    [VEHICLE_DYNAMICS.md](VEHICLE_DYNAMICS.md) 참고.

## 남은 작업

- [ ] `RunUAT.sh BuildCookRun`으로 (GUI 없이 커맨드라인만으로) 새 맵을
      포함한 스탠드얼론 Linux 빌드 패키징 — 에디터를 열지 않고도 지금의
      `simulator/FSDS.sh`처럼 평소에 실행할 수 있도록. **보류** — 지금은
      Play-in-Editor로 컨트롤러 테스트에 충분해서 컨트롤러 작업이 우선.
- [x] ~~약 128개 콘 상한의 정확한 원인 재조사~~ — 해결됨, 실제 캡이
      아니라 QoS/조회 방식 문제였음 (위 "해결됨" 절 참고).
- [x] `VEHICLE_DYNAMICS.md`의 미확인 항목 채우기 — 완료.
- [x] ~~`fsds_controller` ROS2 패키지 완성~~ — Pure Pursuit 스켈레톤 동작
      확인 완료 (ground truth 대비 0.17m 오차). 자세한 내용은
      [PROGRESS_CONTROLLER.md](PROGRESS_CONTROLLER.md).
- [ ] **알아둘 것**: `spline_cones_best_200`의 `blue_cone_locations`/
      `yellow_cone_locations` Blueprint 배열 프로퍼티가 실제 라이브
      RPC(`/testing_only/track`) 데이터와 평균 5m, 최대 9m 정도 어긋나
      있음이 컨트롤러 작업 중 발견됨 — 실제 렌더링된/RPC가 보고하는
      콘 위치와 다른 stale한 값. 원인 추정: 예전에 Construction Script가
      자동 재실행 안 되는 문제를 뷰포트 수동 이동으로 우회했을 때
      (`UE4_소스빌드_가이드.md` 참고), 시각적 메시는 갱신됐지만 이
      배열 프로퍼티 자체는 갱신 안 된 것으로 보임. Python으로 이
      배열을 직접 읽어서 뭔가 할 일이 있다면 (콘 위치 재확인 등),
      **반드시 라이브 RPC 데이터를 대신 쓸 것** — 이 배열은 신뢰 불가.
      근본 원인(왜 배열이 안 갱신됐는지) 자체는 미조사 상태.
- [ ] `SETUP_DEBUG_LOG.md`, `SIMULATOR_GUIDE.md`에 새 소스빌드 절차와
      새 맵 실행 방법 반영 필요 여부 검토

## 지금까지 작성한 유용한 스크립트 (전부
`Formula-Student-Driverless-Simulator/UE4Project/editor_scripts/`에 있음,
에디터의 Python (REPL) 콘솔에서
`exec(open("<repo경로>/UE4Project/editor_scripts/<파일명>").read())`로 실행 —
홈 디렉토리를 깨끗하게 유지하려고 예전에 `~/`에 흩어져 있던 걸 여기로 옮김)

- `ue_inspect_level.py`, `ue_list_actors.py` — 레벨/액터 조사
- `ue_inspect_spline_cones.py`, `ue_inspect_spline_cones2.py` —
  spline_cones 액터 조사
- `ue_populate_spline.py` — 실제 용인 서킷 센터라인을 스플라인에 설정
  (실행 완료)
- `ue_cleanup_and_frame.py` — 중복 액터 제거, 뷰포트 프레이밍을 위해
  메인 스플라인 선택 (실행 완료)
- `ue_save_current.py`, `ue_save_as_new_map.py` — 현재 레벨 저장, 새 맵으로
  저장(셸 복사 트릭) (실행 완료)
- `ue_resize_floor.py` — `floor` 액터 리사이즈 (실행 완료)
- `ue_reposition_start.py`, `ue_adjust_playerstart.py`, `ue_measure_offset.py`,
  `ue_debug_positions.py`, `ue_check_start_cones.py`,
  `ue_inspect_startfinish.py`, `ue_fix_start_position.py`(결함 있던 버전),
  `ue_fix_start_position2.py`, `ue_fix_start_final.py`,
  `ue_fix_start_nearest_pair.py`(최종 정답 — 3D 거리 기준 최근접 쌍 탐색),
  `ue_read_startline_rotation.py` — 시작선/PlayerStart 위치·회전 조사 및
  수정 (실행 완료)
- `ue_full_status_check.py`, `ue_find_referee_registration.py`,
  `ue_check_array_lengths.py`(639+639 확인), `ue_check_custom_cones.py`
  (`None` 확인) — 128개 콘 상한 원인 조사 (실행 완료, 원인 미발견)
- `ue_inspect_vehicle_dynamics.py`/`2`/`3` — `TechnionCarPawn`의 PhysX
  Vehicle 파라미터 추출 (진행 중, [VEHICLE_DYNAMICS.md](VEHICLE_DYNAMICS.md)
  참고)
- `yongin_centerline.csv` (같은 `editor_scripts/` 폴더) — 스플라인을 채우는
  데 쓴 실제 재중심화된 256개 포인트 센터라인 데이터 (미터 단위)
