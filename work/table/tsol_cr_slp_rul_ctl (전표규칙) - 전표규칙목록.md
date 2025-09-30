이거 전표 규칙인데 이것도 차/대 나타나고 있음


좋아요. 지금 구조/데이터를 보면 “차대분개쌍(D/C) 단위의 행”을 저장하려는 의도가 보이는데, PK 구성과 코드/자료형/제약이 그 의도와 어긋나는 부분이 있어요. 바로 손댈 포인트만 콕 집어 정리하고, 개선 DDL 샘플도 드릴게요.

# 개선 포인트 (핵심)

1. **PK 불일치 (중복 허용 문제)**
    

- 데이터는 동일 `ist_cd + asj_tbl_cd + asj_cd`에 대해 `dr_cds_tc=1/2` 두 행이 존재합니다.
    
- 현재 PK는 `(ist_cd, asj_cd, asj_tbl_cd)`라서 **D/C 두 행을 동시에 저장 못 함** → **`dr_cds_tc`를 PK에 포함**하거나, 아예 **한 행에 (debit_account, credit_account)로 합치는 구조** 중 하나로 정해야 합니다.
    
    - 지금처럼 “분개행 단위”로 간다면 **PK = (ist_cd, asj_tbl_cd, asj_cd, dr_cds_tc)** 로 가는 게 자연스럽습니다.
        

2. **코드 컬럼 타입/값 정합성**
    

- `dr_cds_tc`가 실제론 D/C 의미인데 값이 `1/2`로 쓰인 레코드도 보입니다. → **값 도메인 통일(‘D’/‘C’) + CHECK 제약** 필요.
    
- `asj_tc`, `stm_ats_tc`도 코드셋입니다. → **코드마스터 FK** 또는 **CHECK 제약**으로 값 통제하세요.
    

3. **불리언/네트워크/시간 타입 정규화**
    

- `del_yn varchar(1)` → **boolean** (default false).
    
- `fst_enr_ip`, `lst_chg_ip` → **inet**.
    
- `timestamp` → **timestamptz**(서버/타임존 혼동 방지).
    

4. **NOT NULL/길이/도메인 제약 강화**
    

- `asj_nm`, `asj_abv_nm`는 **NOT NULL** 권장(마스터 성격).
    
- `ist_cd(3)`, `asj_tbl_cd(10)`, `asj_cd(10)`은 **업무 규격에 맞는 길이** 재검토(고정 패턴이면 `varchar` 대신 `char`도 고려).
    
- **이름 길이 200**은 과합니다. 100/60 등 현실 길이로 다운, 또는 text로 전환+업무 밸리데이션.
    

5. **감사(등록/수정) 컬럼 공통화**
    

- 동일 패턴 컬럼이 반복되니 **트리거(ON INSERT/UPDATE)** 로 자동 세팅; 수동 기본값은 지양.
    

6. **인덱스 전략**
    

- 조회 패턴 상 빈번한 `(ist_cd, asj_tbl_cd)` / `(ist_cd, asj_cd)` 조합에 **보조 인덱스** 추가.
    
- 이름 검색이 있으면 `asj_nm`에 **GIN(trigram)** 고려.
    

7. **명칭 일관성/국영문 분리**
    

- 표시명/약어/영문명을 분리 (`asj_nm_kr`, `asj_nm_en`, `asj_abbr`)하면 보고서/연계 편해집니다.
    

8. **참조무결성**
    

- `ist_cd` → 기관 마스터 FK, `asj_tbl_cd` → “계정과목표(버전/체계)” 마스터 FK, 각 코드 컬럼 → 코드마스터 FK로 **무결성 보강**.
    

---

# 개선 DDL 예시 (분개행 단위 유지안)

`-- 코드/마스터 테이블 예시 (요지) -- 기관 CREATE TABLE cm_inst (   ist_cd varchar(3) PRIMARY KEY,   ist_nm varchar(100) NOT NULL );  -- 계정과목표(버전/체계) CREATE TABLE cm_asj_table (   asj_tbl_cd varchar(10) PRIMARY KEY,   asj_tbl_nm varchar(100) NOT NULL );  -- 차대 구분 코드 CREATE TABLE cd_drcr (   dr_cds_tc char(1) PRIMARY KEY,  -- 'D' or 'C'   dr_cds_nm varchar(20) NOT NULL );  -- 계정과목 구분 코드 CREATE TABLE cd_asj_tc (   asj_tc varchar(10) PRIMARY KEY,   asj_tc_nm varchar(50) NOT NULL );  -- 결제계정 구분 코드 CREATE TABLE cd_stm_ats_tc (   stm_ats_tc varchar(10) PRIMARY KEY,   stm_ats_nm varchar(50) NOT NULL );  -- 메인 테이블 DROP TABLE IF EXISTS tsol_cr_asj_ctl CASCADE; CREATE TABLE tsol_cr_asj_ctl (   ist_cd        varchar(3)   NOT NULL,   asj_tbl_cd    varchar(10)  NOT NULL,   asj_cd        varchar(20)  NOT NULL,          -- 여유있게 (예: 'LF009', 'S7101' 등)   dr_cds_tc     char(1)      NOT NULL,          -- 'D'/'C'   asj_nm        varchar(120) NOT NULL,          -- 실제 쓰는 길이로 타이트하게   asj_abv_nm    varchar(60),                    -- 약어   asj_tc        varchar(10),                    -- FK to cd_asj_tc   stm_ats_tc    varchar(10),                    -- FK to cd_stm_ats_tc   del_yn        boolean      NOT NULL DEFAULT false,   fst_enr_usid  varchar(32)  NOT NULL,   fst_enr_dtm   timestamptz  NOT NULL DEFAULT now(),   fst_enr_trid  varchar(64)  NOT NULL,   fst_enr_ip    inet         NOT NULL DEFAULT '0.0.0.0',   lst_chg_usid  varchar(32)  NOT NULL,   lst_chg_dtm   timestamptz  NOT NULL DEFAULT now(),   lst_chg_trid  varchar(64)  NOT NULL,   lst_chg_ip    inet         NOT NULL DEFAULT '0.0.0.0',   CONSTRAINT pk_tsol_cr_asj_ctl     PRIMARY KEY (ist_cd, asj_tbl_cd, asj_cd, dr_cds_tc),   CONSTRAINT fk_tsol_cr_asj_ctl_inst     FOREIGN KEY (ist_cd) REFERENCES cm_inst(ist_cd),   CONSTRAINT fk_tsol_cr_asj_ctl_asjtbl     FOREIGN KEY (asj_tbl_cd) REFERENCES cm_asj_table(asj_tbl_cd),   CONSTRAINT fk_tsol_cr_asj_ctl_drcr     FOREIGN KEY (dr_cds_tc) REFERENCES cd_drcr(dr_cds_tc),   CONSTRAINT fk_tsol_cr_asj_ctl_asjtc     FOREIGN KEY (asj_tc) REFERENCES cd_asj_tc(asj_tc),   CONSTRAINT fk_tsol_cr_asj_ctl_stm     FOREIGN KEY (stm_ats_tc) REFERENCES cd_stm_ats_tc(stm_ats_tc),   CONSTRAINT ck_tsol_cr_asj_ctl_nm     CHECK (length(asj_nm) > 0) );  -- 조회 인덱스 CREATE INDEX ix_tsol_cr_asj_ctl_tbl ON tsol_cr_asj_ctl (ist_cd, asj_tbl_cd); CREATE INDEX ix_tsol_cr_asj_ctl_cd  ON tsol_cr_asj_ctl (ist_cd, asj_cd);  -- (선택) 이름 검색 최적화 -- CREATE EXTENSION IF NOT EXISTS pg_trgm; -- CREATE INDEX ix_tsol_cr_asj_ctl_nm_trgm ON tsol_cr_asj_ctl USING gin (asj_nm gin_trgm_ops);`

> 만약 “한 건당 차대쌍을 한 행에” 담고 싶다면:  
> `dr_cds_tc` 컬럼을 없애고 `dr_acct_cd`, `cr_acct_cd` 두 컬럼으로 설계하고, 평가/처분/세부 구분은 별도 컬럼으로 분리하는 방식이 더 적합합니다(분개템플릿 테이블 설계).

---

# 데이터 측면 체크리스트

- **dr_cds_tc 값 통일**: `1/2` → 반드시 `D/C`로 이관.
    
- **의미 중복 제거**: `asj_nm`에 “/”로 두 계정을 붙여 쓰는 데이터가 많습니다(예: “미수금/이자수익”). 이건 **표시문구**일 뿐이므로, 실제 분개 설계는 **행/컬럼으로 계정이 분리**되어 있어야 합니다.
    
- **더미(DUMMY) 데이터**: 운영계 반영 전 **테스트 코드 분리**(별 asj_tbl_cd=‘TEST’) 또는 삭제.
    
- **코드 길이 혼재**(숫자/문자 혼용 ‘14600’, ‘LF009’ 등): `asj_cd`는 **문자형으로 통일**하고, **포맷 규칙**(접두/자리수) 문서화.
    
- **시간대 혼재**: `timestamp` → `timestamptz`로 이관 시점에 **서버/로컬 기준 시간 변환** 주의.
    

---

원하시면, **차대쌍을 한 행으로 묶는 ‘분개템플릿’ 모델**(예: `entry_template(id, ist_cd, tbl_cd, code, dr_acct, cr_acct, ... )`)도 바로 설계해서 드릴게요. 지금 데이터 패턴상 실제 전표 생성 로직엔 그 편이 더 단순할 가능성이 큽니다.

##### 나의 말:

테이블이 다 약어라 못알아듣겠어 full name으로 쓰면서 설명부탁

###### ChatGPT의 말:

15s 동안 생각함

좋아요—약어를 모두 “풀네임(영문 스네이크케이스)”로 풀고, 각 컬럼이 무슨 뜻인지 바로 이해되도록 설명까지 붙여서 정리해 드릴게요. 아래 예시는 이전에 제안했던 개선사항(차/대 구분을 PK에 포함, 타입 정규화 등)을 반영한 **가독성 좋은 DDL**입니다.

# 권장 스키마 (풀네임 + 설명 강화)

`-- 기관, 계정체계 등 참조마스터는 예시 (필요 시 생략/대체 가능) CREATE TABLE cm_institution (   institution_code varchar(3) PRIMARY KEY,        -- 기관 코드   institution_name varchar(100) NOT NULL );  CREATE TABLE cm_account_schema (   account_schema_code varchar(10) PRIMARY KEY,    -- 계정과목 '표/체계' 코드(버전/프레임)   account_schema_name varchar(100) NOT NULL );  -- 차/대 구분 코드 ('D'/'C') CREATE TABLE cd_debit_credit_type (   debit_credit_type char(1) PRIMARY KEY,          -- 'D' or 'C'   debit_credit_name varchar(20) NOT NULL );  -- (선택) 계정과목 구분, 결제계정 구분 코드마스터 CREATE TABLE cd_account_category (   account_category_code varchar(10) PRIMARY KEY,   account_category_name varchar(50) NOT NULL );  CREATE TABLE cd_settlement_account_type (   settlement_account_type_code varchar(10) PRIMARY KEY,   settlement_account_type_name varchar(50) NOT NULL );  ---------------------------------------------------------------- -- 메인 테이블: tsol_cr_asj_ctl (읽기 어려운 약어 → 풀네임으로 변경) ----------------------------------------------------------------  DROP TABLE IF EXISTS tsol_chart_of_accounts_entry CASCADE; CREATE TABLE tsol_chart_of_accounts_entry (   institution_code              varchar(3)   NOT NULL,           -- 기관 코드 (ist_cd)   account_schema_code           varchar(10)  NOT NULL,           -- 계정체계/표 코드 (asj_tbl_cd)   account_code                  varchar(20)  NOT NULL,           -- 계정과목 코드 (asj_cd) 예: 'LF009','S7101','14600'   debit_credit_type             char(1)      NOT NULL,           -- 차/대 구분 ('D','C') (dr_cds_tc)   account_name                  varchar(120) NOT NULL,           -- 계정과목명 (asj_nm)   account_name_abbrev           varchar(60),                      -- 계정과목 약어명 (asj_abv_nm)   account_category_code         varchar(10),                      -- 계정과목 '분류' 코드 (asj_tc)   settlement_account_type_code  varchar(10),                      -- 결제계정 구분 코드 (stm_ats_tc)    is_deleted                    boolean      NOT NULL DEFAULT false,  -- 삭제여부 (del_yn)   created_by_user_id            varchar(32)  NOT NULL,               -- 최초등록사용자ID (fst_enr_usid)   created_at                    timestamptz  NOT NULL DEFAULT now(), -- 최초등록일시 (fst_enr_dtm)   created_by_transaction_id     varchar(64)  NOT NULL,               -- 최초등록거래ID (fst_enr_trid)   created_from_ip               inet         NOT NULL DEFAULT '0.0.0.0', -- 최초등록IP (fst_enr_ip)    updated_by_user_id            varchar(32)  NOT NULL,               -- 최종변경사용자ID (lst_chg_usid)   updated_at                    timestamptz  NOT NULL DEFAULT now(), -- 최종변경일시 (lst_chg_dtm)   updated_by_transaction_id     varchar(64)  NOT NULL,               -- 최종변경거래ID (lst_chg_trid)   updated_from_ip               inet         NOT NULL DEFAULT '0.0.0.0', -- 최종변경IP (lst_chg_ip)    CONSTRAINT pk_tsol_chart_of_accounts_entry     PRIMARY KEY (institution_code, account_schema_code, account_code, debit_credit_type),    CONSTRAINT fk_coa_entry_institution     FOREIGN KEY (institution_code) REFERENCES cm_institution(institution_code),    CONSTRAINT fk_coa_entry_schema     FOREIGN KEY (account_schema_code) REFERENCES cm_account_schema(account_schema_code),    CONSTRAINT fk_coa_entry_drcr     FOREIGN KEY (debit_credit_type) REFERENCES cd_debit_credit_type(debit_credit_type),    CONSTRAINT fk_coa_entry_category     FOREIGN KEY (account_category_code) REFERENCES cd_account_category(account_category_code),    CONSTRAINT fk_coa_entry_settlement     FOREIGN KEY (settlement_account_type_code) REFERENCES cd_settlement_account_type(settlement_account_type_code),    -- 기본 밸리데이션   CONSTRAINT ck_coa_entry_account_name_nonempty CHECK (length(account_name) > 0) );  -- 자주 쓰는 조회를 위한 보조 인덱스 CREATE INDEX ix_coa_entry_by_schema ON tsol_chart_of_accounts_entry (institution_code, account_schema_code); CREATE INDEX ix_coa_entry_by_acct   ON tsol_chart_of_accounts_entry (institution_code, account_code);`

---

## 컬럼별 설명 (한눈에)

- **institution_code**: 기관 식별자(예: 증권사/자회사 코드).
    
- **account_schema_code**: 계정과목 “표/체계(버전)” 코드. 기관 내 여러 버전의 COA를 운용할 때 구분.
    
- **account_code**: 계정과목 코드 자체. 숫자/문자 혼재 가능하므로 `varchar(20)` 권장.
    
- **debit_credit_type**: 차/대 구분. `'D'`(Debit), `'C'`(Credit). **분개행 단위**를 구분하므로 **PK에 포함**.
    
- **account_name**: 계정과목의 정식 명칭.
    
- **account_name_abbrev**: 보고서/화면에서 쓰는 약칭.
    
- **account_category_code**: 계정과목의 상위 분류(자산/부채/자본/수익/비용 or 내부 분류코드 등).
    
- **settlement_account_type_code**: 결제/정산 관점의 구분이 필요할 때(예: 결제계정, 비결제계정…).
    
- **is_deleted**: 논리삭제 여부(기본 false).
    
- __created__ / updated__**: 생성/수정 메타데이터(사용자/트랜잭션/시간/IP).
    

---

## (참고) 기존 약어 ↔ 새 풀네임 매핑표

|기존(약어)|새 컬럼명(풀네임)|의미|
|---|---|---|
|ist_cd|institution_code|기관코드|
|asj_tbl_cd|account_schema_code|계정과목표/체계 코드|
|asj_cd|account_code|계정과목 코드|
|dr_cds_tc|debit_credit_type|차/대 구분 (‘D’/’C’)|
|asj_nm|account_name|계정과목명|
|asj_abv_nm|account_name_abbrev|계정과목 약어명|
|asj_tc|account_category_code|계정과목 구분/분류 코드|
|stm_ats_tc|settlement_account_type_code|결제계정 구분 코드|
|del_yn|is_deleted|삭제여부(논리)|
|fst_enr_usid|created_by_user_id|최초등록사용자ID|
|fst_enr_dtm|created_at|최초등록일시|
|fst_enr_trid|created_by_transaction_id|최초등록거래ID|
|fst_enr_ip|created_from_ip|최초등록IP|
|lst_chg_usid|updated_by_user_id|최종변경사용자ID|
|lst_chg_dtm|updated_at|최종변경일시|
|lst_chg_trid|updated_by_transaction_id|최종변경거래ID|
|lst_chg_ip|updated_from_ip|최종변경IP|

---

## 운영 이관 팁

- **값 표준화**: `dr_cds_tc`가 레거시에 `1/2`로 있다면 이관 시 **‘D’/’C’로 변환**하세요.
    
- **문구 분리**: `account_name` 값에 “미수금/이자수익”처럼 **두 계정을 슬래시로 붙인 표시문구**가 있습니다. 이 표기는 UI용일 뿐 실제 전표 로직은 **분개행(차/대)으로 분리**되므로, 표기는 `account_name`에 두되 키 구조는 위 설계대로 유지하면 됩니다.
    
- **IP/시간대**: `inet`, `timestamptz`로 전환 시 서버 타임존/로그 정책을 문서화해 두면 이후 문제 예방에 좋아요.
    

필요하면 이 스키마에 맞춘 **마이그레이션 스크립트(INSERT … SELECT 변환)** 도 바로 만들어 드릴게요.

##### 나의 말:

왜이게 전표규칙목록이야?

###### ChatGPT의 말:

좋은 질문이에요.  
앞에서 보신 `TSOL_CR_ASJ_CTL` 같은 테이블은 단순히 **계정과목(Chart of Accounts)** 정보를 저장하는 마스터 테이블인데,

👉 지금 말씀하신 **“전표 규칙 목록(Entry Rule List)”**은 개념이 다릅니다.

---

## 📌 계정과목 마스터 vs 전표규칙목록

- **계정과목 마스터 (Chart of Accounts, COA)**
    
    - `account_code`, `account_name`, `account_category` 같은 **계정 자체 정의**를 담음.
        
    - “이런 계정과목이 존재한다”는 사실을 관리.
        
- **전표규칙목록 (Journal Entry Rule List / 분개 규칙 테이블)**
    
    - 특정 거래유형(Transaction Type, 예: “이자수익 발생”, “채권매매”)이 발생했을 때 **차변/대변에 어떤 계정을 사용할지** 정의.
        
    - 즉, “거래유형 + 조건 → (차변계정, 대변계정)” 매핑 룰.
        
    - 한 건의 거래가 입력되면, 이 규칙을 따라 자동으로 분개 전표(Debit/Credit line)를 생성.
        

---

## 📊 예시 (전표규칙목록 테이블)

|rule_id|transaction_type|debit_account_code|credit_account_code|description|
|---|---|---|---|---|
|1|이자수익 발생|미수금|이자수익|“이자 수익을 아직 수취하지 않은 경우”|
|2|채권 매입|채권|현금|“현금으로 채권을 매입하는 경우”|

👉 이렇게 정의된 규칙이 실제로는 **분개템플릿** 역할을 합니다.

---

## 🛠️ 왜 `TSOL_CR_ASJ_CTL`이 전표규칙처럼 보였을까?

- 데이터에 `미수금/이자수익` 같이 **차변/대변 계정이 한 줄에 동시에 표시**되어 있었죠.
    
- 사실 이건 계정 자체가 아니라 **“분개 규칙 표시”**에 가까운 형태예요.
    
- 그래서 “이 테이블은 전표규칙목록이 아닌가?” 하는 혼동이 생기신 거예요.
    

---

✅ 결론:

- **COA(계정과목표)**: 계정 자체 정의 (마스터 테이블).
    
- **전표규칙목록**: 거래유형별 분개 룰 (차·대 계정 조합 정의).
    

---

혹시 원하시는 건, 지금 테이블을 **순수 계정과목 마스터**로 정리하고, 별도로 **전표규칙 테이블**을 설계해서 분리하는 걸 원하시나요?

##### 나의 말:

그러면 단순히 대차를 써 둔 거랑 전표규칙목록이랑 분리하는게 나아?

###### ChatGPT의 말:

결론부터: **네, 분리하는 게 맞습니다.**  
“대차가 함께 적힌 문자열(메모)”과 “전표규칙(자동분개 템플릿)”은 수명·역할·검증 포인트가 달라요. 섞어두면 검색·유효성검사·변경관리·감사가 모두 애매해집니다.

# 왜 분리?

- **정규화/무결성**: 전표규칙은 차·대 계정, 우선순위, 유효기간, 조건(상품/통화/회계기준 등)이 존재합니다. 문자열 필드에 섞어두면 검증/참조무결성 불가.
    
- **버전·유효기간 관리**: 회계정책 변경(FVPL→FVOCI, 헤지여부 등)을 규칙 테이블에서 기간(versioning)으로 관리 가능.
    
- **감사 추적성**: 어떤 거래에 어떤 규칙 버전이 적용됐는지 로그로 남기기 쉬움.
    
- **성능/유지보수**: 룰만 인덱싱·캐시 가능. COA와 엮여도 단순 조인.
    

# 권장 구조(요지)

1. **계정과목 마스터 (Chart of Accounts)**
    

- `account_id(PK)`, `account_code`, `account_name`, `account_type(자산/부채/수익/비용/자본)`, `ifrs_category(FVTPL/FVOCI/AC 등 필요시)`, `currency_scope(원화/외화/공통)`, `is_active`, `valid_from/to`, 감사필드.
    

2. **전표규칙 헤더 (Journal Entry Rule)**
    

- `rule_id(PK)`, `rule_name`, `transaction_type`, `accounting_framework(IFRS/K-GAAP 등)`, `valid_from/to`, `priority`, `status(DRAFT/ACTIVE/RETIRED)`, 감사필드.
    

3. **전표규칙 라인 (Rule Lines)**
    

- `rule_line_id(PK)`, `rule_id(FK)`, `line_side(DEBIT/CREDIT)`, `account_id(FK)`, `amount_basis(원가/공정가/평가손익 등)`, `memo_template`.
    

4. **전표규칙 조건 (Rule Conditions)**
    

- `rule_cond_id(PK)`, `rule_id(FK)`, `product_type(주식/채권/파생…)`, `hedge_type(비헤지/현금흐름/공정가액)`, `classification(FVTPL/FVOCI/AC 등)`, `currency(KRW/USD/*)`, `counterparty_type(은행/증권 등)`, 기타 키.
    

5. **규칙 적용 로그 (Rule Application Log)** – 선택
    

- 전표 생성 시 `rule_id`, `rule_version(=valid_from)`, 입력 파라미터, 산식 결과 저장.
    

6. **메모/서술(선택)**
    

- 단순 안내용 “대차 메모”는 별도 `entry_note` 테이블(전표번호 FK, free text)로 관리. 규칙이 아님을 명확히.
    

# 언제 굳이 안 나눠도 되나?

- 룰이 거의 없고, 자동분개를 쓰지 않으며, 모두 수작업 전표라면 굳이 룰테이블이 필요 없을 수 있습니다.
    
- 그러나 현재처럼 “미수금/이자수익”, “평가손/자산” 등 **반복 패턴**이 많다면 분리가 이득입니다.
    

# 마이그레이션 가이드(압축)

1. 현 테이블에서 `asj_nm` 등 문자열 중 “A/B” 패턴 → 차·대 분해.
    
2. 좌/우를 COA로 매핑해 `account_id` 확보(미매핑은 보류리스트).
    
3. 중복 조합을 묶어 `rule`로 승격, 라인 2건(DEBIT/CREDIT) 생성.
    
4. 이름에 들어간 분류어(매매목적/FVPL/헤지유형/선물/옵션/외화/원화 등)를 **조건 테이블** 컬럼으로 이관.
    
5. 유효기간·우선순위 설정, 샘플 트랜잭션으로 회귀 테스트.
    
6. 전표 생성엔 룰만 사용. 문자열 “대차 메모”는 주석용으로만.
    

# 아주 짧은 DDL 스케치

`-- 1) COA CREATE TABLE coa (   account_id BIGSERIAL PRIMARY KEY,   account_code VARCHAR(20) UNIQUE NOT NULL,   account_name VARCHAR(200) NOT NULL,   account_type VARCHAR(10) NOT NULL,  -- ASSET/LIAB/REV/EXP/EQ   ifrs_class VARCHAR(10),             -- FVTPL/FVOCI/AC/...   currency_scope VARCHAR(10),         -- KRW/FX/ALL   is_active BOOLEAN DEFAULT TRUE,   valid_from DATE DEFAULT CURRENT_DATE,   valid_to DATE );  -- 2) Rule Header CREATE TABLE je_rule (   rule_id BIGSERIAL PRIMARY KEY,   rule_name VARCHAR(200) NOT NULL,   transaction_type VARCHAR(100) NOT NULL,   accounting_framework VARCHAR(20) DEFAULT 'K-IFRS',   priority INT DEFAULT 100,   valid_from DATE NOT NULL,   valid_to DATE,   status VARCHAR(10) DEFAULT 'ACTIVE' );  -- 3) Rule Lines CREATE TABLE je_rule_line (   rule_line_id BIGSERIAL PRIMARY KEY,   rule_id BIGINT REFERENCES je_rule(rule_id),   line_side VARCHAR(6) NOT NULL CHECK (line_side IN ('DEBIT','CREDIT')),   account_id BIGINT NOT NULL REFERENCES coa(account_id),   amount_basis VARCHAR(50),   memo_template VARCHAR(300) );  -- 4) Rule Conditions CREATE TABLE je_rule_condition (   rule_cond_id BIGSERIAL PRIMARY KEY,   rule_id BIGINT REFERENCES je_rule(rule_id),   product_type VARCHAR(30),    -- STOCK/BOND/DERIV/LOAN...   hedge_type VARCHAR(20),      -- NONE/CFH/FAH   classification VARCHAR(10),  -- FVTPL/FVOCI/AC...   currency VARCHAR(10),        -- KRW/USD/*   other_key VARCHAR(50) ); CREATE INDEX ix_rule_cond_rule ON je_rule_condition(rule_id);`

# 요약

- **계정과목(마스터)**, **전표규칙(템플릿)**, **메모/서술**은 각각 분리하세요.
    
- 규칙은 헤더/라인/조건 구조로 만들고, **유효기간·우선순위**와 **회계기준·헤지/분류**를 조건으로 관리하세요.
    
- 이렇게 하면 자동분개 정확도·감사 추적성·유지보수가 확 좋아집니다.