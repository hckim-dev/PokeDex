# 개발·검증 가이드

[README](../README.md) · [아키텍처와 데이터 흐름](architecture.md)

## 실행 환경

| 항목 | 저장소 설정 |
|---|---|
| JDK | 21 (`settings.gradle.kts` 검사, Daemon JVM 기준과 JVM target) |
| Gradle | 9.4.1, 저장소 Wrapper 사용 |
| AGP | 9.1.1 |
| Android SDK | `compileSdk = 37`, `targetSdk = 37` |
| 최소 실행 버전 | API 26(Android 8.0) |
| SDK Build Tools | 명시적 override 없음. AGP 9.1 기본값 36.0.0 |
| IDE | AGP 9.1.1을 지원하는 Android Studio |

JDK 21은 이 프로젝트가 선택한 요구사항입니다. AGP 자체의 최소 JDK 요구사항과는 구분합니다. SDK Build Tools 기본값과 AGP의 호환 범위는 [공식 릴리스 문서](https://developer.android.com/build/releases/agp-9-1-0-release-notes)를 참고합니다.

관련 설정: [SDK·JVM target](../build-logic/convention/src/main/kotlin/com/hckim/pokedex/KotlinAndroid.kt), [targetSdk](../build-logic/convention/src/main/kotlin/AndroidApplicationConventionPlugin.kt), [Gradle Wrapper](../gradle/wrapper/gradle-wrapper.properties), [Daemon JVM](../gradle/gradle-daemon-jvm.properties)

## 로컬 설정

1. SDK Manager에서 SDK Platform 37, Build Tools 36.0.0, Platform Tools를 설치합니다.
2. Android Studio의 Gradle JDK와 CLI의 `JAVA_HOME`을 유효한 JDK 21 경로로 지정합니다.
3. Android Studio에서 SDK 경로를 설정해 루트 `local.properties`를 생성하거나 직접 작성합니다.

```properties
# Windows 예시: 자신의 SDK 설치 경로로 변경
sdk.dir=C:/Users/YOUR_USER/AppData/Local/Android/Sdk

# 선택 사항: 생략하면 아래 URL이 기본값으로 사용됨
BACKEND_URL=https://pokeapi.co/api/v2/
```

`local.properties`는 `.gitignore` 대상입니다. API URL은 따옴표 없이 적고 Retrofit base URL의 마지막 `/`를 유지합니다. 별도 인증 키는 현재 코드에서 사용하지 않습니다. 기본 URL 생성은 [core:network 빌드 설정](../core/network/build.gradle.kts)에서 확인할 수 있습니다.

## 빌드와 설치

프로젝트 루트의 PowerShell에서 실행합니다.

```powershell
.\gradlew.bat --version
.\gradlew.bat :app:assembleDebug
```

APK는 `app/build/outputs/apk/debug/app-debug.apk`에 생성됩니다. 기기나 에뮬레이터를 연결한 경우 다음 명령으로 설치할 수 있습니다.

```powershell
.\gradlew.bat :app:installDebug
```

macOS/Linux에서는 같은 task를 Wrapper로 실행합니다.

```bash
chmod +x gradlew
./gradlew :app:assembleDebug
```

`release`는 현재 R8 최적화가 기본 활성화되고 **debug signingConfig**를 사용합니다. 스토어 배포를 준비한 서명 구성은 아닙니다. 일반적인 로컬 확인에는 `debug`를 사용합니다.

## 자동 테스트

### 로컬 단위 테스트

```powershell
.\gradlew.bat :core:domain:test :core:data:testDebugUnitTest :feature:list:testDebugUnitTest :feature:detail:testDebugUnitTest :feature:favorites:testDebugUnitTest
```

| 위치 | 프로젝트 동작 테스트 수 | 확인하는 범위 |
|---|---:|---|
| [`core:domain`](../core/domain/src/test/kotlin/com/hckim/pokedex/core/domain/usecase) | 5 | 목록·상세·즐겨찾기 관련 Repository 호출과 반환값 전달 |
| [`core:data`](../core/data/src/test/kotlin/com/hckim/pokedex/core/data/repository/DefaultPokemonRepositoryTest.kt) | 4 | 즐겨찾기 삽입·삭제 분기, ID·등록 여부 Flow 전달 |
| [`feature:list`](../feature/list/src/test/kotlin/com/hckim/pokedex/feature/list/PokemonListViewModelTest.kt) | 5 | 검색어·선택 타입·즐겨찾기 ID 상태, toggle 호출, 상세 이동 Effect |
| [`feature:detail`](../feature/detail/src/test/kotlin/com/hckim/pokedex/feature/detail/PokemonDetailViewModelTest.kt) | 2 | 상세 조회 성공과 실패 시 상태 |
| [`feature:favorites`](../feature/favorites/src/test/kotlin/com/hckim/pokedex/feature/favorites/FavoriteListViewModelTest.kt) | 3 | 즐겨찾기 ID 상태, toggle 호출, 상세 이동 Effect |

JUnit 4·MockK·Turbine·`kotlinx-coroutines-test`를 사용합니다. ViewModel 테스트에서는 `Dispatchers.Main`을 테스트 Dispatcher로 바꿉니다. 현재 목록 테스트는 검색 조건 상태를 확인하며, 실제 PagingData 내용이나 `flatMapLatest` 취소 동작까지 검증하지는 않습니다. UseCase는 6개 중 5개에 테스트가 있으며 `GetFavoritePokemonListUseCase`의 개별 테스트는 없습니다.

로컬 테스트의 HTML 보고서는 JVM 모듈의 `build/reports/tests/test/index.html`, Android 모듈의 `build/reports/tests/testDebugUnitTest/index.html`에 생성됩니다.

### Room 계측 테스트

기기 또는 에뮬레이터가 필요합니다. [PokemonDaoTest](../core/database/src/androidTest/kotlin/com/hckim/pokedex/core/database/dao/PokemonDaoTest.kt)는 in-memory Room DB에서 삽입·조회와 즐겨찾기 추가·삭제 2개 시나리오를 확인합니다.

```powershell
.\gradlew.bat :core:database:connectedDebugAndroidTest
```

`app`의 컨텍스트 테스트와 `core:designsystem`의 덧셈 테스트는 기본 예제이며 위의 프로젝트 동작 테스트 수에는 포함하지 않았습니다. MockWebServer 의존성은 선언되어 있으나 현재 네트워크 동작 테스트는 없습니다. RemoteMediator, Migration, Compose UI 및 전체 화면 흐름 테스트도 추가 과제입니다. 저장소에 CI 워크플로는 없습니다.

### 검증 기록

2026-10-23(KST), Windows x64·JDK 21·Gradle 9.4.1 환경에서 아래 명령을 실행했습니다. 로컬 테스트 수는 Gradle의 JUnit XML 결과를 기준으로 집계했습니다.

```powershell
.\gradlew.bat :app:assembleDebug :core:domain:test :core:data:testDebugUnitTest :feature:list:testDebugUnitTest :feature:detail:testDebugUnitTest :feature:favorites:testDebugUnitTest --console=plain
```

| 항목 | 결과 |
|---|---|
| `:app:assembleDebug` | 성공, `app-debug.apk` 생성 |
| 위의 5개 모듈 단위 테스트 task | 19개 통과, 실패·오류·건너뜀 0개 |
| 기본 `BACKEND_URL` 생성 | `https://pokeapi.co/api/v2/` 확인 |
| Room 계측 테스트·기기 UI 동작 | 이번 검증에서 실행하지 않음 |

빌드 성공과 단위 테스트 통과는 실제 기기에서의 네트워크·오프라인·화면 동작 검증을 대신하지 않습니다.

## 수동 확인 순서

아래 항목은 자동 테스트가 다루지 않는 동작을 확인하는 절차입니다. 실제 수행 결과와는 구분합니다.

1. 온라인에서 목록을 열고 스크롤해 다음 페이지가 추가되는지 확인합니다.
2. 첫 페이지 로딩 후 `bulba`를 검색하고 Grass 필터를 적용해 Bulbasaur를 확인합니다. 검색어를 지우고 All을 선택하면 전체 목록으로 돌아갑니다. 아직 저장하지 않은 포켓몬은 검색 대상이 아닙니다.
3. 카드에서 상세로 이동해 타입·능력치·키·몸무게·특성을 확인한 뒤 돌아옵니다.
4. 목록에서 즐겨찾기를 추가하고 즐겨찾기 화면에서 해제해 두 화면의 하트 상태를 확인합니다.
5. 네트워크를 끄고 저장된 상세와 로컬 검색을 확인합니다. 새 페이지와 미캐시 이미지는 로딩되지 않을 수 있습니다.
6. 온라인으로 돌아와 목록의 초기 갱신 전후를 비교합니다. 테스트하려면 앱을 종료한 뒤 다시 실행해 새 Pager의 초기 갱신을 유도합니다. 현재 갱신 시의 즐겨찾기 표시 제약은 [아키텍처 문서](architecture.md)에 설명했습니다.

## 환경 문제 확인

### CLI에서 Java는 실행되지만 Wrapper가 실패하는 경우

`java -version`과 Wrapper는 서로 다른 Java 설치를 가리킬 수 있습니다. Wrapper는 `JAVA_HOME`을 우선 사용하므로 삭제된 JDK 경로가 남아 있는지 확인합니다.

```powershell
$env:JAVA_HOME
Test-Path "$env:JAVA_HOME/bin/java.exe"
java -version
.\gradlew.bat --version
```

현재 PowerShell 세션에만 적용하려면 실제 설치 경로를 지정합니다.

```powershell
$env:JAVA_HOME = 'C:\path\to\jdk-21'
.\gradlew.bat --version
```

### SDK 또는 API URL 관련 오류

`local.properties`의 `sdk.dir`와 설치된 SDK Platform 37을 확인합니다. 네트워크 초기화에 실패하면 `BACKEND_URL` 오타·따옴표·마지막 `/`도 확인합니다.

### 현재 빌드에서 확인한 경고

- **`androidTestImplementation dependencies are ignored because androidTest is disabled`**: Convention Plugin이 `src/androidTest`가 없는 라이브러리의 계측 테스트를 비활성화하지만 공통 설정이나 개별 모듈에 계측 테스트 의존성이 남아 있어 발생합니다. 의존성 선언과 variant 활성화 조건을 함께 정리할 대상입니다.
- **`ExperimentalCoroutinesApi` opt-in marker unresolved**: 공통 Kotlin 설정이 Coroutines에 의존하지 않는 `core:model`에도 Coroutines opt-in 인자를 추가합니다. 해당 compiler option을 필요한 모듈에만 적용하도록 좁힐 수 있습니다.
