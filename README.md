# Connect — Robot Fleet Orchestrator Prototype

> 프론트엔드 엔지니어 인수인계용 문서  
> 파일: `prototype.jsx` (약 7,500줄) / 빌드 결과: `prototype.html`

-----

## 1. 실행 방법

별도 빌드 없이 `prototype.html`을 브라우저에서 바로 열면 동작합니다.

```
prototype.html   ← 이것만 열면 됨
prototype.jsx    ← 소스 (컴포넌트 구조 참고용)
```

> **기술 스택** (CDN 기반, 설치 불필요)
> 
> - React 18 (UMD)
> - Babel Standalone (JSX 변환)
> - Tailwind CSS (CDN)
> - Lucide Icons (UMD)

-----

## 2. 컴포넌트 트리

```
App                          ← 최상위. 네비게이션 state 관리
│
├── Sidebar                  ← 좌측 GNB (아이콘 네비게이션)
├── Header                   ← 상단 바 (타이틀, 커맨드 팔레트 버튼)
├── CommandPalette            ← Cmd+K 글로벌 검색/이동
│
├── MonitoringHome            ← 모니터링 홈 (L562)
│   ├── KpiCard               ← 상단 KPI 카드 4종
│   ├── AlertRow              ← 경보 목록 행
│   ├── [SVG 맵]              ← 인라인 SVG, pan/zoom, 노드·엣지 렌더링
│   └── [우측 패널]
│       ├── Command 탭        ← 작업 지시 (fleet/robot 선택, 템플릿 실행)
│       └── Properties 탭     ← 선택된 로봇/노드/엣지 속성
│           ├── Commission 설정
│           └── Parameter 설정
│
├── InfraStatusView           ← 인프라 현황 (L2177)
├── RobotStatusView           ← 로봇 상태 상세 (L2466)
├── AnalyticsView             ← 통계/분석 (L2953)
│   ├── AreaChart             ← 시간대별 처리량 영역 차트
│   ├── StackedBarChart       ← 작업 유형별 누적 바 차트
│   └── MiniDonut             ← 도넛 차트
│
├── ToolCartView              ← Tool Cart 관리 (L3280)
│   └── AddCartModal          ← Cart 등록 모달
│
├── MapEditorView             ← 맵 에디터 (L3586)
│   └── [좌측 패널]
│       ├── Active Graph      ← Fleet 그래프 다중 선택/색상/복사/삭제
│       ├── Edit Mode         ← Stamp / Build / Paint / Speed
│       ├── Grid              ← Grid 표시 / Snap
│       ├── Inspector         ← 선택 노드·엣지 속성
│       ├── Background Image  ← 도면 오버레이
│       └── Validation        ← 실시간 유효성 검사
│
├── RecipeView                ← 작업 관리 (L4435)
│   ├── RecipeList            ← 템플릿 목록 (L4474)
│   └── RecipeBuilder         ← 템플릿 생성/편집 (L4679)
│       └── [멀티슬롯 스텝]   ← 최대 3 로봇 슬롯, 순차/병렬 모드
│
├── OrderTemplateView         ← 작업 요청 (L5379)
├── SchedulerView             ← 작업 스케줄링 / 간트차트 (L5725)
│   ├── MultiSelectDropdown   ← 다중 선택 드롭다운
│   ├── MultiFleetDropdown    ← Fleet 필터 드롭다운
│   ├── AddOrderDialog        ← 오더 추가 모달
│   └── [Gantt Chart]         ← 드래그앤드롭, 멀티슬롯 배정 모달 포함
│
├── LogsView                  ← 로그/이벤트 (L6669)
└── PermissionsView           ← 권한 관리 (L7029)
```

-----

## 3. 전역 데이터 (Mock Data)

모두 `prototype.jsx` 상단에 `const` 배열로 정의되어 있습니다.

|변수명          |설명           |주요 필드                                            |
|-------------|-------------|-------------------------------------------------|
|`fleets`     |Fleet 목록 (5개)|`id, name, color`                                |
|`robots`     |로봇 목록        |`id, fleet, battery, st`                         |
|`mapNodes`   |맵 노드         |`id, x, y, label, fleets`                        |
|`mapEdges`   |맵 엣지         |`id, from, to, dir`                              |
|`floors`     |층/맵 목록       |`id, label`                                      |
|`actionTypes`|액션 타입 (17종)  |`id, label, icon, color, fleets, defaultDuration`|
|`initOrders` |초기 오더 데이터    |`id, robotId, recipeId, status, start`           |
|`logs`       |로그 데이터       |`id, level, cat, msg, ts, robot`                 |
|`initCarts`  |Tool Cart 데이터|`id, size, line, floor, parts`                   |
|`initUsers`  |사용자 데이터      |`id, name, dept, role, status`                   |
|`initPending`|대기 오더        |`id, fleet, recipe, requestedAt`                 |

-----

## 4. App 최상위 State

```js
const [active, setActive]           // 현재 활성 뷰 (라우팅 역할)
const [cmdOpen, setCmdOpen]         // CommandPalette 열림 여부
const [focusRcp, setFocusRcp]       // 스케줄러에서 포커스할 레시피 ID
const [recipeRev, setRecipeRev]     // 레시피 변경 감지용 revision
const [logsInitCat3, setLogsInitCat3] // 로그 뷰 초기 카테고리
const [isDark, setIsDark]           // 다크모드 (현재 미구현)
const [pendingOrder, setPendingOrder] // 스케줄러로 넘기는 pending 오더
const [globalPopup, setGlobalPopup] // 전역 팝업 (작업 발송 확인 등)
```

**뷰 라우팅** — `active` 값 목록:

```
"home"       → MonitoringHome
"infra"      → InfraStatusView
"robots"     → RobotStatusView
"analytics"  → AnalyticsView
"carts"      → ToolCartView
"map-editor" → MapEditorView
"recipe"     → RecipeView
"order"      → OrderTemplateView
"scheduler"  → SchedulerView
"logs"       → LogsView
"permissions"→ PermissionsView
```

-----

## 5. 주요 데이터 모델

### Recipe (작업 템플릿)

```ts
{
  id: string
  name: string
  fleet: string           // fleet id
  status: "draft" | "published"
  locked: boolean         // 잠금 시 fleet-level 템플릿
  steps: Step[]
}

type Step = {
  nodeId: string
  mode: "sequential" | "parallel"
  slots: Slot[]           // 최대 3개
}

type Slot = {
  actionType: string      // actionTypes[].id 참조
  duration: number        // 초 단위
}
```

### Order (작업 오더)

```ts
{
  id: string
  robotId: string
  recipeId: string
  status: "queued" | "running" | "done" | "error"
  start: number           // 분 단위 (자정 기준 경과)
  groupId?: string        // 멀티슬롯 오더 그룹
  slotIdx?: number        // 슬롯 인덱스 (0, 1, 2)
}
```

### MapNode / MapEdge

```ts
type MapNode = {
  id: string
  x: number               // SVG 좌표 (픽셀)
  y: number
  label: string
  fleets: string[]        // 사용 가능 fleet ids
}

type MapEdge = {
  id: string
  from: string            // node id
  to: string
  dir: "bi" | "uni"
}
```

### MapEditor 내부 Node / Edge (편집 시)

```ts
type EditorNode = {
  id: string
  x: number
  y: number
  name: string
  role: "standard" | "station" | "charger" | "holding" | "holding_candidate" | "siding"
  is_charger: boolean
  is_holding: boolean
  graph_idx: number       // fleet 그래프 인덱스
  capability: null
}

type EditorEdge = {
  id: string
  src: string
  dst: string
  bidir: boolean
  v_max: number | null    // m/s, null = 미설정
  graph_idx: number
  dirCycle: 0 | 1 | 2    // 방향 순환 상태 (→ / ↔ / ←)
}
```

-----

## 6. 주요 유틸 함수

|함수                     |위치              |설명                        |
|-----------------------|----------------|--------------------------|
|`recipeDuration(r)`    |L144            |레시피 전체 소요 시간 계산           |
|`recipeMaxSlots(r)`    |L145            |레시피 최대 슬롯 수               |
|`fmtTime(min)`         |MonitoringHome 내|분 → `HH:MM` 포맷            |
|`computePlacement(...)`|L5216           |간트차트 오더 배치 (충돌 회피)        |
|`recipeDuration(r)`    |L144            |steps.slots 기준 duration 합산|
|`actMeta(id)`          |RecipeBuilder 내 |actionType 메타데이터 조회       |
|`fleetMeta(id)`        |SchedulerView 내 |fleet 색상/이름 조회            |

-----

## 7. 화면별 주요 기능 요약

### MonitoringHome

- 실시간 KPI (로봇 운영/Fleet/작업/처리량)
- SVG 맵: 노드·엣지·로봇 렌더링, 클릭 선택, pan/zoom, 전체화면
- 우측 패널: Command(작업지시) / Properties(속성+Commission)
- 작업 현황 목록: 새창 팝업, Fleet/Robot 필터, 컬럼 정렬

### MapEditorView

- 4가지 Edit Mode: Stamp / Build / Paint / Speed
- Multi-Graph (Fleet별): 다중 선택, 색상 지정, 복사, 삭제
- Ctrl+드래그: 영역 선택 (점선 박스)
- 노드 드래그 이동 (Grid Snap 지원)
- Edge 더블클릭: 방향 순환 (→ / ↔ / ←)
- Ctrl+C/V: 노드 복제
- Delete 키: 선택 노드/엣지 삭제
- Ctrl+Z/Y: Undo/Redo (50단계)
- Background Image 오버레이

### RecipeBuilder

- 멀티슬롯 스텝: 노드당 최대 3 로봇
- 실행 방식: 순차(시간 합산) / 병렬(최대값)
- Flow 블록 드래그앤드롭 순서 변경

### SchedulerView (간트차트)

- 레시피 드래그앤드롭으로 로봇 행에 오더 배치
- 멀티슬롯 레시피 배정 시 모달로 추가 로봇 선택
- 오더 이동/복사/삭제
- Fleet/날짜 필터

-----

## 8. 실제 개발 시 교체 필요 항목

|항목      |현재 (Mock)    |실제 구현 필요            |
|--------|-------------|--------------------|
|데이터     |하드코딩 const 배열|REST API / WebSocket|
|인증      |없음           |JWT / Session       |
|로봇 위치   |정적           |실시간 WebSocket       |
|오더 상태   |정적           |폴링 or SSE           |
|Map Save|미구현 (버튼만)    |API 연동              |
|권한      |UI만 존재       |RBAC 실제 적용          |

-----

## 9. 파일 구조 (실제 개발 시 분리 권장)

```
src/
├── components/
│   ├── common/
│   │   ├── Icon.jsx
│   │   ├── Sidebar.jsx
│   │   ├── Header.jsx
│   │   └── CommandPalette.jsx
│   ├── monitoring/
│   │   ├── MonitoringHome.jsx
│   │   ├── KpiCard.jsx
│   │   └── MapCanvas.jsx
│   ├── map-editor/
│   │   └── MapEditorView.jsx
│   ├── recipe/
│   │   ├── RecipeView.jsx
│   │   ├── RecipeList.jsx
│   │   └── RecipeBuilder.jsx
│   ├── scheduler/
│   │   ├── SchedulerView.jsx
│   │   └── GanttChart.jsx
│   └── ...
├── data/
│   └── mockData.js        ← 현재 prototype.jsx 상단 const 배열들
├── utils/
│   └── helpers.js         ← recipeDuration, computePlacement 등
└── App.jsx
```

-----

> 질문 있으면 편하게 연락주세요.