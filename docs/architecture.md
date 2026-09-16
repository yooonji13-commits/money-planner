# 구조

## 전체 모습

브라우저에서 `index.html` 하나를 열면 그것이 앱 전부다. HTML·CSS·JavaScript가 그 한 파일에
들어 있고, 빌드 단계도 서버 코드도 없다. 정적 호스팅에 파일을 올리는 것이 곧 배포다.

```
브라우저
  └─ index.html  ── 화면·계산·저장 전부
       ├─ localStorage            (이 기기의 저장소, 원본)
       └─ Supabase sync_data 테이블 (기기 사이를 잇는 중계)
```

외부에서 불러오는 것은 CDN의 Supabase 클라이언트 **하나뿐**이다. 그것이 없으면
(`window.supabase`가 없으면) 동기화 기능만 꺼지고 나머지 앱은 그대로 동작한다.

## 화면 구성

전체는 고정 골격 + 한 덩어리 본문이다.

- **헤더** — 날짜, 그리고 동기화 / 내보내기 / 가져오기 / 계산기 / 고정내역 버튼. HTML에 정적으로
  박혀 있고 JavaScript가 `hidden` 속성만 토글한다.
- **`#mp-main`** — 지금 탭의 내용. 탭을 바꾸거나 데이터가 바뀌면 이 안을 **문자열로 통째로 다시
  만든다**(`innerHTML`).
- **하단 탭바** — 다섯 탭. 헤더처럼 정적이다.
- **패널 네 개** (`#mp-sync-panel`, `#mp-password-panel`, `#mp-calculator-panel`,
  `#mp-fixed-panel`) — 모달·팝오버. 본문과 독립적으로 각자 다시 그려진다.

## 화면을 그리는 두 가지 방식 — 통째로와 부분만

이 구분이 이 앱 구조의 핵심이다. 둘을 혼동하면 실기기에서 입력이 끊긴다.

**통째로 그리기 — `render()` → `drawApp()`**

`render()`는 먼저 저장소 읽기 실패 여부를 보고, 실패면 안내 화면으로 대체한다. 아니면
`drawApp()`을 `try`로 감싸 호출한다. `drawApp()`이 지금 탭에 맞는 그리기 함수
(`renderPersonal` / `renderLiving` / `renderSavings` / `renderLoan` / `renderUtility`)를 골라
그 결과 문자열을 `#mp-main`에 넣는다. 그리다 예상 못한 오류가 나면 `render()`의 `catch`가 안내
화면을 띄운다.

**부분만 갈아끼우기 — `update...Display()` / `live...Patch()`**

숫자를 치는 중에는 절대 통째로 그리지 않는다. 통째로 그리면 사용자가 커서를 올려둔 입력칸
자체가 새 노드로 교체되고, 아이폰에서는 그 순간 키패드가 내려간다. 그래서 값이 바뀔 때 **숫자
텍스트와 게이지 폭만** 바꿔 넣는 함수들이 항목마다 따로 있다
(`updateUtilityDisplay`, `updateSavingsDisplay`, `updateLivingDisplay`,
`updatePersonalBalanceDisplay`, `updateLoanDisplay`, `liveUpdateLoanFromDOM`,
`liveUtilityChartPatch`, `updateFixedItemsTotal`).

같은 이유로 실시간 동기화가 원격 데이터를 받아도, 잠금 화면이 떠 있거나 어딘가 입력칸에 커서가
있으면 `applyRemoteState`는 데이터만 반영하고 다시 그리지 않는다.

## 조작이 흐르는 길

조작은 모두 `#mp-root`에 걸린 **위임 리스너 네 개**로 들어온다. 개별 요소에 리스너를 붙이지
않기 때문에, 다시 그려도 리스너를 다시 달 필요가 없다.

- **`click`** — `data-action` 값으로 분기. 맨 앞에서 `delete-`로 시작하는 동작을 가로채 확인
  창을 띄운다(새 삭제 기능이 생기면 자동으로 확인이 붙는다).
- **`input`** — 치는 중. 날짜칸은 형식만 맞추고 **저장하지 않는다**. 금액칸은 천 단위 콤마를
  넣고 커서를 보정한 뒤, 관리비 금액과 고정내역 금액은 여기서 바로 저장한다.
- **`change`** — 손을 뗐을 때. 값을 상태에 확정하고 저장한다.
- **`focusin` / `focusout` / `keydown`** — 숫자칸 커서를 끝으로 보내기, 계좌 정보 커밋,
  Enter로 입력 종료.

탭바 버튼만 예외로 각 버튼에 직접 리스너가 붙어 있다(정적 요소라 다시 만들어지지 않는다).

## 대표 흐름 하나 — 관리비 전기료에 숫자를 칠 때

1. 키를 누르면 `input` 리스너가 받는다. 날짜칸이 아니므로 금액 경로로 간다.
2. 값에서 마이너스를 떼고(개인 탭 기준 잔액만 예외) 천 단위 콤마를 넣은 뒤 커서를 원래 자리로
   되돌린다.
3. `getUtilityMonth(지금 달)`의 해당 항목에 절대값을 넣는다.
4. 화면의 최종 관리비 텍스트를 바꾸고 `liveUtilityChartPatch()`로 그래프 좌표만 옮긴다.
5. `saveState()`로 기기에 저장하고, `showUtilitySavedFlash()`로 "저장됨 ✓"을 잠시 켠다.
6. `saveState()`는 마지막에 `scheduleCloudPush()`를 부른다. 동기화 코드가 있으면 800ms 뒤
   클라우드로 올리며, 그 사이 다시 치면 타이머가 새로 잡힌다.

**여기서 `render()`는 한 번도 불리지 않는다.** 그래서 입력칸과 커서가 그대로 남는다.

## 데이터가 들어오는 세 길

같은 데이터가 세 경로로 들어오고, **세 경로 모두 `normalizeState()` 하나를 지난다.**

```
기기 저장소 읽기 ──┐
클라우드 수신    ──┼─→ normalizeState() ─→ state ─→ render()
백업 파일 가져오기 ─┘
```

- **기기 저장소** — `loadState()`. 원문을 읽지 못하면 `preserveUnreadableRaw()`가 원문을 다른
  키로 복사해 두고 `storageUnreadable` 깃발을 세운다. 그 상태에서는 저장도 클라우드 전송도
  하지 않는다.
- **클라우드** — `applyRemoteState()`. 어느 탭·몇 월을 보고 있는지는 기기별 화면 상태이므로
  원격 값으로 덮지 않는다. 비밀번호도 여기로 들어오지 않는다.
- **백업 파일** — `looksLikeBackup()`으로 머니플래너 백업인지 먼저 판별하고, 아니면 거절한다.

## 저장 경로

**가계부 상태를 저장하는 통로는 `saveState()` 하나다.** `writeStorage()`로 `localStorage`에 쓰고, 실패하면
`notifySaveFailed()`가 안내를 띄우되 **예외를 밖으로 던지지 않는다** — 던지면 같은 처리 안의
뒤따르는 화면 갱신이 중단된다. 쓰기가 성공하면 저장 시각을 남기고 클라우드 전송을 예약한다.

앱이 백그라운드로 가거나 닫힐 때는 `flushPendingBeforeLeave()`가 입력 중이던 값들을 커밋하고
저장한 뒤, 클라우드 전송 대기 시간을 기다리지 않고 바로 올린다.

## 기기 경계

`localStorage`에 들어가는 키 중 **동기화되는 것은 하나뿐**이다.

| 키 | 무엇 | 클라우드로 가는가 |
|---|---|---|
| `mp_data_v1` | 가계부 전체 | **간다** |
| `mp_personal_pw_v1` | 개인 탭 비밀번호 해시 | 가지 않는다 (기기 전용) |
| `mp_sync_code` | 이 기기가 쓰는 6자리 코드 | 가지 않는다 |
| `mp_last_saved_at` | 마지막 저장 시각 | 가지 않는다 (앱 시작 시 원격과 비교하는 용도) |
| `mp_data_v1_unreadable*` | 읽지 못한 원문의 사본 | 가지 않는다 |
