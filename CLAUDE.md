# forms-as — 폼즈 AS 사내전산

단일 `index.html`(HTML+CSS+JS 전부 포함) + Google Apps Script(GAS) 백엔드로 돌아가는 태블릿 AS 관리 시스템.
마스터(민규님, T5)가 운영하며 엔지니어들이 수리 등록·검수·출고·재고·휴가를 전부 이 화면에서 처리한다.

## 구조와 데이터 흐름

- 프론트: `index.html` 하나. 빌드 없음 — 파일을 main에 커밋하면 GitHub Pages로 바로 배포된다.
- 백엔드: GAS 웹앱 (`gasCall({action, ...})` JSONP/fetch). Code.gs는 이 저장소에 없음(구글 측 관리) — **티오더 AS 프로그램 GAS: scriptId `1HbEzUWU0xnPNBBY-0u516wl81k-WronulnCZlhcCGIvoolXpNAp9-uaU`** (사용자 확인 2026-10-23, https://script.google.com/home/projects/1HbEzUWU0xnPNBBY-0u516wl81k-WronulnCZlhcCGIvoolXpNAp9-uaU/edit). `clasp clone`으로 받아 볼 수 있음.
  로컬 사본: `C:\Users\Forms\forms-as-gas` (이 public 저장소 밖, 자체 git — 배포 전 `clasp pull`로 최신 확인).
  - **티오더 고객사 AS 대시보드 서버 GAS: scriptId `1TRM-DUXMkNeGdYOORtcD4GfGn6bY8_4vkvpdkdmbEDz3fRH5kv_rinMV`** (프로젝트명 `voltales-as-backend`, 사용자 확인 2026-10-23, https://script.google.com/home/projects/1TRM-DUXMkNeGdYOORtcD4GfGn6bY8_4vkvpdkdmbEDz3fRH5kv_rinMV/edit). 사내 AS 프로그램 GAS(위)와 별개 — 고객사 대시보드 작업일 때만 이쪽.
  - ⚠ 공개 data/load.json에 passwordMap(직원 비밀번호 해시)·priceSettings.finance가 그대로 실림 (2026-10-23 확인, client=1 응답만 제거). 고치려면 로그인이 load.json의 passwordMap을 쓰는지 먼저 확인.
- 읽기 데이터: GAS가 주기적으로 `data` 브랜치의 `data/load.json`으로 덤프 (`_z` = gzip+base64, entries는 컬럼형 tbl).
  - **중요**: load.json은 용량 절약을 위해 **출고(shipped) 건의 `parts`를 생략**한다. 서버에는 데이터가 있고,
    수정 모달이 열릴 때 상세를 따로 불러온다. 그래서 **부품 사용량 집계는 반드시 `stockLog`(재고 차감 기록) 기준**으로 한다
    — entries.parts 기준으로 집계하면 출고 순간 과거 숫자가 줄어드는 버그가 된다 (v3.10a에서 전환 완료).
  - **stockLog는 서버가 최근 3,000줄만 보관** (현재 속도로 약 10일치). 그보다 오래된 날짜는 '보관 범위 밖'으로
    표시해야 하며(— 표시), 가장 오래된 날은 중간에 잘려 있으므로 그 날도 범위 밖 취급한다.
- stockLog note 규약: 수리 차감 `수리 자동 차감 [시리얼]` / 수정 `수리 수정 추가차감|복구 [시리얼]` /
  DOA `DOA 불량 [증상]` / DOA 취소 `DOA 등록 취소 (MM-DD HH:mm)` / 실사 `재고 실사 조정`(type은 in/out이므로 note로 걸러야 함).

## 배포 규칙

- 버전: 커밋 제목 `vX.XX: 한 줄 요약` (기능 추가 = 마이너 올림, 후속 수정 = `vX.XXa/b`). 별도 버전 상수 없음.
- main에 index.html 커밋 → GitHub Pages 자동 배포. 로컬에서는 `git push`가 그대로 됨.
- 데모 대시보드는 별도 저장소 `tori-forms/forms-as-demo` (구 kicc-as — 중립 브랜딩·예시 데이터 시연용).

## 검증(QA) 관례

jsdom으로 index.html을 통째로 로드해 함수 단위로 검증한다. 패턴:

```js
const {JSDOM}=require('jsdom');
const dom=new JSDOM(html,{runScripts:'dangerously',url:'https://x.test/',beforeParse(w){
  w.fetch=()=>new Promise(()=>{}); w.Chart=function(){this.destroy=()=>{};this.data={}};
}});
// 전역 주입은 반드시 w.eval('entries=...')로 — entries/stockLog 등은 let 선언이라 w.entries= 직접 대입은 무효
```

함정: `uiConfirm=async()=>true; openPromptModal=(t,m,p,cb)=>cb('...')` 스텁 필요(모달이 비동기),
`MASTER_NAME`·`histFilter`는 const라 재할당 금지(필드만 대입), `window.FINANCE`는 직접 주입,
공휴일 픽스처는 `holidayCache['YYYY-MM']`에 실제 공휴일을 넣어야 KR_HOLIDAYS_FALLBACK과 안 충돌,
serializeWorkers가 참조하는 맵(workerDailyTargets·mentoringEndMap 등)을 전부 초기화해야 휴가 저장 테스트가 통과.

## 도메인 규칙 (마스터 방침 — 바꾸기 전에 반드시 확인)

- **파트 체계**: partMap { 이름: 'p1'|'p2'|'all' }, T4 = 파트장(본인 파트만 관리: canManageWorker), T5 = 총괄.
  파트장 화면 범위는 `_t4CanSee(w)`. 수상은 전체 1명 + 파트별 그룹 점수. 호칭 "1파트 ○○○ 파트장님".
- **휴가**: vacationDaysMap { 이름: ['YYYY-MM-DD'(연차), '…H'(반차), '…Q'(반반차)] }.
  **종류 불문 그 날 수리 목표 면제**(v3.16 — 반차도 미달로 안 잡음). 성장분석 분모 가중은 연1·반0.5·반반0.25 유지.
  팀 휴가 달력(전 직원 공유)은 날짜 클릭 → 모달 등록. 사람별 사용량 합계는 **표시 금지**(비교 부담 — 마스터 지시).
- **경영분석**: T5 전용. 개인별 연봉·월 인건비·배분·손익은 기본 ●●● 가림, [👁 연봉 보기]에 2차 암호(_finHash, f.finPin).
  2차 암호 변경·해제는 현재 암호 확인 필수. 간접 인건비는 파트 스코프 배분(파트장 몫 → 본인 파트 직접 인력).
  퇴사자는 f.departures/f.hires로 입·퇴사일 일할. 추천 KPI는 손익분기 하한.
- **B급**: entry.note의 `[B급]` 태그가 원본. `_bBadge(e)`로 시리얼 앞에 칩 표시 (전 화면).
- **재작업 완료**: 담당 본인 + 총괄 + 같은 파트 파트장(canManageWorker).
- **티오더 검사 의뢰 (v3.23~d)**: 수리 항목 3행 메인보드 옆 [🔍 검사요청] 카드(input 아님 — window._irMode, buildChecks가 그림) → 버튼이 [검사 등록]으로, addEntry가 irRegister로 넘김. 체크한 수리 항목 = 수리 필요 항목. 시트 `검사요청`(수리 통계와 분리 — 건수·목표·수당 미포함).
  상태 wait → approved(수리 승인) → converted / declined(수리 거부 — 검사 내역만 보관, 출고 관리로 넘길지는 논의 중). 예상 공임비 = WORKS 단가 합 + 기타작업 자동 규칙(syncEtcRule과 동일)·NB_EXCLUSIVE 적용.
  승인 건은 수리 등록 전까지 탭 배지·배너로 계속 표시. 내역서는 [내역서(엑셀)] = 원본 양식 xlsx(IR_XLSX_TPL, 예시값·작성자 지운 base64)에 JSZip으로 셀 값만 채움(금액 D20 숫자 → 엑셀에서 수정), [인쇄]는 irPrint. 서버는 converted 건 상태 되돌리기 거절.
  1차는 승인을 엔지니어가 대신 누름 — **2차(대기)**: 고객사 대시보드 승인 버튼(서버 권한 확인 필요) + 티미 슬랙 알림·미처리 재알림.
- **품질점수**: 100 − 불합격률×5. 초과건 수당: `_finOverBonusData` (품질 게이트 5%).
- 불출 전산화(엔지니어가 부품 가져갈 때 직접 입력)는 **거부됨** — 대신 주간 장부 대조 카드로 수기 장부와 대조.
- 공휴일은 API 실패 대비 `KR_HOLIDAYS_FALLBACK` 합집합 (10/5 대체공휴일 오탐 사건).

## 보안 수칙

- flex Open API 시크릿 등 비밀 값은 **코드·커밋·채팅에 절대 금지** (GAS 스크립트 속성에만).
- 급여 데이터는 민감 — 화면 가림은 어깨너머 방지 수준이며, 콘솔까지 막는 보호가 아님을 항상 고지.
- 이 저장소는 public이다. 고객사 실명 브랜딩(예: KICC 로고)은 계약 전 노출 금지 — 데모는 forms-as-demo의 중립 버전.

## 진행 중 / 대기 작업

- flex Open API 연동 (Code.gs 받으면), KICC 계약 체결 시 forms-as-demo에 브랜딩 복원 + 실데이터 연동.
- 바코드 불출 스캔 + 부품 라벨 시트 (마스터가 원할 때).
- 후면 필름 소모 기록: 엔지니어들이 수리건 수량을 안 올리는 습관(주 51개 차이) — 교육으로 갈지 '작업 소모' 버튼으로 갈지 미결.
- GAS stockLog 보관 줄 수 확대 검토 (현재 3,000줄 → 주간 대조 범위 한계).
