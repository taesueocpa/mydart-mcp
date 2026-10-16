# MyDART

**MyDART**는 금융감독원 전자공시(OpenDART)를 Claude 같은 AI 에이전트에서 바로 조회하고 분석할 수 있게 해주는 원격 MCP(Model Context Protocol) 서버입니다. 단순한 공시·재무·지분 조회를 넘어, 감사보고서 본문(핵심감사사항·계속기업·내부회계관리제도)을 섹션 단위로 파싱해 제공합니다.

---

## 배경 및 문제의식 (Why MyDART?)

금융감독원의 OpenDART는 훌륭한 API지만, AI 에이전트가 그대로 호출해 실무에 쓰기에는 명확한 한계가 있습니다.

### 기존 OpenDART API의 한계
- **불편한 기업 식별자 (`corp_code` 필수):** 회사명이나 종목코드로 바로 조회할 수 없고, 8자리 고유번호를 먼저 찾아야 합니다.
- **API로 제공되지 않는 감사보고서 본문:** 계속기업 불확실성, 강조사항, 핵심감사사항(KAM)처럼 정작 실무에서 가장 중요하게 보는 문단은 API로 조회할 수 없습니다. 정형 API가 반환하는 감사의견은 `적정 / 한정 / 부적정 / 의견거절` 한 단어뿐입니다.
- **공시 의무 및 맥락의 부재:** 상장 여부나 자산 규모에 따라 제출하는 보고서가 다른데, 이 도메인 지식이 없는 AI는 제출 대상이 아닌 회사의 사업보고서를 찾아 헤매는 등 불필요한 호출을 반복합니다.

### MyDART의 해결 방식

#### 1. 지능형 기업명 매핑 (Entity Resolution)
어떤 형태로 기업을 부르든 서버가 알아서 정확한 고유번호로 변환합니다.
- **영문·약칭 변환:** `KT&G` → 케이티앤지
- **종목코드 매핑:** `005380` → 현대자동차 (우선주 코드는 보통주 발행사로 자동 연결)
- **과거 사명 추적:** `두산중공업` → 두산에너빌리티

#### 2. 공시 일정 계산 및 스마트 라우팅
- 기준연도를 명시하지 않아도 결산월과 공시 제출 기한을 계산해 가장 최신 보고서를 자동으로 찾아냅니다.
- 상장 여부, 자산 규모, 금융업 여부에 맞춰 '사업보고서'와 '감사보고서' 중 적절한 공시로 알아서 라우팅합니다.
- 별도·연결 감사보고서는 물론, 내부회계관리제도 운영실태보고서나 정관 같은 핵심 첨부서류까지 직접 파싱합니다.

#### 3. 감사보고서 구조화 DB (Pre-indexed DB)
FY2020부터의 상장사 감사보고서 원문을 전수 수집해 문단 단위로 정교하게 구조화했습니다.
- **본문 섹션 분리:** 감사의견 · 의견근거 · 핵심감사사항(KAM) · 계속기업 불확실성 · 강조사항 · 기타사항
- **내부회계 파싱:** 내부회계관리제도 검토/감사 의견 · 중요한 취약점
- **메타데이터 결합:** 전기말 자산 규모 구간 · 금융업 여부 · 외부감사인 정보

> OpenDART 정형 API가 주는 데이터도 그대로 지원합니다(주요 재무지표 2종·XBRL 택사노미 양식 제외 — [이유](#참고-및-주의사항-limitations)). 파편화된 API를 15개 도구로 통합했으며, 가치평가(Valuation)에 필요한 시세·WACC 입력값·피어그룹(Peer) 멀티플까지 함께 제공합니다.

### 다른 DART 연동 도구와의 차별점
- **단순 API 래퍼(Wrapper)가 아닙니다:** OpenDART API로 볼 수 있는 데이터는 DART 웹 화면의 일부에 불과합니다. MyDART는 정형 API에 없는 **공시 원문과 첨부문서 본문**까지 직접 파싱해 가져옵니다.
- **효율적인 도구 설계:** MCP 표준 규격을 따르며, AI가 질문 의도에 맞는 최적의 도구를 한 번에 선택할 수 있도록 설계했습니다.
- **Zero-Install (원격 서버):** 로컬에 파이썬이나 패키지를 설치할 필요 없이, 엔드포인트 URL만 등록하면 즉시 연결됩니다.
- **무료 운영:** 누구나 무료로 사용할 수 있습니다.

---

## 사용 예시 및 활용 사례

### 💬 이런 질문을 바로 던질 수 있습니다
- *"2025년 감사보고서에 계속기업 불확실성이 기재된 코스닥 상장사는 어디야?"*
- *"2025년 내부회계관리제도에서 중요한 취약점이 지적된 기업 목록과 구체적인 지적 내용을 알려줘."*
- *"카카오의 핵심감사사항(KAM)이 최근 3년 동안 어떻게 바뀌었는지 비교해줘."*
- *"직전연도 자산 2조 원 이상이면서 삼일회계법인이 감사한 기업들의 외부감사 투입시간이 전년 대비 어떻게 변했어?"*

### 💼 실무 활용 시나리오 (Use Cases)
- **보고서 작성 및 크로스체크:** 보고서 작성 중 전기말·전분기 DART 공시 숫자와 빠른 대사(Reconciliation)
- **KAM 리서치:** 핵심감사사항 선정 시 동종업계 유사회사들의 KAM 기재 사례 비교 검색
- **DCF 가치평가:** WACC 산출을 위한 무위험이자율, 베타($\beta$), 피어그룹 재무정보 및 멀티플 즉시 도출
- **감사인 이력 추적:** 특정 기업의 과거 외부감사인 교체 이력 및 감사보수 추이 조회

---

## Quick Start

### 1. eocpa token 발급

[eocpa.kr/token](https://eocpa.kr/token)에서 폼을 제출하면 이메일로 토큰을 받습니다. 토큰 하나로 MyDART · MyKIFRS · MyFSS를 모두 쓸 수 있습니다.

### 2. 커넥터(MCP) 등록

엔드포인트 URL은 `https://mydart.eocpa.kr/mcp` (Streamable HTTP)입니다.

**① claude.ai (웹)**
`Settings` → `Connectors` → `Add custom connector`의 URL 입력란에 토큰을 포함한 주소를 붙여넣습니다.
```text
https://mydart.eocpa.kr/mcp?key=<eocpa_token>
```

**② Claude Desktop · Claude Code**
`mcp-remote`를 사용해 토큰을 헤더로 전달합니다. (URL에 토큰이 남지 않아 안전합니다.)
```json
{
  "mcpServers": {
    "mydart": {
      "command": "npx",
      "args": [
        "-y",
        "mcp-remote",
        "https://mydart.eocpa.kr/mcp",
        "--header",
        "x-dart-api-key:${EOCPA_TOKEN}"
      ],
      "env": {
        "EOCPA_TOKEN": "<eocpa_token>"
      }
    }
  }
}
```

**③ 커스텀 헤더를 지원하는 클라이언트 (Cursor 등)**
요청 헤더에 `x-dart-api-key: <eocpa_token>` 또는 `Authorization: Bearer <eocpa_token>`을 직접 설정합니다.

### 3. 질문하기

> *"카카오 25년 감사보고서 핵심감사사항 알려줘"*

한국어로 질문하면 AI가 문맥에 맞는 도구를 스스로 호출하고, 결과를 분석해 답변합니다.

---

## 제공 도구 (Tools)

총 15개의 도구를 제공합니다. 사용자가 도구를 직접 호출할 일은 없으며, 자연어로 질문하면 AI가 적절한 도구를 선택해 실행합니다. 아래 예시에는 어떤 질문(주석)에 어떤 파라미터가 호출되는지 함께 적었습니다. *(예시의 `20260318001423`은 카카오 2025년 사업보고서 접수번호입니다.)*

| 분류 | 도구명 | 설명 |
| :--- | :--- | :--- |
| **기업·공시** | `find_corp_code` | 회사명·종목코드 → DART 고유번호 조회 |
| | `search_disclosures` | 공시 목록 검색 (기업·기간·공시유형·보고 사유 필터링) |
| | `download_document` | 공시 원문 마크다운 변환 (감사보고서·주석 발췌 및 원문 키워드 검색) |
| | `get_attachments` | 첨부 문서 목록·본문 텍스트·페이지 이미지 조회 |
| **재무·정기보고서** | `get_company_profile` | 기업 개황 및 공시 제출 대상 여부 확인 |
| | `get_financials` | 재무제표 조회 (주요 계정·전체 계정·공시 원본 XBRL) |
| | `get_periodic_report` | 사업보고서 주요 항목 28종 조회 (배당·최대주주·임원보수 등) |
| **주요사항·증권신고서** | `get_corporate_event` | 주요사항보고서 36종 + 지분공시 2종 조회 |
| | `get_registration_statement` | 증권신고서 6종 요약 (공모 조건·인수인·합병비율 등) |
| **감사** | `get_audit_report` | 감사보고서 본문 섹션별 분할 추출 |
| | `get_audit_profile` | 감사인·감사의견·감사보수·업무수행이사 요약 |
| | `get_going_concern` | 계속기업 불확실성 점검 (재무적 징후 5가지 × 감사보고서 기재 대조) |
| | `get_internal_control` | 내부회계관리제도 검토/감사 여부, 의견 및 중요한 취약점 추출 |
| | `search_audit_findings` | 감사의견·KAM·취약점 등 조건별 기업 역검색 (스크리닝) |
| **가치평가** | `get_valuation_data` | 주가·금리 시계열, WACC 입력값, 피어그룹 멀티플 조회 |

---

### `find_corp_code`
회사명·종목코드·고유번호로 DART 고유번호(`corp_code`) 후보를 검색합니다. 다른 도구들은 `corp` 인자에 회사명이나 종목코드를 그대로 넣어도 작동하므로, 이 도구는 주로 이름이 비슷한 동명이인 기업을 구분할 때 사용됩니다.
```javascript
// "SK하이닉스 고유번호 알려줘"
find_corp_code({ query: "SK하이닉스" })

// 종목코드로도 조회 가능 (우선주 코드는 보통주 발행사로 자동 매핑)
find_corp_code({ query: "000660" })

// "'동양'이 들어가는 회사 후보 다 보여줘"
find_corp_code({ query: "동양", limit: 20 })
```

### `search_disclosures`
공시 목록에서 접수번호(`rcept_no`)를 검색합니다. 기업·기간·공시유형으로 필터링하며, 주요사항보고서는 단순 제목 일치가 아니라 DART 표준 '보고 사유'가 정확히 일치하는 공시만 추려냅니다.
```javascript
// "삼성전자 최근 한 달 공시 보여줘"
search_disclosures({ corp: "삼성전자", days: 30 })

// "작년 유상증자 결정 공시 전부 찾아줘"
search_disclosures({ preset: "rights_offering", start: "2025-01-01", end: "2025-12-31" })

// "최근 한 달 전환사채 발행 결정" — 매도·만기전취득 같은 사후 거래는 제외
search_disclosures({ preset: "cb_issuance", days: 30 })

// "최근 6개월 내 사업보고서를 제출한 기업 전부"
search_disclosures({ preset: "annual_report", days: 180 })
```

### `download_document`
공시 원문을 표와 제목 구조가 살아 있는 마크다운(Markdown)으로 변환합니다. 사업보고서 내에 포함된 별도·연결감사보고서만 골라 읽거나, 주석을 번호별로 쪼개 필요한 항목만 발췌할 수 있습니다.
```javascript
// "이 사업보고서 원문 읽어줘"
download_document({ rcept_no: "20260318001423" })

// 목차(제목 + 주석 번호 목록)만 먼저 확인
download_document({ rcept_no: "20260318001423", role: "consolidated_audit", sections: [] })

// "카카오 연결감사보고서에서 특수관계자 주석만 보여줘"
download_document({ rcept_no: "20260318001423", role: "consolidated_audit", sections: ["특수관계자"] })

// "원문에서 횡령·배임 키워드 찾아줘" — 본문과 감사보고서 전체에서 표/문단 단위로 발췌
download_document({ rcept_no: "20260318001423", find: ["횡령", "배임"] })

// 지원 role: main_body(본문) · separate_audit(감사보고서) · consolidated_audit(연결감사보고서) · internal_control(내부회계)
```

### `get_attachments`
공시에 첨부된 문서(정관·영업보고서·내부회계 운영실태보고서 등)의 목록과 본문을 가져옵니다. 표가 이미지로만 들어 있거나 스캔본·직인만 있는 문서는 페이지 이미지로 조회합니다.
```javascript
// 첨부 문서 목록 조회
get_attachments({ rcept_no: "20260318001423" })

// 첨부 문서 본문을 마크다운으로 변환 (dcm_no는 목록 조회 결과에서 확인)
get_attachments({ rcept_no: "20260318001423", dcm_no: "11142684", mode: "text" })

// 표가 이미지이거나 스캔본·도장뿐일 때 — 페이지 이미지로 조회
get_attachments({ rcept_no: "20260318001423", dcm_no: "11142684", mode: "images" })
```

### `get_company_profile`
업종·대표자·설립일·결산월 등 기업 개황을 조회합니다. 해당 기업의 재무·감사 자료가 어떤 형태로 공시되는지(사업보고서 제출 대상인지, 감사보고서만 제출하는지)도 함께 알려줍니다.
```javascript
// "카카오 기업 개황 알려줘"
get_company_profile({ corp: "카카오" })
```

### `get_financials`
재무제표 수치를 조회합니다. 기업명과 연도로 호출하면 최신 정정공시까지 반영된 값을, 접수번호(`rcept_no`)로 호출하면 해당 공시 시점 그대로의 XBRL 원본과 계정 합산 검증 결과를 반환합니다. 모든 금액은 임의 가공 없이 공시된 값 그대로 제공됩니다.
```javascript
// "삼성전자 2025년 매출·영업이익" — 주요 계정 (연결·별도)
get_financials({ corp: "삼성전자", year: 2025 })

// "삼성전자·SK하이닉스·현대차 실적 비교해줘"
get_financials({ corp: ["삼성전자", "SK하이닉스", "현대자동차"], year: 2025 })

// "SK하이닉스 2025년 연결 재무제표 전체 계정"
get_financials({ corp: "SK하이닉스", year: 2025, scope: "full" })

// 현금흐름표·자본변동표 지정 조회 (BS · IS · CIS · CF · SCE)
get_financials({ corp: "SK하이닉스", year: 2025, scope: "full", sj_div: ["CF", "SCE"] })

// "2026년 반기 실적" — report: q1 · half · q3 · annual
get_financials({ corp: "삼성전자", year: 2026, report: "half" })

// "특정 공시본 그대로의 재무제표 + 합산 검증" — 추후 정정이 반영되기 전의 XBRL 원본
get_financials({ rcept_no: "20260318001423" })
```

### `get_periodic_report`
사업·분기·반기보고서의 정형 항목 28종을 조회합니다. 단일 항목 또는 여러 항목을 배열로 묶어 한 번에 호출할 수 있습니다.
```javascript
// "삼성전자 2025년 배당 내역"
get_periodic_report({ corp: "삼성전자", year: 2025, report_type: "dividends" })

// "카카오 최대주주가 누구야?" — shareholders = 최대주주·최대주주 변동·소액주주·주식총수 통합 조회
get_periodic_report({ corp: "카카오", report_type: "shareholders" })

// "현대차 임원 보수" — executive_compensation = 임원 보수 관련 6개 항목 통합 조회
get_periodic_report({ corp: "현대자동차", report_type: "executive_compensation" })

// 여러 항목을 한 번에 조회
get_periodic_report({ corp: "카카오", report_type: ["treasury_stock", "other_company_investment"] })

/* 조회 가능한 항목 (report_type, 28종)
- 주주: largest_shareholder · largest_shareholder_changes · minority_shareholders · total_stocks
- 임직원: executives · employees · outside_director_changes
- 보수: executive_compensation_total · executive_compensation_individual · individual_pay_top5 ·
        unregistered_executive_comp · director_total_comp_approval · director_total_comp_by_type
- 회계감사: auditor_opinion · audit_service_contract · non_audit_service_contract
- 자본: capital_increase_decrease · dividends · treasury_stock
- 자금 사용: private_placement_fund_use · public_offering_fund_use
- 타법인출자: other_company_investment (종속·관계기업 지분 포함)
- 채무증권: debt_securities_issuance · cp_unredeemed · short_term_bond_unredeemed ·
            corp_bond_unredeemed · hybrid_capital_unredeemed · contingent_capital_unredeemed
*/
```

### `get_corporate_event`
주요사항보고서 36종(증자·감자·사채 발행·자기주식·합병·분할 등)과 지분공시 2종(5% 대량보유·임원/주요주주 소유보고)을 조회합니다. 여러 종류를 함께 호출하면 시간순 타임라인으로 병합해 보여줍니다.
```javascript
// "카카오 최근 3년 자본 조달 흐름" — 자본거래 25종을 시간순 타임라인으로 병합
get_corporate_event({ corp: "카카오", mode: "timeline" })

// "2020년부터 자사주 취득·처분 내역"
get_corporate_event({ corp: "카카오", event_type: ["treasury_acquisition", "treasury_disposal"], start: "2020-01-01" })

// "누가 지분을 늘리고 줄였나?" — 5% 대량보유 · 임원/주요주주 소유보고
get_corporate_event({ corp: "카카오", event_type: "holdings" })

/* 조회 가능한 항목 (event_type, 주요사항 36종 + 지분공시 2종)
- 구조조정/소송: 부도·영업정지·회생절차·해산 / 채권은행 관리절차 / 소송
- 자본조달: 유상·무상증자 · 감자 / 전환사채·신주인수권부사채·교환사채·조건부자본증권
- 자사주/지배구조: 자기주식 취득·처분 · 신탁계약 / 합병·분할·분할합병·주식교환
- 자산양수도/기타: 영업·유형자산·타법인주식·주권관련사채권 양수도 / 해외 상장·상장폐지
- 지분공시: 5% 대량보유 · 임원/주요주주 소유보고
*/
```

### `get_registration_statement`
증권신고서의 주요 정보를 신고서 단위로 재구성해 제공합니다 (공모가액·인수인·자금 사용목적·합병비율·주식매수청구권 등).
```javascript
// "대한항공 유상증자 발행 조건"
get_registration_statement({ corp: "대한항공", statement_type: "equity" })

/* statement_type (6종)
equity(지분증권) · debt(채무증권) · depositary(증권예탁증권) · merger(합병) · stock_exchange(주식교환·이전) · division(분할)
*/
```

### `get_audit_report`
감사보고서 본문을 감사의견·의견근거·핵심감사사항(KAM)·강조사항·기타사항·계속기업·내부회계 섹션으로 분리해 추출합니다. 핵심감사사항은 주제별(수익인식·영업권 손상·재고평가 등)로 자동 분류되며, 연결·별도 감사보고서를 함께 조회합니다.
```javascript
// "카카오 25년 감사보고서 핵심감사사항 알려줘"
get_audit_report({ corp: "카카오", year: 2025 })

// 필요한 섹션만 선택 조회 — opinion · basis · kam · emphasis · other_matter · going_concern …
get_audit_report({ corp: "카카오", year: 2025, sections: ["opinion", "kam"] })
```

### `get_audit_profile`
감사인·감사의견(연결/별도)·강조사항·핵심감사사항 건수·감사보수·비감사보수·외부감사 실시내용·업무수행이사를 한눈에 요약합니다.
```javascript
// "카카오 감사인·감사의견·감사보수·업무수행이사 한 번에 요약해줘"
get_audit_profile({ corp: "카카오", year: 2025 })
```

### `get_going_concern`
회계감사기준서 570이 예시하는 재무적 징후 가운데 5가지(자본잠식 · 부의 운전자본 · 연속 영업손실 · 음의 영업현금흐름 · 이자보상배율 1 미만)를 재무 수치로 자동 판정하고, 실제 감사보고서 내 계속기업 불확실성 기재 여부와 대조합니다. **재무적 징후가 켜졌는데 감사보고서 기재가 없는 기업**이 원문을 최우선으로 확인해야 할 대상입니다.
```javascript
// "스타코링크 계속기업 불확실성 여부와 재무적 근거까지 확인해줘"
get_going_concern({ corp: "스타코링크", year: 2025 })
```

### `get_internal_control`
내부회계관리제도 보고서가 '검토' 대상인지 '감사' 대상인지 본문을 바탕으로 판정하고, 의견 및 중요한 취약점을 추출해 재무제표 감사의견과 대조합니다.
```javascript
// "카카오 내부회계관리제도 — 검토야 감사야? 취약점 지적된 거 있어?"
get_internal_control({ corp: "카카오", year: 2025 })
```

### `search_audit_findings`
특정 기업을 지정하지 않아도 감사의견·핵심감사사항·계속기업·내부회계·감사인·자산규모 등의 조건으로 사전 인덱싱된 상장사 감사보고서 DB에서 기업을 스크리닝합니다. 사업보고서를 제출하는 비상장사는 `market: "unlisted"` 옵션으로 조회할 수 있습니다.
```javascript
// "25년 감사의견 의견거절 받은 코스닥 상장사"
search_audit_findings({ year: 2025, market: "kosdaq", opinion: "disclaimer" })

// "계속기업 불확실성이 언급된 코스닥 상장사"
search_audit_findings({ year: 2025, market: "kosdaq", going_concern: true })

// "내부회계 중요한 취약점이 있는 상장사"
search_audit_findings({ year: 2025, icfr_material_weakness: true })

// "삼일회계법인이 감사했고 강조사항이 있는 코스피 회사"
search_audit_findings({ year: 2025, market: "kospi", auditor: "삼일", emphasis: true })

// 여러 해를 한 번에 조회 — opinion: unqualified · qualified · adverse · disclaimer · modified(변형의견 전체)
search_audit_findings({ year: [2024, 2025], opinion: "modified" })

// 사업보고서를 제출하는 비상장사 중 한정의견 기업
search_audit_findings({ year: 2025, market: "unlisted", opinion: "qualified" })
```

### `get_valuation_data`
가치평가(Valuation) 데이터를 `kind` 파라미터로 구분해 제공합니다.
- `market`: 개별 종목 주가·시장 지수·금리 시계열
- `wacc`: $R_f$ · $\beta$ · MRP · SRP · $K_d$ · 자본구조 (항목별 출처 및 기준일 포함)
- `peers`: 동종업계(KSIC 기준) 피어그룹의 EV/EBITDA · PER · PBR · PSR · 무차입 $\beta$
```javascript
// "에스에프에이 최근 1년 주가"
get_valuation_data({ kind: "market", corp: "에스에프에이" })

// 지수·금리 시계열 조회
get_valuation_data({ kind: "market", index: "kospi", freq: "monthly" })
get_valuation_data({ kind: "market", rate_series: "ktb10y" })

// "SK하이닉스 WACC 입력값" — 신용등급·목표 자본구조·MRP는 파라미터로 변경 가능
get_valuation_data({ kind: "wacc", corp: "SK하이닉스", credit_rating: "AA0" })

// "반도체장비 업종 유사회사 멀티플"
get_valuation_data({ kind: "peers", ksic: "29271" })
get_valuation_data({ kind: "peers", corp: "카카오" })
```

---

## 웹에서 바로 보기 (MyDART Viewer)

[**MyDART Viewer**](https://mydart.eocpa.kr/viewer)에서는 사전 인덱싱된 감사보고서 DB를 웹 테이블 형태로 바로 열람할 수 있습니다. 시장·감사의견·핵심감사사항·내부회계·감사인 조건으로 필터링하고, 기업별 다년도 감사인 및 감사의견 추이를 한눈에 비교해 보세요.

## 참고 및 주의사항 (Limitations)

- **원문 대조 권장:** 파싱 및 섹션 분류는 알고리즘에 의한 자동 추출 결과입니다. 실제 감사 판단이나 의사결정의 근거로 삼기 전에 반드시 원문(`get_audit_report`, `download_document`)과 대조하시기 바랍니다.
- **비표준 서식 한계:** 표준 감사보고서 양식과 다른 일부 공시에서는 특정 항목이 빈값으로 반환될 수 있습니다. 값이 비어 있다는 것은 해당 내역이 '없음'이 아니라 파서가 '인식하지 못함'을 의미할 수 있습니다.
- **연차 보고서 중심:** 감사 관련 도구는 회계감사를 받는 '사업보고서(연차)'와 '감사보고서'만 다룹니다. 분기·반기보고서는 감사가 아닌 '검토' 대상이므로, 원문은 `download_document`, 재무 수치는 `get_financials`로 조회해 주세요.
- **지원하지 않는 OpenDART API 3종:**
  - **주요 재무지표 2종(단일회사·다중회사):** 금감원이 XBRL 재무제표로 계산해 주는 수익성·안정성·성장성·활동성 지표(예: 영업이익률)입니다. 2023년 3분기 보고서부터만 제공되고 응답에 계산식이 없습니다. 기간 제약 없이 근거가 분명하도록, 지표는 `get_financials`가 주는 공시 계정 원본에서 직접 계산합니다.
  - **XBRL 택사노미 재무제표양식:** 회사 데이터가 아니라 금감원 회계포탈의 IFRS 공시용 표준 계정과목 체계(양식)입니다. 회사가 실제로 쓴 계정 구조는 `get_financials`(`rcept_no`)가 그 공시의 XBRL 원본에서 직접 복원하므로 따로 넣지 않았습니다. XBRL 원본 파일 자체는 지원합니다.

## 더 보기 (Links)

- **서비스 소개:** [eocpa.kr/mydart](https://eocpa.kr/mydart)
- **함께 쓰는 MCP 서비스:**
  - [**MyKIFRS**](https://github.com/taesueocpa/mykifrs-mcp) — 회계기준·질의회신 검색
  - [**MyFSS**](https://github.com/taesueocpa/myfss-mcp) — 금감원·금융위 회계감독 자료 검색

## 라이선스 (License)

이 저장소의 문서는 [MIT 라이선스](LICENSE)를 따릅니다. (MyDART 서버 소스 코드는 본 저장소에 포함되어 있지 않습니다.) 모든 공시 데이터의 출처는 금융감독원 전자공시시스템(OpenDART)입니다.

© 2026 eocpa.kr