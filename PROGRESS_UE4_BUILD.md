# 실차 스케일 용인 서킷 — Unreal Engine 소스 빌드 진행상황

2026-09-04 기준 상태. 목표: FSDS를 Unreal Engine 소스에서 직접 빌드해서
실제 스케일(축소 없는) 용인 서킷을 (`CustomMap` 축소판 대신) 실제 레벨로
쓸 수 있게 만드는 것 — 패키지된 바이너리 `CustomMap`의 약 128개 콘 상한과
좁은 월드 범위 한계를 우회.

## 완료

1. **Unreal Engine 4.27을 `~/UnrealEngine`에 소스 빌드** (Editor,
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

## 남은 작업

- [ ] 에디터에서 `YonginCircuitMap`을 로드해서 이후 작업은 이 파일 기준으로
      진행
- [ ] `floor` StaticMeshActor를 실제 트랙 범위에 맞게 리사이즈/재배치
      (bbox ≈ 822m × 476m, 중심 ≈ (224, -78)m)
- [ ] `PlayerStart`, `StartFinishLine`, `Referee`를 새 트랙의 실제
      시작점에 맞게 재배치; `Referee`의 `Cones` 링크가 여전히 콘을 채운
      스플라인 액터를 올바르게 가리키는지 확인
- [ ] Play-in-Editor로 실제 주행 테스트
- [ ] `RunUAT.sh BuildCookRun`으로 (GUI 없이 커맨드라인만으로) 새 맵을
      포함한 스탠드얼론 Linux 빌드 패키징 — 에디터를 열지 않고도 지금의
      `simulator/FSDS.sh`처럼 평소에 실행할 수 있도록
- [ ] `fsds_ros2_bridge`로 검증: `/testing_only/track`에 실제 콘이 **전부**
      나오는지 확인 (일부만이 아니라 — `CustomMap`의 캡 문제처럼 화면상
      멀쩡해 보여도 몰래 누락될 수 있으니 화면만 믿지 말 것), 차량이 월드
      밖으로 떨어지지 않고 실제 범위 전체를 주행할 수 있는지 확인
- [ ] `SETUP_DEBUG_LOG.md`, `SIMULATOR_GUIDE_KR.md`에 새 소스빌드 절차와
      새 맵 실행 방법 반영

## 지금까지 작성한 유용한 스크립트 (전부 `~/`에 있음, 에디터의
Python (REPL) 콘솔에서 `exec(open("~/<파일명>").read())`로 실행)

- `ue_inspect_level.py`, `ue_list_actors.py` — 레벨/액터 조사
- `ue_inspect_spline_cones.py`, `ue_inspect_spline_cones2.py` —
  spline_cones 액터 조사
- `ue_populate_spline.py` — 실제 용인 서킷 센터라인을 스플라인에 설정
  (실행 완료)
- `ue_cleanup_and_frame.py` — 중복 액터 제거, 뷰포트 프레이밍을 위해
  메인 스플라인 선택 (실행 완료)
- `ue_save_current.py` — 현재 레벨 저장 (실행 완료)
- `/home/ailab/yongin_centerline.csv` — 스플라인을 채우는 데 쓴 실제
  재중심화된 256개 포인트 센터라인 데이터 (미터 단위)
