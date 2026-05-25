# 청취계산기 (chungchi_cal) — CLAUDE.md

## 프로젝트 개요

**청취 10a당 생산량 계산기** — 농작물 현장 청취조사 시 10a(아르)당 생산량을 환산하고, KOSIS 국가통계포털 기준값과 비교하는 모바일 웹 도구입니다. 농업 현장 조사원이 스마트폰에서 직접 사용하는 것을 전제로 설계되었습니다.

- **배포 URL:** `ubaldos00-ship-it.github.io/` (GitHub Pages 정적 배포)
- **홈 화면:** `ubaldos00-ship-it.github.io/home`
- **현재 운영 파일:** `index.html` (v40, 2026-05-25)

---

## 파일 구조

```
chungchi_cal/
├── index.html              # 운영 버전 (v40) — 가장 최신, 기능 완전체
├── 청취계산기_v3.html       # v3 리디자인 — CSS 변수 기반 클린 UI, 저장 기능 없음
└── new(청취계산기).html     # 경량 심플 버전 — 저장 기능 없음
```

> **핵심:** `index.html`이 실제 서비스 파일입니다. 새 기능 추가나 수정은 이 파일에 합니다.

---

## 기술 스택

빌드 시스템, 패키지 매니저, 프레임워크 **없음**. 순수 HTML + CSS + JavaScript 단일 파일입니다.

### 외부 CDN 의존성 (index.html)

| 라이브러리 | 버전 | 용도 |
|---|---|---|
| Chart.js | 4.4.1 | KOSIS 5년 꺾은선 그래프 |
| chartjs-plugin-datalabels | 2.2.0 | 그래프 위 수치 레이블 |
| html2canvas | 1.4.1 | 결과 이미지 캡처/저장 |

`청취계산기_v3.html`과 `new(청취계산기).html`은 html2canvas만 사용합니다.

---

## 핵심 도메인 지식

### 계산 공식

```
10a당 생산량(kg) = 수확량 합계(kg) ÷ 재배면적(㎡) × 1,000
```

### 단위 환산

- **1평 = 3.3058㎡** (상수: `PY2M2 = 3.3058`)
- 1a(아르) = 100㎡, 10a = 1,000㎡

### 지원 작물 및 기본 수확단위

| 작물 | 기본단위 | 단위중량(kg) |
|---|---|---|
| 사과, 배 | 컨테이너 | 18 |
| 고추 | 근 | 0.6 |
| 참깨 | 되 | 1.2 |
| 고구마, 가을감자 | 박스 | 10 |
| 겉보리, 쌀보리, 밀 | 톤백 | 500 |

### 지원 지역

**UI에서 선택 가능:** 대구, 경북, 강원 (기본값: 경북)

`index.html`의 `KD` 데이터 객체에는 전국 18개 시도 데이터가 모두 포함되어 있습니다.

### KOSIS 데이터

출처: KOSIS 농작물생산조사 (2021~2025). 연도별 10a당 생산량(kg/10a) 하드코딩.

- **전년값:** 배열에서 마지막 유효값 (`gLast` / `lastValid` 함수)
- **평년값:** 유효값들의 산술평균 (`gAvg` / `calcAvg` 함수)
- `null`은 해당 지역·연도 데이터 없음을 의미합니다

---

## index.html 아키텍처

### 전역 상태 변수

```js
let sR = '경북';   // 선택된 지역 (index.html)
let sC = null;     // 선택된 작물명 문자열 (index.html)

let selectedRegion = '경북';  // 선택된 지역 (v3/new)
let selectedCrop   = null;    // 선택된 작물 객체 (v3/new)
let qtyRows        = [];      // 수확량 행 배열 (v3/new)
```

### 주요 함수 (index.html)

| 함수 | 역할 |
|---|---|
| `selReg(b)` | 지역 버튼 선택 |
| `selCrop(b)` | 작물 버튼 선택, 드럼 피커 재빌드 포함 |
| `cA(inp)` / `cAR(inp)` | 평↔㎡ 양방향 환산 |
| `uAT()` | 재배면적 합계 업데이트 |
| `bldHR()` / `addHR()` | 수확량 행 추가 |
| `rRow(r)` | 수확량 행 단위 × 수량 계산 |
| `uHT()` | 수확량 합계 업데이트 |
| `uRes()` | 10a당 생산량 계산·표시 |
| `uAll()` | uKT + uRC + uCh 일괄 업데이트 |
| `uKT()` | KOSIS 꺾은선 그래프 + 통계표 렌더 |
| `uRC()` | 최종 결과 요약 카드 렌더 |
| `uCh()` | 지역별 수평 바 차트 렌더 |
| `drawKosisLine(rgs)` | Chart.js 인스턴스 생성/교체 |
| `saveImg()` | html2canvas 캡처 후 저장 모달 |
| `resetAll()` | 전체 초기화 |
| `goHome()` | 데이터 클리어 후 홈 이동 |

### 저장/불러오기 시스템 (index.html 전용)

localStorage 키:

| 키 | 내용 |
|---|---|
| `chungchiAutoSave_v1` | 자동저장 (JSON) |
| `chungchiSaveList` | 수동 저장 목록 메타데이터 (JSON 배열) |
| `chungchiSave_[timestamp]` | 수동 저장 개별 데이터 |

- 수동 저장은 최대 30개 유지 (초과 시 오래된 것 자동 삭제)
- `_ccCollect()` — 현재 DOM 상태 수집 → JSON
- `_ccApply(d)` — JSON 데이터 → DOM 복원
- `ccAutoSave()` — `_ccRestoring`, `_ccInitDone`, `_allowLeave` 플래그 확인 후 저장

### 자동저장 보호 플래그

```js
window._ccRestoring   // true: 복원 중 — 새 저장 차단
window._ccInitDone    // false: 초기 복원 완료 전 — 새 저장 차단
window._allowLeave    // true: 홈 이동/초기화 중 — 이탈 저장 차단
```

이 플래그를 우회하면 저장 데이터가 빈 DOM으로 덮어씌워질 수 있습니다.

### 드럼 피커 UI

`WO` 객체에 작물별 선택 가능한 중량 옵션이 정의되어 있습니다. 옵션이 2개 이상이면 드럼 피커(터치 스크롤 방식), 1개이면 일반 숫자 입력창이 렌더됩니다.

---

## UI/스타일 컨벤션

### 공통

- **최대 너비:** 430px (모바일 퍼스트, 데스크톱에서는 중앙 정렬)
- **기본 배경:** `#27500A` (짙은 녹색 헤더), `#f4f7f4` (본문)
- **주조색:** `#27500A` (딥 그린), `#27ae60` (라이트 그린)
- 폰트: `Apple SD Gothic Neo`, `Noto Sans KR`, `Malgun Gothic` 순 fallback

### index.html CSS 클래스 패턴

CSS는 미니파이된 단자 클래스명 방식입니다:

| 클래스 | 역할 |
|---|---|
| `.card` | 섹션 카드 컨테이너 |
| `.ct` | 카드 타이틀 바 |
| `.brg` / `.brg.on` | 지역 선택 버튼 / 선택됨 |
| `.bc` / `.bc.on` | 작물 선택 버튼 / 선택됨 |
| `.ar` | 재배면적 입력 행 |
| `.hr2` | 수확량 입력 행 |
| `.pi` / `.si` | 평(坪) 입력 / ㎡ 입력 |
| `.hq` / `.hw` / `.hres` | 수확수량 / 단위중량 / 환산결과 |
| `.needs-input` | 미입력 시 황색 깜빡임 강조 |
| `.tb` | 소계 요약 박스 |
| `.rc` | 최종 결과 요약 카드 |
| `.cc` | 비교 데이터 카드 |
| `.rb` | 행 삭제 버튼 (빨간 원형) |

### 청취계산기_v3.html / new(청취계산기).html

CSS 커스텀 프로퍼티(`--accent`, `--text`, `--bg` 등) 기반의 클린 스타일입니다. 상태 클래스는 `.active`로 통일되어 있습니다.

---

## 작업 가이드라인

### 데이터 업데이트 (KOSIS 새 연도 추가)

**index.html:**
```js
// 1. KD 객체의 각 작물·지역 배열에 새 연도 값을 추가합니다
const KD = {
  '사과': { '경북': [1876, 2108, 1648, 2024, 2053, /* 2026 신규값 */], ... }
};
// 2. 그래프 labels 배열을 찾아 연도를 추가합니다
labels: ['2021','2022','2023','2024','2025', '2026']
// 3. 통계표 헤더의 연도 <th>도 업데이트합니다
```

**청취계산기_v3.html / new(청취계산기).html:**
```js
const YEARS = ['2021','2022','2023','2024','2025']; // 연도 추가
const CROPS = [
  { id:'apple', d:{'대구':[...], '경북':[...], '강원':[...]} }
  // 각 배열에 새 값 추가
];
```

### 새 작물 추가

1. `KD` (또는 `CROPS`) 객체에 작물 데이터 추가
2. `CU` 객체에 기본 수확단위/중량 추가
3. `WO` 객체에 선택 가능한 중량 옵션 배열 추가
4. HTML 버튼 그리드에 `<button class="bc" data-crop="...">` 추가

### 새 지역 추가

1. `KD` 객체 각 작물에 해당 지역 배열 추가
2. `EM` 객체에 지역 이모지 추가
3. HTML 지역 선택 버튼(`<button class="brg">`) 추가

### iOS 호환 주의사항

- `touchstart` / `touchmove` 이벤트는 `passive:false`로 처리해야 `preventDefault()` 동작
- `beforeunload`는 iOS Safari에서 미지원 → `pagehide` + `visibilitychange`로 보완
- 뒤로가기 차단은 `history.pushState` 기반 4중 방어 구조 (함부로 제거하지 마세요)
- 입력 포커스 후 키보드 표시를 위해 `setTimeout(..., 30)` + `inp.click()` 패턴 사용

---

## 버전 관리 패턴

파일 내 헤더에 버전과 날짜가 직접 기입됩니다:

```html
<span class="hd-ver">(v40: 2026-05-25 06:48)</span>
```

기능 변경 시 버전 번호와 날짜를 함께 업데이트하세요.

---

## 배포

별도 빌드 단계 없음. `index.html`을 직접 편집 후 GitHub에 push하면 GitHub Pages에 즉시 반영됩니다.

```bash
git add index.html
git commit -m "설명"
git push -u origin <브랜치명>
```
