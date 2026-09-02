# 시간 블록 스케줄러 (Time Block Scheduler)

## 프로젝트 개요

`index.html` **단일 파일**로 이루어진 바닐라 JS 웹앱. 빌드 도구·번들러·npm 없음.
브라우저로 파일을 직접 열거나 정적 호스팅에 올려서 사용한다.

주간/일간 타임블록 스케줄을 드래그로 만들고, Firebase Realtime Database에 계정별로 동기화한다.

## 파일 구조

`index.html` 한 파일 안에 전부 들어있다.

| 영역 | 위치 | 내용 |
|---|---|---|
| CSS | `<style>` 블록 | CSS 변수 팔레트 → 컴포넌트 스타일 → `@media (max-width: 768px)` 모바일 오버라이드 |
| HTML | `<body>` | 로그인 오버레이 / 헤더 / 그리드 / 메모 패널 / 모달 4종(일별통계·주간통계·템플릿·아카이브) |
| JS | 첫 번째 `<script>` | 앱 전체 로직 |
| SW | 두 번째 `<script>` | Blob URL로 서비스워커 인라인 등록 (오프라인 캐시) |

**수정 시 원칙**: 파일을 분리하지 말 것. 사용자가 HTML 파일 하나만 들고 다니는 형태를 전제로 만들어졌다.

## 핵심 데이터 모델

```js
블록 = { id, label, mins, slots, col, si, done, track }
```

- `col` — 0~6 (월~일)
- `si` — **10분 단위** 인덱스. `si * 10` = 시작 시각(05:00 기준 경과 분). 0~114
- `mins` — 블록 길이(분). 10분 단위로 snap
- `slots` — `mins / 10` (사실상 파생값, 레거시)
- `track` — 0 = 계획, 1 = 실제. **일간 뷰의 좌/우 2트랙**
- `done` — 완료 체크

메모 블록 = `{ id, color, title, body, ts }` (일자별 배열)

## 좌표계 규칙 (가장 헷갈리는 부분)

세 가지 단위가 섞여 있으니 반드시 구분할 것:

| 단위 | 이름 | 값 |
|---|---|---|
| 표시 슬롯 | 30분 | `N_SLOTS = 38`, DOM의 `.sl` 요소 |
| 내부 배치 | 10분 | `N_FINE = 114`, 블록의 `si`가 쓰는 단위 |
| 픽셀 | `PX` | 10분당 픽셀. 기본 10px, 줌으로 변동 |

- 픽셀 변환: `top = si * PX`, `height = (mins / 10) * PX - 1`
- 30분 슬롯 → fine si: `si30 * 3`
- 시간 범위는 `S_HR = 5` ~ `E_HR = 24` 고정. 하루 `TOTAL_MINS = 1140`

## 카테고리 시스템

**라벨 문자열에 키워드가 포함되는지**로 카테고리를 판정한다 (`catOf`). 별도 카테고리 필드 없음.

```
업무 / 공부 / 운동 / 식사 / 휴식 / 취침 → 매칭 없으면 "기타"
```

색상은 `CATEGORIES` 배열(JS)과 `:root` CSS 변수 **두 곳에 중복 정의**되어 있다. 색을 바꾸면 양쪽 모두 고쳐야 한다.

## 저장 구조

Firebase Realtime Database (프로젝트: `my-scheduler-e97b8`, Google 로그인)

```
users/{uid}/weeks/{weekKey}     주간 블록      weekKey = "week_YYYYMMDD" (그 주 월요일)
users/{uid}/memos/{weekKey}/d{n} 일자별 메모   n = 0~6
users/{uid}/templates/{id}       템플릿
users/{uid}/tpl_index            템플릿 목록
```

- 모든 쓰기는 **Firebase + localStorage 동시 저장**. 읽기는 Firebase 우선, 실패/미로그인 시 localStorage 폴백
- localStorage 키 접두사: `scheduler_`(주간), `memo_blocks_`(메모), `sched_tpl_`·`sched_tpl_index`(템플릿)
- 실시간 리스너(`attachFirebaseListener`)가 다른 기기의 변경을 반영. 단 **드래그/리사이즈 중, 입력창 열림, 내가 5초 내 저장** 시에는 무시한다 (중복 렌더 방지)

## 주요 함수 지도

| 기능 | 함수 |
|---|---|
| 그리드/축 생성 | `buildAxis`, `buildGrid`, `rebuildGridSize`, `syncAxisSpacer` |
| 블록 생성 | `attachDragCreate`, `openDragCreateInput`, `handleInp`, `createBlockAt` |
| 렌더 | `renderBlock`, `startEditLabel` |
| 충돌 해결 | `resolveSlot` ← **가장 중요**. 실패 시 `null` 반환 |
| 드래그/리사이즈 | `finishDrag`, `finishResize`, `previewShift`, `clearShift` |
| 뷰 전환 | `switchView('week'\|'day')`, `setDayView(di)` |
| 통계 | `openStats(dayIdx)`, `openWeekStats` |
| 저장/로드 | `autoSave`, `saveWeekData`, `loadWeekData`, `loadWeek`, `attachFirebaseListener` |
| 메모 | `initMemoPanel`, `addMemoBlock`, `persistMemo`, `moveMemoToNextDay` |
| 템플릿 | `saveTemplate`, `applyTemplate`, `renderTplList` |

### resolveSlot 동작

같은 `col` + 같은 `track`끼리만 **분 단위**로 겹침을 검사한다.
원하는 위치가 막히면 ① 겹친 블록 바로 아래 → ② 위/아래 최근접 빈 자리 순으로 탐색하고, 빈 자리가 없으면 `null`을 반환한다.
호출부는 `null`을 반드시 처리해야 한다 (토스트 띄우고 원상 복구).

## 통계 집계 규칙

일별·주간 통계 모두 **`track === 0`(계획)만 집계**한다. 실제(track 1)는 제외.
완료율만 `done` 플래그를 쓴다.

## 뷰 모드

- **주간**: 7열 그리드. 블록을 요일 간 이동 가능
- **일간**: 1열 + `.dual-track` 클래스. 좌=계획(track 0), 우=실제(track 1). 우측에 메모 패널
- 모바일(`window.innerWidth <= 768`)은 일간 모드로 시작하고 메모 패널은 기본 닫힘

## 단축키

`T` 오늘 · `W` 주간 · `D` 일간 · `←`/`→` 이전/다음 (일간이면 하루, 주간이면 한 주)

## 수정할 때 주의사항

1. **이벤트 리스너 중복 등록** — 목록을 다시 그리는 함수(`renderTplList`, `openArchive`)에서 컨테이너에 `addEventListener`를 붙이면 열 때마다 누적된다. 리스너는 한 번만 등록하고 위임(delegation)으로 처리할 것
2. **`loadWeekData`, `loadMemoBlocks`, `getAllSavedKeys`는 async** — `await` 없이 쓰면 Promise가 그대로 들어가 조용히 망가진다
3. **Firebase 경로는 반드시 `fb*Path()` 헬퍼를 통해서** — uid가 빠지면 계정 간 데이터가 섞인다
4. **삭제는 Firebase와 localStorage 양쪽 모두** 처리할 것
5. `renderBlock`은 기존 DOM을 지우고 새로 만든다. 블록 요소에 외부에서 상태를 붙이면 재렌더 시 날아간다

## 테스트

자동화된 테스트 없음. 브라우저에서 직접 확인한다:
- 블록 생성(클릭/드래그) → 이동 → 리사이즈 → 삭제
- 주간 ↔ 일간 전환, 트랙 간 이동
- 새로고침 후 데이터 유지, 다른 기기/탭에서 동기화
- 모바일 폭(≤768px)에서 레이아웃

## 커밋

의미 있는 변경 단위로 커밋한다. 커밋 메시지는 한국어.
