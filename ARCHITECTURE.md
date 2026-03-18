# bansong350 아키텍처 연결 맵

> **프로젝트**: 반송 소나무 350그루 판매
> **최종 업데이트**: 2026-03-18

---

## 파일 구조

```
bansong350/
├── ARCHITECTURE.md                                    # 이 파일
├── dashboard.html                                     # 판매 현황 대시보드
├── 반송 소나무 350그루 판매 프로젝트 - 현황 정리.md       # 매물 기본 정보 + 시세
├── 반송 350그루 판매 - 업체 연락처 종합 리스트.md          # 46개 업체 (7순위)
│
├── docs/                                              # 프로젝트 관리 문서
│   ├── Requirements.md                                # 요구사항
│   ├── Architecture.md                                # 판매 프로세스 구조
│   ├── Decision.md                                    # 의사결정 기록
│   ├── Features.json                                  # 작업 추적 (6개 기능)
│   └── UserStories.md                                 # 시나리오/체크리스트
│
└── _dev/                                              # 비배포 개발 폴더
    ├── 00_planning/                                   # 기획 문서
    ├── 10_docs/                                       # 추가 문서
    ├── 20_tests/                                      # 테스트
    ├── 30_archive/                                    # 아카이브
    ├── 40_sessions/                                   # 세션 요약
    ├── activeContext.md                                # 작업 기억
    └── context-health-log.md                          # 가드레일 점검
```

## 핵심 데이터 흐름

```
[매물 정보] → [업체 연락] → [견적 수집] → [비교/협상] → [계약/인도]
    ↑              ↑             ↓              ↓
현황 정리.md   연락처 리스트.md  dashboard.html  Decision.md
```

## 단일 진실 공급원 (SSOT)

| 정보 | SSOT 파일 |
|------|----------|
| 매물 규격/시세 | 반송 소나무 350그루 판매 프로젝트 - 현황 정리.md |
| 업체 연락처 | 반송 350그루 판매 - 업체 연락처 종합 리스트.md |
| 진행 상태 | dashboard.html + docs/Features.json |
| 의사결정 | docs/Decision.md |
| 작업 기억 | _dev/activeContext.md |
| 업체 수집 이력 | dashboard.html 내 `updateLog` (Supabase `update_log` 컬럼 동기화) |

---

## 업체 정보 수집 방법 (KOSCA 스크래핑)

### 개요

대한전문건설협회(KOSCA) 시공능력평가 조회 페이지에서 브라우저 MCP를 사용하여 업체 리스트를 수집한다.

### 수집 절차

1. **KOSCA 페이지 접속**: `https://www.kosca.or.kr/const/proconst/search.do?menuId=MENU001126`
2. **검색 조건 설정** (브라우저 MCP `form_input` 사용):
   - 지역 셀렉트(`ref_20`): 대구=`22`, 경북=`37`, 경남=`38` 등 (value 코드)
   - 업종: `조경식재·시설물` (value=`65`) — 이미 기본 선택됨
   - 시공능력평가액 하한: `3000000` (30억, 천원 단위)
   - 시군구: `시군구 전체` (기본값)
3. **검색 실행**: `ref_29` (검색 버튼) 클릭
4. **데이터 추출** (JS 실행):
   ```javascript
   // 데이터 테이블: document.querySelectorAll('table.tb')[1] (두 번째 table.tb)
   // 컬럼: td[0]=상호, td[1]=대표, td[2]=주소, td[3]=업종, td[4]=시공능력평가액(천원)
   // 행 onclick: proConstView('memberNo') — 상세 페이지 접근용
   var t = document.querySelectorAll('table.tb')[1];
   var rows = t.querySelectorAll('tr');
   for (var i = 1; i < rows.length; i++) {
     var c = rows[i].querySelectorAll('td');
     // c[0].textContent = 상호명
     // c[2].textContent = 주소 (지역 추출용)
     // c[4].textContent = 시공능력평가액 (천원, 쉼표 제거 필요)
   }
   ```
5. **페이지 이동**: `goPage(n)` JS 함수 호출 → 2초 대기 → 동일 추출 반복
6. **페이지네이션 확인**: `.paging` 요소 텍스트에서 총 페이지 수 파악

### 데이터 변환 규칙

| 항목 | KOSCA 원본 | dashboard.html 저장 |
|------|-----------|-------------------|
| 시공능력평가액 | 천원 단위 (예: `6,493,394`) | **만원 단위** = 천원 ÷ 10 (예: `649339`) |
| 표시 (억) | - | `evalAmount / 10000` → `65억` |
| 지역(region) | 전체 주소 | 시/도 + 구/군만 (예: `대구 달서구`, `경북 구미시`) |
| phone | 목록에 없음 | `"-"` (상세 페이지에서 별도 수집) |
| detail | 대표자명 (목록) | `"대표 홍길동"` 또는 `""` (목록 수집 시 생략 가능) |

### dashboard.html DATA 배열 형식

```javascript
{
  id: Number,        // 기존 마지막 id + 1부터 순차
  p: 4,              // 우선순위 4 = 경북/대구
  g: "경북/대구",     // 그룹명
  name: "업체명",
  phone: "-",        // 상세 수집 전까지 "-"
  region: "대구 달서구",
  evalAmount: 649339, // 만원 단위
  detail: "",        // 목록만 수집 시 빈 문자열
  source: "KOSCA"    // 출처 태그
}
```

### 수집 모드

| 모드 | 설명 | 수집 항목 |
|------|------|----------|
| **목록만** | 검색 결과 테이블에서 추출 | name, region, evalAmount |
| **상세 포함** | `proConstView(memberNo)` 상세 페이지 접근 | + phone, fax, homepage, 대표자 |

> **주의**: 사용자가 "목록만 업데이트해줘"라고 하면 상세 페이지 접근 없이 목록 데이터만 수집.
> 상세 데이터는 사용자의 명시적 요청이 있을 때만 수집.

### 수집 완료 후 필수 작업

1. `dashboard.html`의 `DATA` 배열에 항목 추가 (주석으로 출처/날짜/건수 표기)
2. `updateLog`에 bulk 타입 로그 자동 추가 (boot 함수에 one-time 체크)
3. Supabase `bansong_state.update_log` 동기화 확인

### 수집 이력

| 날짜 | 지역 | 건수 | ID 범위 | 비고 |
|------|------|------|---------|------|
| 2026-03-18 | 경북 | 39건 | 46~84 | 30억+ 조경, 목록만 |
| 2026-03-18 | 대구 | 26건 | 85~110 | 30억+ 조경, 목록만 |
