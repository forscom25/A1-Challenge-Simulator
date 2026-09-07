# Unreal Engine 소스 빌드로 실제 스케일 용인 서킷 만들기

이 문서는 FSDS(Formula Student Driverless Simulator)를 미리 빌드된 바이너리가
아니라 **Unreal Engine 4.27 소스에서 직접 빌드**해서, 실제 스케일(약 2.5km)의
용인 서킷을 시뮬레이터 레벨로 직접 만든 과정을 정리한 문서입니다. 대상
독자는 이 시스템을 이어받아 수정하거나 처음부터 다시 재현해야 하는 팀원입니다.

관련 문서: `SETUP_DEBUG_LOG.md` (시뮬레이터/ROS2 브릿지 기본 설치),
`SIMULATOR_GUIDE.md`/`SIMULATOR_GUIDE_KR.md` (평소 실행 방법),
`Formula-Student-Driverless-Simulator/maps/YonginCircuit/` (커스텀 맵 CSV
방식 — 이 문서가 다루는 소스 빌드 방식과는 별개의, 더 가벼운 대안).

## 왜 소스 빌드가 필요했나

미리 빌드된 바이너리(`v2.2.0`)의 `CustomMap` 레벨은:
1. 소형 FS 트랙 기준으로 만들어져 있어 실제 서킷 규모(약 850m × 900m)를
   렌더링하지 못함
2. 전체 콘 개수가 약 128개로 하드캡 걸려있음 (화면상이 아니라
   `/testing_only/track` RPC로 직접 확인해서 발견함)

두 한계 모두 패키지된 `.pak` 안에 컴파일되어 들어있는 Blueprint 로직이라
`.ini`나 커맨드라인으로 바꿀 수 없음. 실제 서킷 스케일을 제대로 쓰려면
Unreal Editor로 직접 레벨을 만들어야 함 — 그래서 Epic 계정 연동, 소스 클론,
엔진 빌드까지 진행.

## 1. Unreal Engine 4.27 소스 빌드

### 사전 준비 (사용자가 직접 해야 하는 부분)
- unrealengine.com에서 Epic Games 계정을 만들고 GitHub 계정과 연동 (EULA 동의 필요)
- 이 머신에서 `gh auth login` (브라우저 device-code 방식) → `gh auth setup-git`
  으로 git 인증 연결 — 토큰이 대화 내용에 노출되지 않는 안전한 방식

### 빌드

**주의**: `UnrealEngine/`은 (다른 대용량 디렉토리들과 달리) 홈 디렉토리
바로 밑 `~/UnrealEngine`에 그대로 둠 — 한 번 빌드하고 나면 UBT가
`Intermediate/Build/.../Makefile.bin` 같은 빌드 캐시 파일에 절대경로를
그대로 박아 넣기 때문에, 나중에 옮기면 다음 증분 빌드 때 불필요하게 여러
모듈이 재컴파일될 수 있음 (실제로 겪어본 뒤 다시 원위치로 되돌림). 클론할
때부터 최종적으로 둘 위치에 바로 클론할 것.

```bash
git clone --depth=1 -b 4.27 https://github.com/EpicGames/UnrealEngine.git ~/UnrealEngine
cd ~/UnrealEngine
./Setup.sh              # 서드파티 바이너리 의존성 다운로드 (~12GB)
./GenerateProjectFiles.sh
```

**중요**: `make` (인자 없이, 기본적으로 7개 엔진 타겟을 한번에 병렬 빌드)로
빌드하면 안 됨. 여러 타겟이 동시에 `UnrealHeaderTool`이라는 공용
의존성을 빌드하려고 경쟁하면서, Epic의 Linux 진행률 표시 래퍼 스크립트가
깨지는 레이스 컨디션이 있음 (`ERROR: Unepxected line:` 같은 이상한 메시지와
함께 특정 타겟들이 아무 에러 메시지 없이 조용히 누락됨 — 실제로 7개 중
3개만 완료되고 4개(`UE4Editor` 포함!)가 그냥 사라지는 문제를 겪음). 대신
**타겟을 하나씩** 빌드:
```bash
bash Engine/Build/BatchFiles/Linux/Build.sh UE4Editor Linux Development
bash Engine/Build/BatchFiles/Linux/Build.sh ShaderCompileWorker Linux Development
bash Engine/Build/BatchFiles/Linux/Build.sh UnrealLightmass Linux Development
```
`UE4Editor`만 있으면 에디터는 뜨지만, `ShaderCompileWorker`(런타임 셰이더
컴파일)와 `UnrealLightmass`(라이트매스 빌드)는 에디터가 실제로 필요로 하므로
같이 빌드해둠. `UnrealFrontend`/`UnrealInsights` 등은 우리 목적에는 불필요해서
생략.

## 2. FSDS 프로젝트(Blocks 게임 모듈 + AirSim 플러그인) 빌드

타겟 이름은 `UE4Project/Source/BlocksEditor.Target.cs`에서 확인:
```bash
cd ~/UnrealEngine
bash Engine/Build/BatchFiles/Linux/Build.sh BlocksEditor Linux Development \
  -project="<repo경로>/UE4Project/FSOnline.uproject" -waitmutex
```

### 버그 1: AirLib이 플러그인 폴더에 없음

```
fatal error: 'common/AirSimSettings.hpp' file not found
```
이전에는 ROS2 브릿지용 cmake 빌드만 했었고 (`AirLib`을 그 자리에서
`add_subdirectory`로 참조), `AirSim/build.sh`(빌드 결과물을
`UE4Project/Plugins/AirSim/Source/AirLib`로 복사하는 스크립트)를 이
프로젝트에서 한 번도 실행한 적이 없었음. `AirSim/build.sh`를 실행하면 해결.

### 버그 2: 진짜 ABI 불일치 — `pthread_cond_clockwait`

`build.sh`를 실행해서 AirLib을 다시 빌드해도, 최종 링크 단계에서:
```
ld.lld: error: undefined symbol: pthread_cond_clockwait
>>> referenced by __mutex_base:511 (Engine/Source/ThirdParty/Linux/LibCxx/include/c++/v1/__mutex_base:511)
>>>               client.cc.o: ... in archive .../AirLib/deps/rpclib/lib/librpc.a
```

**원인 분석**:
- `AirSim/build.sh`는 이 머신의 시스템 `clang++-12`로 `librpc.a`를 빌드함
- 하지만 UE4Editor 플러그인(`.so`)의 최종 링크는 UE4가 번들로 갖고 있는
  자체 툴체인(`Engine/Extras/.../v19_clang-11.0.1-centos7`)을 **명시적
  `--sysroot=`로 지정**해서 사용 — 이 sysroot는 CentOS7 시절의 오래된
  glibc 기반이라 `pthread_cond_clockwait`(glibc 2.30부터 추가된 함수)가 없음
- `librpc.a`를 컴파일할 때 시스템 clang이 (sysroot 지정 없이) 호스트의
  최신 glibc 2.35 헤더를 보고 "이 함수 있네" 하고 그걸 써버린 게 문제 —
  즉 **컴파일 시점**의 헤더/sysroot와 **링크 시점**의 sysroot가 다른 게 원인

**해결**: `AirLib`/`rpclib`을 시스템 clang이 아니라 **UE4가 번들로 갖고
있는 clang과 정확히 같은 `--sysroot`**로 다시 빌드:
```bash
cd Formula-Student-Driverless-Simulator/AirSim
rm -rf build_debug
UE4_BIN=~/UnrealEngine/Engine/Extras/ThirdPartyNotUE/SDKs/HostLinux/Linux_x64/v19_clang-11.0.1-centos7/x86_64-unknown-linux-gnu/bin
SYSROOT=~/UnrealEngine/Engine/Extras/ThirdPartyNotUE/SDKs/HostLinux/Linux_x64/v19_clang-11.0.1-centos7/x86_64-unknown-linux-gnu
LIBCXX=~/UnrealEngine/Engine/Source/ThirdParty/Linux/LibCxx
export CC="$UE4_BIN/clang"
export CXX="$UE4_BIN/clang++"
export CXXFLAGS="-target x86_64-unknown-linux-gnu --sysroot=$SYSROOT -nostdinc++ -I$LIBCXX/include -I$LIBCXX/include/c++/v1"
export LDFLAGS="-target x86_64-unknown-linux-gnu --sysroot=$SYSROOT -nodefaultlibs -L$LIBCXX/lib/Linux/x86_64-unknown-linux-gnu -lc++ -lc++abi -lm -lc -lgcc_s -lgcc"
mkdir -p build_debug && cd build_debug
cmake ../cmake -DCMAKE_BUILD_TYPE=Debug
make -j16
```
빌드 후 `nm librpc.a | grep pthread_cond`로 확인하면 해당 심볼 참조 자체가
사라진 것을 볼 수 있음 (다른 pthread 함수로 대체되거나 아예 그 코드
경로를 안 타게 됨). 이후 `build.sh`가 하던 것과 동일하게 결과물을
`AirLib/lib`, `AirLib/deps/rpclib/lib`에 복사하고
`UE4Project/Plugins/AirSim/Source`로 rsync.

`AirSim.Build.cs`에 Linux용 `pthread` 시스템 라이브러리 링크도 추가함
(근본 원인은 아니었지만, 정상적인 관례이기도 하고 혹시 몰라 유지):
```csharp
else if (Target.Platform == UnrealTargetPlatform.Linux)
{
    PublicSystemLibraries.Add("pthread");
}
```

빌드 성공 후 `UE4Editor "<project>.uproject" -log`로 에디터가 정상적으로
뜨는 것 확인.

## 3. Python Editor Scripting으로 실제 스케일 레벨 만들기

### 준비: Editor Scripting Utilities 플러그인 활성화

기본 Python 콘솔(`unreal.EditorLevelLibrary` 등)은 별도 플러그인이 필요함.
`FSOnline.uproject`의 `Plugins` 목록에 추가:
```json
{ "Name": "EditorScriptingUtilities", "Enabled": true }
```
추가 후 에디터 재시작 필요. 이후 에디터 하단 **Output Log** 패널에서
드롭다운을 `Python (REPL)`로 바꾸면 파이썬 명령을 직접 입력 가능.
파일로 작성한 스크립트는 `exec(open("경로").read())`로 실행.

### `TrainingMap` 구조 조사

Python으로 액터 목록/속성을 직접 조회:
```python
actors = unreal.EditorLevelLibrary.get_all_level_actors()
for a in actors:
    print(a.get_name(), a.get_class().get_name())
```
확인된 주요 액터:
- `floor` (`StaticMeshActor`): 바닥. Landscape가 아니라 단순 평면 메시라서
  `Scale`만 바꾸면 쉽게 확장 가능 (기본 타일 1m×1m, 현재 200×250 스케일 =
  200m×250m)
- `spline_cones_best_200` (`spline_cones_C`): 콘 배치용 스플라인 액터.
  `docs/map-tutorial.md`에 문서화된 표준 맵 제작 방식 — `CustomMap`의
  불투명한(속을 알 수 없는) Blueprint와 달리, 이건 실제로 하드캡이 없음을
  확인함 (스플라인 포인트 30개일 때 콘 메시 컴포넌트 192개 → 일정 간격으로
  스플라인을 재샘플링해서 자동으로 콘을 배치하는 구조)
- `Referee` (`RefereeBP_C`), `StartFinishLine` (`StartFinishLine_C`),
  `PlayerStart_1` — 표준 레퍼리/출발선/스폰 지점

### 실제 용인 서킷 센터라인 데이터 준비

`mgeo_to_fsds.py`(용인 서킷 MGeo 변환 스크립트, 자세한 내용은
`maps/YonginCircuit/mgeo-conversion-guide.md` 참고)의 센터라인 추출 로직을
재사용해서, 실제 좌표를 시작점 기준 (0,0)으로 재중심화하고 10m 간격으로
리샘플링한 256개 포인트를 CSV로 저장
(`UE4Project/editor_scripts/yongin_centerline.csv`, 전체 루프 길이 2563.5m).

### 스플라인에 실제 데이터 채우기

```python
spline_comp = spline_cones_actor.get_component_by_class(unreal.SplineComponent)
spline_comp.clear_spline_points(update_spline=False)
world_points = [unreal.Vector(x*100, y*100, z) for x, y in centerline_points]  # m -> cm
spline_comp.set_spline_points(world_points, unreal.SplineCoordinateSpace.WORLD, True)
spline_comp.set_closed_loop(True, True)
```

### 함정: Construction Script가 자동으로 재실행되지 않음

`spline_cones_C`는 Blueprint의 **Construction Script**가 스플라인 데이터를
읽어서 실제 콘 메시 컴포넌트들을 생성하는 구조. 그런데 Python으로
`set_spline_points()`를 호출해도 콘 메시가 갱신되지 않음 (컴포넌트 개수가
그대로 192개로 유지됨). `unreal.EditorLevelLibrary`나 액터 인스턴스
어디에도 `rerun_construction_scripts` 같은 함수가 Python에 노출되어 있지
않음 (`AActor::RerunConstructionScripts()`는 C++ 전용이고
`BlueprintCallable`로 노출되어 있지 않은 것으로 보임).

**해결**: 실제 에디터 UI를 통한 트랜스폼 변경(마우스로 뷰포트에서 액터를
살짝 이동)은 `PostEditMove` 훅을 거쳐서 Construction Script를 정상적으로
재실행시킴. 그래서:
1. Python으로 스플라인 포인트 데이터만 먼저 설정
2. 액터를 선택한 뒤, 뷰포트에서 `W`(이동 기즈모)로 살짝 드래그

이렇게 하니 콘 메시 컴포넌트가 192개 → **1278개**로 정상적으로 재생성됨
(256개 포인트, 10m 간격에 맞게 스케일). 뷰포트에서 실제 용인 서킷과 동일한
형태(헤어핀, 에스자 구간 등)의 트랙 윤곽선을 육안으로 확인함.

### 새 맵으로 저장하기

`unreal.EditorLevelLibrary`에는 "다른 이름으로 저장" API가 없음
(`save_current_level_as` 같은 함수 없음). 대신:
1. `unreal.EditorLevelLibrary.save_current_level()`로 일단 현재(수정된)
   상태를 `TrainingMap.umap`에 저장 — **원본을 임시로 덮어씀**
2. 셸에서 그 파일을 새 이름(`YonginCircuitMap.umap`)으로 복사
3. `TrainingMap.umap`은 git으로 추적되고 있으므로 (커밋 안 하고 그대로
   두었다면) `git checkout -- .../TrainingMap.umap`으로 원본 복원
4. 이후 새 맵 파일을 기준으로 계속 작업

## 4. 이후 진행 상황 (완료 및 남은 작업)

`floor` 리사이즈, `PlayerStart`/`StartFinishLine`/`Referee` 재배치,
Play-in-Editor 전체 랩 주행 테스트까지 전부 완료함. 다만
`fsds_ros2_bridge`로 실제 연결해서 확인한 결과 **`/testing_only/track`에
콘이 전부 나오지 않고 약 128개로 상한이 걸림** — 소스 스플라인의 콘
배열 자체는 완전한 상태(1278개)인데도 RPC 응답에서만 잘림. 정확한
원인은 찾지 못했고, 컨트롤러 로직 테스트에는 시각적으로 완전한 트랙이면
충분하다고 판단해 우선순위를 낮추고 보류함. 조사 시도 전체 내역은
[PROGRESS_UE4_BUILD.md](PROGRESS_UE4_BUILD.md)의 "미해결: 약 128개 콘
상한이 새 레벨에서도 재현됨" 절 참고.

남은 작업:
- `RunUAT.sh BuildCookRun`으로 커맨드라인 기반 스탠드얼론 빌드 패키징
  (에디터 없이 평소처럼 `FSDS.sh`류로 실행 가능하도록) — 보류, 지금은
  Play-in-Editor로 충분해서 컨트롤러 스켈레톤 작업이 우선
- 약 128개 콘 상한의 정확한 원인 재조사 — 보류
- `SETUP_DEBUG_LOG.md`, `SIMULATOR_GUIDE.md`에 새 빌드 절차와 새 맵 실행
  방법 반영 필요 여부 검토

진행 상황과 남은 작업의 최신 상태는 `PROGRESS_UE4_BUILD.md` 참고.
