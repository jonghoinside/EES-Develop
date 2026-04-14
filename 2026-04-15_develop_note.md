# BGF CDC 시뮬레이션 환경 구축
### WCS/WES SP 분석 및 자동 출고 시뮬 설계

> **작업 기간:** 2026-04-08 ~ 2026-04-14
> **대상 시스템:** BGF 부산 물류센터(CDC) WCS/WES

---

## 1. 목표

**SP_TSK_CORE_MATCH_TOTE_ORDER_PRC** (이하 SP_MATCH_TOTE) 를 활용하여
MSC(자동창고 크레인)의 출고 오더 생성 흐름을 시뮬레이션하고,
**연속 반복 실행 가능한 시뮬레이션 환경**을 구축한다.

---

## 2. 시스템 구조

### 2-1. 설비 흐름

```mermaid
flowchart LR
    subgraph MSC["🏭 MSC 자동창고"]
        RACK["RACK\n15,360셀\n(4라인×4BANK×60BAY×16LVL)"]
        CRANE["크레인 64대\nMSC_I_A01 ~ MSC_I_D16"]
    end

    subgraph CONVEYOR["컨베이어 라인"]
        LIFT["LIFT 출고"]
        CV["CONVEYOR\nMAIN LOOP"]
    end

    subgraph ODS["📦 ODS 스테이션"]
        F["F층 스테이션\nF11~F46"]
        M["M층 스테이션\nM11~M55"]
    end

    RACK -->|TOTE 보관\n7,229셀 점유| CRANE
    CRANE -->|MSC_OUT| LIFT
    LIFT --> CV
    CV --> F
    CV --> M
    F -->|피킹 완료 후 재입고| RACK
    M -->|피킹 완료 후 재입고| RACK
```

### 2-2. 핵심 데이터 원본

| 테이블 | 출처 | 행 수 | 역할 |
|--------|------|------:|------|
| `TOTE_MAPPING` | BGF_CDC → CTAS 복사 | 102,276 | 출고 대상 TOTE-스테이션 매핑 |
| `TOTE_INVENTORY` | BGF_CDC → CTAS 복사 | 9,295 | MSC 창고 실제 재고 현황 |

---

## 3. 시뮬레이션 전체 실행 구조

```mermaid
flowchart TD
    INIT["📄 INIT_SIM_BASE_DATA.sql\n──────────────────\n최초 1회 실행\nRACK 전체 셀 15,360개 세팅\n설비 설정 MSC 64대 · ODS 51개"]

    RESET["🔄 SP_RESET_SIMUL\n──────────────────\nSTEP 1. 이전 결과 삭제\nSTEP 2. 장비 상태 초기화\nSTEP 3. 주문 데이터 재생성"]

    MATCH["⚙️ SP_MATCH_TOTE\n──────────────────\nWCS_ORDER_MST 생성\nMSC_OUT_READY 64건\nWCS_ORDER_DTL WAIT 64건"]

    STEPA["🟡 STEP A\n크레인 출고 시작\n──────────────────\nORDER_TYPE → MSC_OUT_START\nCMD_STATUS → WORKING\nTOTE_STATUS 50 → 55\nWES_LOCATION → EMPTY"]

    STEPB["🟠 STEP B\nTOTE ODS 도착\n──────────────────\nCMD_STATUS → COMPLETE\nTOTE_STATUS 55 → 80\nBOX_ROUTE '20' → '30'"]

    STEPC["🟢 STEP C\nODS 분배 완료\n──────────────────\nTOTE_STATUS 80 → 90\nBOX_ROUTE '30' → '40'\nBATCH 전 완료 시 STATUS '90'"]

    INIT -->|1회 실행 후 고정| RESET
    RESET -->|시뮬 시작| MATCH
    MATCH --> STEPA
    STEPA --> STEPB
    STEPB --> STEPC
    STEPC -->|TOTE 재입고\n다음 회차 반복| MATCH

    style INIT fill:#dbeafe,stroke:#3b82f6,color:#1e3a5f
    style RESET fill:#f0fdf4,stroke:#22c55e,color:#14532d
    style MATCH fill:#fef9c3,stroke:#eab308,color:#713f12
    style STEPA fill:#fff7ed,stroke:#f97316,color:#7c2d12
    style STEPB fill:#fff7ed,stroke:#ef4444,color:#7f1d1d
    style STEPC fill:#f0fdf4,stroke:#16a34a,color:#14532d
```

---

## 4. 구축 작업 상세

### 4-1. SP_RESET_SIMUL — 시뮬 초기화 프로시저

```mermaid
flowchart TD
    S1["🗑️ STEP 1\n이전 결과 삭제\n──────────────\nDELETE\nWCS_ORDER_MST\nWCS_ORDER_DTL\nDAS_WORK_ORDER_RESULT\nTEMP_ASSIGN_LOG"]

    S2A["① 전체 EMPTY 리셋\nUPDATE WES_LOCATION\nSTORED_CONTAINER_ID = NULL\nLOCATION_STATUS = 'EMPTY'"]
    S2B["② STOCK 셀 복원\nMERGE from TOTE_INVENTORY\n동일 셀 TOTE 경합 시\nTOTE_MAPPING 소속 우선"]

    S2["🔧 STEP 2\n상태 초기화\n──────────────\nWCS_STATUS_MSC 64대\nREADY=1 AUTO=1 ACTIVE=1\n\nWES_LOCATION 2단계 처리"]

    S3["📋 STEP 3\n주문 데이터 재생성\n──────────────\nTRUNCATE → INSERT\nWES_WAVE_BATCH_INF 7건\nWES_TOTE_MST 4,592건\nWES_BOX_ROUTE 65,625건\nWES_WORK_STN_MST 65,625건\nDAS_WORK_ORDER 9,389건"]

    S1 --> S2
    S2 --> S2A --> S2B
    S2B --> S3

    style S1 fill:#fee2e2,stroke:#ef4444,color:#7f1d1d
    style S2 fill:#fef9c3,stroke:#eab308,color:#713f12
    style S2A fill:#fff7ed,stroke:#f97316
    style S2B fill:#fff7ed,stroke:#f97316
    style S3 fill:#dbeafe,stroke:#3b82f6,color:#1e3a5f
```

> **성능 개선 포인트:** STEP 3 기존 `DELETE` → `EXECUTE IMMEDIATE 'TRUNCATE TABLE'` 교체
> WES_BOX_ROUTE 65,625건 DELETE 시 수 분 소요 → TRUNCATE로 즉시 완료

### 4-2. INIT_SIM_BASE_DATA.sql — WES_LOCATION RACK 전체 셀

```mermaid
graph LR
    subgraph RACK_STRUCTURE["WES_LOCATION 생성 구조"]
        LINE["4 라인\nA · B · C · D"]
        BANK["× 4 BANK\n라인당 BANK 1~4"]
        BAY["× 60 BAY"]
        LVL["× 16 LVL"]
        TOTAL["= 15,360 셀\nLOCATION_STATUS\n= EMPTY"]
    end
    LINE --> BANK --> BAY --> LVL --> TOTAL

    subgraph BANK_MAP["GLOBAL_BANK 매핑"]
        A["A라인: BANK 1~4"]
        B["B라인: BANK 5~8"]
        C["C라인: BANK 9~12"]
        D["D라인: BANK 13~16"]
    end

    style TOTAL fill:#d1fae5,stroke:#10b981,color:#064e3b
```

**LOCATION_ID 규칙:** `MSC_I.{GLOBAL_BANK}.{BAY}.{LVL}`
**예시:** A라인 BANK2, BAY5, LVL3 → `MSC_I.2.5.3`

| 테이블 | 건수 | 내용 |
|--------|-----:|------|
| `WES_BASE_STN_MST` | 51 | ODS 스테이션 (F층/M층) |
| `WCS_CFG_EQP_ID` | 64 | 크레인 설비 MSC_I_A01 ~ D16 |
| `WES_CFG_DEACTIVE` | 68 | MSC 64 + LIF_OUT 4 |
| `WCS_CFG_CV_SETTING` | 4 | 컨베이어 MAIN LOOP (A~D) |
| `WCS_CFG_LIMIT_CNT` | 2 | 스테이션 부하 제한 (F/M, LIMIT=10) |
| `WCS_STATUS_MSC` | 64 | 크레인 초기 정상 상태 |

---

## 5. 트러블슈팅

```mermaid
flowchart TD
    BUG1["🔴 버그 1\nSP_MATCH_TOTE 실행 후\nWCS_ORDER_MST 0건"]
    BUG1_CAUSE["원인\nWES_WAVE_BATCH_INF\nCENTER_CODE = 'BGF' 하드코딩\n↕\nFN_GET_BUSAN_CDC_CD = '980'\n불일치 → BATCH 루프 미진입"]
    BUG1_FIX["✅ 해결\nV_CENTER_CODE := FN_GET_BUSAN_CDC_CD\n변수 사용으로 교체"]

    BUG2["🔴 버그 2\nORA-01403\nNO_DATA_FOUND"]
    BUG2_CAUSE["원인\nWES_CFG_DEACTIVE\nEQP_GUBUN = 'LIF' 잘못 세팅\n↕\nSP는 'LIF_OUT' 조회\n→ 데이터 없음"]
    BUG2_FIX["✅ 해결\n'LIF' → 'LIF_OUT' 수정\nINIT_SIM_BASE_DATA.sql 소스 반영"]

    PERF["🟡 성능 문제\nSP_RESET_SIMUL 무한루프 현상"]
    PERF_CAUSE["원인\nDELETE 65,625건\nundo 로그 전량 생성\n→ 수 분 이상 소요"]
    PERF_FIX["✅ 해결\nEXECUTE IMMEDIATE\n'TRUNCATE TABLE'\n교체 → 즉시 완료"]

    BUG1 --> BUG1_CAUSE --> BUG1_FIX
    BUG2 --> BUG2_CAUSE --> BUG2_FIX
    PERF --> PERF_CAUSE --> PERF_FIX

    style BUG1 fill:#fee2e2,stroke:#ef4444
    style BUG2 fill:#fee2e2,stroke:#ef4444
    style PERF fill:#fef9c3,stroke:#eab308
    style BUG1_FIX fill:#d1fae5,stroke:#10b981
    style BUG2_FIX fill:#d1fae5,stroke:#10b981
    style PERF_FIX fill:#d1fae5,stroke:#10b981
```

---

## 6. SP_MATCH_TOTE 동작 원리

### 오더 생성 판단 흐름 (Pass 조건 3가지)

```mermaid
flowchart TD
    START(["⚙️ SP_MATCH_TOTE 실행"])
    BATCH["BATCH 루프\nWES_WAVE_BATCH_INF\nBATCH_STATUS='40'"]
    STN["STN 루프\nVW_STN_WORK_CNT"]
    CNT{CNT < 10\n스테이션 여유?}
    SKIP_STN(["⏩ 다음 스테이션"])

    PASS2{Pass 2\nCURRENT_CNT\n≥ LIMIT_CNT?}
    SKIP2(["⏩ 스테이션 SKIP"])

    BOX["BOX 선정\nWES_BOX_ROUTE\nORDER_CHECK='N'\nROUTE_SEQ 순"]

    PASS1{Pass 1\nMSC_OUT_READY\n≥ 1?}
    SKIP1(["⏩ MSC SKIP"])

    PASS3{Pass 3\nMSC_OUT_CNT\n+ AILE_OUT_CNT\n> 5?}
    SKIP3(["⏩ MSC SKIP"])

    ORDER["✅ 오더 생성\nWCS_ORDER_MST INSERT\n ORDER_TYPE = MSC_OUT_READY\nWCS_ORDER_DTL INSERT\n CMD_STATUS = WAIT\nWES_TOTE_MST TOTE_STATUS = 50\nWES_BOX_ROUTE WORK_STATUS = '20'"]
    EXIT(["🔚 스테이션당 1건 EXIT"])

    START --> BATCH --> STN --> CNT
    CNT -->|No| SKIP_STN
    CNT -->|Yes| PASS2
    PASS2 -->|Yes| SKIP2
    PASS2 -->|No| BOX --> PASS1
    PASS1 -->|Yes| SKIP1
    PASS1 -->|No| PASS3
    PASS3 -->|Yes| SKIP3
    PASS3 -->|No| ORDER --> EXIT

    style ORDER fill:#d1fae5,stroke:#10b981,color:#064e3b
    style SKIP_STN fill:#f1f5f9,stroke:#94a3b8
    style SKIP1 fill:#f1f5f9,stroke:#94a3b8
    style SKIP2 fill:#f1f5f9,stroke:#94a3b8
    style SKIP3 fill:#f1f5f9,stroke:#94a3b8
```

**실행 결과 (정상 동작 확인)**

| 테이블 | 건수 | 값 |
|--------|-----:|-----|
| `WCS_ORDER_MST` | **64건** | ORDER_TYPE = MSC_OUT_READY |
| `WCS_ORDER_DTL` | **64건** | CMD_STATUS = WAIT |

---

## 7. 시뮬레이션 단계 설계

### TOTE 상태 전이

```mermaid
stateDiagram-v2
    direction LR

    [*] --> s40 : SP_RESET_SIMUL
    s40 --> s50 : SP_MATCH_TOTE\n출고지시
    s50 --> s55 : STEP A\n크레인 출고 시작
    s55 --> s80 : STEP B\nTOTE ODS 도착
    s80 --> s90 : STEP C\nODS 분배 완료
    s90 --> s40 : 재입고

    s40 : TOTE_STATUS 40\n보관창고 적재완료
    s50 : TOTE_STATUS 50\n출고지시
    s55 : TOTE_STATUS 55\n크레인 이동중
    s80 : TOTE_STATUS 80\nODS 작업중
    s90 : TOTE_STATUS 90\n분배완료
```

### BOX_ROUTE · CMD_STATUS 상태 전이

```mermaid
stateDiagram-v2
    direction LR

    [*] --> w10 : SP_RESET_SIMUL
    w10 --> w20 : SP_MATCH_TOTE\nORDER_CHECK='Y'
    w20 --> w30 : STEP B\nODS 도착
    w30 --> w40 : STEP C\n분배 완료
    w40 --> [*] : BATCH 완료\nBATCH_STATUS '90'

    w10 : WORK_STATUS '10'\nWAIT
    w20 : WORK_STATUS '20'\nMOVE / 진행중
    w30 : WORK_STATUS '30'\nODS 작업중
    w40 : WORK_STATUS '40'\nCOMPLETE
```

```mermaid
stateDiagram-v2
    direction LR

    [*] --> WAIT : SP_MATCH_TOTE\nINSERT
    WAIT --> WORKING : STEP A
    WORKING --> COMPLETE : STEP B

    WAIT : CMD_STATUS\nWAIT
    WORKING : CMD_STATUS\nWORKING
    COMPLETE : CMD_STATUS\nCOMPLETE\n※ VW_STN_WORK_CNT 감소
```

### 단계별 업데이트 테이블

| 단계 | 대상 테이블 | 변경 컬럼 | 변경 값 |
|:----:|------------|----------|---------|
| **STEP A** | `WCS_ORDER_MST` | ORDER_TYPE | `MSC_OUT_READY` → `MSC_OUT_START` |
| **STEP A** | `WCS_ORDER_DTL` | CMD_STATUS | `WAIT` → `WORKING` |
| **STEP A** | `WES_TOTE_MST` | TOTE_STATUS | `50` → `55` |
| **STEP A** | `WES_LOCATION` | STORED_CONTAINER_ID / LOCATION_STATUS | `NULL` / `EMPTY` |
| **STEP B** | `WCS_ORDER_DTL` | CMD_STATUS | `WORKING` → `COMPLETE` |
| **STEP B** | `WES_TOTE_MST` | TOTE_STATUS | `55` → `80` |
| **STEP B** | `WES_BOX_ROUTE` | WORK_STATUS | `'20'` → `'30'` |
| **STEP C** | `WES_TOTE_MST` | TOTE_STATUS | `80` → `90` |
| **STEP C** | `WES_BOX_ROUTE` | WORK_STATUS | `'30'` → `'40'` |
| **STEP C** | `WES_WAVE_BATCH_INF` | BATCH_STATUS | `40` → `90` *(전 BOX 완료 시)* |

---

## 8. 담당자 질의응답 — 확정 사항

```mermaid
mindmap
  root((설계 확정\n사항))
    MSC RACK 구조
      4라인 × 4BANK × 60BAY × 16LVL
      총 15,360셀 ✅
    ODS EQP_ID
      ODS_M35 형식
      STATION_ID 기반 ✅
    TOTE 식별
      TOTE_NO = TOTE_ID
      동일값 사용 허용 ✅
    TOTE 순환 구조
      MSC 출고
      여러 스테이션 순회
      재입고 ✅
    CMD_STATUS 체계
      시뮬 문자열 사용
      WAIT / WORKING / COMPLETE ✅
    보류 항목
      MC_VALUE 불필요
      DAS_WORK 일단 보류
```

---

## 9. 현재 상태 및 다음 단계

```mermaid
gantt
    title BGF CDC 시뮬레이션 구축 진행 현황
    dateFormat YYYY-MM-DD
    section 완료
        SP_MATCH_TOTE 분석          :done, 2026-04-07, 1d
        원본 데이터 분석 및 CTAS     :done, 2026-04-08, 1d
        SP_RESET_SIMUL 구축          :done, 2026-04-08, 1d
        버그 수정 및 성능 개선       :done, 2026-04-09, 1d
        WES_LOCATION RACK 재설계     :done, 2026-04-10, 1d
        SP_MATCH_TOTE 정상 동작 확인 :done, 2026-04-10, 1d
        담당자 Q&A 및 설계 확정      :done, 2026-04-11, 3d
    section 진행 예정
        SP_SIM_STEP_A 개발           :active, 2026-04-14, 2d
        SP_SIM_STEP_B 개발           :2026-04-16, 2d
        SP_SIM_STEP_C 개발           :2026-04-18, 2d
        연속 시뮬 루프 검증          :2026-04-20, 2d
```

### 완료

- [x] `INIT_SIM_BASE_DATA.sql` — RACK 전체 셀 15,360개 포함 완성
- [x] `SP_RESET_SIMUL` — 3단계 초기화 정상 동작 확인
- [x] `SP_MATCH_TOTE` — 64건 오더 생성 정상 확인
- [x] 시뮬레이션 STEP A / B / C 설계 확정

### 다음 작업

- [ ] `SP_SIM_STEP_A` 개발 — 크레인 출고 시작 처리
- [ ] `SP_SIM_STEP_B` 개발 — TOTE ODS 도착 처리
- [ ] `SP_SIM_STEP_C` 개발 — ODS 분배 완료 처리
- [ ] 연속 시뮬 루프 실행 검증

---

*BGF CDC 시뮬레이션 환경 구축 — NNT 작업 내용 정리*
