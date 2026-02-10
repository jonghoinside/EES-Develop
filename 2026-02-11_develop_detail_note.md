# EES ↔ EM 인터페이스 상세 명세

> 작성일: 2026-02-10
> 기준: API 명세 v4 (2026-01-26) + 현재 구현 상태
> 이전 버전: `EES_EM_Interface.md`

---

## 1. 시스템 아키텍처

```mermaid
graph LR
    subgraph EES ["EES (Client)"]
        UI["UI<br/>customtkinter"]
        CTRL["Controller<br/>Flask :6000"]
        WK["OdsWorker<br/>상태머신 + 큐"]
    end

    subgraph DB ["Oracle DB"]
        SP["SP_SFA_ODS_ARRIVE_TOTE_PRC"]
        FN["FN_ITEM_PICK_DURATION_SEC"]
        VW1["VW_SFA_INV_BOX"]
        VW2["VW_SFA_LOCATION"]
        MST["WES_BASE_STN_MST"]
    end

    subgraph EM ["EM (Server)"]
        EM_SRV["EM 서버<br/>Flask :7000"]
        DECANT_W["DECANT 작업자"]
        ODS_W["ODS 작업자"]
    end

    UI -- "requests<br/>POST/GET" --> EM_SRV
    EM_SRV -- "HTTP POST<br/>이벤트 콜백" --> CTRL
    CTRL -- "위임" --> WK
    WK -- "DB 조회" --> SP
    WK -- "DB 조회" --> FN
    UI -- "DB 조회" --> VW1
    UI -- "DB 조회" --> VW2
    UI -- "DB 조회" --> MST
    EM_SRV --- DECANT_W
    EM_SRV --- ODS_W
```

---

## 2. 작업 Phase 흐름

```mermaid
flowchart TD
    P0["Phase 0: 초기화<br/>POST /api/reset"]
    P1["Phase 1: 데이터 적재<br/>(Pause 상태에서)"]
    P2["Phase 2: 실행<br/>POST /api/play"]
    P3["Phase 3: DECANT 작업<br/>(Running 상태)"]
    P4["Phase 4: ODS 작업<br/>(Running 상태)"]
    P5["Phase 5: 백업<br/>(Pause 상태에서)"]

    P0 -->|"Pause 상태 진입"| P1
    P1 -->|"데이터 준비 완료"| P2
    P2 -->|"시뮬레이션 시작"| P3
    P2 -->|"시뮬레이션 시작"| P4
    P3 -.->|"반복"| P3
    P4 -.->|"반복"| P4
    P3 -->|"POST /api/pause"| P5
    P4 -->|"POST /api/pause"| P5
    P5 -->|"재개 필요 시"| P2
    P5 -->|"초기화 필요 시"| P0

    style P0 fill:#e74c3c,color:#fff
    style P1 fill:#f39c12,color:#fff
    style P2 fill:#27ae60,color:#fff
    style P3 fill:#3498db,color:#fff
    style P4 fill:#8e44ad,color:#fff
    style P5 fill:#95a5a6,color:#fff
```

### Phase별 필요 상태

| Phase | 필요 상태 | 설명 |
|:---:|:---:|:---|
| 0 | - | 언제든 호출 가능 (단, Reset은 Pause 상태에서만 성공) |
| 1 | **Pause** | 데이터 적재는 일시정지 상태에서만 가능 |
| 2 | **Pause** | Play로 Running 전환 |
| 3~4 | **Running** | 실제 작업 수행 (DECANT, ODS 동시 진행) |
| 5 | **Pause** | 백업 조회는 일시정지 + 추가 안전 조건 필요 |

---

## 3. Phase 0~2: 초기화 → 데이터 적재 → 실행

```mermaid
sequenceDiagram
    participant EES as EES (Client :6000)
    participant EM as EM (Server :7000)
    participant DB as Oracle DB

    rect rgb(231, 76, 60, 0.1)
        Note over EES,EM: Phase 0 — 초기화
        EES->>+EM: POST /api/reset
        EM-->>-EES: 200 OK
        Note right of EM: Pause 상태로 진입
    end

    rect rgb(243, 156, 18, 0.1)
        Note over EES,EM: Phase 1 — 데이터 적재 (Pause 상태에서만 가능)

        EES->>DB: SELECT FROM VW_SFA_LOCATION
        DB-->>EES: msc_id, barcode, bank, bay, level
        EES->>+EM: POST /api/stock/create<br/>{count, data: [{msc_id, barcode, bank, bay, level}]}
        EM-->>-EES: 200 OK
        Note right of EM: MSC 셀에 Tote 적재

        EES->>DB: SELECT TOTE_ID FROM VW_SFA_INV_BOX
        DB-->>EES: barcode 목록
        EES->>+EM: POST /api/source/create<br/>{count, data: [barcode]}
        EM-->>-EES: 200 OK
        Note right of EM: DECANT Source Buffer 생성

        Note over EES,EM: ↓ 미구현 API ↓

        EES-->>+EM: POST /api/target/create<br/>{count, data: [barcode]}
        EM-->>-EES: 200 OK
        Note right of EM: ODS Target Buffer 생성

        EES-->>+EM: POST /api/ods/create<br/>{data: [{station_id, channel_id, barcode}]}
        EM-->>-EES: 200 OK
        Note right of EM: ODS 채널에 Tote 매핑
    end

    rect rgb(39, 174, 96, 0.1)
        Note over EES,EM: Phase 2 — 실행
        EES->>+EM: POST /api/play
        EM-->>-EES: 200 OK
        Note right of EM: Running 상태로 전환
    end
```

### Phase 1 API Request Body 상세

| API | Body |
|:---|:---|
| **Create Stock** | `{count: int, data: [{msc_id: str, barcode: str, bank: int, bay: int, level: int}]}` |
| **Create Source** | `{count: int, data: [barcode: str]}` |
| **Create Target** | `{count: int, data: [barcode: str]}` |
| **Create ODS** | `{data: [{station_id: str, channel_id: str, barcode: str}]}` |

### Phase 1 NG 조건 정리

| API | NG 조건 |
|:---|:---|
| Create Stock | Pause 아닌 경우, msc_id 미존재, barcode 중복, count!=data수, 복합키 중복, Cell 미존재 |
| Create Source | Pause 아닌 경우, 투입구 미지정, count!=data수, barcode 중복 |
| Create Target | Create Source와 동일 |
| Create ODS | channel에 Tote 있음, station_id 미존재, channel_id 미존재, barcode 중복, 복합키 중복 |

---

## 4. Phase 3: DECANT 작업 흐름

```mermaid
sequenceDiagram
    participant EES as EES (Client :6000)
    participant EM as EM (Server :7000)

    Note right of EM: DECANT 작업자: Trigger 감지<br/>Idle → Loading → N초 대기

    rect rgb(52, 152, 219, 0.1)
        Note over EES,EM: ① EM → EES: Ready 통보
        EM->>+EES: POST /api/decant/source/ready<br/>{station_id, barcode}
        EES-->>-EM: 200 OK (body 없음)
    end

    Note left of EES: 자동 응답 생성<br/>decant_time = random(5~10)초

    rect rgb(52, 152, 219, 0.1)
        Note over EES,EM: ② EES → EM: Loading 지시 (별도 스레드)
        EES->>+EM: POST /api/decant/source/loading<br/>{station_id, barcode, decant_time}
        EM-->>-EES: 200 OK
    end

    Note right of EM: 작업자: Working<br/>decant_time 만큼 대기<br/>→ Releasing → Tote 투입

    rect rgb(52, 152, 219, 0.1)
        Note over EES,EM: ③ EM → EES: 완료 통보
        EM->>+EES: POST /api/decant/source/completed<br/>{station_id, barcode}
        EES-->>-EM: 200 OK
        Note left of EES: 로그 기록만 수행
    end
```

### DECANT 작업자 상태 머신

```mermaid
stateDiagram-v2
    [*] --> Idle

    Idle --> Loading: Trigger 위치 감지
    Loading --> ReadyAPI: N초 대기 후
    ReadyAPI --> Working: EES가 Loading API 호출
    Working --> Releasing: decant_time 경과
    Releasing --> Idle: Tote 투입 완료

    state ReadyAPI {
        direction LR
        [*] --> EM_calls_Ready
        EM_calls_Ready --> EES_responds_Loading
    }

    note right of Loading: EM 내부 상태
    note right of Working: decant_time 동안 작업 수행
```

### DECANT Loading NG 조건

| 조건 | 설명 |
|:---|:---|
| 실행 상태 아닌 경우 | Running 상태에서만 호출 가능 |
| station_id 미존재 | 유효하지 않은 스테이션 |
| barcode 불일치 | Ready에서 보낸 barcode와 다름 |
| decant_time < 0 | 음수 시간 불가 |
| 작업자 상태 != Loading | 작업자가 Loading 대기 중이 아님 |

---

## 5. Phase 4-A: ODS Source Pickup (피킹)

```mermaid
sequenceDiagram
    participant EM as EM (Server :7000)
    participant EES as EES (Client :6000)
    participant DB as Oracle DB

    Note right of EM: ODS 작업자: Trigger 감지<br/>3초 정지 → Working

    rect rgb(142, 68, 173, 0.1)
        Note over EM,DB: ① EM → EES: Ready + EES의 DB 조회 + 응답
        EM->>+EES: POST /api/ods/source/ready<br/>{station_id, barcode}

        EES->>+DB: SP_SFA_ODS_ARRIVE_TOTE_PRC<br/>(station_id, barcode)
        DB-->>-EES: (cd, msg) — WCS 도착 통보

        EES->>+DB: FN_ITEM_PICK_DURATION_SEC<br/>(station_id, barcode)
        DB-->>-EES: pickup_time (초)

        EES-->>-EM: 200 {station_id, pickup_time}
        Note right of EM: 응답 body에<br/>pickup_time 포함!
    end

    Note right of EM: 작업자: pickup_time 만큼<br/>피킹 작업 수행

    rect rgb(142, 68, 173, 0.1)
        Note over EM,DB: ② EM → EES: 피킹 완료
        EM->>+EES: POST /api/ods/source/completed<br/>{station_id, barcode}
        EES-->>-EM: 200 OK
        Note left of EES: Worker: Working → Idle<br/>큐 다음 작업 확인
    end
```

### ODS Source Ready 응답 형식

```json
// Request (EM → EES)
{ "station_id": "FST01", "barcode": "TOTE001" }

// Response (EES → EM) ← 반드시 body에 데이터 반환!
{ "station_id": "FST01", "pickup_time": 12.5 }
```

---

## 6. Phase 4-B: ODS Target Release (배출)

```mermaid
sequenceDiagram
    participant EES as EES (Client :6000)
    participant EM as EM (Server :7000)

    Note left of EES: Worker 큐에 Release 등록<br/>Idle 시 큐에서 꺼내 실행

    rect rgb(142, 68, 173, 0.1)
        Note over EES,EM: ① EES → EM: Release 명령
        EES->>+EM: POST /api/ods/target/release<br/>{station_id, channel_id,<br/>barcode, loading_time}
        EM-->>-EES: 200 OK
        Note right of EM: 작업자: Idle → Releasing<br/>loading_time 대기 → Tote 투입
    end

    Note right of EM: loading_time 경과

    rect rgb(142, 68, 173, 0.1)
        Note over EES,EM: ② EM → EES: Release 완료 콜백
        EM->>+EES: POST /api/ods/target/release/completed<br/>{station_id, channel_id, barcode}
        EES-->>-EM: 200 OK
        Note left of EES: Worker: Releasing → Idle<br/>큐 다음 작업 확인
    end
```

### Release NG 조건

| 조건 | 설명 |
|:---|:---|
| 실행 상태 아닌 경우 | Running 상태에서만 호출 가능 |
| station_id 미존재 | 유효하지 않은 스테이션 |
| channel_id 미존재 | 유효하지 않은 채널 |
| Tote 미매핑 | 해당 채널에 매핑된 Tote가 없음 |
| barcode 불일치 | 매핑된 Tote의 barcode와 다름 |
| loading_time < 0 | 음수 시간 불가 |

---

## 7. Phase 4-C: ODS Target Mapping (채널 매핑)

```mermaid
sequenceDiagram
    participant EES as EES (Client :6000)
    participant EM as EM (Server :7000)

    Note left of EES: Worker 큐에 Mapping 등록<br/>Idle 시 큐에서 꺼내 실행

    rect rgb(142, 68, 173, 0.1)
        Note over EES,EM: ① EES → EM: Mapping 명령
        EES->>+EM: POST /api/ods/target/mapping<br/>{station_id, channel_id, mapping_time}
        EM-->>-EES: 200 OK
        Note right of EM: 작업자: Idle → Mapping<br/>mapping_time 대기 → Channel에 Mapping
    end

    Note right of EM: mapping_time 경과

    rect rgb(142, 68, 173, 0.1)
        Note over EES,EM: ② EM → EES: Mapping 완료 콜백
        EM->>+EES: POST /api/ods/target/mapping/completed<br/>{station_id, channel_id, barcode}
        EES-->>-EM: 200 OK
        Note left of EES: Worker: Mapping → Idle<br/>큐 다음 작업 확인
    end
```

### Mapping NG 조건

| 조건 | 설명 |
|:---|:---|
| 실행 상태 아닌 경우 | Running 상태에서만 호출 가능 |
| station_id 미존재 | 유효하지 않은 스테이션 |
| channel_id 미존재 | 유효하지 않은 채널 |
| channel에 이미 Mapping됨 | 중복 매핑 불가 |
| mapping_time < 0 | 음수 시간 불가 |

---

## 8. Phase 5: 백업 조회

```mermaid
sequenceDiagram
    participant EES as EES (Client :6000)
    participant EM as EM (Server :7000)

    rect rgb(149, 165, 166, 0.1)
        Note over EES,EM: 일시정지 전환
        EES->>+EM: POST /api/pause
        EM-->>-EES: 200 OK
    end

    rect rgb(149, 165, 166, 0.1)
        Note over EES,EM: 재고 백업 (미구현)
        EES-->>+EM: GET /api/stock
        EM-->>-EES: 200 {count, data: [{msc_id, barcode, bank, bay, level}]}
    end

    rect rgb(149, 165, 166, 0.1)
        Note over EES,EM: ODS 백업 (미구현)
        EES-->>+EM: GET /api/ods
        EM-->>-EES: 200 {data: [{station_id, channel_id, barcode}]}
    end
```

### 백업 NG 조건 (Stock / ODS 공통)

| 조건 | 설명 |
|:---|:---|
| 일시정지 아닌 경우 | Pause 상태에서만 조회 가능 |
| 프로젝트 내 Tote 존재 | 적재 상태 Tote 제외, 이동중 Tote가 있으면 불가 |
| MV가 Tote 적재중 | 운반차가 Tote를 들고 있으면 불가 |
| MV ID 미할당 | 운반차 ID가 할당되지 않은 상태 |
| MV 명령 수행중 | 운반차가 명령을 수행 중이면 불가 |

---

## 9. ODS Worker 상태 머신

```mermaid
stateDiagram-v2
    [*] --> Idle

    Idle --> Working: Ready ODS Source 수신<br/>(on_ready_source)
    Working --> Idle: Completed ODS Source 수신<br/>(on_completed_source)

    Idle --> Releasing: 큐에서 Release 작업 실행<br/>(_execute TARGET_RELEASE)
    Releasing --> Idle: Completed Release 수신<br/>(on_completed_release)

    Idle --> Mapping: 큐에서 Mapping 작업 실행<br/>(_execute TARGET_MAPPING)
    Mapping --> Idle: Completed Mapping 수신<br/>(on_completed_mapping)

    Working --> Error: API 호출 실패
    Releasing --> Error: API 호출 실패
    Mapping --> Error: API 호출 실패
    Error --> Idle: Reset 명령

    note right of Idle: _try_next() 호출<br/>큐에 작업 있으면 실행
    note right of Error: 큐 전체 클리어<br/>Reset으로만 해제
```

---

## 10. ODS Worker 큐 스케줄링

```mermaid
flowchart TD
    subgraph 입력 ["작업 입력"]
        READY["EM → Ready 수신"]
        REL_REQ["enqueue_release()"]
        MAP_REQ["enqueue_mapping()"]
    end

    subgraph 분기 ["처리 경로"]
        READY -->|"즉시 실행<br/>(큐 거치지 않음)"| ON_READY["on_ready_source()<br/>state → Working"]
        REL_REQ -->|"큐 등록"| QUEUE["task_queue<br/>(deque)"]
        MAP_REQ -->|"큐 등록"| QUEUE
    end

    QUEUE --> TRY{"_try_next()<br/>state == Idle?"}
    TRY -->|"No"| WAIT["대기<br/>(Completed 수신 시 재시도)"]
    TRY -->|"Yes"| POPLEFT["popleft()"]

    POPLEFT --> EXEC{"task_type?"}
    EXEC -->|"TARGET_RELEASE"| RELEASE["state → Releasing<br/>_call_em(/api/ods/target/release)"]
    EXEC -->|"TARGET_MAPPING"| MAPPING["state → Mapping<br/>_call_em(/api/ods/target/mapping)"]

    ON_READY --> COMP_S["Completed Source 수신"]
    RELEASE --> COMP_R["Completed Release 수신"]
    MAPPING --> COMP_M["Completed Mapping 수신"]

    COMP_S --> IDLE["state → Idle<br/>_try_next()"]
    COMP_R --> IDLE
    COMP_M --> IDLE
    IDLE --> TRY

    style ON_READY fill:#3498db,color:#fff
    style RELEASE fill:#e67e22,color:#fff
    style MAPPING fill:#8e44ad,color:#fff
    style QUEUE fill:#2c3e50,color:#fff
    style IDLE fill:#27ae60,color:#fff
```

---

## 11. 전체 통합 시퀀스 (Happy Path)

```mermaid
sequenceDiagram
    actor USER as 사용자
    participant EES as EES
    participant EM as EM
    participant DB as Oracle DB

    Note over USER,DB: === Phase 0: 초기화 ===
    USER->>EES: [RESET 버튼]
    EES->>EM: POST /api/reset
    EM-->>EES: 200

    Note over USER,DB: === Phase 1: 데이터 적재 ===
    USER->>EES: [CREATE STOCK 버튼]
    EES->>DB: SELECT VW_SFA_LOCATION
    DB-->>EES: 재고 데이터
    EES->>EM: POST /api/stock/create
    EM-->>EES: 200

    USER->>EES: [CREATE SOURCE 버튼]
    EES->>DB: SELECT VW_SFA_INV_BOX
    DB-->>EES: Tote 바코드
    EES->>EM: POST /api/source/create
    EM-->>EES: 200

    Note over USER,DB: === Phase 2: 실행 ===
    USER->>EES: [PLAY 버튼]
    EES->>EM: POST /api/play
    EM-->>EES: 200

    Note over USER,DB: === Phase 3: DECANT (자동 반복) ===
    loop DECANT Tote마다 반복
        EM->>EES: POST /api/decant/source/ready
        EES-->>EM: 200
        EES->>EM: POST /api/decant/source/loading
        EM-->>EES: 200
        EM->>EES: POST /api/decant/source/completed
        EES-->>EM: 200
    end

    Note over USER,DB: === Phase 4: ODS (자동 반복) ===
    loop ODS Tote마다 반복
        EM->>EES: POST /api/ods/source/ready
        EES->>DB: SP + FN 호출
        DB-->>EES: pickup_time
        EES-->>EM: 200 {pickup_time}
        EM->>EES: POST /api/ods/source/completed
        EES-->>EM: 200
    end
```

---

## 12. API 전체 요약

### EES → EM (12개)

| # | API | Method | Endpoint | Body | 구현 |
|:---:|:---|:---:|:---|:---|:---:|
| 1 | Play | POST | `/api/play` | - | ✅ |
| 2 | Pause | POST | `/api/pause` | - | ✅ |
| 3 | Reset | POST | `/api/reset` | - | ✅ |
| 4 | Create Stock | POST | `/api/stock/create` | `{count, data: [{msc_id, barcode, bank, bay, level}]}` | ✅ |
| 5 | Create Source Buffer | POST | `/api/source/create` | `{count, data: [barcode]}` | ✅ |
| 6 | Loading DECANT Source | POST | `/api/decant/source/loading` | `{station_id, barcode, decant_time}` | ✅ |
| 7 | Create Target Buffer | POST | `/api/target/create` | `{count, data: [barcode]}` | ❌ |
| 8 | Create ODS | POST | `/api/ods/create` | `{data: [{station_id, channel_id, barcode}]}` | ❌ |
| 9 | Backup Stock | GET | `/api/stock` | - | ❌ |
| 10 | Backup ODS | GET | `/api/ods` | - | ❌ |
| 11 | Release ODS Target | POST | `/api/ods/target/release` | `{station_id, channel_id, barcode, loading_time}` | ❌ |
| 12 | Mapping ODS Target | POST | `/api/ods/target/mapping` | `{station_id, channel_id, mapping_time}` | ❌ |

### EM → EES (5개)

| # | API | Method | Endpoint | Request Body | Response Body | 구현 |
|:---:|:---|:---:|:---|:---|:---|:---:|
| 1 | Ready DECANT Source | POST | `/api/decant/source/ready` | `{station_id, barcode}` | - | ✅ |
| 2 | Completed DECANT Source | POST | `/api/decant/source/completed` | `{station_id, barcode}` | - | ✅ |
| 3 | Ready ODS Source | POST | `/api/ods/source/ready` | `{station_id, barcode}` | **`{station_id, pickup_time}`** | ✅ |
| 4 | Completed ODS Source | POST | `/api/ods/source/completed` | `{station_id, barcode}` | - | ✅ |
| 5 | Completed Release ODS Target | POST | `/api/ods/target/release/completed` | `{station_id, channel_id, barcode}` | - | ✅ |
| 6 | Completed Mapping ODS Target | POST | `/api/ods/target/mapping/completed` | `{station_id, channel_id, barcode}` | - | ✅ |

### 구현 현황

```mermaid
pie title API 구현 현황 (총 17개)
    "구현 완료" : 12
    "미구현" : 5
```

| 구분 | 구현 | 미구현 | 합계 |
|:---|:---:|:---:|:---:|
| EES → EM | 6 | 6 | 12 |
| EM → EES | 6 | 0 | 6 |
| **합계** | **12** | **6** | **18** |

---

## 13. 통신 패턴 분류

```mermaid
graph LR
    subgraph P1 ["패턴 1: 이벤트-반응"]
        EM1["EM: Ready"] -->|"POST"| EES1["EES: 수신"]
        EES1 -->|"자동 POST"| EM1b["EM: Loading"]
    end

    subgraph P2 ["패턴 2: 동기 요청-응답"]
        EM2["EM: Ready"] -->|"POST"| EES2["EES: DB 조회"]
        EES2 -->|"200 + body"| EM2b["EM: pickup_time 수신"]
    end

    subgraph P3 ["패턴 3: 비동기 명령-콜백"]
        EES3["EES: 명령"] -->|"POST"| EM3["EM: 작업 시작"]
        EM3 -->|"완료 후 POST"| EES3b["EES: Completed 수신"]
    end

    style P1 fill:#3498db,color:#fff,stroke:#2980b9
    style P2 fill:#8e44ad,color:#fff,stroke:#7d3c98
    style P3 fill:#e67e22,color:#fff,stroke:#d35400
```

| 패턴 | 적용 대상 | 방향 | 특징 |
|:---|:---|:---:|:---|
| **이벤트-반응** | DECANT | EM→EES→EM | Ready 수신 후 EES가 자동으로 Loading 역호출 |
| **동기 요청-응답** | ODS Source Pickup | EM→EES | Ready의 HTTP 응답 body에 pickup_time 포함 |
| **비동기 명령-콜백** | ODS Release / Mapping | EES→EM→EES | 명령 후 EM이 작업 완료 시 Completed 콜백 |

---

## 14. 미구현 API 구현 로드맵

```mermaid
gantt
    title 미구현 API 구현 우선순위
    dateFormat X
    axisFormat %s

    section Phase 1 - 데이터 적재
    POST /api/target/create   :a1, 0, 1
    POST /api/ods/create      :a2, 0, 1

    section Phase 4 - ODS 작업
    POST /api/ods/target/release  :a3, 1, 2
    POST /api/ods/target/mapping  :a4, 1, 2

    section Phase 5 - 백업
    GET /api/stock            :a5, 2, 3
    GET /api/ods              :a6, 2, 3
```

| 우선순위 | API | Phase | 의존성 |
|:---:|:---|:---:|:---|
| 1 | `POST /api/target/create` | 1 | 없음 — ODS Target Buffer 생성 |
| 1 | `POST /api/ods/create` | 1 | 없음 — ODS 채널 사전 매핑 |
| 2 | `POST /api/ods/target/release` | 4 | Worker 큐에서 호출 (enqueue_release 연결 필요) |
| 2 | `POST /api/ods/target/mapping` | 4 | Worker 큐에서 호출 (enqueue_mapping 연결 필요) |
| 3 | `GET /api/stock` | 5 | 모든 Phase 4 작업 완료 후 |
| 3 | `GET /api/ods` | 5 | 모든 Phase 4 작업 완료 후 |

---

## 15. 공통 규칙

### HTTP 통신 규칙
- Content-Type: `application/json`
- 모든 속성은 필수, null/whitespace 불가
- 성공: HTTP 200 (body 없거나 데이터 포함)
- 실패: 상황별 상태코드 + `{"message": "실패 사유"}`

### 바코드 중복 검사
- 모든 생성 API(`create` 계열)에서 **등록된 바코드 전체**에 대해 수행
- Stock, Source, Target, ODS 모든 영역에서 교차 검사

### Error 상태 처리
- API 호출 실패 또는 잘못된 요청 시 Worker가 Error 상태 진입
- Error 상태에서는 신규 작업/큐 작업 모두 수행 불가
- 해제 조건: 시뮬레이션 초기화 (`POST /api/reset`)
