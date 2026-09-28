# Hexa Merge WebGL/테스트/워크플로우 상세 분석

**작성일**: 2026-02-23
**분석 대상**: Assets/WebGLTemplates, Assets/Plugins, tests/, workflow.md, docs/qa_20260222.txt
**프로젝트 구조**: Unity → WebGL 빌드 → Playwright E2E 테스트

---

## 1. WebGL 템플릿 구조 (Assets/WebGLTemplates/HexaMerge/)

### 1.1 파일 구조
```
Assets/WebGLTemplates/HexaMerge/
├── index.html          (메인 HTML 템플릿)
├── index.html.meta     (Unity 메타)
├── thumbnail.png       (템플릿 썸네일)
└── thumbnail.png.meta
```

### 1.2 index.html 상세 분석

#### 메타 태그 & 초기화
```html
<meta charset="utf-8">
<meta http-equiv="Content-Type" content="text/html; charset=utf-8">
<meta name="viewport" content="width=device-width, initial-scale=1.0, user-scalable=no">
```
- **언어**: 한국어 (lang="ko")
- **뷰포트**: 모바일 비활성화 (user-scalable=no)
- **배경**: 검은색 (#000)

#### CSS 스타일링
| 요소 | 목적 | 스타일 |
|------|------|--------|
| `#unity-container` | 캔버스 래퍼 | flex center, 100% w/h |
| `#loading-screen` | 로딩 화면 | 고정 오버레이, 페이드 아웃 가능 |
| `#loading-bar` | 진행률 표시 | 그라데이션 (FF69B4→FF1493) |
| `#warning-banner` | 오류 배너 | 빨강, 숨김 기본값, 5초 타임아웃 |

#### JavaScript 초기화 로직

**1. 전역 객체 설정**
```javascript
window.__adsRemovedState = false;  // IAP 광고 제거 상태
window.gamebridge = {              // 플랫폼 호환성 객체
  isAdsRemoved: () => window.__adsRemovedState,
  isBannerReady: () => true,
  isRewardedReady: () => true
};
```

**2. Unity↔JS 메시지 브리지**
```javascript
window.unityMessageCallbacks = {};  // 콜백 저장소

window.SendMessageFromUnity(message) {
  // 1. JSON 파싱
  // 2. callbackId 기반 콜백 실행 (1회용)
  // 3. adsRemoved/bannerHidden 이벤트 처리
  // 4. CustomEvent 발송 (Playwright 테스트용)
}

window.sendToUnity(objectName, methodName, value)  // JS→Unity
window.getGameState()                               // Promise 기반 상태 조회
```

**3. Unity 로더**
```javascript
const config = {
  dataUrl, frameworkUrl, codeUrl, memoryUrl, symbolsUrl,  // 빌드 아티팩트
  streamingAssetsUrl,
  companyName, productName, productVersion,
  showBanner: function(msg, type)  // 에러 배너
}
```

**4. 로딩 진행률 콜백**
- 진행률 0~100% 업데이트
- 로딩 완료 후 loading-screen 페이드 아웃
- 오류 발생 시 빨강 텍스트 표시

#### 에러 처리 (TC-PLAT-035)
```javascript
window.onerror = function(message, source, lineno, colno, error)
```
- 전역 미처리 에러 포착
- 콘솔 로깅
- 배너 표시는 하지 않음 (명시적 호출만)

### 1.3 주요 특징
| 항목 | 값 |
|------|-----|
| **charset** | UTF-8 |
| **언어** | 한국어 |
| **로딩 UI** | 진행바 + 텍스트 |
| **로딩 색상** | Pink/Magenta |
| **Canvas ID** | unity-canvas |
| **IAP 지원** | gamebridge 객체 |
| **테스트 지원** | CustomEvent 방식 |

---

## 2. JS 플러그인 & 브릿지 코드 (Assets/Plugins/WebGL/)

### 2.1 파일 구조
```
Assets/Plugins/WebGL/
├── HexaMergeBridge.jslib       (C# P/Invoke 플러그인)
└── HexaMergeBridge.jslib.meta
```

### 2.2 HexaMergeBridge.jslib 상세

#### 함수 1: SendMessageToJS
**목적**: Unity → JS 메시지 전달

```javascript
SendMessageToJS: function(messagePtr) {
    var message = UTF8ToString(messagePtr);

    // 3가지 채널로 전송
    1. window.SendMessageFromUnity(message)      // 템플릿 콜백
    2. window.onUnityMessage(JSON.parse(message)) // 레거시 호환
    3. window.dispatchEvent(CustomEvent)          // Playwright 테스트
}
```

**마샬링**: UTF-8 포인터 → 문자열 변환
**이벤트명**: 'unityMessage'
**에러 처리**: JSON 파싱 실패 시 경고만 출력

#### 함수 2: SetWindowProperty
**목적**: Unity에서 JS window 객체에 속성 설정

```javascript
SetWindowProperty: function(keyPtr, valuePtr) {
    var key = UTF8ToString(keyPtr);
    var value = UTF8ToString(valuePtr);
    try {
        window[key] = JSON.parse(value);  // JSON 파싱 시도
    } catch(e) {
        window[key] = value;               // 실패 시 문자열로 저장
    }
}
```

**용도**: 게임 설정, 사용자 정보 등 동적 저장

#### 함수 3: CallWindowCallback
**목적**: Unity에서 JS 함수 호출

```javascript
CallWindowCallback: function(callbackNamePtr, valuePtr) {
    var callbackName = UTF8ToString(callbackNamePtr);
    var value = UTF8ToString(valuePtr);
    if (typeof window[callbackName] === 'function') {
        window[callbackName](value);
    }
}
```

**안전성**: 함수 존재 여부 확인 후 호출

#### 함수 4: RegisterHexaTestAPI
**목적**: Playwright 테스트용 API 등록

```javascript
RegisterHexaTestAPI: function() {
    window.HexaTest = {
        triggerSpawnAnimation(count),
        triggerMerge(q1, r1, q2, r2),
        triggerCombo(count),
        triggerWaveAnimation(direction),
        triggerScreenTransition(from, to),
        isAnimationPlaying(),
        getBlockScale(q, r),
        getBlockAlpha(q, r),
        getAnimationState(),
        getFPS(),
        setBoardState(stateJson)
    }
}
```

**콜백 시스템**:
- ID 기반 Promise 패턴
- Unity → JS CustomEvent('unityMessage')로 응답
- `__hexaTestCallback` 플래그로 테스트 콜백 식별

### 2.3 마샬링 패턴
| 타입 | 전달 방식 | 예시 |
|------|---------|------|
| 문자열 | UTF8ToString(ptr) | "message" |
| JSON | JSON.parse(str) | {q:0, r:1, v:2} |
| 좌표 | "q,r" 구분 | "0,1" |
| 애니메이션 ID | "id\|param" 구분 | "ht_123\|param" |

---

## 3. 빌드 헬퍼 (Assets/Editor/BuildHelper.cs)

### 3.1 구조
```
BuildHelper
├── WebGL 빌드
│   ├── BuildWebGLDev()      → BuildWebGL(true)
│   └── BuildWebGLRelease()  → BuildWebGL(false)
├── Android 빌드
│   ├── BuildAndroidAPKDev()
│   ├── BuildAndroidAPKRelease()
│   └── BuildAndroidAABRelease()
└── 유틸리티
    └── OpenBuildFolder()
```

### 3.2 WebGL 빌드 설정

**경로**: `Build/WebGL/`

**빌드 설정값**:
```csharp
PlayerSettings.WebGL.compressionFormat = WebGLCompressionFormat.Disabled;  // 정적 호스팅용
PlayerSettings.WebGL.dataCaching = true;                                   // 데이터 캐싱
PlayerSettings.WebGL.linkerTarget = WebGLLinkerTarget.Wasm;                // Wasm 대상
PlayerSettings.WebGL.exceptionSupport = WebGLExceptionSupport.None;        // 예외 미지원
PlayerSettings.WebGL.template = "PROJECT:HexaMerge";                       // 커스텀 템플릿
PlayerSettings.stripEngineCode = true;                                     // 코드 스트리핑
```

**Development vs Release**:
| 설정 | Development | Release |
|------|-------------|---------|
| 디버그 심볼 | O | X |
| 프로파일러 연결 | O | X |
| 빌드 시간 | 빠름 | 느림 |
| 파일 크기 | 큼 | 작음 |

**호출 전 처리**:
```csharp
SceneSetup.SetupGameScene();  // 씬 직렬화 동기화
```

**빌드 후**:
- 성공 메시지 + 크기 출력 (MB 단위)
- 실패 메시지 + 결과 코드 출력

### 3.3 Android 빌드 설정
- **API 레벨**: 22 (최소) ~ 33 (대상)
- **스크립팅 백엔드**: IL2CPP
- **아키텍처**: ARM64
- **AAB/APK**: 선택식

---

## 4. 테스트 인프라 (tests/)

### 4.1 디렉토리 구조
```
tests/
├── package.json              (npm 메타데이터)
├── package-lock.json         (의존성 고정)
├── playwright.config.ts      (테스트 설정)
├── helpers/
│   └── unity-bridge.ts       (Unity↔JS 헬퍼)
├── specs/
│   ├── ad-reward.spec.ts           (광고 리워드)
│   ├── animation.spec.ts           (애니메이션 + 스냅샷)
│   ├── animation.spec.ts-snapshots/ (스냅샷 이미지)
│   ├── audio.spec.ts               (오디오)
│   ├── game-state.spec.ts          (게임 상태)
│   ├── hex-grid.spec.ts            (그리드 구조)
│   ├── iap.spec.ts                 (인앱 구매)
│   ├── merge-system.spec.ts        (머지 로직)
│   ├── platform.spec.ts            (플랫폼 호환성)
│   ├── scoring.spec.ts             (점수 계산)
│   ├── ui-components.spec.ts       (UI + 스냅샷)
│   └── ui-components.spec.ts-snapshots/
└── helpers/                  (744 KB 전체)
```

### 4.2 package.json
```json
{
  "name": "hexa-merge-e2e-tests",
  "version": "1.0.0",
  "scripts": {
    "test": "npx playwright test",
    "test:headed": "npx playwright test --headed",
    "test:debug": "npx playwright test --debug",
    "test:ui": "npx playwright test --ui",
    "test:hex-grid": "npx playwright test specs/hex-grid.spec.ts",
    "test:merge": "npx playwright test specs/merge-system.spec.ts",
    "test:scoring": "npx playwright test specs/scoring.spec.ts",
    "test:game-state": "npx playwright test specs/game-state.spec.ts",
    "serve": "npx http-server ../Build/WebGL -p 8080 --cors -c-1",
    "report": "npx playwright show-report"
  }
}
```

**의존성**:
- `@playwright/test@^1.49.0`
- `http-server@^14.1.1`
- `typescript@^5.5.0`

### 4.3 Playwright 설정

**파일**: playwright.config.ts

**주요 설정값**:
```typescript
testDir: './specs'                           // 테스트 위치
timeout: 120_000                             // 각 테스트 타임아웃 (2분)
expect: { timeout: 30_000 }                  // expect 타임아웃 (30초)
retries: 1                                   // 자동 재시도 1회
fullyParallel: false, workers: 1             // 순차 실행 (Unity WebGL 제약)
baseURL: 'http://localhost:8080'             // 개발 서버
```

**WebGL 설정**:
```typescript
launchOptions: {
  args: ['--use-gl=angle', '--use-angle=swiftshader']  // GPU 가속 활성화
}
```

**리포팅**:
```typescript
reporter: ['html', 'list']                   // HTML + 콘솔 리포트
screenshot: 'only-on-failure'                // 실패 시만 스크린샷
trace: 'on-first-retry'                      // 첫 재시도 시 트레이스
video: 'on-first-retry'                      // 첫 재시도 시 비디오
```

**웹 서버**:
```typescript
webServer: {
  command: 'npx http-server ../Build/WebGL -p 8080 --cors -c-1',
  port: 8080,
  timeout: 30_000,
  reuseExistingServer: true  // 기존 서버 재사용
}
```

### 4.4 Unity Bridge 헬퍼 (helpers/unity-bridge.ts)

#### 타입 정의
```typescript
interface CellInfo {
  q: number;
  r: number;
  v: number;  // 타일 값 (0=빈 셀)
}

interface GameStatePayload {
  callbackId: string;
  state: 'Ready' | 'Playing' | 'Paused' | 'GameOver';
  score: number;
  highScore: number;
  cells: CellInfo[];  // 항상 25개
}

type UnityEvent =
  | { event: 'stateChanged'; state: string }
  | { event: 'merge'; value: number; count: number; score: number }
  | { event: 'scoreChanged'; score: number }
```

#### 핵심 메서드

| 메서드 | 목적 | 반환값 |
|--------|------|--------|
| `waitForUnityLoad()` | Unity 인스턴스 대기 | void |
| `loadAndStartGame()` | 로드 + 게임 시작 | void |
| `sendMessage(obj, method, value)` | JS→Unity | void |
| `getGameState()` | 게임 상태 조회 | Promise<GameStatePayload> |
| `tapCell(q, r)` | 셀 탭 시뮬레이션 | void |
| `waitForEvent(name, predicate, timeout)` | 이벤트 대기 | Promise<UnityEvent> |
| `getCollectedEvents()` | 수집된 이벤트 조회 | Promise<UnityEvent[]> |
| `getNonEmptyCells()` | 비어있지 않은 셀 | Promise<CellInfo[]> |
| `getCurrentScore()` | 현재 점수 | Promise<number> |

#### 좌표 유틸리티
```typescript
getAllGridCoords(): Array<{q, r}>
// 25셀 다이아몬드 그리드: 1-2-3-4-5-4-3-2-1
// r: -4~+4 (9개 행)
// 행별 q 범위 동적 계산
```

#### 이벤트 수집 버퍼
```typescript
window.__unityEvents = []  // 테스트 중 발생한 모든 이벤트 저장
```

### 4.5 테스트 커버리지

#### hex-grid.spec.ts (25셀 구조)
- [x] Unity WebGL 로드 검증
- [x] 25개 셀 존재 확인
- [x] 다이아몬드 좌표 (1-2-3-4-5-4-3-2-1) 검증
- [x] 초기 타일 값 유효성 (2, 4, 8, 16)
- [x] 초기 보드 완전 채움 확인

#### game-state.spec.ts (상태 관리)
- [x] StartNewGame → Playing 전환
- [x] 새 게임 시 점수 0 초기화
- [x] 보드 25셀 타일 배치
- [x] 재시작 시 상태 초기화
- [x] GameStatePayload 필드 검증
- [x] highScore 정수 검증

#### merge-system.spec.ts (머지 로직)
- [x] 빈 셀 탭 → 머지 없음
- [x] 단독 셀 탭 → 머지 없음
- [x] 인접 같은 값 탭 → 머지 성공
- [x] 머지 후 셀 제거 확인
- [x] 머지 후 보드 상태 일관성

#### scoring.spec.ts (점수 계산)
- [x] 새 게임 점수 0
- [x] 머지 시 점수 증가
- [x] scoreChanged 이벤트 발생
- [x] 점수 누적 검증
- [x] 콤보 점수 배수

#### animation.spec.ts (애니메이션 + 스냅샷)
- [x] 머지 애니메이션 재생
- [x] 콤보 애니메이션 (x2, x4, x5 글로우)
- [x] 큰 점수 팝업
- [x] 스냅샷 비교 (4개 이미지)

#### audio.spec.ts (사운드)
- [x] 게임 시작 음량
- [x] 버튼 클릭 음량
- [x] 타일 드롭 음량
- [x] 머지 음량 및 지속 시간
- [x] 게임 오버 글리산도
- [x] CrystalNote 노이즈 + 서브톤

#### ui-components.spec.ts (UI + 스냅샷)
- [x] 메뉴 버튼 표시
- [x] 게임 오버 버튼
- [x] 확인 팝업 모달
- [x] 버튼 클릭 이벤트
- [x] 스냅샷 비교 (2개 이미지)

#### iap.spec.ts (인앱 구매)
- [x] gamebridge 객체 존재
- [x] isAdsRemoved() 호출
- [x] 광고 제거 상태 변경
- [x] 광고 배너 준비 상태

#### ad-reward.spec.ts (광고 리워드)
- [x] 보상 광고 준비 상태
- [x] 광고 시청 후 보상
- [x] 보상 점수 적용

#### platform.spec.ts (플랫폼 호환성)
- [x] Chromium WebGL 렌더링
- [x] 뷰포트 1280×720
- [x] 로딩 진행률 표시
- [x] 게임 로드 완료

### 4.6 실행 워크플로우
```bash
# 1. WebGL 빌드
npm run serve           # http-server 시작 (8080 포트)

# 2. 테스트 실행 (별도 터미널)
npm test                # 전체 테스트 (headless)
npm test:hex-grid       # 그리드만 테스트
npm test:merge          # 머지만 테스트
npm test -- --headed    # UI 보며 테스트

# 3. 결과 확인
npm run report          # HTML 리포트 보기
```

### 4.7 테스트 신뢰성
| 요소 | 설정 | 목적 |
|------|------|------|
| 타임아웃 | 120초 | SwiftShader 저FPS 감안 |
| 재시도 | 1회 | 플리키 테스트 감지 |
| 단일 워커 | 1개 | Unity 동시 접속 불가 |
| 리사이즈 | 1280×720 | 표준 데스크톱 뷰포트 |

---

## 5. 리소스 에셋 목록 (Assets/Resources, ScriptableObjects, Materials, Sprites)

### 5.1 폰트 (Assets/Resources/Fonts/)
```
Fonts/
└── NunitoExtraBold.ttf  (한글 표시용)
```

**용도**: UI 텍스트 렌더링
**가중치**: Extra Bold (900)
**지원**: 한글/영문

### 5.2 ScriptableObjects (Assets/ScriptableObjects/)
```
ScriptableObjects/
└── TileColorConfig.asset  (타일 색상 매핑)
```

**포함 내용**:
- 타일 값별 색상 정의 (2, 4, 8, 16, ...)
- RGB 값 저장
- Serialized JSON 형식

**예시 구조** (추정):
```json
{
  "colors": {
    "2": "#FF69B4",
    "4": "#FF1493",
    "8": "#...",
    ...
  }
}
```

### 5.3 Materials (Assets/Materials/)
**현황**: 비어있음 (초기 상태)
**예상 용도**: 향후 셰이더/텍스처 매터리얼

### 5.4 Sprites (Assets/Sprites/)
**현황**: 비어있음 (초기 상태)
**예상 용도**: UI 버튼, 아이콘, 왕관 등

### 5.5 웹 재구현 시 활용 가능한 부분
| 리소스 | 재사용 가능 | 비고 |
|--------|-----------|------|
| Nunito 폰트 | ✓ | 웹 폰트로 직접 활용 |
| 타일 색상 설정 | ✓ | JSON 추출하여 CSS 변수화 |
| 게임 로직 | ✗ | Unity C#→JavaScript 재구현 필요 |
| UI 에셋 | △ | PNG/Sprite 추출 후 웹 최적화 |

---

## 6. 워크플로우 및 QA 항목 (workflow.md, qa_20260222.txt)

### 6.1 workflow.md 개요

**파일 크기**: 252 bytes
**인코딩**: 가능한 인코딩 이슈 (한글 깨짐)
**주요 내용** (추정):
1. 게임 로직 설계 및 분석
2. 기능별 기술 분석
3. UI 제작 및 테스트
4. 성능 최적화
5. 메모리 최적화 및 추가 기능
6. 웹 배포 및 CI/CD
7. 추가 개선 사항

### 6.2 QA 체크리스트 (qa_20260222.txt)

**작성일**: 2026-02-22
**완료율**: 13/17 (76%)

#### 사운드 퀄리티 개선 (8/8 완료)
| 항목 | 상태 | 내용 |
|------|------|------|
| 1 | ✓ | CrystalNote 노이즈 트랜지언트 (+3% 밴드패스, 0.35 강도) |
| 2 | ✓ | CrystalNote 서브톤 (0.5x 주파수, 0.12 강도) |
| 3 | ✓ | 음량 계층화: 탭(0.5) < 드롭(0.55) < 머지(0.8~0.9) < 오버(1.0) |
| 4 | ✓ | 지속시간: 머지 0.40→0.20s, 오버 0.60→1.2s, 탭 0.10→0.07s |
| 5 | ✓ | 게임오버 3단계 글리산도 (E5→C4→C3, 1.2초) |
| 6 | ✓ | 게임시작 0.35, 버튼 0.45 음량 |
| 7 | ✓ | 깊이그룹 간격 0.17→0.15s, NumberUp 0.17→0.14s |
| 8 | ✓ | WebGL Release + gh-pages 배포 (ca82906) |

#### 왕관 디자인 개선 (5/5 완료)
| 항목 | 상태 | 내용 |
|------|------|------|
| 9 | ✓ | 벤치마크 450장 + 스크린샷 분석 (에이전트팀 병렬) |
| 10 | ✓ | 5피크 왕관 (3피크→5피크, V자 밸리 + 원형 팁) |
| 11 | ✓ | 텍스처 56×40→80×60px (고해상도) |
| 12 | ✓ | 표시 28×20→32×24px, Y offset 24→28px |
| 13 | ✓ | WebGL Release + gh-pages 배포 (f35f7bc) |

#### 미완료 항목 (0/4)
| 항목 | 상태 | 내용 |
|------|------|------|
| 14 | [ ] | 폰트 스타일 분위기 개선 |
| 15 | [ ] | 모바일 브라우저 메뉴 UI 크기/겹침 수정 |
| 16 | [ ] | 메뉴 아이콘 디자인 개선 |
| 17 | [ ] | 메뉴 아이콘 기능 실제 구현 |

**배포 기록**:
- ca82906: 사운드 최적화 후 배포
- f35f7bc: 왕관 디자인 변경 후 배포

**진행 방식**: 에이전트팀 설계 검토 → 구현 → 테스트 → 수정 → 메모리 업데이트 → 원격 배포

---

## 7. 핵심 아키텍처 다이어그램

### 7.1 런타임 메시지 흐름

```
┌─────────────────────────────────────────────────────────┐
│                    User (Browser)                       │
│           (Playwright 테스트 또는 사용자)                 │
└────────────────┬────────────────────────────────────────┘
                 │
                 │ HTML + JS
                 ↓
┌─────────────────────────────────────────────────────────┐
│              index.html (템플릿)                         │
│  ┌───────────────────────────────────────────────────┐  │
│  │ • Loading Screen (진행률)                         │  │
│  │ • Unity Canvas                                   │  │
│  │ • gamebridge 객체                                │  │
│  │ • SendMessageFromUnity 콜백                      │  │
│  │ • unityMessage CustomEvent 수신                  │  │
│  └───────────────────────────────────────────────────┘  │
└────┬──────────────────────────────────────┬─────────────┘
     │                                      │
     │ window.unityInstance.SendMessage()   │ Unity → JS
     │                                      │
     ↓                                      ↓
┌─────────────────────────────────────────────────────────┐
│              Unity WebGL (Wasm)                         │
│  ┌───────────────────────────────────────────────────┐  │
│  │ • GameManager (WebGLBridge)                      │  │
│  │ • GameplayController                            │  │
│  │ • HexGrid, MergeSystem                           │  │
│  │ • AudioManager, AnimationManager                 │  │
│  └───────────────────────────────────────────────────┘  │
└────┬──────────────────────────────────────┬─────────────┘
     │ JS_TapCell(q,r)                      │
     │ JS_StartNewGame()                    │ HexaMergeBridge.jslib
     │ JS_GetGameState(callbackId)          │ SendMessageToJS()
     │                                      │
     ↓                                      ↓
┌─────────────────────────────────────────────────────────┐
│           HexaMergeBridge.jslib                         │
│  ┌───────────────────────────────────────────────────┐  │
│  │ • SendMessageToJS (JSON 파싱)                     │  │
│  │ • SetWindowProperty (동적 저장)                   │  │
│  │ • CallWindowCallback (함수 호출)                  │  │
│  │ • RegisterHexaTestAPI (테스트 API)                │  │
│  └───────────────────────────────────────────────────┘  │
└────┬──────────────────────────────────────────────────────┘
     │
     │ CustomEvent('unityMessage', { detail: data })
     │
     ↓
┌─────────────────────────────────────────────────────────┐
│            Playwright Test Harness                      │
│  ┌───────────────────────────────────────────────────┐  │
│  │ • UnityBridge Helper                             │  │
│  │ • Event Listener: window.addEventListener()      │  │
│  │ • Assertion: expect(...)                         │  │
│  └───────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────┘
```

### 7.2 빌드 파이프라인

```
┌──────────────────────────┐
│  Unity Editor (C#)       │
│  • GameScene.unity       │
│  • 37개 C# 스크립트      │
│  • Assets/Resources      │
│  • Assets/Materials      │
└────────┬─────────────────┘
         │
         │ BuildHelper.BuildWebGL()
         │
         ↓
┌──────────────────────────────────────┐
│  빌드 전 처리                        │
│  • SceneSetup.SetupGameScene()       │
│  • 씬 직렬화 동기화                  │
└────────┬─────────────────────────────┘
         │
         ↓
┌──────────────────────────────────────┐
│  Unity Build Pipeline                │
│  • IL2CPP 컴파일                     │
│  • 코드 스트리핑                     │
│  • Wasm 생성                         │
└────────┬─────────────────────────────┘
         │
         ↓
┌──────────────────────────────────────┐
│  Build/WebGL/ 출력                   │
│  ├── Build/*.wasm                    │
│  ├── Build/*.data                    │
│  ├── Build/*.js                      │
│  └── index.html (HexaMerge 템플릿)  │
└────────┬─────────────────────────────┘
         │
         │ http-server (localhost:8080)
         │
         ↓
┌──────────────────────────────────────┐
│  Playwright E2E 테스트               │
│  • 자동 로드                         │
│  • getGameState() 폴링               │
│  • 스냅샷 비교                       │
│  • 리포트 생성                       │
└──────────────────────────────────────┘
```

### 7.3 좌표계 및 25셀 레이아웃

```
큐브 좌표 (q, r) 기반 육각형 그리드

행(r)별 구성:
  r=-4:           (0,-4)                           [1셀]
  r=-3:        (-1,-3) (0,-3)                      [2셀]
  r=-2:     (-1,-2) (0,-2) (1,-2)                 [3셀]
  r=-1:  (-2,-1) (-1,-1) (0,-1) (1,-1)            [4셀]
  r=0: (-2,0) (-1,0) (0,0) (1,0) (2,0)           [5셀]  ← 중심
  r=1:  (-1,1) (0,1) (1,1) (2,1)                  [4셀]
  r=2:     (0,2) (1,2) (2,2)                      [3셀]
  r=3:        (0,3) (1,3)                         [2셀]
  r=4:           (0,4)                            [1셀]

총 25셀 = 1+2+3+4+5+4+3+2+1

6방향 인접 오프셋:
  (+1,-1), (+1,0), (0,+1), (-1,+1), (-1,0), (0,-1)
```

---

## 8. 웹 순수 버전 재구현 시 활용 가능한 부분

### 8.1 활용 가능 (권장)
| 항목 | 파일 | 방식 |
|------|------|------|
| **템플릿 구조** | index.html | 직접 사용 가능 (수정 최소화) |
| **메시지 프로토콜** | HexaMergeBridge.jslib 로직 | JS로 재구현 |
| **테스트 프레임워크** | playwright.config.ts | 그대로 사용 가능 |
| **좌표계** | unity-bridge.ts `getAllGridCoords()` | 복사 후 사용 |
| **이벤트 시스템** | CustomEvent 패턴 | 동일하게 구현 |
| **폰트** | Nunito Extra Bold | 웹 폰트 사용 |
| **색상 설정** | TileColorConfig.asset | JSON 추출 |

### 8.2 재구현 필요
| 항목 | 사유 |
|------|------|
| **게임 로직** | Unity C# → JavaScript (HexGrid, MergeSystem 등) |
| **애니메이션** | Unity Animator → HTML5/CSS3 또는 Three.js |
| **오디오** | Unity AudioClip → Web Audio API |
| **렌더링** | Unity Graphics → Canvas 2D 또는 WebGL |
| **저장소** | Unity PlayerPrefs → localStorage/IndexedDB |

### 8.3 권장 재구현 순서
```
1. 메시지 프로토콜 재정의 (JSON 구조 통일)
2. 게임 상태 관리 (GameState, CellInfo)
3. 좌표계 및 HexGrid 로직
4. 머지 시스템 (MergeSystem.cs 로직 포팅)
5. 렌더링 엔진 (Canvas 2D 또는 WebGL)
6. 애니메이션 시스템
7. 오디오 시스템
8. UI 및 인터랙션
9. 테스트 마이그레이션 (기존 스펙 활용)
```

---

## 9. 빌드 & 배포 체크리스트

### 9.1 로컬 개발
```bash
# 1. WebGL 빌드
Unity Editor → HexaMerge > Build > WebGL (Development)

# 2. 테스트 서버 실행
cd tests
npm run serve

# 3. 테스트 실행 (다른 터미널)
npm test
npm test:hex-grid  # 특정 테스트
npm test -- --headed --debug  # UI 보며 디버깅

# 4. 결과 확인
npm run report
```

### 9.2 배포 (gh-pages)
```bash
# 1. Release 빌드
Unity Editor → HexaMerge > Build > WebGL (Release)

# 2. 빌드 폴더 검증
ls Build/WebGL/Build/
# → .wasm, .data, .js 파일 확인

# 3. gh-pages 브랜치에 배포
git add Build/WebGL/
git commit -m "feat: WebGL 빌드 배포"
git push origin main  # main에서 gh-pages 자동 배포 설정 필요

# 4. 배포 확인
https://<github-user>.github.io/<repo>/
```

### 9.3 성능 최적화 항목
| 항목 | 현재 설정 | 개선 방안 |
|------|---------|---------|
| 압축 | 비활성화 | gzip 활성화 (정적 호스팅 호환) |
| 캐싱 | 활성화 | Service Worker 추가 |
| 메모리 | 기본값 | 프로파일링 후 최적화 |
| 렌더링 | WebGL | CPU 부하 모니터링 |

---

## 10. 주요 발견사항 및 권장사항

### 10.1 강점
✓ **일관된 메시지 프로토콜**: JSON 기반 단방향/양방향 통신
✓ **포괄적 테스트**: 11개 spec 파일, 게임로직 전반 커버
✓ **표준화된 빌드**: Unity BuildHelper로 일관된 빌드 프로세스
✓ **명확한 좌표계**: 25셀 다이아몬드 구조 잘 정의
✓ **온전한 QA 프로세스**: 체크리스트 기반 완료 추적

### 10.2 개선 가능 영역
△ **workflow.md 인코딩**: 한글 텍스트 깨짐 문제
△ **Materials/Sprites**: 아직 미활용 디렉토리
△ **미완료 QA**: 폰트, 모바일 UI, 메뉴 아이콘 (4개)
△ **테스트 속도**: 120초 타임아웃으로 인한 느린 실행

### 10.3 웹 버전 구현 시 주의사항
1. **메시지 프로토콜**: index.html의 SendMessageFromUnity 로직 참고
2. **좌표 매핑**: HexCoord(q, r) 시스템 정확히 구현
3. **이벤트 순서**: merge → scoreChanged → stateChanged 순서 보장
4. **타임아웃**: 네트워크/렌더링 지연 감안 (30초 이상)
5. **브라우저 호환**: SwiftShader 테스트로 저사양 디바이스 검증

---

## 11. 파일 크기 요약

| 경로 | 크기 | 파일 수 |
|------|------|--------|
| Assets/WebGLTemplates/ | 11 KB | 4개 |
| Assets/Plugins/WebGL/ | 6 KB | 2개 |
| tests/ | 744 KB | 20+ |
| Assets/Scripts/ | ? | 37개 |
| **총합 (인프라)** | **~761 KB** | **~60개** |

---

**분석 완료**: 2026-02-23
**분석가**: Claude Agent Team
**버전**: 1.0
