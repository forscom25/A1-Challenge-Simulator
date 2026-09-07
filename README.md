# A1 Challenge Simulator — 팀 문서

이 저장소는 팀이 실제로 진행 중인 FSDS(Formula Student Driverless
Simulator) 작업을 위한 **문서 전용** 저장소입니다. 시뮬레이터 프로젝트
자체는 일부러 포함하지 않습니다 — FS-Driverless 원본 클론, 미리 빌드된
바이너리, Unreal Engine 소스/빌드 결과물 전부 용량이 크고 대부분 서드파티
콘텐츠라서, `Formula-Student-Driverless-Simulator/`, `simulator/`,
`~/UnrealEngine`에 로컬로만 존재하며 이 저장소에서는 추적하지 않습니다
(`.gitignore` 참고). `UnrealEngine/`은 빌드 시스템이 절대경로를 빌드
캐시에 기록하기 때문에 (`Intermediate/Build/.../Makefile.bin` 등) 한 번
빌드한 뒤에는 옮기지 않음 — 옮기면 다음 증분 빌드 때 불필요하게 여러
모듈이 재컴파일될 수 있어서, 처음 클론한 위치(`~/UnrealEngine`)에
그대로 둠. 반면 `ue_*.py` 같은 소형 Python 스크립트는 이런 제약이
없어서 `Formula-Student-Driverless-Simulator/UE4Project/editor_scripts/`로
옮겨서 관리 (아래 UE4 소스빌드 가이드 참고).

여기 있는 것은 팀원이 처음부터 다시 파악하지 않고도 **이해하고, 재현하고,
확장할 수 있도록** 필요한 모든 것입니다: 환경을 어떻게 구축했는지, 그
과정에서 만난 모든 실제 버그와 해결 방법, 평소 사용법, 현재 진행상황까지.

## 문서 지도

| 문서 | 용도 | 종류 |
|---|---|---|
| **[README.md](README.md)** | 지금 보고 있는 이 문서 — 저장소 개요와 색인 | 색인 |
| **[SETUP_DEBUG_LOG.md](SETUP_DEBUG_LOG.md)** | 아무것도 없는 머신에서 시뮬레이터 + ROS2 Humble 브릿지를 띄우기까지: NVIDIA 드라이버, ROS2, colcon 빌드. 만난 모든 버그(드라이버 버전 불일치, 경로 공백 버그, 풀스크린 크래시)와 해결 방법을 겪은 순서대로 기록. | 설치 기록 (역사적, 계속 덧붙이는 형태) |
| **[SIMULATOR_GUIDE.md](SIMULATOR_GUIDE.md)** | "그냥 실행하고 싶다" — 시뮬레이터와 ROS2 브릿지를 평소에 실행하는 방법, 센서 추가하는 법, 알아두면 좋은 함정들. 위의 최초 설치가 이미 끝났다고 가정. | 사용 가이드 (상시 최신화) |
| **[YONGIN_MAP_GUIDE.md](YONGIN_MAP_GUIDE.md)** | 실제 용인 서킷(MGeo 대회 데이터)을 FSDS가 읽을 수 있는 콘 트랙 CSV로 변환한 방법, 그리고 아래 소스빌드 작업의 계기가 된 패키지 바이너리 `CustomMap`의 스케일/콘 개수 한계. | 사용 가이드 |
| **[UE4_소스빌드_가이드.md](UE4_소스빌드_가이드.md)** | 패키지된 바이너리 대신 Unreal Engine 4.27 소스로 FSDS를 직접 빌드해서, 실제 스케일 그대로의 용인 서킷을 진짜 레벨로 만드는 방법 (`CustomMap`의 한계를 우회). 만난 모든 버그(AirLib 스테이징 누락, 진짜 glibc/ABI 불일치)와 해결법, Python으로 레벨을 제작한 과정까지 포함. | 설치 가이드 + 버그 기록 |
| **[PROGRESS_UE4_BUILD.md](PROGRESS_UE4_BUILD.md)** | Unreal Engine 소스빌드 작업의 실시간 진행상황: 완료된 것, 막힌 것, 남은 것. 작업이 진행되는 동안 계속 갱신하고, 마무리되면 남길 가치가 있는 내용은 위 가이드로 옮긴 뒤 이 파일은 정리(종료 표시 또는 삭제). | 진행상황 트래커 (살아있는 문서, 특정 작업 범위에 한정) |
| **[VEHICLE_DYNAMICS.md](VEHICLE_DYNAMICS.md)** | FSDS 차량이 쓰는 Unreal PhysX Vehicle 시스템의 동역학 충실도(타이어 슬립, 서스펜션, 엔진/변속기, 공력 드래그)와 `TechnionCarPawn` Blueprint의 실제 파라미터 값. 컨트롤러(Pure Pursuit → MPC) 설계 시 시뮬레이터가 가정하는 차량 모델을 참고하기 위한 문서. | 참고 문서 (상시 최신화) |
| **[PROGRESS_CONTROLLER.md](PROGRESS_CONTROLLER.md)** | `fsds_controller` ROS2 패키지(Pure Pursuit 스켈레톤) 진행상황: 좌표계/조향 부호/경로 이탈 복구/참조 경로 데이터 등 겪은 버그와 해결 과정, 최종 검증 결과(0.17m 오차), 남은 작업. | 진행상황 트래커 (살아있는 문서, 특정 작업 범위에 한정) |
| **[PRESENTATION.md](PRESENTATION.md)** | 진행상황 발표용 슬라이드 초안 (맵 제작, 컨트롤러 설계, 차량 동역학, MPC 다음 단계). | 발표 자료 (일회성 산출물) |

## 문서 간 관계

```
README.md  ──────────────────────────────────── 아래 모든 문서의 색인
   │
   ├─ SETUP_DEBUG_LOG.md ─────────── 1단계: 빈 머신 → 시뮬레이터+브릿지 실행
   │      └─ SIMULATOR_GUIDE.md ──── 1단계가 끝난 뒤 평소 사용법
   │
   ├─ YONGIN_MAP_GUIDE.md ─────────── 실제 트랙 데이터 → FSDS CSV (패키지
   │                                  바이너리의 CustomMap과 함께 사용;
   │                                  거기서 스케일/콘개수 한계에 부딪힘 — 아래로 이어짐)
   │
   └─ UE4_소스빌드_가이드.md ────────── 2단계: 그 한계를 넘기 위해 UE4
          ├─ PROGRESS_UE4_BUILD.md      소스에서 직접 빌드, 실시간 진행상황
          ├─ VEHICLE_DYNAMICS.md        차량 물리 모델 조사 (컨트롤러 설계 참고용)
          ├─ PROGRESS_CONTROLLER.md     3단계: Pure Pursuit 컨트롤러 스켈레톤
          └─ PRESENTATION.md            진행상황 발표 자료
```

## 새 문서를 추가할 때 참고할 규칙

- **`*_LOG.md`**: 무엇을 시도했고 무엇이 어떻게 깨졌는지, 겪은 순서 그대로
  적는 역사적 기록. 이런 문서는 다시 쓰지 말고 뒤에 덧붙일 것.
- **`*_GUIDE.md`**: "이걸 어떻게 하는가"를 다루는 상시 갱신되는 참고
  문서. 항상 최신 상태를 유지하고, 오래된 안내를 새 안내 옆에 그냥
  남겨두지 말고 그 자리에서 수정할 것.
- **`PROGRESS_*.md`**: 진행 중인 하나의 명확한 작업 범위에 대한 살아있는
  진행상황/TODO 트래커. 이런 문서는 원래 임시로 존재하는 것 — 추적하던
  작업이 끝나면 남길 가치가 있는 내용은 관련 가이드로 옮기고 트래커
  자체는 정리(삭제하거나 종료됐다고 표시).
- 모든 문서는 한국어로 작성. 영문 버전은 따로 유지하지 않음 — 예전에
  존재했던 영문/`_KR` 쌍 구조는 정리하고 한국어 단일 버전으로 통합함.
