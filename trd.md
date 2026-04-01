# HQ Account - 전표입력 프로그램 TRD (Technical Requirements Document)

> 버전: v1.0 | 최종 수정일: 2026-04-01

## 1. 시스템 아키텍처

### 1.1 개요
단일 HTML 파일로 구성된 SPA(Single Page Application) 형태의 클라이언트 전용 웹 애플리케이션.
서버 없이 브라우저의 localStorage를 데이터 저장소로 사용한다.

### 1.2 아키텍처 다이어그램

```
┌──────────────────────────────────────────────────┐
│                  브라우저 (Chrome/Edge)            │
│                                                    │
│  ┌──────────────────────────────────────────────┐ │
│  │          전표입력_프로그램.html                 │ │
│  │                                                │ │
│  │  ┌─────────┐  ┌─────────┐  ┌──────────────┐  │ │
│  │  │  HTML   │  │   CSS   │  │  JavaScript  │  │ │
│  │  │ (UI 구조)│  │ (스타일) │  │  (비즈니스   │  │ │
│  │  │         │  │         │  │    로직)      │  │ │
│  │  └─────────┘  └─────────┘  └──────┬───────┘  │ │
│  │                                     │          │ │
│  │                              ┌──────▼───────┐  │ │
│  │                              │ localStorage │  │ │
│  │                              │ ┌───────────┐│  │ │
│  │                              │ │hq_vouchers││  │ │
│  │                              │ ├───────────┤│  │ │
│  │                              │ │hq_accounts││  │ │
│  │                              │ └───────────┘│  │ │
│  │                              └──────────────┘  │ │
│  └──────────────────────────────────────────────┘ │
│                                                    │
│  ┌──────────────────────────────────────────────┐ │
│  │              CSV 파일 다운로드                  │ │
│  │  (Blob URL → <a> download 트리거)              │ │
│  └──────────────────────────────────────────────┘ │
└──────────────────────────────────────────────────┘
```

---

## 2. 기술 스택 상세

### 2.1 프론트엔드

| 항목 | 상세 |
|------|------|
| HTML | HTML5, `lang="ko"` |
| CSS | CSS3, CSS Custom Properties (변수) |
| JavaScript | ES6+ (Vanilla JS, 프레임워크 없음) |
| 외부 의존성 | 없음 |

### 2.2 런타임 환경

| 항목 | 요구사항 |
|------|---------|
| 브라우저 | Chrome 80+, Edge 80+, Firefox 78+, Safari 14+ |
| localStorage | 최소 5MB 지원 필요 |
| 화면 해상도 | 최소 1280×720 권장 |

---

## 3. 파일 구조

```
HQ-Account/
├── 전표입력_프로그램.html       # 메인 애플리케이션 (단일 파일)
├── prd.md                      # 제품 요구사항 정의서
├── 전표입력_프로그램_기획서.md   # 프로젝트 기획서
├── trd.md                      # 기술 요구사항 정의서 (본 문서)
├── 2025년재무제표.pdf           # 참고: 비용 계정과목 목록 기준
└── 2026 월별 지출 내역_v0.1.pdf # 참고: 출력물 형태 기준
```

### 3.1 HTML 파일 내부 구조

```
전표입력_프로그램.html
├── <head>
│   └── <style>                  # 전체 CSS (약 170줄)
├── <body>
│   ├── <nav class="sidebar">   # 좌측 사이드바 네비게이션
│   ├── <div class="main">      # 메인 컨텐츠 영역
│   │   ├── <div id="page-input">     # 전표입력 페이지
│   │   ├── <div id="page-search">    # 전표조회 페이지
│   │   ├── <div id="page-report">    # 월별내역표 페이지
│   │   └── <div id="page-accounts">  # 계정과목관리 페이지
│   ├── <div class="toast">     # Toast 알림
│   ├── <div id="modalContainer"> # 모달 컨테이너
│   └── <script>                 # 전체 JavaScript (약 550줄)
```

---

## 4. 데이터 저장소 설계

### 4.1 localStorage 키 구조

| 키 | 데이터 타입 | 설명 |
|----|-----------|------|
| `hq_vouchers` | JSON Array | 전표 데이터 |
| `hq_accounts` | JSON Array | 계정과목 데이터 |

### 4.2 전표 스키마

```typescript
interface Voucher {
  id: number;       // Auto Increment (전역 nextId 변수)
  date: string;     // "YYYY-MM-DD"
  account: string;  // 계정과목명 (accounts.name 참조)
  amount: number;   // 원 단위 양의 정수
  memo: string;     // 적요 (빈 문자열 허용)
}
```

### 4.3 계정과목 스키마

```typescript
interface Account {
  id: number;        // Auto Increment (전역 acctNextId 변수)
  category: string;  // "수익" | "판매비와관리비" | "영업외수익" | "영업외비용" | "세금"
  name: string;      // 계정과목명 (고유)
  note: string;      // 비고 (빈 문자열 허용)
  active: boolean;   // 활성 여부
}
```

### 4.4 데이터 관계

- 전표의 `account` 필드는 계정과목의 `name` 필드를 문자열로 참조
- 계정과목명 변경 시 해당 이름을 참조하는 모든 전표의 `account` 필드를 일괄 업데이트
- 계정과목 삭제 없음 (비활성화만 가능) → 기존 전표의 참조 무결성 보장

### 4.5 ID 생성 전략

```javascript
// 전표 ID
let nextId = vouchers.length > 0
  ? Math.max(...vouchers.map(v => v.id)) + 1
  : 1;

// 계정과목 ID
let acctNextId = accounts.length > 0
  ? Math.max(...accounts.map(a => a.id)) + 1
  : 1;
```

- 앱 초기화 시 기존 데이터의 최대 ID + 1로 설정
- 단일 사용자 환경이므로 동시성 충돌 없음

### 4.6 초기 데이터

| 구분 | 동작 |
|------|------|
| 계정과목 | localStorage에 `hq_accounts`가 없으면 DEFAULT_ACCOUNTS (19개) 자동 등록 |
| 전표 | localStorage에 `hq_vouchers`가 비어있으면 샘플 데이터 (14건) 자동 등록 |

---

## 5. 모듈 설계

### 5.1 전역 상태 변수

| 변수명 | 타입 | 설명 |
|--------|------|------|
| `vouchers` | Array | 전표 데이터 (localStorage 동기화) |
| `accounts` | Array | 계정과목 데이터 (localStorage 동기화) |
| `nextId` | Number | 다음 전표 ID |
| `acctNextId` | Number | 다음 계정과목 ID |
| `editingId` | Number\|null | 수정 중인 전표 ID |
| `editingAcctId` | Number\|null | 수정 중인 계정과목 ID |
| `csvPendingData` | Array | CSV 미리보기 대기 데이터 |
| `CATEGORY_ORDER` | Array | 분류 정렬 순서 |

### 5.2 함수 목록

#### 저장 함수
| 함수명 | 설명 |
|--------|------|
| `saveToStorage()` | 전표 데이터를 localStorage에 저장 |
| `saveAccountsToStorage()` | 계정과목 데이터를 localStorage에 저장 |

#### 초기화 함수
| 함수명 | 설명 |
|--------|------|
| `init()` (IIFE) | 앱 초기화: 날짜 설정, 연도 드롭다운, 샘플 데이터, 렌더링 |
| `loadSampleData()` | 샘플 전표 14건 등록 |

#### 네비게이션
| 함수명 | 설명 |
|--------|------|
| `showPage(page)` | 페이지 전환 (input/search/report/accounts) |

#### 계정과목 드롭다운
| 함수명 | 설명 |
|--------|------|
| `buildAccountSelect(selectId, includeAll)` | 드롭다운 동적 생성 (활성 계정만, 분류별 optgroup) |
| `refreshAllAccountSelects()` | 전체 계정과목 드롭다운 갱신 |

#### 전표 CRUD
| 함수명 | 설명 |
|--------|------|
| `saveVoucher()` | 전표 저장 (신규/수정 분기) |
| `resetForm()` | 전표 입력 폼 초기화 |
| `editVoucher(id)` | 수정 모드 진입 (폼에 데이터 채움) |
| `confirmDelete(id)` | 삭제 확인 모달 표시 |
| `deleteVoucher(id)` | 전표 삭제 실행 |

#### CSV 일괄 입력
| 함수명 | 설명 |
|--------|------|
| `downloadCsvTemplate()` | CSV 양식 파일 다운로드 |
| `parseCSV(text)` | CSV 텍스트 파싱 → 배열 변환 |
| `validateCsvRow(row)` | CSV 행 유효성 검증 |
| `importCSV()` | CSV 파일 읽기 + 미리보기 표시 |
| `confirmCsvImport()` | 유효한 CSV 행만 일괄 저장 |
| `cancelCsvImport()` | CSV 미리보기 취소 |

#### 전표 조회
| 함수명 | 설명 |
|--------|------|
| `searchVouchers()` | 필터 조건으로 전표 검색 + 결과 렌더링 |
| `resetSearch()` | 검색 필터 초기화 |

#### 월별 내역표
| 함수명 | 설명 |
|--------|------|
| `generateReport()` | 연도별 계정과목×월 피벗 테이블 생성 |
| `exportCSV()` | 내역표를 CSV 파일로 다운로드 |

#### 계정과목 관리
| 함수명 | 설명 |
|--------|------|
| `saveAccount()` | 계정과목 저장 (신규/수정, 이름 변경 시 전표 연동) |
| `resetAccountForm()` | 계정과목 폼 초기화 |
| `editAccount(id)` | 계정과목 수정 모드 진입 |
| `toggleAccountActive(id)` | 계정과목 활성/비활성 토글 |
| `renderAccountTable()` | 계정과목 목록 렌더링 |

#### 유틸리티
| 함수명 | 설명 |
|--------|------|
| `fmt(n)` | 숫자를 천 단위 콤마 문자열로 변환 |
| `formatAmountInput(el)` | 금액 입력 필드 실시간 콤마 포맷팅 |
| `parseAmount(s)` | 콤마 포함 문자열 → 숫자 변환 |

#### UI 컴포넌트
| 함수명 | 설명 |
|--------|------|
| `showToast(msg)` | Toast 알림 표시 (2초 자동 사라짐) |
| `closeModal()` | 모달 닫기 |
| `renderTodayTable()` | 오늘 입력 전표 목록 렌더링 |

---

## 6. UI 설계

### 6.1 CSS 디자인 토큰

```css
:root {
  --primary: #2563eb;        /* 기본 액션 색상 (파란색) */
  --primary-hover: #1d4ed8;  /* 기본 hover */
  --danger: #dc2626;         /* 위험/삭제 (빨간색) */
  --danger-hover: #b91c1c;
  --success: #16a34a;        /* 성공/활성 (초록색) */
  --bg: #f8fafc;             /* 배경색 */
  --card: #ffffff;           /* 카드 배경 */
  --border: #e2e8f0;         /* 테두리 */
  --text: #1e293b;           /* 본문 텍스트 */
  --text-secondary: #64748b; /* 보조 텍스트 */
  --sidebar-bg: #1e293b;     /* 사이드바 배경 (어두운) */
  --sidebar-text: #cbd5e1;   /* 사이드바 텍스트 */
  --sidebar-active: #2563eb; /* 사이드바 활성 메뉴 */
}
```

### 6.2 레이아웃 구조

```
┌─────────┬──────────────────────────────────┐
│         │            TopBar               │
│ Sidebar │──────────────────────────────────│
│ (220px) │                                  │
│         │         Content Area             │
│  메뉴:   │       (스크롤 가능)              │
│  전표입력│                                  │
│  전표조회│     ┌─────────────────────┐      │
│  월별내역│     │      Card 1        │      │
│  계정관리│     └─────────────────────┘      │
│         │     ┌─────────────────────┐      │
│         │     │      Card 2        │      │
│         │     └─────────────────────┘      │
└─────────┴──────────────────────────────────┘
```

### 6.3 컴포넌트 목록

| 컴포넌트 | CSS 클래스 | 설명 |
|----------|-----------|------|
| 사이드바 | `.sidebar` | 좌측 고정 네비게이션 |
| 카드 | `.card` | 콘텐츠 섹션 컨테이너 |
| 테이블 | `table` | 데이터 목록 표시 |
| 버튼 | `.btn`, `.btn-primary`, `.btn-outline`, `.btn-danger`, `.btn-edit` | 액션 버튼 |
| 뱃지 | `.badge`, `.badge-blue`, `.badge-green`, `.badge-red`, `.badge-gray` | 상태/분류 표시 |
| 폼 그리드 | `.form-grid` | 입력 폼 레이아웃 (4컬럼) |
| 요약 바 | `.summary-bar` | 건수/합계 표시 |
| Toast | `.toast` | 알림 메시지 |
| 모달 | `.modal-overlay`, `.modal` | 확인 다이얼로그 |
| 내역표 | `.report-table` | 월별 피벗 테이블 전용 스타일 |

---

## 7. 핵심 알고리즘

### 7.1 월별 피벗 테이블 생성

```
입력: 연도(year)
처리:
  1. vouchers에서 해당 연도 전표 필터링 (date.startsWith(year))
  2. pivot 객체 생성: { 계정과목명: [12개월 금액 배열] }
  3. 각 전표에서 월 추출 (date.slice(5,7)) → pivot[account][month-1] += amount
  4. accounts 배열 순서를 기준으로 행 정렬
  5. 행별 합계(rowTotal), 월별 합계(monthTotals), 총합계(grandTotal) 계산
출력: HTML 테이블 렌더링
```

### 7.2 CSV 파일 생성

```
입력: 테이블 DOM (#reportTable)
처리:
  1. BOM 문자(\uFEFF) 추가
  2. 테이블의 모든 행(rows) 순회
  3. 각 셀의 textContent를 큰따옴표로 감싸고 콤마로 연결
출력: Blob → URL.createObjectURL → <a> download 트리거
```

### 7.3 CSV 파일 파싱 (일괄 입력)

```
입력: CSV 파일 텍스트
처리:
  1. 줄바꿈으로 분리, 빈 줄 제거
  2. 첫 행(헤더) 건너뜀
  3. 각 행을 콤마로 분리, 따옴표 제거
  4. { date, account, amount, memo } 객체로 변환
유효성 검증:
  - 날짜: /^\d{4}-\d{2}-\d{2}$/ 정규식
  - 계정과목: accounts에서 활성 항목 존재 확인
  - 금액: 양수 확인
```

### 7.4 계정과목명 변경 시 전표 연동

```
입력: 변경된 계정과목 (oldName → newName)
처리:
  1. 중복 이름 확인 (다른 ID에 동일 이름 존재 여부)
  2. vouchers 배열에서 account === oldName인 전표 모두 찾기
  3. 해당 전표들의 account를 newName으로 변경
  4. vouchers와 accounts 모두 localStorage에 저장
```

---

## 8. 보안 고려사항

### 8.1 현재 (1단계)

| 항목 | 상태 | 비고 |
|------|------|------|
| 인증/인가 | 없음 | 로컬 단독 사용 |
| 데이터 암호화 | 없음 | localStorage 평문 저장 |
| XSS 방지 | 부분 적용 | innerHTML 사용 시 사용자 입력 이스케이프 필요 |
| CSRF | 해당 없음 | 서버 없음 |

### 8.2 2단계 전환 시 필수 사항
- 서버 사이드 인증/인가
- API 토큰 관리 (.env, 클라이언트 노출 금지)
- 입력값 서버 사이드 검증
- HTTPS 적용

---

## 9. 성능 고려사항

### 9.1 localStorage 용량

| 항목 | 예상 크기 |
|------|----------|
| 전표 1건 | ~100 bytes |
| 계정과목 1건 | ~80 bytes |
| localStorage 제한 | ~5MB (브라우저 기본) |
| 최대 전표 수 (추정) | ~50,000건 |

### 9.2 렌더링 성능
- innerHTML 일괄 교체 방식 (개별 DOM 조작 대비 효율적)
- 전표 조회 시 전체 배열 필터링 → 수천 건 수준에서 성능 이슈 없음
- 월별 내역표는 최대 계정과목 수 × 12 셀 → 경량

### 9.3 알려진 제한사항
- 전표 수가 수만 건을 초과하면 localStorage 직렬화/역직렬화에 지연 발생 가능
- 페이지네이션 미구현 → 전표 조회 시 결과가 많으면 DOM 렌더링 지연 가능
- 브라우저 데이터 초기화 시 모든 데이터 유실

---

## 10. 테스트 전략

### 10.1 수동 테스트 체크리스트

#### 전표입력
- [ ] 정상 입력 후 저장 → 오늘 목록에 반영
- [ ] 필수값 누락 시 Toast 오류 메시지
- [ ] 수정 모드 진입 → 폼에 기존 데이터 채워짐
- [ ] 수정 저장 → 데이터 반영
- [ ] 삭제 → 모달 확인 → 삭제 완료
- [ ] 폼 초기화 버튼 동작

#### CSV 일괄 입력
- [ ] CSV 파일 선택 → 미리보기 표시
- [ ] 유효한 행: OK 뱃지 / 오류 행: 오류 내용 뱃지
- [ ] 확인 등록 → 유효한 행만 저장, 오류 건수 Toast 표시
- [ ] 양식 다운로드 → UTF-8 CSV 파일 생성

#### 전표조회
- [ ] 기간 필터 동작
- [ ] 계정과목 필터 동작
- [ ] 검색 결과 건수/합계 정확성
- [ ] 초기화 버튼 동작

#### 월별내역표
- [ ] 연도 선택 → 해당 연도 데이터만 표시
- [ ] 행 합계 / 열 합계 / 총합계 정확성
- [ ] CSV 다운로드 → 엑셀에서 한글 정상 표시

#### 계정과목관리
- [ ] 신규 등록 → 목록 반영 + 드롭다운 반영
- [ ] 중복 이름 등록 시 오류 메시지
- [ ] 이름 변경 → 기존 전표 계정과목 자동 업데이트
- [ ] 사용안함 토글 → 드롭다운에서 숨김
- [ ] 사용안함 항목 표시 체크박스 동작

---

## 11. 2단계 기술 전환 계획

### 11.1 기술 스택 변경

| 구분 | 1단계 (현재) | 2단계 (계획) |
|------|-------------|-------------|
| 프론트엔드 | 단일 HTML + Vanilla JS | React + TypeScript (Vite) |
| UI | 커스텀 CSS | Ant Design |
| 상태관리 | 전역 변수 + localStorage | TanStack Query |
| 백엔드 | 없음 | NocoDB (REST API) |
| 데이터 저장 | localStorage | NocoDB (Docker) |
| 출력 | CSV | CSV + PDF (jsPDF) |
| 배포 | 파일 복사 | Docker Compose (Nginx + NocoDB) |

### 11.2 데이터 마이그레이션
- localStorage → NocoDB 테이블로 데이터 이관 도구 필요
- 계정과목 → NocoDB SingleSelect 옵션으로 변환
- 전표 → NocoDB 레코드로 일괄 INSERT

### 11.3 API 설계 (NocoDB 기반)

| 작업 | Method | Endpoint |
|------|--------|----------|
| 전표 목록 조회 | GET | `/api/v2/tables/{tableId}/records` |
| 전표 등록 | POST | `/api/v2/tables/{tableId}/records` |
| 전표 수정 | PATCH | `/api/v2/tables/{tableId}/records/{id}` |
| 전표 삭제 | DELETE | `/api/v2/tables/{tableId}/records/{id}` |
