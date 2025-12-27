# **Temporal 분산 오케스트레이션 플랫폼 심층 기술 분석 보고서: 다중 에이전트 관점의 아키텍처 및 기능 명세**

## **1\. 서론: 분산 시스템의 새로운 패러다임과 보고서의 목적**

현대 소프트웨어 아키텍처가 마이크로서비스(Microservices)와 클라우드 네이티브(Cloud-Native) 환경으로 진화함에 따라, 서비스 간의 신뢰성 있는 통신과 상태 관리(State Management)는 엔지니어링의 핵심 난제로 부상했습니다. Temporal은 이러한 분산 시스템의 본질적인 불확실성—네트워크 지연, 서비스 중단, 인프라 장애—을 '내구성 있는 실행(Durable Execution)'이라는 추상화 계층을 통해 해결하고자 하는 플랫폼입니다.

본 보고서는 Temporal 라이브러리의 기술적 실체를 **시스템을 구성하는 주요 에이전트(Agent)**—워크플로우(Workflow), 액티비티(Activity), 워커(Worker), 서버(Server), 클라이언트(Client)—의 관점에서 해체하고 재조립하여 심층 분석합니다. 단순한 기능 나열을 넘어, 각 에이전트가 시스템 내에서 수행하는 역할, 상호작용 프로토콜, 그리고 실패 시 복구 메커니즘을 상세히 규명함으로써, 기업의 기술 의사결정권자가 Temporal 도입 시 고려해야 할 아키텍처적 장단점과 운영 전략을 수립하는 데 필요한 포괄적인 근거 자료를 제공하는 것을 목적으로 합니다.1

## ---

**2\. 아키텍처 개요: 중앙 집중식 오케스트레이션과 에이전트 기반 실행**

Temporal의 아키텍처는 전통적인 메시지 큐 기반의 비동기 시스템이나 단순한 작업 스케줄러와는 근본적으로 다른 접근 방식을 취합니다. 핵심은 \*\*상태의 중앙 집중화(Server)\*\*와 \*\*실행의 분산화(Worker)\*\*라는 이분법적 구조에 있습니다. 이 구조 하에서 시스템은 크게 네 가지 주요 에이전트로 나뉘며, 각 에이전트는 엄격하게 정의된 프로토콜을 통해 소통합니다.4

### **2.1 시스템 구성 요소 및 역할 분담**

Temporal 시스템을 유기체에 비유하자면, 서버는 기억과 명령을 담당하는 '뇌'에 해당하며, 워커는 실제 물리적 행동을 수행하는 '근육'에 해당합니다. 클라이언트는 외부 자극을 전달하는 '감각 기관'으로 볼 수 있습니다.

| 에이전트 (Agent) | 주요 역할 (Primary Role) | 책임 범위 (Responsibilities) | 통신 프로토콜 |
| :---- | :---- | :---- | :---- |
| **Server** | 상태 관리 및 중개 (State & Broker) | 이벤트 히스토리 저장, 태스크 큐 관리, 타이머 관리, 샤딩 및 복제 | gRPC (Frontend) |
| **Worker** | 비즈니스 로직 실행 (Execution) | 워크플로우/액티비티 코드 호스팅, 태스크 폴링, 결정론적 리플레이 수행 | Polling / gRPC |
| **Workflow** | 오케스트레이션 로직 (Orchestration) | 비즈니스 프로세스 흐름 정의, 보상 트랜잭션 관리, 상태 전이 제어 | Deterministic API |
| **Activity** | 부작용 실행 (Side Effect) | 외부 API 호출, DB 접근, 파일 I/O 등 비결정론적 작업 수행 | Implementation Specific |
| **Client** | 트리거 및 제어 (Control) | 워크플로우 시작/종료/취소, 시그널 전송, 상태 쿼리 및 업데이트 요청 | gRPC |

이러한 역할 분담은 \*\*확장성(Scalability)\*\*과 \*\*신뢰성(Reliability)\*\*이라는 두 마리 토끼를 잡기 위해 설계되었습니다. 서버는 비즈니스 로직을 전혀 알 필요가 없으므로 범용적인 확장이 가능하며, 워커는 상태를 저장하지 않으므로(Stateless) 언제든 교체되거나 수평 확장될 수 있습니다.

## ---

**3\. 워커 에이전트 (Worker Agent): 실행의 최전선과 폴링 메커니즘**

워커 에이전트는 Temporal 시스템에서 가장 능동적인 구성 요소입니다. 개발자가 작성한 코드가 실제로 구동되는 런타임 환경이며, Temporal 서버와 끊임없이 소통하며 작업을 가져와 처리하고 결과를 보고합니다. 워커의 효율적인 구성과 튜닝은 전체 시스템의 처리량(Throughput)과 지연 시간(Latency)을 결정짓는 핵심 요소입니다.4

### **3.1 롱 폴링(Long Polling) 기반의 작업 획득 구조**

워커는 서버가 작업을 밀어넣어주기를 기다리는 수동적인 존재가 아닙니다. 워커는 능동적으로 서버의 \*\*태스크 큐(Task Queue)\*\*에 접속하여 "처리할 작업이 있는가?"를 묻는 **롱 폴링** 방식을 사용합니다.

* **방화벽 및 NAT 친화적 아키텍처:** 이 방식의 가장 큰 장점은 서버가 워커의 IP 주소나 위치를 알 필요가 없다는 점입니다. 워커는 사설 네트워크 내부, Kubernetes 클러스터, 혹은 개발자의 로컬 노트북 어디서든 실행될 수 있으며, 아웃바운드(Outbound) 연결만 가능하다면 서버와 통신할 수 있습니다. 이는 하이브리드 클라우드 환경이나 보안이 엄격한 기업 네트워크 환경에서 Temporal 도입을 용이하게 합니다.  
* **배압(Backpressure) 조절:** 워커가 자신의 처리 용량(Capacity)에 맞춰 작업을 요청하므로, 서버의 부하가 급증해도 워커가 과부하로 쓰러지는 것을 방지할 수 있습니다. 워커가 바쁘면 폴링 빈도를 줄이거나 멈춤으로써 자연스럽게 유입 속도가 조절됩니다.6

### **3.2 워커 엔티티와 슬롯 관리 (Slot Management)**

하나의 워커 프로세스 내부에는 여러 개의 '워커 엔티티(Worker Entity)'가 존재할 수 있습니다. 각 엔티티는 특정 태스크 큐를 전담하여 처리합니다. 워커의 성능은 \*\*슬롯(Slot)\*\*이라는 개념으로 관리됩니다.7

* **워크플로우 태스크 슬롯 vs 액티비티 태스크 슬롯:** 워커는 워크플로우 진행을 담당하는 슬롯과 액티비티 실행을 담당하는 슬롯을 별도로 관리합니다. 이는 무거운 액티비티 작업이 워크플로우의 경량 오케스트레이션 작업을 차단(Block)하지 않도록 하기 위함입니다.  
* **슬롯 고갈과 폴링 중단:** worker\_task\_slots\_available 메트릭이 0이 되면 워커는 더 이상 해당 유형의 태스크를 폴링하지 않습니다. 이는 시스템이 안정적으로 동작하고 있음을 의미하기도 하지만, 반대로 특정 작업이 슬롯을 점유한 채 반환하지 않고 있다면(예: 데드락, 무한 대기) 전체 파이프라인이 멈추는 원인이 됩니다. 따라서 SDK 메트릭을 통한 슬롯 가용성 모니터링은 운영의 필수 요소입니다.7  
* **Eager Workflow Start:** 최신 Temporal 버전(Server 1.29+ 및 호환 SDK)에서는 지연 시간을 최소화하기 위해 **Eager Workflow Start** 기능을 지원합니다. 이는 워크플로우 시작 요청을 받은 서버가 태스크 큐를 거치지 않고, 요청을 보낸 클라이언트(만약 워커 역할도 겸하고 있다면)에게 즉시 첫 번째 태스크를 할당하는 최적화 기법입니다.8

### **3.3 튜닝 및 성능 최적화 전략**

워커의 성능을 최적화하기 위해서는 단순한 CPU/메모리 증설 외에도 정교한 설정이 필요합니다.9

* **캐시 크기 (Workflow Cache Size):** 워커는 자주 실행되는 워크플로우의 상태를 메모리에 캐싱(Sticky Execution)하여, 매번 전체 이벤트 히스토리를 다시 받아오는 오버헤드를 줄입니다. 캐시 적중률(Cache Hit Ratio)을 높이기 위해 워커의 메모리 용량에 맞춰 캐시 크기를 적절히 조절해야 합니다.  
* **폴러 수 (Max Concurrent Pollers):** 동시에 서버에 폴링 요청을 보내는 고루틴/스레드의 수입니다. 네트워크 대기 시간이 길다면 폴러 수를 늘려 처리량을 높일 수 있지만, 과도한 폴러는 서버의 매칭 서비스(Matching Service)에 부하를 줄 수 있으므로 주의해야 합니다.

## ---

**4\. 워크플로우 에이전트 (Workflow Agent): 결정론적 오케스트레이션의 미학**

워크플로우 에이전트는 비즈니스 로직의 '순서'와 '규칙'을 정의하는 사령관입니다. Temporal에서 워크플로우는 단순히 코드를 순차적으로 실행하는 것이 아니라, 이벤트 히스토리를 기반으로 상태를 재구성하는 **리플레이(Replay)** 메커니즘 위에서 동작합니다. 이 독특한 실행 모델은 개발자에게 \*\*결정론(Determinism)\*\*이라는 엄격한 제약을 요구합니다.10

### **4.1 결정론(Determinism)의 원칙과 리플레이 메커니즘**

워크플로우 코드는 실행 도중 서버 장애나 워커 프로세스 재시작 등으로 인해 언제든지 중단될 수 있습니다. Temporal은 중단된 지점부터 다시 실행하기 위해, 워크플로우의 처음부터 코드를 재실행(Replay)하며 과거에 기록된 이벤트 결과값들을 주입합니다.

* **동일한 입력, 동일한 명령:** 리플레이 시 워크플로우 코드는 과거 실행과 정확히 동일한 순서로 동일한 명령(Command)을 생성해야 합니다. 만약 코드 변경이나 비결정론적 요소(예: System.currentTimeMillis(), UUID.randomUUID(), 외부 API 직접 호출)로 인해 생성되는 명령의 순서나 내용이 달라지면, 워커는 NonDeterministicError를 발생시키고 해당 워크플로우를 차단합니다.12  
* **샌드박싱과 언어별 제약:** 각 언어별 SDK는 이러한 결정론을 강제하거나 돕기 위해 다양한 기법을 사용합니다.  
  * **Java/Go:** 개발자가 직접 주의해야 하는 부분이 많지만, 정적 분석 도구나 린터(Linter)를 통해 비결정론적 코드 사용을 경고합니다.  
  * **TypeScript:** Node.js의 vm 모듈 혹은 v8-isolate를 사용하여 워크플로우 코드를 완전히 격리된 샌드박스 환경에서 실행합니다. 이를 통해 전역 변수 오염이나 비결정론적 API 접근을 원천적으로 차단합니다.14  
  * **Python:** asyncio 이벤트 루프를 기반으로 하며, 워크플로우 로직 내에서의 외부 통신을 엄격히 제한합니다.15

### **4.2 이벤트 히스토리와 상태 관리**

워크플로우의 '상태'는 메모리가 아닌 이벤트 히스토리에 존재합니다.

* **이벤트 소싱(Event Sourcing):** 모든 상태 변화는 이벤트로 기록됩니다 (WorkflowExecutionStarted, ActivityTaskScheduled, TimerFired 등). 워커는 이 로그를 읽어 현재의 메모리 상태를 복원합니다.16  
* **히스토리 크기 제한과 Continue-As-New:** 단일 워크플로우의 이벤트 히스토리가 너무 커지면(기본 50,000 이벤트 또는 50MB), 리플레이 속도가 느려지고 시스템 성능이 저하됩니다. 이를 해결하기 위해 Temporal은 Continue-As-New 패턴을 제공합니다. 이는 현재 실행을 종료하고, 최신 상태(State)만을 입력값으로 넘겨 완전히 새로운 워크플로우 실행(Run ID 변경)을 시작하는 방식입니다. 이를 통해 히스토리 사이즈를 초기화하면서 무한 루프 워크플로우를 구현할 수 있습니다.16

### **4.3 자식 워크플로우 (Child Workflow)와 파티셔닝**

복잡하고 방대한 작업을 단일 워크플로우에서 처리하면 히스토리 제한에 걸릴 위험이 큽니다. 이때 **자식 워크플로우**를 사용하여 로직을 분할(Partitioning)합니다.17

* **격리된 실패 도메인:** 자식 워크플로우는 부모와 독립적인 이벤트 히스토리를 가집니다. 부모는 자식의 시작과 종료 이벤트만 기록하므로, 수천 개의 액티비티를 수행하는 작업을 여러 자식 워크플로우로 나누면 부모의 히스토리를 가볍게 유지할 수 있습니다.  
* **부모 종료 정책 (Parent Close Policy):** 부모가 취소되거나 종료될 때 자식의 운명을 결정할 수 있습니다 (ABANDON, REQUEST\_CANCEL, TERMINATE). ABANDON을 사용하면 부모가 죽더라도 자식은 계속 실행되어, 비동기적인 후처리가 필요한 작업에 유용합니다.

## ---

**5\. 액티비티 에이전트 (Activity Agent): 부작용의 캡슐화와 신뢰성 보장**

액티비티 에이전트는 워크플로우의 지시를 받아 외부 세계와 상호작용하는 역할을 합니다. API 호출, 데이터베이스 트랜잭션, 파일 처리 등 실패 가능성이 높고 비결정론적인 작업은 반드시 액티비티로 캡슐화되어야 합니다. Temporal은 액티비티의 안정적인 실행을 위해 정교한 타임아웃 및 재시도 정책을 제공합니다.18

### **5.1 타임아웃(Timeout)의 수학과 전략**

액티비티의 실패를 감지하는 것은 분산 시스템에서 매우 어렵습니다. "응답이 없는 것"이 "작업 중"인지 "장애 발생"인지 구분할 수 없기 때문입니다. Temporal은 이를 네 가지 타임아웃 조합으로 해결합니다.

| 타임아웃 종류 | 정의 및 목적 | 설정 전략 |
| :---- | :---- | :---- |
| **Schedule-To-Start** | 큐에 들어간 후 워커가 픽업하기까지의 대기 시간. | 워커 풀의 백로그(Backlog)나 다운 타임을 감지합니다. 일반적으로 짧게 설정하지 않으며, 모니터링 지표로 주로 활용합니다. |
| **Start-To-Close** | \*\*단일 시도(Attempt)\*\*의 실행 시간 제한. | 워커 프로세스가 작업 도중 크래시(Crash)되는 것을 감지하는 핵심 수단입니다. 예상 실행 시간보다 약간 여유 있게 설정하여, 워커 장애 시 빠르게 재시도하도록 해야 합니다. |
| **Schedule-To-Close** | 전체 실행(재시도 포함)의 총 시간 제한. | 비즈니스 요구사항(SLA)에 따라 설정합니다. (예: "결제 처리는 5분 내에 완료되어야 한다.") |
| **Heartbeat Timeout** | 워커가 "살아있음"을 보고해야 하는 주기. | 장기 실행 작업(Long-running activity)에서 필수적입니다. 작업 시간이 길 경우 Start-To-Close를 길게 잡아야 하는데, 이때 워커 장애 감지가 늦어질 수 있습니다. 하트비트를 사용하면 긴 작업 중에도 워커 다운을 즉각 감지할 수 있습니다. |

**분석적 통찰:** Start-To-Close 타임아웃을 설정하지 않고 기본값(무한대)으로 두는 것은 가장 흔한 실수 중 하나입니다. 이 경우 워커가 작업을 수행하다가 전원이 꺼지면, 서버는 해당 작업이 영원히 실행 중인 것으로 간주하여 재시도를 트리거하지 않습니다. 따라서 모든 액티비티에는 적절한 Start-To-Close 타임아웃이 명시되어야 합니다.6

### **5.2 하트비트(Heartbeating)와 비동기 완료**

하트비트는 단순한 생존 신고 이상의 기능을 수행합니다.

* **진행 상황 저장 (Checkpointing):** 액티비티는 하트비트를 보낼 때 현재 진행률이나 상태 데이터(Payload)를 함께 보낼 수 있습니다. 액티비티가 실패하고 재시도될 때, 이 데이터를 읽어와 처음부터가 아닌 중단된 지점부터 작업을 재개(Resume)할 수 있습니다. 이는 대용량 파일 처리나 머신러닝 학습과 같은 작업에서 엄청난 효율성을 제공합니다.18  
* **비동기 완료 (Async Completion):** 어떤 액티비티는 워커가 직접 완료할 수 없고, 외부 시스템의 콜백(Callback)이나 사람의 승인을 기다려야 할 수 있습니다. 이때 워커는 액티비티를 시작만 하고 즉시 반환하지 않은 채 종료될 수 있으며, 나중에 외부 시스템이 Temporal 클라이언트를 통해 해당 액티비티를 Complete 시킬 수 있습니다. 이는 "인적 개입(Human-in-the-Loop)" 프로세스 구현의 핵심 패턴입니다.20

## ---

**6\. 서버 에이전트 (Server Agent): 클러스터링, 지속성, 그리고 확장성**

Temporal 서버는 시스템의 심장부로서, 데이터의 일관성을 보장하고 대규모 트래픽을 처리하기 위한 복잡한 내부 아키텍처를 가지고 있습니다. 서버는 크게 네 가지 독립적인 서비스로 구성되며, 각각 개별적으로 스케일링이 가능합니다.5

### **6.1 내부 서비스 아키텍처**

1. **Frontend Service:** 외부(클라이언트, 워커)와의 통신 관문입니다. 인증/인가, 속도 제한(Rate Limiting), 요청 라우팅을 담당합니다. 상태를 가지지 않으므로 로드 밸런서 뒤에서 쉽게 수평 확장이 가능합니다.  
2. **History Service:** 시스템에서 가장 중요한 컴포넌트로, 워크플로우의 상태 전이(State Transition)를 처리하고 데이터베이스에 기록합니다. **샤딩(Sharding)** 기술을 사용하여 수많은 워크플로우 ID를 여러 History 호스트에 분산시킵니다. 각 샤드는 특정 워크플로우 ID 범위에 대한 소유권을 가지며, 이를 통해 데이터베이스 락 경합을 최소화하고 처리량을 극대화합니다.  
3. **Matching Service:** 태스크 큐를 관리합니다. 워크플로우나 액티비티 태스크가 생성되면 이를 메모리 내의 큐에 담고, 폴링하러 온 워커에게 전달합니다. 큐의 데이터를 DB에 쓰지 않고 메모리에서 바로 전달하는 'Sync Match' 기능을 통해 지연 시간을 획기적으로 줄입니다.  
4. **Worker Service:** Temporal 내부 시스템용 워크플로우(예: 아카이빙, 교차 클러스터 복제 등)를 실행하는 백그라운드 워커입니다.

### **6.2 다중 클러스터 복제 (Multi-Cluster Replication)와 데이터 일관성**

엔터프라이즈 환경에서는 재해 복구(DR)를 위해 다중 리전/다중 클러스터 구성을 요구합니다. Temporal은 이를 위해 **XDC (Cross Data Center) Replication**을 지원합니다.21

* **비동기 복제와 결과적 일관성:** 클러스터 간 데이터 복제는 비동기적으로 이루어집니다. 따라서 글로벌 네임스페이스 환경에서는 **CAP 이론** 중 **AP(Availability \+ Partition Tolerance)** 시스템에 가깝습니다. 즉, 네트워크 단절 시에도 서비스 가용성을 유지하지만, 데이터의 순간적인 불일치는 허용합니다.  
* **충돌 해결 (Conflict Resolution):** Active 클러스터가 장애로 인해 Passive 클러스터로 절체(Failover)될 때, 아직 복제되지 않은 이벤트가 있을 수 있습니다. 이때 Temporal은 \*\*벡터 시계(Vector Clock)\*\*와 유사한 버전 관리 메커니즘을 사용합니다. 각 클러스터는 고유한 버전 값을 가지며, 충돌 발생 시 가장 높은 버전을 가진 히스토리를 '진실(Source of Truth)'로 채택하고, 낮은 버전의 히스토리는 폐기하거나 재조정합니다. 이 과정에서 일부 진행 상황이 롤백될 수 있으므로, 애플리케이션 설계 시 이를 고려해야 합니다.

### **6.3 가시성(Visibility)과 아카이빙(Archival)**

Temporal은 워크플로우의 실행 상태뿐만 아니라, 비즈니스 데이터를 기반으로 한 검색 기능도 제공합니다.22

* **이중 지속성 모델:** 실행 히스토리는 Cassandra/MySQL/PostgreSQL 같은 트랜잭션 DB에 저장되지만, 검색을 위한 '가시성 레코드'는 Elasticsearch와 같은 검색 엔진에 별도로 인덱싱됩니다. 이를 통해 "지난 24시간 동안 '서울' 지역에서 실패한 '배달' 주문"과 같은 복잡한 쿼리가 가능해집니다.  
* **아카이빙:** 완료된 워크플로우는 보존 기간(Retention Period)이 지나면 DB에서 삭제됩니다. 컴플라이언스 준수나 장기 분석을 위해, 삭제 전 데이터를 S3나 GCS 같은 저렴한 오브젝트 스토리지로 내보내는 아카이빙 기능을 설정할 수 있습니다.24

## ---

**7\. 클라이언트 에이전트 (Client Agent): 통합과 상호작용 패턴**

클라이언트 에이전트는 외부 시스템이 Temporal 워크플로우를 제어하는 리모콘입니다. 워크플로우를 시작하고, 상태를 조회하며, 실행 중인 프로세스에 개입합니다.

### **7.1 통신 패턴: 시그널, 쿼리, 업데이트**

실행 중인 워크플로우와의 상호작용은 Temporal의 강력한 기능 중 하나입니다. 단순한 호출을 넘어, 실행 중인 프로세스의 흐름을 동적으로 변경할 수 있습니다.25

| 기능 | 통신 방식 | 목적 및 특징 | 일관성 모델 |
| :---- | :---- | :---- | :---- |
| **Query** | 동기식 (Read-only) | 워크플로우의 현재 내부 상태 조회. 상태 변경 불가. | 강한 일관성 (Strong Consistency) \- 워커가 최신 이벤트를 처리한 후 응답. |
| **Signal** | 비동기식 (Write) | 워크플로우에 데이터 주입. 반환값 없음 (Fire-and-forget). | 결과적 일관성 \- 시그널이 큐에 들어가고 나중에 처리됨. |
| **Update** | 동기식 (Write+Read) | 워크플로우 상태 변경 요청 후, **처리 완료 및 결과 반환**까지 대기. | 강한 일관성 \- 유효성 검사(Validator)를 거쳐 잘못된 요청 거부 가능. 시그널과 쿼리의 장점 결합. |

**분석적 통찰:** 최신 Temporal 버전에서 도입된 **Workflow Update**는 시그널의 한계를 극복했습니다. 시그널은 보낸 후 처리가 잘 되었는지 알 방법이 없어 별도의 액티비티나 쿼리로 확인해야 했으나, 업데이트는 RPC 호출처럼 요청-응답 패턴을 완벽하게 지원하므로 클라이언트 코드의 복잡성을 크게 줄여줍니다.27

### **7.2 Nexus: 경계를 넘어서는 연결**

마이크로서비스 아키텍처가 고도화되면서, 서로 다른 팀이 관리하는 Temporal 네임스페이스 간, 혹은 완전히 다른 클러스터 간의 호출이 필요해졌습니다. **Temporal Nexus**는 이를 위한 차세대 기능입니다.28

* **서비스로서의 워크플로우:** Nexus를 사용하면 특정 네임스페이스의 워크플로우를 HTTP 엔드포인트처럼 외부에 노출할 수 있습니다. 호출자(Caller)는 상대방이 Temporal을 쓰는지 알 필요 없이 표준화된 인터페이스로 요청을 보내고, Nexus 게이트웨이가 이를 받아 내부 워크플로우 실행이나 시그널로 변환합니다.  
* **장점:** 팀 간의 결합도를 낮추고(Decoupling), 각자의 배포 주기와 네임스페이스 격리 정책을 유지하면서도 협업할 수 있는 '서비스 메쉬(Service Mesh)'와 유사한 역할을 수행합니다.

## ---

**8\. 고급 개발 패턴 및 SDK 기능 매트릭스**

Temporal을 단순한 스케줄러가 아닌 애플리케이션 플랫폼으로 활용하기 위한 핵심 디자인 패턴과 언어별 지원 현황입니다.

### **8.1 사가(Saga) 패턴: 분산 트랜잭션의 정석**

마이크로서비스 환경에서 ACID 트랜잭션을 보장하는 것은 불가능합니다. Temporal은 **사가 패턴**을 코드 레벨에서 직관적으로 구현할 수 있게 지원합니다.31

* **구현:** try-catch-finally 블록을 사용하여, 액티비티 실행(정방향 트랜잭션) 후 보상 액티비티(역방향 트랜잭션)를 스택에 쌓습니다. 만약 중간에 에러가 발생하면, catch 블록에서 스택에 쌓인 보상 액티비티들을 역순으로 실행하여 시스템 상태를 롤백합니다.  
* **Temporal의 강점:** 일반적인 사가 구현체는 보상 트랜잭션 실행 중에 시스템이 다운되면 롤백 자체가 실패하여 데이터 불일치가 발생할 수 있습니다. 하지만 Temporal은 보상 액티비티 실행 자체도 내구성 있게(Durable) 보장하므로, \*\*"반드시 성공하는 롤백"\*\*을 구현할 수 있습니다.

### **8.2 엔티티(Entity) 패턴: 상태를 가진 액터**

워크플로우를 프로세스가 아닌 '객체'로 바라보는 관점입니다. 예를 들어, "사용자 ID: 1234"에 해당하는 워크플로우를 하나 띄워두고, 이를 해당 사용자의 상태를 관리하는 전용 서버처럼 활용합니다.16

* **구조:** 무한 루프를 돌며 시그널을 대기합니다. 시그널이 오면 상태를 변경하고 필요한 액티비티를 수행합니다.  
* **동시성 제어:** 특정 엔티티에 대한 모든 요청이 단일 워크플로우 인스턴스로 들어오므로, 별도의 DB 락(Lock) 없이도 자연스럽게 동시성 제어(Serialization)가 이루어집니다. 이는 재고 관리나 포인트 시스템 등에서 강력한 위력을 발휘합니다.

### **8.3 SDK 기능 지원 매트릭스**

모든 SDK가 동일한 속도로 발전하는 것은 아닙니다. 도입 언어 결정 시 다음 매트릭스를 고려해야 합니다.34

| 기능 (Feature) | Go SDK | Java SDK | Python SDK | TypeScript SDK | .NET SDK |
| :---- | :---- | :---- | :---- | :---- | :---- |
| **기반 (Core)** | Native Go | Native Java | Rust Core 기반 | Rust Core 기반 | Rust Core 기반 |
| **Workflow Update** | Stable | Stable | Pre-release | Pre-release | Pre-release |
| **Nexus** | 지원 | 지원 | 개발 중 | 개발 중 | 개발 중 |
| **Generics/Type Safety** | 우수 | 우수 | 동적 타이핑 (Type hints) | 우수 | 우수 |
| **샌드박싱 방식** | 린터/주의 필요 | 린터/주의 필요 | 런타임 제한 | v8-isolate (강력) | 런타임 제한 |

**Go/Java SDK**는 가장 역사가 길고 기능이 풍부하며, 네이티브로 구현되어 성능이 뛰어납니다. **Python/TypeScript/.NET SDK**는 Rust로 작성된 'Core SDK'를 공유하는 구조로 되어 있어, 새로운 기능이 Rust Core에 구현되면 빠르게 동시 적용되는 장점이 있습니다. 특히 TypeScript SDK는 V8 엔진의 격리 기능을 활용하여 가장 안전한 샌드박스 환경을 제공합니다.

## ---

**9\. 운영 및 의사결정을 위한 제언**

Temporal 도입은 기술 스택의 변화를 넘어 아키텍처의 패러다임 시프트를 의미합니다. 의사결정권자는 다음 사항을 고려해야 합니다.

### **9.1 도입의 장점 (Pros)**

1. **코드 복잡성 제거:** 재시도, 타임아웃, 비동기 통신, 상태 저장 등의 복잡한 인프라 코드를 비즈니스 로직에서 완전히 제거할 수 있습니다. 개발자는 "성공 경로(Happy Path)"에만 집중하면 됩니다.  
2. **가시성 확보:** 분산된 마이크로서비스들의 실행 흐름을 중앙에서 추적하고 모니터링할 수 있어 문제 해결(Troubleshooting) 속도가 획기적으로 빨라집니다.  
3. **운영 탄력성:** 워커의 무중단 배포, 스케일링이 용이하며, 장애 발생 시 데이터 유실 없이 자동 복구되는 시스템을 구축할 수 있습니다.

### **9.2 도입 시 고려사항 및 비용 (Cons)**

1. **학습 곡선:** '결정론적 실행', '이벤트 히스토리', '버전 관리' 등 생소한 개념에 대한 개발자들의 학습 비용이 높습니다. 잘못된 코드 작성(비결정론적 코드)은 런타임 장애로 직결됩니다.  
2. **운영 오버헤드:** 자체 호스팅(Self-hosted) 시 Cassandra/MySQL, Elasticsearch, Prometheus, Grafana 등 관리해야 할 인프라 요소가 많습니다. 관리형 서비스(Temporal Cloud) 사용 시 비용 모델을 검토해야 합니다.  
3. **지연 시간 (Latency):** 모든 단계가 DB 트랜잭션을 거치므로, 밀리초(ms) 단위의 초저지연 응답이 필요한 실시간 트레이딩 시스템 등에는 적합하지 않습니다. (단, Eager Workflow Start 등으로 개선 중)

### **9.3 결론**

Temporal은 "실패는 선택이 아닌 필수"라는 분산 시스템의 현실을 있는 그대로 수용하고, 이를 시스템 레벨에서 우아하게 처리하는 플랫폼입니다. 복잡한 비즈니스 프로세스, 장기 실행 작업, 그리고 높은 신뢰성이 요구되는 금융 및 커머스 트랜잭션 시스템에 있어 Temporal은 현존하는 가장 강력한 오케스트레이션 솔루션입니다. 본 보고서에 상세히 기술된 각 에이전트의 특성과 제약 사항을 면밀히 검토하여 아키텍처에 반영한다면, 견고하고 확장 가능한 차세대 시스템을 구축하는 데 결정적인 기여를 할 것입니다.

#### **참고 자료**

1. Understanding Temporal: Building Resilient Distributed Systems | by Dilip Tadepalli, 12월 27, 2025에 액세스, [https://medium.com/@tadepalli.dilip/understanding-temporal-building-resilient-distributed-systems-cca5b561f46d](https://medium.com/@tadepalli.dilip/understanding-temporal-building-resilient-distributed-systems-cca5b561f46d)  
2. The definitive guide to Durable Execution \- Temporal, 12월 27, 2025에 액세스, [https://temporal.io/blog/what-is-durable-execution](https://temporal.io/blog/what-is-durable-execution)  
3. The distributed machine | Temporal, 12월 27, 2025에 액세스, [https://temporal.io/blog/the-distributed-machine](https://temporal.io/blog/the-distributed-machine)  
4. What is a Temporal Worker? | Temporal Platform Documentation, 12월 27, 2025에 액세스, [https://docs.temporal.io/workers](https://docs.temporal.io/workers)  
5. How the Temporal Platform Works, 12월 27, 2025에 액세스, [https://temporal.io/how-it-works](https://temporal.io/how-it-works)  
6. Activity poller becomes inactive, activities stuck in PENDING\_ACTIVITY\_STATE\_SCHEDULED state \- Temporal Community Forum, 12월 27, 2025에 액세스, [https://community.temporal.io/t/activity-poller-becomes-inactive-activities-stuck-in-pending-activity-state-scheduled-state/16974](https://community.temporal.io/t/activity-poller-becomes-inactive-activities-stuck-in-pending-activity-state-scheduled-state/16974)  
7. Workers not polling for tasks \- Temporal Community Forum, 12월 27, 2025에 액세스, [https://community.temporal.io/t/workers-not-polling-for-tasks/16411](https://community.temporal.io/t/workers-not-polling-for-tasks/16411)  
8. Worker performance | Temporal Platform Documentation, 12월 27, 2025에 액세스, [https://docs.temporal.io/develop/worker-performance](https://docs.temporal.io/develop/worker-performance)  
9. An introduction to Worker tuning \- Temporal, 12월 27, 2025에 액세스, [https://temporal.io/blog/an-introduction-to-worker-tuning](https://temporal.io/blog/an-introduction-to-worker-tuning)  
10. System Design: A Breakdown of Temporal's Internal Architecture by Sanil Khurana | Data Science Collective \- Medium, 12월 27, 2025에 액세스, [https://medium.com/data-science-collective/system-design-series-a-step-by-step-breakdown-of-temporals-internal-architecture-52340cc36f30](https://medium.com/data-science-collective/system-design-series-a-step-by-step-breakdown-of-temporals-internal-architecture-52340cc36f30)  
11. Temporal Workflow | Temporal Platform Documentation, 12월 27, 2025에 액세스, [https://docs.temporal.io/workflows](https://docs.temporal.io/workflows)  
12. Temporal Fundamentals Part IV: Workflows \- Keith Tenzer's Blog, 12월 27, 2025에 액세스, [https://keithtenzer.com/temporal/Temporal\_Fundamentals\_Workflows/](https://keithtenzer.com/temporal/Temporal_Fundamentals_Workflows/)  
13. Workflow Determinism \- Developer Corner \- Temporal Community Forum, 12월 27, 2025에 액세스, [https://community.temporal.io/t/workflow-determinism/4027](https://community.temporal.io/t/workflow-determinism/4027)  
14. Comparisons between the go/js sdks \- Community Support \- Temporal, 12월 27, 2025에 액세스, [https://community.temporal.io/t/comparisons-between-the-go-js-sdks/10446](https://community.temporal.io/t/comparisons-between-the-go-js-sdks/10446)  
15. Temporal Python SDK | Durable Asyncio Event Loop, 12월 27, 2025에 액세스, [https://temporal.io/blog/durable-distributed-asyncio-event-loop](https://temporal.io/blog/durable-distributed-asyncio-event-loop)  
16. Managing very long-running Workflows with Temporal, 12월 27, 2025에 액세스, [https://temporal.io/blog/very-long-running-workflows](https://temporal.io/blog/very-long-running-workflows)  
17. Child Workflows | Temporal Platform Documentation, 12월 27, 2025에 액세스, [https://docs.temporal.io/child-workflows](https://docs.temporal.io/child-workflows)  
18. Detecting Activity failures | Temporal Platform Documentation, 12월 27, 2025에 액세스, [https://docs.temporal.io/encyclopedia/detecting-activity-failures](https://docs.temporal.io/encyclopedia/detecting-activity-failures)  
19. Understanding the 4 types of Activity timeouts in Temporal, 12월 27, 2025에 액세스, [https://temporal.io/blog/activity-timeouts](https://temporal.io/blog/activity-timeouts)  
20. Webinar | Automation of Human-in-the-loop Workflows \- Temporal, 12월 27, 2025에 액세스, [https://pages.temporal.io/webinar-automation-of-human-in-the-loop-workflows-with-temporal.html](https://pages.temporal.io/webinar-automation-of-human-in-the-loop-workflows-with-temporal.html)  
21. Multi-Cluster Replication | Temporal Platform Documentation, 12월 27, 2025에 액세스, [https://docs.temporal.io/temporal-service/multi-cluster-replication](https://docs.temporal.io/temporal-service/multi-cluster-replication)  
22. Search Attributes | Temporal Platform Documentation, 12월 27, 2025에 액세스, [https://docs.temporal.io/search-attribute](https://docs.temporal.io/search-attribute)  
23. Self-hosted Visibility feature setup | Temporal Platform Documentation, 12월 27, 2025에 액세스, [https://docs.temporal.io/self-hosted-guide/visibility](https://docs.temporal.io/self-hosted-guide/visibility)  
24. Self-hosted Archival setup | Temporal Platform Documentation, 12월 27, 2025에 액세스, [https://docs.temporal.io/self-hosted-guide/archival](https://docs.temporal.io/self-hosted-guide/archival)  
25. Temporal Workflow message passing \- Signals, Queries, & Updates, 12월 27, 2025에 액세스, [https://docs.temporal.io/encyclopedia/workflow-message-passing](https://docs.temporal.io/encyclopedia/workflow-message-passing)  
26. Handling Signals, Queries, & Updates | Temporal Platform ..., 12월 27, 2025에 액세스, [https://docs.temporal.io/handling-messages](https://docs.temporal.io/handling-messages)  
27. Announcing a new operation: Workflow Update \- Temporal, 12월 27, 2025에 액세스, [https://temporal.io/blog/announcing-a-new-operation-workflow-update](https://temporal.io/blog/announcing-a-new-operation-workflow-update)  
28. Temporal Nexus \- Java SDK Feature Guide, 12월 27, 2025에 액세스, [https://docs.temporal.io/develop/java/nexus](https://docs.temporal.io/develop/java/nexus)  
29. Temporal Nexus \- Python SDK Feature Guide, 12월 27, 2025에 액세스, [https://docs.temporal.io/develop/python/nexus](https://docs.temporal.io/develop/python/nexus)  
30. Temporal Nexus \- .NET SDK Feature Guide, 12월 27, 2025에 액세스, [https://docs.temporal.io/develop/dotnet/nexus](https://docs.temporal.io/develop/dotnet/nexus)  
31. Temporal Use Cases and Design Patterns, 12월 27, 2025에 액세스, [https://docs.temporal.io/evaluate/use-cases-design-patterns](https://docs.temporal.io/evaluate/use-cases-design-patterns)  
32. Mastering Saga patterns for distributed transactions in microservices \- Temporal, 12월 27, 2025에 액세스, [https://temporal.io/blog/mastering-saga-patterns-for-distributed-transactions-in-microservices](https://temporal.io/blog/mastering-saga-patterns-for-distributed-transactions-in-microservices)  
33. The purpose of this code sample is to illustrate some of the interesting properties of the Entity Lifecycle Pattern (aka Entity Workflow) using the Temporal Go SDK. \- GitHub, 12월 27, 2025에 액세스, [https://github.com/temporal-sa/temporal-entity-lifecycle-go](https://github.com/temporal-sa/temporal-entity-lifecycle-go)  
34. About Temporal SDKs | Temporal Platform Documentation, 12월 27, 2025에 액세스, [https://docs.temporal.io/encyclopedia/temporal-sdks](https://docs.temporal.io/encyclopedia/temporal-sdks)  
35. .NET SDK | Temporal, 12월 27, 2025에 액세스, [https://temporal.io/change-log/product-area/net-sdk](https://temporal.io/change-log/product-area/net-sdk)  
36. Go SDK \- Temporal, 12월 27, 2025에 액세스, [https://temporal.io/change-log/product-area/go-sdk](https://temporal.io/change-log/product-area/go-sdk)  
37. When will Nexus be supported in the Python SDK? \- Temporal Community Forum, 12월 27, 2025에 액세스, [https://community.temporal.io/t/when-will-nexus-be-supported-in-the-python-sdk/17543](https://community.temporal.io/t/when-will-nexus-be-supported-in-the-python-sdk/17543)