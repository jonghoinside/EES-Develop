---
marp: true
theme: default
mermaid: true
---

# BGF 부산 물류센터 — EES ↔ EM 시스템 설계

> 2026-02-24 | API 명세 v4 기준 · 문서 v4 (v3 대비: ODS 폴링·DB 프로시저·동시성 추가)

---

## 1. 시스템 구조

```mermaid
graph LR
    subgraph EES ["EES (Client :6000)"]
        direction TB
        UI["UI 제어판<br/>customtkinter"]
        CTRL["Flask 수신 서버"]
        WORKER["ODS Worker<br/>상태머신 + 작업큐"]
        POLL["ODS 폴링 스레드<br/>1초 주기"]
        UI --- WORKER
        UI --- POLL
        CTRL --- WORKER
    end

    subgraph EM ["EM (Server :7000)"]
        direction TB
        EM_SRV["Flask 수신 서버"]
        DW["DECANT 작업자"]
        OW["ODS 작업자"]
        EM_SRV --- DW
        EM_SRV --- OW
    end

    DB[("Oracle DB<br/>WCS 연동")]

    UI -- "명령 전송<br/>(HTTP POST/GET)" --> EM_SRV
    EM_SRV -- "이벤트 콜백<br/>(HTTP POST)" --> CTRL
    WORKER <-. "SP/FN 호출" .-> DB
    POLL <-. "VW 조회 + SP 호출" .-> DB

    style EES fill:#2c3e50,color:#ecf0f1,stroke:#2c3e50
    style EM fill:#1a5276,color:#ecf0f1,stroke:#1a5276
    style DB fill:#7d6608,color:#ecf0f1,stroke:#7d6608
```

| 구성요소 | 역할 |
|:---|:---|
| **EES** | 설비 제어 클라이언트 — 명령 전송 + 이벤트 수신 + DB 연동 |
| **EM** | 설비 시뮬레이터 서버 — 명령 수신 + 작업 완료 시 이벤트 콜백 |
| **Oracle DB** | WCS 연동 — 피킹 시간 조회, 토트 도착/완료 통보, ODS 채널 상태 폴링 |

---

## 2. 운영 흐름 (Phase)

```mermaid
sequenceDiagram
    actor USER as 운영자
    participant EES as EES
    participant EM as EM
    participant DB as Oracle DB

    rect rgb(231, 76, 60, 0.15)
        Note over EES,EM: Phase 0 · 초기화
        USER->>EES: RESET
        EES->>EM: POST /api/reset
        Note right of EES: ODS 폴링 중단<br/>바코드 채번 초기화
    end

    rect rgb(243, 156, 18, 0.15)
        Note over EES,EM: Phase 1 · 데이터 적재 (Pause 상태)
        USER->>EES: CREATE STOCK / SOURCE
        EES->>EM: POST /api/stock/create
        EES->>EM: POST /api/source/create
    end

    rect rgb(39, 174, 96, 0.15)
        Note over EES,EM: Phase 2 · 실행
        USER->>EES: PLAY
        EES->>EM: POST /api/play
        Note right of EES: ODS 폴링 자동 시작
    end

    rect rgb(52, 152, 219, 0.15)
        Note over EES,EM: Phase 3 · DECANT 작업 (자동 반복)
        loop 투입 Tote마다
            EM->>EES: Ready (Tote 준비됨)
            EES->>EM: Loading (디캔트 지시, decant_time 랜덤)
            EM->>EES: Completed (완료)
        end
    end

    rect rgb(142, 68, 173, 0.15)
        Note over EES,EM: Phase 4 · ODS 작업 (자동 반복)
        loop 피킹 Tote마다
            EM->>EES: Ready (Tote 도착)
            EES->>DB: SP_SFA_ODS_ARRIVE_TOTE_PRC
            EES-->>EM: 200 {pickup_time}
            EM->>EES: Completed (피킹 완료)
            EES->>DB: SP_SFA_ODS_WORK_FINISH_PRC
            EES->>DB: SP_TSK_CORE_ORD_WORK_RESULT_PRC
        end
        loop ODS 채널 폴링 (1초 주기)
            EES->>DB: VW_SFA_DOS_CHANNEL_STATUS 조회
            EES->>DB: SP_SFA_ODS_MAPPING_TARGET_TOTE_PRC
            EES->>EM: POST /api/ods/create
        end
        loop Release 자동 체크
            EES->>DB: VW_SFA_SHIP_BOX 조회
            EES->>EM: POST /api/ods/target/release
            EM->>EES: Completed Release
        end
    end

    rect rgb(149, 165, 166, 0.15)
        Note over EES,EM: Phase 5 · 백업
        USER->>EES: PAUSE
        EES->>EM: GET /api/stock · GET /api/ods
        Note right of EES: ODS 폴링 자동 중단<br/>JSON 파일 저장
    end
```

---

## 3. 통신 패턴

시스템은 작업 유형에 따라 **3가지 통신 패턴**을 사용합니다.

```mermaid
graph TB
    subgraph P1 ["이벤트 → 반응<br/>(DECANT)"]
        direction LR
        A1["EM"] -->|"① Ready"| B1["EES"]
        B1 -->|"② Loading"| A1
    end

    subgraph P2 ["동기 요청 → 응답<br/>(ODS Source Pickup)"]
        direction LR
        A2["EM"] -->|"① Ready"| B2["EES"]
        B2 -->|"② 200 + pickup_time"| A2
    end

    subgraph P3 ["비동기 명령 → 콜백<br/>(ODS Release / Mapping)"]
        direction LR
        B3["EES"] -->|"① 명령"| A3["EM"]
        A3 -->|"② Completed"| B3
    end

    subgraph P4 ["DB 폴링 → 자동 명령<br/>(ODS Create)"]
        direction LR
        D4[("Oracle DB")] -->|"① 채널 상태 감지"| B4["EES"]
        B4 -->|"② /api/ods/create"| A4["EM"]
    end

    style P1 fill:#3498db,color:#fff
    style P2 fill:#8e44ad,color:#fff
    style P3 fill:#e67e22,color:#fff
    style P4 fill:#27ae60,color:#fff
```

| 패턴 | 대상 | 핵심 |
|:---|:---|:---|
| **이벤트-반응** | DECANT | EM이 Ready → EES가 자동으로 Loading 역호출 |
| **동기 요청-응답** | ODS Pickup | EM이 Ready → EES가 DB 조회 후 pickup_time 응답 |
| **비동기 명령-콜백** | ODS Release/Mapping | EES가 명령 → EM이 작업 후 Completed 콜백 |
| **DB 폴링-자동 명령** | ODS Create | EES가 1초 주기로 DB 폴링 → 조건 충족 시 자동 명령 |

---

## 4. ODS Worker 상태 머신

각 ODS 스테이션마다 Worker 인스턴스가 **상태 머신 + 작업 큐**로 동작합니다.

```mermaid
stateDiagram-v2
    [*] --> Idle
    Idle --> Working: Source Pickup<br/>(EM이 Ready 호출)
    Idle --> Releasing: Target Release<br/>(큐에서 실행)
    Idle --> Mapping: Target Mapping<br/>(큐에서 실행)

    Working --> Idle: Completed\n→ WORK_FINISH_PRC\n→ WORK_RESULT_PRC\n→ Release 체크
    Releasing --> Idle: Completed
    Mapping --> Idle: Completed

    Working --> Error: DB 프로시저 실패
    Releasing --> Error: 실패
    Mapping --> Error: 실패
    Error --> Idle: Reset
```

### ODS Source Completed 처리 흐름 (auto worker)

```
Completed 수신
    → WORK_FINISH_PRC(station_id, barcode)   ← 작업 완료 등록
    → WORK_RESULT_PRC(station_id|barcode)     ← 작업 결과 통보
    → VW_SFA_SHIP_BOX 조회                    ← Release 대상 체크
        → 해당 station 데이터 있으면 enqueue_release()
```

- **Source Pickup** — EM 이벤트로 즉시 진입 (큐 거치지 않음)
- **Release / Mapping** — 작업 큐(deque)에 등록 후 Idle일 때 순차 실행
- **Error** — Reset으로만 해제, 큐 전체 클리어
- **auto/manual** — EM 측에서 관리, EES는 UI 체크박스로 상태만 표시

---

## 5. DB 연동 현황

### 프로시저 / 함수 호출 목록

| # | 이름 | 종류 | 호출 시점 | 실패 시 |
|:--|:---|:----|:---|:---|
| 1 | `SP_SFA_ODS_ARRIVE_TOTE_PRC` | 프로시저 | ODS Ready 수신 (on_ready_source) | 400 반환 |
| 2 | `SP_SFA_ODS_WORK_FINISH_PRC` | 프로시저 | ODS Completed 수신 (auto만) | 400 반환 |
| 3 | `SP_TSK_CORE_ORD_WORK_RESULT_PRC` | 프로시저 | WORK_FINISH 직후 (auto만), PI_PARAMS=`"station_id\|barcode"` | 400 반환 |
| 4 | `SP_SFA_ODS_MAPPING_TARGET_TOTE_PRC` | 프로시저 | ODS 폴링 시 각 row별 (채널 매핑 전) | API 중단 |
| 5 | `XX_SFA_FN_ITEM_PICK_DURATION_SEC` | 함수 | ODS Ready 수신 (auto만, pickup_time 조회) | default 5.0s |

### 조회 뷰

| 뷰 | 조회 시점 | 용도 |
|:---|:---|:---|
| `VW_SFA_DOS_CHANNEL_STATUS` | ODS 폴링 (1초 주기) | 빈 채널 감지 → ODS Create |
| `VW_SFA_SHIP_BOX` | ODS Source Completed 후 | Release 대상 체크 |
| `WES_BASE_STN_MST` | 앱 시작 시 | ODS Station ID 목록 로드 |

### 업데이트 테이블

| 테이블 | 업데이트 시점 | 변경 내용 |
|:---|:---|:---|
| `WES_SHIP_BOX_MST` | Release 큐 등록 직후 | `SHIP_TOTE_STATUS = 30` |

---

## 6. ODS 자동 폴링 구조

```mermaid
graph TD
    PLAY["PLAY 버튼"] -->|"폴링 시작"| THREAD["폴링 스레드<br/>daemon, 1초 주기"]
    PAUSE["PAUSE / RESET"] -->|"Event.set()"| STOP["폴링 중단"]

    THREAD --> QUERY["VW_SFA_DOS_CHANNEL_STATUS 조회<br/>(manual worker 제외)"]
    QUERY -->|"빈 채널 감지"| PARSE["LOCATION_ID 파싱<br/>F11-L1-01 → station=F11, channel=L11"]
    PARSE --> BARCODE["바코드 채번<br/>25000001 ~ 25899999"]
    BARCODE --> SPROC["SP_SFA_ODS_MAPPING_TARGET_TOTE_PRC<br/>실패 시 해당 row 중단"]
    SPROC -->|"성공"| API["POST /api/ods/create"]
    API --> THREAD

    style THREAD fill:#27ae60,color:#fff
    style STOP fill:#e74c3c,color:#fff
```

- **manual worker 제외**: UI 체크박스로 manual 설정된 station은 NOT IN 필터로 폴링 제외
- **바코드 채번**: `ods_barcode_seq` 카운터 (RESET 시 25000001로 초기화)
- **LOCATION_ID 파싱**: `'F11-L1-01'` → `station_id='F11'`, `channel_id='L11'`

---

## 7. API 현황

### EES → EM (12개)

| 분류 | API | 상태 |
|:---|:---|:---:|
| 제어 | Play · Pause · Reset | ✅ ✅ ✅ |
| 데이터 적재 | Create Stock · Create Source Buffer | ✅ ✅ |
| 데이터 적재 | Create Target Buffer | △ 비활성화 |
| ODS | Create ODS | ✅ 자동 폴링 |
| DECANT | Loading DECANT Source | ✅ |
| ODS 명령 | Release ODS Target | ✅ 자동 |
| ODS 명령 | Mapping ODS Target | △ 주석 처리 |
| 백업 조회 | Backup Stock · Backup ODS | ✅ ✅ |

### EM → EES (5개) — 전체 구현 완료

| 분류 | API |
|:---|:---|
| DECANT | Ready · Completed |
| ODS | Ready · Completed Source · Completed Release · Completed Mapping |

```mermaid
pie title 구현 현황 (17개)
    "구현 완료 (13)" : 13
    "자동화 대체 (2)" : 2
    "비활성화 (2)" : 2
```

---

## 8. 동시성 설계

### Flask → Worker 스레드 안전

```
Flask Worker Thread          OdsWorker._lock (threading.Lock)
      │                             │
      ├─ on_ready_source()    ─── with self._lock: ──→ 상태 전환
      ├─ on_completed_source() ─── with self._lock: ──→ 상태 전환 + DB 호출
      ├─ on_completed_release() ── with self._lock: ──→ 상태 전환
      ├─ on_completed_mapping() ── with self._lock: ──→ 상태 전환
      └─ reset()             ─── with self._lock: ──→ 전체 초기화
```

### tkinter 로그 스레드 안전

```
Flask / Worker Thread          Main Thread (tkinter)
      │                              │
      └─ log(msg)                    │
           └─ queue.put(msg)    ──→ _process_log_queue() [10ms 주기]
                                      └─ _update_log_ui() [위젯 업데이트]
```

- Flask/Worker 스레드에서 tkinter 위젯 직접 조작 금지 → `queue.Queue` 경유
- ODS 폴링 스레드: `threading.Event` 기반 중단 신호, daemon=True
- EM API 호출 (Release/Mapping): `threading.Thread(daemon=True)` 별도 스레드
