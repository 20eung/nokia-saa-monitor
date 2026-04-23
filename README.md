# Nokia 멀티 세대 장비 품질 모니터링 자동화

# Nokia SAA-driven Performance Monitoring System

## 1. 프로젝트 배경 및 목표

- **배경** : 7210 SAS, 7705 SAR, 7750 SR 등 다양한 세대의 Nokia 장비 운영 중.
- **목표** : Epipe(L2 P2P) 서비스의 품질(지연, 손실)을 실시간 수집하고, 이를 시각화하여 관제 업무에 활용.
- **특이사항** : 구형 장비(TiMOS v6.x, v7.x)의 SNMP 성능 이슈로 인해 Python 기반의 SSH Scraper(Push 방식) 아키텍처 채택.

---

## 2. 대상 네트워크 환경

| 장비 모델   | 주요 OS 버전      | 특이사항                        |
| :---------- | :---------------- | :------------------------------ |
| 7210 SAS-M  | TiMOS-B 7.0.R4    | 리소스 제약 심함, Classic CLI   |
| 7210 SAS-Sx | TiMOS-B 22.3.R3   | 최신형, Classic CLI             |
| 7705 SAR    | TiMOS-B 6.1.R7    | 셀룰러 백홀 장비, Classic CLI   |
| 7750 SR     | TiMOS-B 12.0.R9   | 표준 백본 장비, Classic CLI     |
| 7750 SR     | TiMOS-C 20.10.R13 | 최신 MD-CLI (Model-Driven) 적용 |

---

## 3. 시스템 아키텍처 (End-to-End Flow)

용역 업체가 구축해야 할 데이터 파이프라인은 다음과 같습니다.

```mermaid
graph TD
    subgraph "Field (Nokia Devices)"
        Target_Devices["7210 SAS / 7705 SAR / 7750 SR"]
        CLI_Old["Classic CLI (7705/7210/SR 12.x)"]
        CLI_New["MD-CLI (7750 SR 20.x)"]
        
        Target_Devices --> CLI_Old
        Target_Devices --> CLI_New
    end

    subgraph "Data Collection Layer"
        Collector["Python Collector App\n(Netmiko / TextFSM)"]
    end

    subgraph "Data Storage Layer"
        Telegraf["Telegraf\n(HTTP Listener v2)"]
        InfluxDB[("InfluxDB v1.8\n(Time Series DB)")]
    end

    subgraph "Visualization & Alerting"
        Grafana["Grafana\n(Dashboard & Monitoring)"]
        External_Alerts["Alert System\n(Slack, Email, n8n)"]
    end

    %% 연결
    CLI_Old -->|SSH Scrape| Collector
    CLI_New -->|SSH Scrape| Collector
    
    Collector -->|HTTP Push (JSON)| Telegraf
    Telegraf -->|Write Metrics| InfluxDB
    
    InfluxDB -->|Query| Grafana
    Grafana -->|Trigger| External_Alerts
```

---

## 4. 핵심 요구 사항 (Technical Spec)

### ① 데이터 수집 (Python Application)

- **라이브러리** : Netmiko (SSH), TextFSM (Parsing), Requests (HTTP Push).
- **동시성** : 수집 대상 장비 수에 따라 concurrent.futures를 이용한 멀티스레딩 구현 필수.
- **버전 대응** : TiMOS-B(Classic)와 TiMOS-C(MD-CLI)의 명령어 차이를 인식하는 로직 구현.
- **예외 처리** : 장비 접속 불가, 명령어 타임아웃, CPU 부하 시 재시도 로직 포함.

### ② 데이터 저장 및 처리 (Telegraf & InfluxDB v1.8)

- **Telegraf** : inputs.http_listener_v2를 활성화하여 Python 데이터를 수용.
- **InfluxDB** : Retention Policy(보관 기간)를 설정하여 데이터 무한 증식 방지.
- **Data Schema** :
  - **Tags** : host, epipe_id, port, service_type
  - **Fields** : avg_rtt, max_rtt, jitter, loss_count, if_in_octets, if_out_octets

### ③ 시각화 (Grafana)

- **통합 대시보드**: 트래픽(Traffic)과 품질(Latency/Loss)을 동일 시간축에서 비교 분석할 수 있도록 구성.
- **품질 지표 정의**:
  - **지연(Latency)** : Avg RTT 기준 (ms)
  - **손실(Loss)** : Packet Loss Count 및 Loss %
  - **상태(Health)** : SAP Error 및 Port Status
- **알람 연동**: 임계치 초과 시 Slack/Email/n8n Webhook 연동.

## 5. 용역 업체에 대한 가이드 (Advice)

- **보안**: 장비 접속 계정은 최소 권한(Read-only)을 부여하되, SAA 설정을 자동화해야 할 경우 config 권한이 필요할 수 있음을 명시.
- **성능**: 7210 SAS-M과 같은 저사양 장비는 수집 주기를 5분 이상으로 설정하거나, 필요한 데이터만 콕 집어서 가져오는 최적화된 Query를 사용할 것.
- **자율성**: 기존 시스템과 분리된 별도 VM(또는 컨테이너) 환경에서 end-to-end로 구축한 후 최종 검수.

---

# 참고자료 : 실제 구현 예제

## 1. 장비 측 설정: ETH-CFM 및 SAA 프로비저닝

품질 측정을 위해서는 Epipe 서비스 양 끝단에 **MEP(Maintenance Endpoint)** 가 상호 설정되어야 합니다.

### A. Classic CLI (7210, 7705, 7750 구형 버전)

가장 많은 비중을 차지하는 방식입니다.

```
# 1. CFM 도메인 및 어소시에이션 설정 (전역)
/configure eth-cfm
    domain 4 format none level 4
        association 1 format icc-based name "epipe100"
            bridge-identifier "100"
            exit
        exit
    exit

# 2. 서비스 내 MEP 생성 (Epipe ID 100 기준)
/configure service epipe 100
    sap 1/1/1:100 create
        eth-cfm
            mep 1 domain 4 association 1 direction down
                no shutdown
            exit
        exit
    exit
```

### B. SAA 테스트 자동 생성 (핵심 로직)

용역 업체는 Python을 이용해 아래 명령어를 자동으로 생성하고 입력해야 합니다.

```
/configure saa
    test "PM_EPIPE_100" owner "NetDevOps"
        type eth-cfm-two-way-delay
        # 상대방 장비의 MEP ID를 지정
        dest-mep 2 domain 4 association 1
        source-mep 1
        fc "ef"              # 우선순위 보장 (매우 중요)
        send-inter-packet-delay 10
        num-packets 10
        interval 60          # 60초마다 테스트 실행
        no shutdown
    exit
```

---

## 2. Python 수집기 구현 상세 (Step-by-Step)

업체가 개발해야 할 수집기(Collector)의 내부 로직은 다음과 같은 정교함이 필요합니다.

### Step 1: 서비스 리스트 및 상태 체크

먼저 장비에서 운영 중인 Epipe 리스트를 확보합니다.

- **명령어** : show service service-using epipe
- **체크 사항** : Admin/Oper Status가 Up인 서비스만 필터링하여 불필요한 에러 로그 방지.

### Step 2: 최신 품질 데이터 추출 및 파싱

가장 효율적인 데이터 추출 명령어는 latest 옵션을 활용하는 것입니다.

- **명령어** : show saa test-results "PM_EPIPE_100" owner "NetDevOps" latest

**[추출해야 할 정규표현식(Regex) 타겟 정보]**

```
표준 출력 예시:
-------------------------------------------------------------------------------
Test Name             : PM_EPIPE_100
Owner                 : NetDevOps
Last Run Time         : 2026/04/23 10:45:12
-------------------------------------------------------------------------------
Two-way Delay (ms)    : Min = 1.24, Max = 3.52, Avg = 1.85, Jitter = 0.42
Loss Packets          : 0
```

---

## 3. 리소스 관리 및 안정성 가이드 (RFP 필수 포함)

용역 업체가 간과하기 쉬운 노후 장비 보호 대책입니다.

1. **세션 제한 (Session Limit)** : 7210 SAS-M과 같은 장비는 동시 가동 가능한 SAA 세션 수에 한계가 있습니다. 한 대의 장비에 Epipe가 너무 많을 경우, 우선순위가 높은(Gold/Silver 등급) 서비스만 측정하거나 수집 주기를 10분 이상으로 늘리는 로직을 포함시켜야 합니다.

2. **SSH 커넥션 풀링** : 매번 접속하고 끊는 방식은 장비 CPU에 부담을 줍니다. 한 번 접속했을 때 해당 장비의 모든 SAA 데이터를 긁어오는 방식으로 최적화해야 합니다.

3. **타임아웃 처리** : 7705 SAR 같은 모델은 응답이 느릴 수 있으므로 SSH 타임아웃을 최소 30초 이상으로 설정하도록 지시하십시오.

---

## 4. 최종 결과물 정의 (Checklist)

용역 업체로부터 최종적으로 받아야 할 결과물입니다.

- **Collector App** : Python 소스 코드 (버전별 분기 로직 포함)
- **Parsing Templates** : Nokia CLI 파싱용 TextFSM 또는 Regex 파일
- **Telegraf Config** : HTTP Listener 및 InfluxDB 연동 설정 파일
- **Grafana Dashboard** : \* 장비별/포트별 통합 뷰어
  - 임계치 기반 알람 설정 (Latency > 50ms 등)
  - 회선별 SLA 리포트 (Monthly 가동률 자동 계산)
