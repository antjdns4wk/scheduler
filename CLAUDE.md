# 시간 블록 스케줄러 (Time Block Scheduler)

## 프로젝트 개요

`index.html` **단일 파일**로 이루어진 바닐라 JS 웹앱. 빌드 도구·번들러·npm 없음.
브라우저로 파일을 직접 열거나 정적 호스팅에 올려 쓴다. PWA 매니페스트가 인라인(data URI)으로 들어있다.

주간/일간 타임블록 스케줄을 드래그로 만들고, **벤자민 프랭클린 3-5-7-9 법칙**에 맞춰 시간 배분을
점검하며, Firebase Realtime Database에 계정별로 동기화한다.

## 파일 구조

`index.html` 한 파일 안에 전부 들어있다. (약 5,500줄)

| 영역 | 위치 | 내용 |
|---|---|---|
| head | ~1–13 | PWA 메타 + 인라인 manifest |
| CSS | `<style>` | CSS 변수 팔레트 → 컴포넌트 → `@media (max-width: 768px)` 모바일 오버라이드 |
| HTML | `<body>` | 로그인 오버레이 / 헤더 / 3-5-7-9 밸런스 바 / 그리드 / 메모 패널 / 모달 6종 |
| JS | `<script>` | 앱 전체 로직 |

모달: 일별통계(`modalOverlay`) · 주간·월간통계(`wstatOverlay`) · 템플릿(`tplOverlay`) ·
아카이브(`archiveOverlay`) · 설정(`setOverlay`) · 검색(`srchOverlay`)

**수정 시 원칙**: 파일을 분리하지 말 것. 사용자가 HTML 파일 하나만 들고 다니는 형태를 전제로 만들어졌다.

> 서비스워커는 v2에서 제거됐다 (blob URL 등록을 브라우저가 거부). 오프라인은 localStorage 폴백으로만 동작한다.

## 핵심 데이터 모델

```js
블록 = { id, label, mins, slots, col, si, am, done, track, rid? }
```

- `col` — 0~6 (월~일)
- `si` — **10분 단위** 인덱스. 화면 표시용 상대 좌표. `S_HR`이 바뀌면 같이 바뀐다
- `am` — **자정 기준 절대 분 앵커**. 저장/복원의 기준이며 시간 범위 변경에도 안전
- `mins` — 블록 길이(분). 10분 단위 snap
- `track` — 0 = 계획, 1 = 실제 (일간 뷰의 좌/우 2트랙)
- `done` — 완료 체크
- `rid` — 매주 반복 정의 ID (있으면 라벨에 🔁 표시)

**`si`와 `am`의 관계**: `am = S_HR*60 + si*10`. 저장 시 `withAnchor(b)`로 `am`을 붙이고,
로드 시 `normalizeBlock(b)`으로 현재 `S_HR` 기준 `si`를 다시 계산한다.
**블록을 저장하는 코드는 반드시 `withAnchor`를, 읽는 코드는 `normalizeBlock`을 통과시켜야 한다.**

메모 블록 = `{ id, color, title, body, ts }` (일자별 배열)

**메모의 순서는 배열 순서이고, 배열은 DOM 순서에서 만들어진다.**
`persistMemo()`가 `#memoScroll` 안의 `.memo-block`을 위에서 아래로 훑어 저장하므로,
순서를 바꾸려면 DOM 노드를 옮긴 뒤 `persistMemo()`만 부르면 된다 (드래그 정렬이 이 방식).

메모 색은 `dataset.color`(저장값)와 CSS 변수 `--mb-color`(왼쪽 띠) 두 곳에 반영된다.
둘을 따로 건드리지 말고 `applyMemoColor(blockEl, dotEl, col, preview)`를 쓸 것
(`preview: true`면 화면만 바꾸고 `dataset.color`는 건드리지 않는다).

**떠 있는 팝오버는 위치를 한 번만 쓸 것.** 크기를 재려고 다른 위치(`visibility:hidden`,
화면 밖 좌표)에 붙였다가 옮기면 반영이 한 프레임 늦어, 곧바로 이어지는 드래그가
히트테스트에서 그 요소를 놓친다. `openColorPicker`는 최종 위치로 붙인 뒤
화면 밖으로 넘칠 때만 보정한다.

## 좌표계 규칙 (가장 헷갈리는 부분)

| 단위 | 이름 | 값 |
|---|---|---|
| 표시 슬롯 | 30분 | `N_SLOTS`, DOM의 `.sl` 요소 |
| 내부 배치 | 10분 | `N_FINE`, 블록의 `si`가 쓰는 단위 |
| 절대 앵커 | 분 | `am`, 자정 기준 |
| 픽셀 | `PX` | 10분당 픽셀. 기본 10px, 줌으로 변동 |

- 픽셀 변환: `top = si * PX`, `height = (mins / 10) * PX - 1`
- 30분 슬롯 → fine si: `si30 * 3`
- **`S_HR` / `E_HR` / `N_SLOTS` / `N_FINE` / `TOTAL_MINS`는 `let`이고 설정에서 바뀐다.**
  상수로 취급하면 안 되고, 범위를 바꾼 뒤에는 `recalcRange()`를 불러야 한다

## 3-5-7-9 그룹 시스템

하루 24시간 = 업무 9h · 수면 7h · 여가 5h · 자기계발 3h.

```
라벨 → catOf() → 카테고리 키 → catGroup() → 3-5-7-9 그룹
```

- `GROUPS` (4개, 고정) — 그룹별 색·목표시간을 정의
- `CATEGORIES` — 사용자가 설정에서 편집. 각 항목은 `{ key, group, color, bg, aliases[] }`
- `catOf(label)` — **키 직접 매칭 우선, 그다음 별칭(aliases) 매칭**. 어디에도 안 걸리면 `기타`
- 블록 배경 = 카테고리 연한색, 컬러바 = 그룹색 (`--blk-group`)
- 화면 범위 밖 시간(예: 24:00–05:00)은 `impliedSleep()`으로 **수면에 자동 가산**된다.
  통계 숫자가 안 맞아 보이면 이것부터 의심할 것

## 저장 구조

Firebase Realtime Database (프로젝트: `my-scheduler-e97b8`, Google 로그인)

```
users/{uid}/weeks/{weekKey}       주간 블록     weekKey = "week_YYYYMMDD" (그 주 월요일)
users/{uid}/memos/{weekKey}/d{n}  일자별 메모   n = 0~6
users/{uid}/templates/{id}        템플릿
users/{uid}/tpl_index             템플릿 목록
users/{uid}/settings              시간 범위·알림
users/{uid}/categories_v4         카테고리 정의
users/{uid}/recurring             매주 반복 정의
```

- 모든 쓰기는 **Firebase + localStorage 동시 저장**. 읽기는 Firebase 우선, 실패/미로그인 시 localStorage 폴백
- **`fbReady()`로 가드할 것** — `db`만 확인하면 로그인 전에도 `users/local/...`로 요청이 나가
  전부 `PERMISSION_DENIED`로 실패하며 헛된 왕복이 생긴다
- localStorage 키: `scheduler_`(주간) · `memo_blocks_`(메모) · `sched_tpl_`·`sched_tpl_index`(템플릿) ·
  `app_settings_v2` · `user_categories_v4` · `recurring_defs_v1` · `live_timer_v1` · `notified_YYYYMMDD`
- 실시간 리스너(`attachFirebaseListener`)가 다른 기기의 변경을 반영. 단 **드래그/리사이즈 중,
  입력창 열림, 내가 5초 내 저장** 시에는 무시한다 (중복 렌더 방지)
- 여러 주를 훑는 화면(아카이브·검색·월간통계)은 **`loadAllWeeks()`를 쓸 것.**
  주마다 `loadWeekData()`를 부르면 Firebase 왕복이 주 수만큼 생긴다

## 주요 함수 지도

| 기능 | 함수 |
|---|---|
| 그리드/축 | `buildAxis`, `buildGrid`, `rebuildGridSize`, `syncAxisSpacer`, `recalcRange` |
| 블록 생성 | `attachDragCreate`, `openDragCreateInput`, `handleInp`, `createBlockAt` |
| 렌더 | `renderBlock`, `startEditLabel` |
| 충돌 해결 | `resolveSlot` ← **가장 중요**. 실패 시 `null` 반환 |
| 드래그/리사이즈 | `finishDrag`, `finishResize`, `previewShift`, `startAltCopyDrag`, `duplicateBlock` |
| 좌표 변환 | `withAnchor`, `normalizeBlock`, `blockRange`, `clockOfMins` |
| 그룹/카테고리 | `catOf`, `catGroup`, `grpOf`, `groupTarget`, `impliedSleep`, `renderCatChips` |
| 밸런스 바 | `updateRtLeg`, `applyGroupFilter`, `franklinSection` |
| 통계 | `openStats(dayIdx)`, `openWeekStats`, `openMonthStats(y, m)` |
| 저장/로드 | `autoSave`, `saveWeekData`, `loadWeekData`, `loadAllWeeks`, `loadWeek`, `attachFirebaseListener` |
| 메모 | `initMemoPanel`, `addMemoBlock`, `persistMemo`, `moveMemoToNextDay`, `finishMemoDrag` |
| 템플릿 | `saveTemplate`, `applyTemplate`, `renderTplList` (전부 async) |
| 타이머 | `startTimer`, `commitTimer`, `updateLiveTimerBlock` |
| 반복 | `toggleRecurring`, `applyRecurringToWeek`, `loadRecurringFromCloud` |
| 설정/검색 | `openSettings`, `applySettings`, `applyRange`, `openSearch`, `runSearch` |

### resolveSlot 동작

같은 `col` + 같은 `track`끼리만 **분 단위**로 겹침을 검사한다.
원하는 위치가 막히면 ① 겹친 블록 바로 아래 → ② 위/아래 최근접 빈 자리 순으로 탐색하고,
빈 자리가 없으면 `null`을 반환한다.
호출부는 `null`을 반드시 처리해야 한다 (토스트 띄우고 원상 복구).

**블록을 새 위치에 놓는 모든 경로는 `resolveSlot`을 통과해야 한다.** 빠뜨리면 블록이 겹쳐 쌓인다.

## 통계 집계 규칙

- 카테고리/그룹 집계는 **`track === 0`(계획)** 기준. `track === 1`(실제)은 따로 집계해 비교 표시
- 완료율은 `done` 플래그, 이행률은 실제/계획 비율
- 밸런스 바(`updateRtLeg`)는 **일간이면 그날, 주간이면 주 전체** 기준으로 스코프가 바뀐다

## 뷰 모드

- **주간**: 7열 그리드. 블록을 요일 간 이동 가능
- **일간**: 1열 + `.dual-track`. 좌=계획(track 0), 우=실제(track 1). 우측에 메모 패널
- 모바일(`window.innerWidth <= 768`)은 일간 모드로 시작, 메모 패널 기본 닫힘

## 단축키 · 조작

`T` 오늘 · `W` 주간 · `D` 일간 · `←`/`→` 이전/다음
`Alt`+드래그 = 블록 복사 · 더블클릭 = 이름 수정 · 길게 누르기 = 이름 수정(모바일)
블록 호버 액션: `⧉` 복사 · `↻` 매주 반복 · `▶` 타이머 시작 · `×` 삭제

## 수정할 때 주의사항

1. **`S_HR`/`E_HR`은 변수다** — 하드코딩된 `5`, `24`, `1140`을 새로 넣지 말 것.
   `TOTAL_MINS`, `N_FINE`을 쓰고, 범위를 바꿨으면 `recalcRange()` → `buildAxis()` → `buildGrid()` 순
2. **저장은 `withAnchor`, 로드는 `normalizeBlock`** — 빠뜨리면 시간 범위 변경 시 블록이 엉뚱한 곳으로 간다
3. **`normalizeBlock`은 범위 밖 블록을 경계로 클램프한다** — 여러 개가 같은 자리에 몰릴 수 있으므로
   범위 변경 후에는 `resolveSlot`으로 재배치해야 한다 (`applyRange` 참고)
4. **다른 주(week)의 데이터를 화면에 띄울 때는 `weekOffset`도 함께 옮길 것** —
   안 그러면 다음 `autoSave()`가 현재 주 키에 덮어써 데이터가 사라진다
5. **Firebase 접근은 `fbReady()` 가드** — 로그인 전 요청은 전부 실패한다
6. **여러 주를 훑을 때는 `loadAllWeeks()`** — 주마다 `loadWeekData()`는 N번 왕복
7. **이벤트 리스너 중복 등록** — 목록을 다시 그리는 함수는 `archiveWired`/`tplWired` 같은 1회 등록
   가드를 쓴다. 렌더 함수 안에서 컨테이너에 `addEventListener`를 붙이면 열 때마다 누적된다
8. **async 함수를 `await` 없이 쓰지 말 것** — `loadWeekData`, `loadAllWeeks`, `loadMemoBlocks`,
   `getTplIndex`, `loadTplData`, `deleteTpl`, `renderTplList`이 전부 async다
9. **`renderBlock`은 기존 DOM을 지우고 새로 만든다** — 블록 요소에 외부에서 상태를 붙이면 재렌더 시 날아간다
10. **`.blk-lbl`은 `.blk-name` + `.blk-time` 두 span을 갖는다** — `textContent`로 통째로 덮으면 서식이 깨진다
11. **사용자 입력을 `innerHTML` 템플릿에 넣을 때는 `esc()`** — 블록 라벨, 템플릿 이름, 카테고리 키,
    메모 제목/본문이 대상이다
12. **삭제는 Firebase와 localStorage 양쪽 모두** 처리할 것

## 테스트

자동화된 테스트 없음. 로컬 서버로 띄워서 확인한다 (`file://`은 localStorage가 제한될 수 있음):

```bash
python -m http.server 8971 --bind 127.0.0.1
```

체크리스트:
- 블록 생성(클릭/드래그) → 이동 → 리사이즈 → Alt 복사 → 삭제
- 일간 보기 메모: 손잡이로 순서 바꾸기 → 새로고침 후 순서 유지 · 창 크기 변경 후에도 패널 유지
- 주간 ↔ 일간 전환, 트랙 간 이동, 3-5-7-9 밸런스 바 스코프 변화
- 설정에서 시간 범위 변경 후 블록 위치·겹침
- 아카이브에서 지난 주 불러오기 → 편집 → 현재 주 데이터가 멀쩡한지
- 타이머 시작/정지 → 실제 트랙 기록
- 새로고침 후 데이터 유지, 다른 기기/탭에서 동기화
- 모바일 폭(≤768px) 레이아웃

## 커밋

의미 있는 변경 단위로 커밋한다. 커밋 메시지는 한국어.
