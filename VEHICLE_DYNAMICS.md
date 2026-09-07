# FSDS 차량 동역학 — PhysX Vehicle 시스템과 Blueprint 파라미터

FSDS의 차량(`ACarPawn`)이 실제로 어느 수준의 물리 시뮬레이션을 쓰는지,
그리고 그 세부 파라미터가 어디에 어떻게 박혀 있는지 정리한 문서.
컨트롤러(Pure Pursuit → MPC) 설계 시 "시뮬레이터가 어떤 차량 모델을
가정하고 있는가"를 알아야 튜닝 방향을 잡을 수 있어서 조사함.

## 1. 개요 — 단순 운동학 모델이 아니라 PhysX 강체 물리

FSDS 차량은 `ACarPawn : public AWheeledVehicle`로 정의되고, 내부적으로
Unreal Engine 4의 **PhysX Vehicle 시스템**
(`UWheeledVehicleMovementComponent4W`)을 그대로 사용함. 즉 "위치를
운동학적으로 갱신하는 자전거 모델(bicycle model)"이 아니라, 실제
질량·관성·서스펜션·타이어 슬립·엔진/변속기 RPM까지 갖춘 4륜 강체
차량 시뮬레이션임. 중요한 결과:

- 조향/가속 입력이 즉시 위치 변화로 반영되지 않음 — 타이어 슬립, 엔진
  토크 전달, 서스펜션 반응 지연이 전부 물리적으로 시뮬레이션됨.
- 저마찰(슬립) 상황, 하중 이동(load transfer), 관성 때문에 생기는
  언더/오버스티어 같은 현상이 자연스럽게 나타날 수 있음 — 순수
  기구학적 Pure Pursuit로는 못 잡아내는 오차 요인.
- 이런 이유로 최종 컨트롤러를 MPC로 가는 계획이 합리적임: MPC는 (필요시)
  차량의 동역학적 제약을 모델에 직접 반영할 수 있지만, Pure Pursuit는
  기구학적 가정 위에서만 동작하므로 스켈레톤/기초 검증용으로 적합.

## 2. ROS2로 노출되는 것

`fsds_ros2_bridge`가 퍼블리시하는 차량 상태 토픽들:

- **`CarState`**: `speed`, `gear`, `rpm`, `maxrpm`, `handbrake` — 엔진/변속기
  상태가 실제로 존재하고 노출된다는 뜻. 단순 등가속 모델이 아님.
- **`/wheel_states`** (`WheelState[4]`): 각 바퀴별 `rpm`, `steering_angle` —
  4륜 개별 텔레메트리. 두 앞바퀴의 조향각이 서로 다를 수 있음(애커먼
  지오메트리 여부는 `WheelSetups`의 개별 조향 파라미터에 달려 있음 —
  §4 참고).

`fs_msgs/ControlCommand` 입력:

- `throttle` ∈ [0, 1], `brake` ∈ [0, 1], `steering` ∈ [-1, 1] (정규화값).
- `steering`의 ±1은 FSDS 문서 기준 차량의 최대 조향각 **±25°**에 대응.
  **(정정)** 이전에는 이게 바퀴 자체의 PhysX `SteerAngle` 물리적
  한계(`40°`로 추정)와는 별개의 값이라고 적었었는데, Wheel Class
  Blueprint를 직접 확인한 결과(§4) 앞바퀴의 실제 `Steer Angle`은
  **`25.0°`**로, `ControlCommand`의 ±25°와 정확히 일치함 — `40°`는 잘못된
  값이었음. 다만 §3의 `Steering Curve`(속도별 배율, 거의 평평하게 ~0.5)가
  추가로 곱해지므로, 실제 순간 최대 조향각은 속도에 따라 25°보다 작을 수
  있음(대략 25°×~0.5 ≈ 12.5° 근처로 추정, §4 참고).

## 3. `TechnionCarPawn` Blueprint — 확인된 파라미터

처음엔 Python(`unreal.get_default_object` + `get_editor_property`,
`ue_inspect_vehicle_dynamics2.py`)으로 최상위 스칼라 값만 읽었고, 중첩된
구조체(`EngineSetup`/`TransmissionSetup`/`DifferentialSetup`)는 파이썬
콘솔에서 펼쳐지지 않았음. 이후 에디터 GUI로 전환해서 훨씬 빠르게 확인함
— `TechnionCarPawn` Blueprint를 열고 **Components** 탭에서 이동체 컴포넌트
선택 → **Details** 패널에서 각 카테고리(`Engine Setup`,
`Transmission Setup`, `Differential Setup`, `Steering Setup`,
`Vehicle Setup`)를 펼쳐서 스크린샷으로 확인함.

### 차대/질량 (Vehicle Setup)

| 파라미터 | 값 | 비고 |
|---|---|---|
| `Mass` | 255.0 | kg 추정. 실제 FS차(약 200~250kg 완주 중량)와 비슷한 스케일 |
| `Drag Coefficient` | 0.3 | **공력 드래그가 모델링되어 있음** (초기에 "공력 없음"으로 잘못 판단했던 것을 정정) |
| `Chassis Width` | 144.0 | cm 추정 |
| `Chassis Height` | 112.0 | cm 추정 |
| `Reverse As Brake` | false | 후진 입력이 브레이크로 겹쳐 쓰이지 않음 — 전/후진이 명확히 분리 |
| `InertiaTensorScale` | (x=1.0, y=1.333, z=1.5) | 기본 박스 관성 텐서에 곱해지는 스케일. yaw축(z) 관성이 가장 크게 스케일됨 |
| `MinNormalizedTireLoad` / `MaxNormalizedTireLoad` | 0.0 / 3.0 | 타이어 하중 민감도 범위 — 하중 이동에 따른 그립 변화가 최대 3배까지 반영될 수 있음 |
| `Wheel Setups` | 4 elements | 각 원소가 `Wheel Class`(별도 Blueprint 에셋) 참조 — 아직 미확인, §4 참고 |

### 엔진 (Engine Setup)

| 파라미터 | 값 |
|---|---|
| `Max RPM` | 13000.0 |
| `MOI` (관성 모멘트) | 0.01 |
| `Damping Rate Full Throttle` | 0.15 |
| `Damping Rate Zero Throttle Clutch Engaged` | 0.01 |
| `Damping Rate Zero Throttle Clutch Disengaged` | 0.35 |

**Torque Curve**: RPM(x) 대 토크(y, Nm 추정) 곡선. 0 RPM에서 약 384로
시작해서 완만히 오르내리다가 약 6000~9000 RPM 구간에서 피크(≈512)를
찍고, 이후 최대 RPM(13000)에 가까워지며 다시 384 근처로 떨어지는 형태 —
오토바이 엔진 특유의 고회전형 토크 커브(FS 차량이 흔히 쓰는 모터사이클
엔진 스왑과 일치하는 모양).

### 디퍼렌셜 (Differential Setup)

| 파라미터 | 값 |
|---|---|
| `Differential Type` | Open 4W |
| `Front Rear Split` | 0.65 |
| `Front Left Right Split` / `Rear Left Right Split` | 0.5 / 0.5 |
| `Centre Bias` / `Front Bias` / `Rear Bias` | 1.3 / 1.3 / 1.3 |

`Front Rear Split = 0.65`는 UE4 기준 "0.5보다 크면 앞쪽에 더 많이" 분배한다는
뜻 — 즉 토크의 65%가 앞바퀴로 감. 레이싱카치고는 특이하게 앞바퀴 편향인데,
FS 차량은 보통 RWD를 가정하는 경우가 많아서, 이 값이 실제로 튜닝된 값인지
아니면 UE4 기본 VehicleAdv 템플릿에서 물려받은 미조정 기본값인지 불명확함
— 컨트롤러 튜닝 시 참고만 하고 과신하지 말 것. 또한 `Differential Type`이
**Open**이라서 `Bias` 값들(1.3)은 LSD/토크벡터링 디퍼렌셜에서나 의미가
있고 Open 타입에서는 사실상 작동하지 않을 가능성이 높음 — 즉 코너링 중
가속 시 휠스핀이 걸리면 안쪽 바퀴로 토크가 쏠리는 개방형 디퍼렌셜
특유의 거동(오픈 디퍼렌셜 특유의 불균등 토크 분배)이 나타날 수 있음.

### 변속기 (Transmission Setup)

| 파라미터 | 값 |
|---|---|
| `Automatic Transmission` | true |
| `Gear Switch Time` | 0.0 (변속 즉시 완료) |
| `Gear Auto Box Latency` | 0.2 (오토박스가 변속을 결정하기까지 지연) |
| `Final Ratio` (파이널 드라이브비) | 3.0 |
| `Clutch Strength` | 10.0 |
| 전진 기어 | **1단만 설정됨** — Gear 1: ratio 5.5, down 0.5, up 0.9 |
| `ReverseGearRatio` | -4.0 |
| `NeutralGearUpRatio` | 0.15 |

전진 기어가 **1단뿐**이라는 게 중요한 포인트 — 사실상 단일 기어(변속
없는) 구동계로 동작함. 자동변속기라 해도 갈아탈 상위 기어 자체가 없으므로,
컨트롤러 입장에서는 기어 변속 지연/RPM 밴드 전환 같은 복잡성을 걱정할
필요 없이 "스로틀 → (거의) 직결 토크 전달"에 가깝게 취급 가능.

### 조향 (Steering Setup)

| 파라미터 | 값 |
|---|---|
| `Steering Curve` | 속도(x축, 단위 불명 — 0~144) 대 조향 배율(y축, 0.44~0.56) — 그래프가 거의 평평, 대략 ~0.5 부근에서 고정 |
| `Ackermann Accuracy` | 1.0 (완벽한 애커먼 지오메트리) |

속도에 따른 조향각 축소(스피드 센서티브 스티어링)가 사실상 거의
없다는 뜻(배율이 전 속도 구간에서 거의 고정 ~0.5). 앞바퀴의 실제
`Steer Angle` baseline은 **25.0°**로 확인됨(§4) — 즉 순간 최대 조향각은
대략 25°×~0.5 ≈ 12.5° 근처로 추정 (그래프가 완전히 평평하지는 않아서
속도에 따라 약간 달라짐, §4 참고). `Ackermann Accuracy = 1.0`은
안쪽/바깥쪽 바퀴 조향각이 이상적인 애커먼 지오메트리를 따른다는 뜻 —
Pure Pursuit가 가정하는 기구학적 자전거 모델과 비교적 잘 맞아떨어지는
조건.

### Avoidance (사용 안 함)

`Use RVOAvoidance = false`, `Avoidance Weight = 0.0` — UE4
`MovementComponent`가 기본으로 갖고 있는 AI 회피 관련 필드로,
플레이어가 직접 조종하는 이 차량에서는 비활성 상태. 컨트롤러 설계와
무관, 참고할 필요 없음.

## 4. 바퀴/타이어 (`Wheel Class` — 앞/뒤 별도 Blueprint)

`Wheel Setups`의 각 원소가 참조하는 `Wheel Class`를 직접 열어서 확인함
— 앞바퀴와 뒷바퀴가 서로 다른 Wheel Class 에셋을 씀.

| 파라미터 | 앞바퀴 | 뒷바퀴 |
|---|---|---|
| `Offset` (X,Y,Z) | (0,0,0) | (0,0,0) |
| `Shape Radius` | 20.0 | 20.0 |
| `Shape Width` | 19.0 | 19.0 |
| `Mass` | 9.0 | 9.0 |
| `Damping Rate` | 0.25 | 0.25 |
| `Affected by Handbrake` | false | **false** (확인 완료 — 뒷바퀴도 false) |
| **`Steer Angle`** | **25.0°** | **0.0°** (조향 안 됨 — 앞바퀴 조향 전용, 4륜조향 아님) |
| `Tire Config` | `FormulaFrontTire` | `FormulaBackTire` |
| `Lat Stiff Max Load` | 4.0 | 4.0 |
| `Lat Stiff Value` | 48.0 | 48.0 |
| `Long Stiff Value` | 400.0 | 400.0 |
| `Suspension Max Raise` / `Max Drop` | 2.5 / 2.5 | 2.5 / 2.5 |
| `Suspension Natural Frequency` | 5.92 | 6.07 |
| `Suspension Damping Ratio` | 0.7 | 0.7 |
| `Sweep Type` | SimpleAndComplex | SimpleAndComplex |
| `Max Brake Torque` | 350.0 | 180.0 |
| `Max Hand Brake Torque` | 3000.0 | 3000.0 |

**정정**: §2/§3에서 언급했던 바퀴 `Steer Angle` 추정값(`40°`)은
틀렸음 — 실제로는 **`25.0°`**로, `ControlCommand`의 정규화 범위(±25°)와
정확히 일치함. 뒷바퀴는 조향각이 아예 `0°`라서 순수 전륜조향(FWS)
구성.

### `Tire Config` 에셋 자체의 마찰 설정

`Lat/Long Stiff Value`와는 별개 레이어 — `Wheel Class`의 `Tire Config`
필드가 실제로 가리키는 에셋(`FormulaFrontTire`/`FormulaBackTire`, §4
표의 `Tire Config` 행 참고)을 직접 열면 지면의 `PhysicalMaterial`
종류별 마찰 배율을 정의하는 `Friction Scale` / `Tire Friction Scales`가
있음:

| 파라미터 | `FormulaFrontTire` | `FormulaBackTire` |
|---|---|---|
| `Friction Scale` (기본값) | 1.4 | 1.4 |
| `Tire Friction Scales` (표면별 override) | 없음 (0 elements) | 없음 (0 elements) |

**앞뒤가 완전히 동일**(1.4, override 없음) — 표면 종류와 무관하게 항상
균일한 ×1.4 마찰 배율이 적용됨. 즉 이 레이어에서는 앞/뒤 그립 차이가
전혀 없음 (그립 차이가 있다면 §4의 `Lat/Long Stiff Value` 쪽에서 나는
것이고, 이쪽 `Tire Config` 레이어와는 무관).

**참고 — 실제로는 안 쓰이는 다른 에셋 쌍 발견**: 콘텐츠 브라우저에
`Vehicle_Front_TireConfig`/`Vehicle_Back_TireConfig`라는 또 다른
`Tire Config` 에셋 쌍이 있는데, `Friction Scale`이 각각 2.0/3.0이고
`NonSlippery`(×1.0)/`Slippery`(×0.7) 표면별 override까지 갖고 있음 —
겉보기엔 더 정교해 보이지만, `TechnionCarPawn`의 `Wheel Class`가 실제로
참조하는 건 위 표의 `FormulaFrontTire`/`FormulaBackTire`쪽이라 이 에셋들은
**사용되지 않는 것으로 보임** (UE4 스톡 VehicleAdv 샘플 콘텐츠에서 남아있는
잔재로 추정). 혼동하지 않도록 이름으로 구분해서 기록해둠.

**주목할 점**:
- **앞뒤 타이어가 다른 에셋**(`FormulaFrontTire` / `FormulaBackTire`)이지만
  실제 `Lat Stiff Value`(48.0)와 `Long Stiff Value`(400.0)는 앞뒤 완전히
  동일함 — 타이어 강성 자체는 앞뒤 차이가 없고, 별도 에셋으로 나뉜 건
  §4 하단의 `Tire Config`(마찰 배율) 쪽 확장 여지를 위한 구조로 보임.
- **제동 배분이 앞쪽으로 치우침**(`Max Brake Torque` 앞 350 vs 뒤 180) —
  실제 차량의 제동 시 하중 이동(앞으로 쏠림)을 반영한 현실적인 세팅.
- **서스펜션 고유진동수가 앞(5.92)보다 뒤(6.07)가 약간 더 큼** — 뒤가
  약간 더 뻣뻣한 스프링 세팅.
- **핸드브레이크가 사실상 비활성**: 앞/뒤 모든 바퀴의 `Affected by
  Handbrake`가 `false`임 — `Max Hand Brake Torque=3000.0`이 양쪽에
  설정되어 있어도 어느 바퀴에도 실제로 걸리는 바퀴가 없어서, `handbrake`
  입력을 줘도 제동 효과가 없을 가능성이 높음. `CarState.handbrake`
  토픽으로 상태 자체는 노출되지만(§2), 실제 물리적 제동으로 이어지는지는
  PIE에서 직접 확인 필요 — 컨트롤러가 핸드브레이크에 의존하는 로직을
  쓸 계획이라면 반드시 검증할 것.
- 두 `Wheel Class`의 `Offset`이 전부 (0,0,0)이라서, **휠베이스는 이
  Blueprint 필드들만으로는 계산 불가** — 실제 바퀴 위치는 스켈레탈 메시
  (`FormulaMesh_Skeleton`)의 본(bone) 트랜스폼에 baked되어 있음. 정확한
  휠베이스가 필요하면 (a) 스켈레톤 에디터에서 앞/뒤 바퀴 본 위치를 직접
  확인하거나, (b) PIE 중에 Python으로
  `SkeletalMeshComponent.get_bone_location()`을 앞/뒤 바퀴 본 이름으로
  각각 호출해서 거리 계산.

## 5. 컨트롤러 설계에 대한 시사점

- Pure Pursuit 스켈레톤 단계에서는 위 세부 동역학을 무시하고 순수
  기구학적 추종만 해도 됨 — 목적이 인터페이스/토폴로지 검증이므로.
- **1단 고정 변속기**(§3) 덕분에 엔진/기어 모델은 걱정할 부분이 적음 —
  RPM 밴드나 변속 타이밍을 신경 쓸 필요 없이 스로틀을 거의 직결 토크로
  취급 가능.
- **Open 디퍼렌셜**(§3) + `MaxNormalizedTireLoad=3.0`처럼 하중 이동에
  따른 그립 변화 폭이 큰 조합이라, 고속 코너링 중 가속 시 기구학적
  자전거 모델과 실제 시뮬레이터 거동이 벌어질 수 있음. 필요하면 단순
  동역학적 자전거 모델(선형 타이어 모델 + yaw 관성) 정도로 확장하는
  것을 고려.
- **순수 전륜조향**(§4, 뒷바퀴 `Steer Angle=0`) + `Ackermann Accuracy = 1.0`
  + 거의 평평한 `Steering Curve`(§3, ~0.5 배율) 조합은 Pure Pursuit가
  가정하는 표준 자전거 모델(전륜조향, 후륜 고정)과 정확히 일치함 —
  기구학적 가정이 시뮬레이터 구성과 잘 맞는 좋은 신호. 다만 실제 순간
  최대 조향각은 25°가 아니라 대략 25°×~0.5 ≈ 12.5° 근처로 봐야 함.
- `Front Rear Split = 0.65`(앞바퀴 편향)가 튜닝된 값인지 기본값인지
  불명확 — RWD를 가정하고 컨트롤러를 설계하지 말 것 (§3 참고).
- **제동력이 앞쪽에 더 실림**(§4, 앞 350 vs 뒤 180 Max Brake Torque) —
  급제동 시 하중 이동과 함께 요(yaw) 안정성에 영향을 줄 수 있는 요소.
- `ControlCommand`의 `steering` 정규화 범위(±25°, §2)는 바퀴의 실제
  `Steer Angle`(25°, §4)과 1:1로 맞아떨어짐 — 컨트롤러가 산출한 조향각을
  그대로 `steering = angle_deg / 25.0`으로 정규화해서 보내면 됨. 다만
  `Steering Curve`의 속도별 배율까지 고려하면 실제 도달 각도는 그보다
  작을 수 있음.

## 6. 관련 문서

- 128개 콘 상한 이슈처럼 이 조사와 별개로 진행된 UE4 소스빌드/레벨 제작
  경과는 [PROGRESS_UE4_BUILD.md](PROGRESS_UE4_BUILD.md) 참고.
- 컨트롤러 스켈레톤(`fsds_controller` ROS2 패키지) 진행 상황도
  `PROGRESS_UE4_BUILD.md`에서 함께 추적 중.
