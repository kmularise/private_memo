
좋아요—BOS 회계 리팩토링을 **API 단위로 나눠**보면, 도메인 경계+CQRS(쓰기/읽기 분리) 관점이 깔끔합니다. 아래는 “실무에 바로 쓸 수 있는” 권장 분할안과 핵심 엔드포인트 예시예요.

# 1) 제안 아키텍처(도메인별 API 경계)

1. **Event Ingest API** – 원천 거래/평가 이벤트 수집
    
2. **Identifier API** – 회계처리식별자/CoA 세트 결정
    
3. **Posting API** – 전표 규칙 적용(차/대/부호/금액 산출)
    
4. **Journal API** – 전표 헤더/라인 저장·확정·취소(트랜잭션 경계)
    
5. **Netting API** – 결제/결산 목적 라인 네팅(규칙/수동)
    
6. **Rules Admin API** – CoA/규칙/금액타입/평가영역 설정(버전·유효기간)
    
7. **Reconciliation API** – 보유원장·손익·잔액 대사 및 리포트
    
8. **Interface Adapter API** – 외부 회계시스템 전송/재전송/상태조회
    
9. **Workflow Hook API** – 승인/반려/결제 모듈 연동용 웹훅(상태 머신)
    
10. **Reporting API (Query/BFF)** – 화면/리포트 전용 조회(조인·집계, 읽기 모델)
    

> 각 서비스는 **자체 DB** 보유(스키마 분리), 이벤트(예: Kafka)로 느슨 결합. 쓰기모델(1~8)과 읽기모델(10) 분리.

---

# 2) 핵심 API별 책임 & 대표 엔드포인트

## (1) Event Ingest API

- 책임: 포지션 이벤트 수신/정규화/중복방지.
    
- 주요 모델: `event_id`, `position_tx_id`, `event_type`, `valuation_candidates`, `amount_breakdown[]`(금액타입별), `as_of`.
    
- 엔드포인트
    
    - `POST /events` (idempotent; `Idempotency-Key`)
        
    - `GET /events/{eventId}`
        
    - `GET /events?positionTxId=&dateFrom=&dateTo=`
        

## (2) Identifier API

- 책임: 속성 조합 → **회계처리식별자**·CoA 세트 결정(우선순위·유효기간 반영).
    
- 엔드포인트
    
    - `POST /identify` (입력: 상품유형/회계관리그룹/유가증권분류/채권분류/거래유형/평가영역)
        
    - `GET /identifiers/{id}`
        
    - `POST /resolve-batch` (대량)
        

## (3) Posting API

- 책임: **전표 규칙**(평가영역·CoA 세트별) 적용 → 라인 산출.
    
- 엔드포인트
    
    - `POST /postings/preview` (이벤트→헤더/라인 미리보기)
        
    - `POST /postings/materialize` (전표 후보 생성 요청; 결과는 Journal API로 전달 or 이벤트 발행)
        
- 출력 라인 필드: `dr_cr`, `account_code`, `amount`, `currency`, `amount_type`, `line_attrs{dept, counterparty, account_id…}`, `rule_version`.
    

## (4) Journal API

- 책임: 전표 **저장/확정/취소**의 트랜잭션 경계, 스냅샷/감사로그.
    
- 엔드포인트
    
    - `POST /journals` (헤더+라인 저장; 상태 `DRAFT`)
        
    - `POST /journals/{id}:confirm` (상태 `CONFIRMED`)
        
    - `POST /journals/{id}:void` (원전표 참조 **마이너스 전표** 자동 생성 옵션)
        
    - `GET /journals/{id}`, `GET /journals?status=&valuationArea=&coaSet=&bundleId=`
        

## (5) Netting API

- 책임: 결제/결산 네팅(통화/계좌/카운터파티/상품그룹 기준), 규칙/수동.
    
- 엔드포인트
    
    - `POST /nettings/preview` (대상 라인 집합→미리보기)
        
    - `POST /nettings` (네팅 묶음 생성 → `bundle_id`)
        
    - `DELETE /nettings/{bundleId}` (해제)
        
    - `GET /nettings/{bundleId}`
        

## (6) Rules Admin API

- 책임: **설정화**(하드코딩 금지): CoA 세트, 전표규칙, 금액타입, 평가영역, 회계식별자.
    
- 엔드포인트(모두 버전/유효기간 관리)
    
    - `POST/PUT/GET /rules/journal` (헤더/라인 DSL 또는 테이블 스키마)
        
    - `POST/PUT/GET /rules/identifiers`
        
    - `POST/PUT/GET /catalog/coa-sets`, `/catalog/accounts`, `/catalog/amount-types`, `/catalog/valuation-areas`
        
    - `POST /rules/simulate` (변경 영향 분석/디프)
        

## (7) Reconciliation API

- 책임: **보유원장·손익·잔액 대사** 자동화, 실패 케이스 리포팅.
    
- 엔드포인트
    
    - `POST /recon/positions-vs-ledger` (자산 누적=장부가)
        
    - `POST /recon/pnl-rollup` (손익 집계 일치성)
        
    - `GET /recon/reports?date=&scope=`
        

## (8) Interface Adapter API

- 책임: 외부 회계시스템별 **포맷 변환/전송/상태 동기화**.
    
- 엔드포인트
    
    - `POST /adapters/{system}/dispatch` (입력: `journal_id` or `bundle_id`)
        
    - `GET /adapters/{system}/status?externalRef=`
        
    - `POST /adapters/{system}/replay` / `:cancel` (정책 상 허용 시)
        

## (9) Workflow Hook API

- 책임: 승인/반려 등 **상태머신** 훅(외부 결재 연동).
    
- 엔드포인트
    
    - `POST /hooks/approval-callback` (payload: `journal_id`, `action=APPROVED|REJECTED`, `actor`)
        
    - `GET /workflows/{journalId}` (히스토리)
        

## (10) Reporting API (BFF/Query)

- 책임: 화면 전용 질의 모델(조인·집계·필터), UI 성능 보장.
    
- 엔드포인트
    
    - `GET /q/journals` (다중 필터+페이지)
        
    - `GET /q/journals/{id}/trace` (식별자/규칙 적용 근거)
        
    - `GET /q/nettings/{bundleId}/preview`
        
    - `GET /q/pnl-summary`, `/q/balance-recon`
        

---

# 3) 이벤트(비동기) 모델

- 토픽 예시
    
    - `event.ingested`, `identifier.resolved`, `posting.materialized`
        
    - `journal.confirmed`, `journal.voided`
        
    - `netting.created`, `adapter.dispatched`, `adapter.ack`
        
    - `rules.version.published`, `recon.report.ready`
        
- 이점: **확장성/감사성/리플레이**, 사이트별 커스텀 어댑터 플러깅 용이.
    

---

# 4) 공통 설계 원칙

- **버전·유효기간 기반 설정**(룰/CoA/식별자/금액타입) + **결정성 보장**(입력+룰버전=출력 고정)
    
- **Idempotency-Key**, 멱등 응답, **분산 트레이싱(traceId)**, **감사 로그**(입력/룰 버전/출력 스냅샷)
    
- **취소 전표 표준화**: 원전표 링크, 상쇄 라인 자동 생성, 외부 정책 대응
    
- **통화 정책**: 원화/외화 동시 라인 지원, 환율 소스·시점 명시
    
- **권한/보안**: 서비스 간 mTLS, JWT(서비스·사용자 클레임), 역할(조회/확정/취소/전송/룰관리)
    
- **스키마 레지스트리**(Avro/JSON Schema) + 계약 테스트(Consumer-Driven)
    

---

# 5) DTO 예시(간단)

**전표규칙 라인(Rules Admin)**

`{   "rule_id": "JR-IFRS-001",   "version": "2025-09-01",   "valuation_area": "IFRS",   "coa_set": "LOCAL",   "event_type": "INTEREST_RECEIVED",   "lines": [     { "side": "DR", "account_code": "112100", "amount_type": "RECEIVABLE_INTEREST", "sign": "+", "attrs": {"dept_required": true} },     { "side": "CR", "account_code": "510100", "amount_type": "INTEREST_INCOME", "sign": "+", "attrs": {} }   ] }`

**전표 결과(Journal API)**

`{   "journal_id": "J20250926-000123",   "status": "CONFIRMED",   "valuation_area": "IFRS",   "coa_set": "LOCAL",   "header": { "bundle_id": "BN-001", "position_tx_id": "PTX-789", "bus_trans_id": "BT-456" },   "lines": [     { "line_no": 1, "dr_cr": "DR", "account_code": "112100", "amount": {"value": 10000000, "ccy": "KRW"}, "amount_type": "RECEIVABLE_INTEREST" },     { "line_no": 2, "dr_cr": "CR", "account_code": "510100", "amount": {"value": 10000000, "ccy": "KRW"}, "amount_type": "INTEREST_INCOME" }   ],   "rule_version": "JR-IFRS-001@2025-09-01" }`

---

# 6) 마이그레이션/릴리즈 전략

- **Strangler 패턴**: As-Is는 그대로, 신규 **Posting/Journal/Adapter**를 병행 도입
    
- 초기엔 **기본 회계규칙(001)**만 적용 → 리포트 비교 → 점진적으로 **평가영역/CoA세트/네팅** 확장
    
- 각 배포마다 **Rules Simulator**로 영향 분석 후 프로덕션 활성화(피처 플래그)
    

---

# 7) 화면(BFF) 연결 팁

- BFF는 **Reporting API** 단일 창구로 묶고, 쓰기 작업은 개별 서비스로 직접 호출
    
- 상세 화면에 **룰 트레이스(Identifier·Rule 선택 근거)** 노출 → 디버깅/감사 용이
    
- 네팅 UI는 `preview → commit`의 2단계 패턴으로 안전성 확보
    

---

필요하시면 이 구조로 **OpenAPI 명세(yaml)**와 **서비스별 ERD**를 바로 뽑아드릴게요. 우선 어떤 API부터 정의할까요—`Posting/Journal`(핵심 경계)부터 갈까요, 아니면 `Rules Admin`(하드코딩 제거)부터?