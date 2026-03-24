# BGF 부산 CDC 시뮬레이션 런처

---

## 1. 배경 및 목적

시뮬레이션을 실행하려면 **4개 프로세스를 정해진 순서대로 수동 실행**해야 했음.
순서를 빠뜨리거나 상태 확인이 어려운 문제 → **GUI 런처로 일원화**

```mermaid
flowchart LR
    subgraph AS-IS
        direction TB
        A1[WCS_COLLECTOR\n수동 실행] --> A2[bridge_v1.1\n수동 실행]
        A2 --> A3[main_conv_higher\n수동 실행]
        A3 --> A4[시뮬레이션\n수동 실행]
    end

    subgraph TO-BE
        B1[런처 실행 버튼 클릭] --> B2[자동 순서 실행\n상태 모니터링]
    end

    AS-IS -->|개선| TO-BE
```

---

## 2. 시스템 구성

```mermaid
graph TD
    Launcher["🖥️ 시뮬레이션 런처\n(simulation_launcher.py)"]

    subgraph MiddleWare
        WCS["WCS_COLLECTOR\n(WS_COLLECTOR.exe)"]
    end

    subgraph PythonServer
        Bridge["bridge_v1.1.exe"]
        Conv["main_conv_higher.exe"]
        Mon["monitor.exe\n시각화"]
    end

    subgraph Automod
        Sim["bgf_cdc_250811.exe\n시뮬레이션 본체"]
    end

    Launcher -->|실행 / 상태 감시| WCS
    Launcher -->|실행 / 상태 감시| Bridge
    Launcher -->|실행 / 상태 감시| Conv
    Launcher -->|실행 N회| Sim
    Launcher -->|초기 실행 시 자동 오픈| Mon
```

---

## 3. 주요 기능 및 기술 스택

### 실행 흐름

```mermaid
flowchart TD
    Click([실행 버튼 클릭])
    Click --> Mode{초기 실행 / 재실행}

    Mode -->|초기 실행| S1[미실행 서버 자동 시작]
    Mode -->|재실행| S2{서버 상태 확인\n10초 주기}

    S2 -->|정상| Keep[현재 프로세스 유지]
    S2 -->|비정상| Restart[종료 후 재시작]
    S2 -->|미실행| NewStart[새로 시작]

    S1 & Keep & Restart & NewStart --> SimLoop

    SimLoop["시뮬레이션 N회 반복 실행\n(bgf_cdc_250811.exe -w)"]
    SimLoop --> Done([완료])
```

### 기술 스택

```mermaid
mindmap
  root((런처))
    GUI
      customtkinter
      다크모드 · blue 테마
      1100×800 리사이즈 가능
    프로세스 관리
      psutil
      CREATE_NEW_CONSOLE
      별도 CMD 창 분리 실행
    설정 관리
      JSONC 파싱
      3개 설정 파일 일괄 저장
      UI에서 직접 수정 가능
    언어 · 환경
      Python 3
      Windows 11
```

---

## 4. 데모

<!-- 스크린샷 / 영상 삽입 -->
