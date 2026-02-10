# BGF 부산 물류센터 — EES ↔ EM 시스템 설계

> 2026-02-10 | API 명세 v4 기준

---

## 1. 시스템 구조

```mermaid
graph LR
    subgraph EES ["EES (Client :6000)"]
        direction TB
        UI["UI 제어판<br/>customtkinter"]
        CTRL["Flask 수신 서버"]
        WORKER["ODS Worker<br/>상태머신 + 작업큐"]
        UI --- WORKER
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

    style EES fill:#2c3e50,color:#ecf0f1,stroke:#2c3e50
    style EM fill:#1a5276,color:#ecf0f1,stroke:#1a5276
    style DB fill:#7d6608,color:#ecf0f1,stroke:#7d6608
```

**양방향 HTTP 통신** — EES와 EM이 각각 Flask 서버를 운영하며, 서로에게 HTTP 요청을 보내는 구조

| 구성요소 | 역할 |
|:---|:---|
| **EES** | 설비 제어 클라이언트 — 명령 전송 + 이벤트 수신 + DB 연동 |
| **EM** | 설비 시뮬레이터 서버 — 명령 수신 + 작업 완료 시 이벤트 콜백 |
| **Oracle DB** | WCS 연동 — 피킹 시간 조회, 토트 도착 통보 |

---

## 2. 운영 흐름 (Phase)

```mermaid
sequenceDiagram
    actor USER as 운영자
    participant EES as EES
    participant EM as EM

    rect rgb(231, 76, 60, 0.15)
        Note over EES,EM: Phase 0 · 초기화
        USER->>EES: RESET
        EES->>EM: POST /api/reset
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
    end

    rect rgb(52, 152, 219, 0.15)
        Note over EES,EM: Phase 3 · DECANT 작업 (자동 반복)
        loop 투입 Tote마다
            EM->>EES: Ready (Tote 준비됨)
            EES->>EM: Loading (디캔트 지시)
            EM->>EES: Completed (완료)
        end
    end

    rect rgb(142, 68, 173, 0.15)
        Note over EES,EM: Phase 4 · ODS 작업 (자동 반복)
        loop 피킹 Tote마다
            EM->>EES: Ready (Tote 도착)
            EES-->>EM: 200 {pickup_time}
            EM->>EES: Completed (피킹 완료)
        end
    end

    rect rgb(149, 165, 166, 0.15)
        Note over EES,EM: Phase 5 · 백업
        USER->>EES: PAUSE
        EES->>EM: GET /api/stock · GET /api/ods
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

    style P1 fill:#3498db,color:#fff
    style P2 fill:#8e44ad,color:#fff
    style P3 fill:#e67e22,color:#fff
```

| 패턴 | 대상 | 핵심 |
|:---|:---|:---|
| **이벤트-반응** | DECANT | EM이 Ready → EES가 자동으로 Loading 역호출 |
| **동기 요청-응답** | ODS Pickup | EM이 Ready → EES가 DB 조회 후 pickup_time 응답 |
| **비동기 명령-콜백** | ODS Release/Mapping | EES가 명령 → EM이 작업 후 Completed 콜백 |

---

## 4. ODS Worker 상태 머신

각 ODS 스테이션마다 Worker 인스턴스가 **상태 머신 + 작업 큐**로 동작합니다.

```mermaid
stateDiagram-v2
    [*] --> Idle
    Idle --> Working: Source Pickup<br/>(EM이 Ready 호출)
    Idle --> Releasing: Target Release<br/>(큐에서 실행)
    Idle --> Mapping: Target Mapping<br/>(큐에서 실행)

    Working --> Idle: Completed
    Releasing --> Idle: Completed
    Mapping --> Idle: Completed

    Working --> Error: 실패
    Releasing --> Error: 실패
    Mapping --> Error: 실패
    Error --> Idle: Reset
```

- **Source Pickup** — EM 이벤트로 즉시 진입 (큐 거치지 않음)
- **Release / Mapping** — 작업 큐(deque)에 등록 후 Idle일 때 순차 실행
- **Error** — Reset으로만 해제, 큐 전체 클리어

---

## 5. API 현황

### EES → EM (12개)

| 분류 | API | 상태 |
|:---|:---|:---:|
| 제어 | Play · Pause · Reset | ✅ ✅ ✅ |
| 데이터 적재 | Create Stock · Create Source | ✅ ✅ |
| 데이터 적재 | Create Target · Create ODS | ❌ ❌ |
| DECANT | Loading DECANT Source | ✅ |
| ODS 명령 | Release ODS Target · Mapping ODS Target | ❌ ❌ |
| 백업 조회 | Backup Stock · Backup ODS | ❌ ❌ |

### EM → EES (6개) — 전체 구현 완료

| 분류 | API |
|:---|:---|
| DECANT | Ready · Completed |
| ODS | Ready · Completed Source · Completed Release · Completed Mapping |

```mermaid
pie title 구현 현황 (18개)
    "구현 완료 (12)" : 12
    "미구현 (6)" : 6
```
