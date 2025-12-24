# **AI Model Serving Platform \- 시스템 요건 정의서 (SRS)**

**Project Name:** AI Serving Foundation

**Version:** 1.3.2 (Clarified Purpose & Foundation)

**Date:** 2025-05-22

**Status:** Draft

## **1\. 개요 (Overview)**

### **1.1 목적 (Project Purpose)**

본 프로젝트의 궁극적인 목표는 **"변화하는 모델(Variable)"과 "변하지 않는 인프라(Invariant)"를 철저히 분리**하여, 전사적 차원의 \*\*AI 서비스 표준 기반(Foundation)\*\*을 구축하는 것이다.

AI 모델은 끊임없이 발전하고 교체되지만, 이를 서빙하기 위해 필수적인 **장시간 비동기 처리, 보안(Security), 프록시(Proxy), 통신 프로토콜, 모니터링** 등의 요소는 변하지 않는 공통 영역이다. 본 플랫폼은 이러한 **공통 난제들을 미리 해결해 둔 견고한 토대**를 제공함으로써, 향후 어떤 AI 모델이 도입되더라도 중복 개발 없이 즉시 최상의 퍼포먼스와 안정성을 상속받아 서비스로 제공될 수 있도록 한다.

### **1.2 핵심 철학 (Core Philosophy)**

1. **Immutable Foundation (변하지 않는 기반):**  
   * 개별 모델을 개발할 때마다 API 서버, 인증, 큐잉 시스템을 새로 구축하는 비효율을 제거한다.  
   * 개발자는 오직 "모델의 추론 로직"에만 집중하고, 나머지 **가용성, 확장성, 보안성**은 플랫폼이 전담한다.  
2. **Domain-Specific Optimization (도메인 특화 최적화):**  
   * 단순히 엔진만 다르게 쓰는 것을 넘어, \*\*도메인별 병목 구간(Bottleneck)\*\*을 정확히 타격하여 최적화한다.  
     * *Vision:* CPU 이미지 디코딩 병목 제거.  
     * *RecSys:* 피처 조회(Feature Lookup) 지연 시간 최소화.  
     * *LLM:* 생성 속도(TPS) 및 메모리 대역폭 최적화.  
3. **Pipeline Orchestration:**  
   * AI 서비스는 단일 모델 호출로 끝나지 않는다. 여러 모델이 연결된 **DAG(Directed Acyclic Graph)** 형태의 복합 파이프라인(Ensemble)을 지원해야 한다.  
4. **Performance First:**  
   * 물리적 추론 속도뿐만 아니라, 캐싱, 커널 최적화, 트래픽 제어를 통해 사용자 체감 지연 시간(End-to-End Latency)을 최소화한다.

## **2\. 시스템 아키텍처 (System Architecture)**

### **2.1 기술 스택 (Tech Stack)**

| Layer | Component | Technology | Role |
| :---- | :---- | :---- | :---- |
| **Gateway** | Ingress | **Nginx** | **(Immutable)** SSL Termination, Request Routing, Rate Limiting, CORS |
| **App** | API Server | **FastAPI** | **(Immutable)** Async Request Handling, Validation (Pydantic), Auth |
| **Orchestration** | Workflow Engine | **Temporal** | **(Immutable)** State Management, Priority Queueing, Retry Policy (장시간 작업 보장) |
| **Optimization** | Feature Store | **Feast / Redis** | **(New)** RecSys/Tabular 모델을 위한 실시간 피처 서빙 |
|  | Pre-processing | **NVIDIA DALI** | **(New)** Vision 모델을 위한 GPU 가속 전처리 (CPU 병목 제거) |
| **Inference** | **Adaptive Execution** | **Plug & Play Backends** | **Concept:** "Right Engine for Right Model" 모든 모델을 하나의 엔진에 가두지 않고, 각 모델의 연산 특성(CNN, Transformer, Tree)에 물리적으로 가장 최적화된 **전용 런타임(Runtime)을 동적으로 부착**하여 구동한다. *(예: Triton, vLLM, SGLang, Trellis, Custom Containers)* |
| **Data** | Metadata DB | **PostgreSQL** | User Data, Job History, Model Meta Info |
| **Monitor** | Observability | **Prometheus / Grafana** | Metrics Collection, Dashboarding (w/ dcgm-exporter) |

### **2.2 논리적 데이터 흐름**

Client → Nginx → FastAPI → (If RecSys: Feature Store Lookup) → Temporal Service → Worker → Inference Adapter → Specific Engine (Triton Ensemble / vLLM)

## **3\. 기능 요건 (Functional Requirements)**

### **3.1 API Gateway & Application Layer**

* **REQ-API-01 (Async Protocol):** 모든 추론 API는 비동기 처리를 기본으로 하며, 긴 작업(Long-running)에 대해 Job ID를 즉시 반환해야 한다.  
* **REQ-API-02 (Streaming):** LLM 생성 응답은 SSE(Server-Sent Events) 또는 WebSocket을 통해 토큰 단위 스트리밍을 지원해야 한다.  
* **REQ-API-03 (Validation):** 모든 입력 요청은 Pydantic 모델을 통해 타입 및 값의 유효성이 검증되어야 한다.

### **3.2 Orchestration Layer (Temporal)**

* **REQ-ORC-01 (Workflow Definition):** 모든 AI 작업은 Workflow 단위로 정의되어야 한다.  
* **REQ-ORC-02 (Retry Policy):**  
  * **System Error:** Exponential Backoff 정책으로 최대 3\~5회 재시도.  
  * **OOM (Out Of Memory):** 별도 에러로 분류하여 배치 사이즈를 줄여 재시도(Fallback)하거나 더 큰 GPU 큐로 라우팅.  
* **REQ-ORC-03 (Priority Queueing):** 사용자 등급(Tier) 또는 작업 중요도에 따라 Temporal Task Queue의 우선순위를 동적으로 조정해야 한다.

### **3.3 Inference Layer (Polyglot Strategy)**

* **REQ-INF-01 (LLM Standard):** 텍스트 생성 모델은 **OpenAI API Compatible Interface**를 준수해야 한다.  
* **REQ-INF-02 (General Standard):** 비-LLM 모델은 **Triton Inference Server** 및 **KServe V2 Protocol**을 따른다.  
* **REQ-INF-03 (Ensemble Support):** **(New)** 단일 추론이 아닌 전처리 \-\> 모델1 \-\> 모델2 \-\> 후처리 과정을 Triton Ensemble Backend 또는 Business Logic Workflow 내에서 처리할 수 있어야 한다.  
* **REQ-INF-04 (Backend Agnostic Extensibility):** **(Critical)** 시스템은 특정 엔진에 종속되지 않아야 하며, 향후 등장할 새로운 모델(3D Gen, Video Gen 등)을 위해 **새로운 컨테이너 기반 백엔드를 규격화된 인터페이스(Adapter)만 맞추면 즉시 추가(Plug-in)** 할 수 있는 구조여야 한다.

## **4\. 인터페이스 및 프로토콜 표준 (Interface Standards)**

### **4.1 Internal Protocol (Temporal Worker ↔ Engine)**

| Model Type | Recommended Engine | Protocol Standard | Example Payload |
| :---- | :---- | :---- | :---- |
| **LLM** | vLLM, SGLang | **OpenAI Chat Completions** | {"messages": \[...\], "temperature": 0.7} |
| **Vision** | Triton (TensorRT) | **KServe V2 (gRPC)** | {"inputs": \[{"name": "image\_tensor", "shape": \[1, 3, 640, 640\], "datatype": "FP16", "data": \[...\]}\]} |
| **Tabular** | Triton (RAPIDS FIL) | **KServe V2 (gRPC)** | {"inputs": \[{"name": "features", "shape": \[1, 50\], "datatype": "FP32", "data": \[...\]}\]} |

## **5\. 비기능 요건 (Non-Functional Requirements)**

### **5.1 성능 및 확장성 (Performance & Scalability)**

* **REQ-PERF-01 (Latency Budget):**  
  * **Vision/RecSys:** P99 Latency **50ms 이내** (실시간성 중요).  
  * **LLM:** TTFT 500ms 이내.  
* **REQ-SCALE-01 (Auto-scaling):**  
  * **API/Worker:** CPU/Memory 기준 HPA.  
  * **Inference:** GPU Duty Cycle 또는 **Request Queue Depth(대기열 길이)** 기반 스케일링.

### **5.2 운영 및 모니터링 (Operations)**

* **REQ-OPS-01 (GPU Monitoring):** dcgm-exporter 필수 (GPU 활용률, VRAM 사용량, **PCIe 대역폭**).  
* **REQ-OPS-02 (Logging):** 입력 파라미터, 출력 토큰 수, 처리 시간, 모델 버전 로깅.

## **8\. 도메인별 성능 최적화 전략 (Domain-Specific Optimization) \- *Refined***

각 모델 타입이 겪는 고유한 병목 현상을 해결하기 위한 기술 요건.

* **REQ-OPT-01 (Vision: GPU Pre-processing):** **(New)**  
  * 고해상도 이미지 디코딩 및 리사이징(Resize) 과정에서 발생하는 CPU 병목을 막기 위해 **NVIDIA DALI**를 사용하여 전처리를 GPU에서 수행해야 한다.  
  * 가능한 모든 Vision 모델은 **TensorRT**로 컴파일하여 FP16 추론 속도를 극대화한다.  
* **REQ-OPT-02 (RecSys/Tabular: Low Latency Feature):** **(New)**  
  * 추천/분류 모델(XGBoost, LightGBM)의 추론 속도 확보를 위해 **RAPIDS FIL (Forest Inference Library)** 백엔드를 사용한다.  
  * 실시간 추론 시 필요한 피처 데이터는 DB가 아닌 **Redis/Feast** 기반의 Feature Store에서 조회하여 I/O Latency를 5ms 미만으로 유지한다.  
* **REQ-OPT-03 (Audio/General: Dynamic Batching):** **(New)**  
  * 길이가 제각각인 입력(오디오, 문장)이 들어오더라도 효율적으로 처리할 수 있도록 Triton의 **Dynamic Batching** 기능을 필수적으로 활성화한다.  
  * 설정된 지연 시간(예: 50ms) 내에 들어온 요청을 자동으로 묶어 GPU 활용률을 높인다.  
* **REQ-OPT-04 (LLM: Generative Optimization):**  
  * LLM 서빙 시에는 **Speculative Decoding** 및 \*\*Semantic Caching(유사도 캐싱)\*\*을 적용하여 생성 속도를 높인다.

## **9\. 고급 배포 및 안정성 (Advanced Deployment & Reliability)**

기능 제공의 연속성을 보장하기 위한 고급 운영 전략.

* **REQ-DEP-01 (Shadow Deployment):**  
  * 새 모델 배포 시, 사용자 트래픽을 복제하여 새 모델에 입력하고, **결과 정합성과 성능(Latency)을 검증**한 뒤에 실 서비스에 투입한다.  
* **REQ-DEP-02 (Circuit Breaker & Fallback):**  
  * 특정 모델의 에러율이 임계치를 넘으면(예: 1분간 5회 실패), 즉시 트래픽을 차단하고 \*\*경량화 모델(Fallback Model)\*\*이나 **룰 기반 응답**으로 대체하여 서비스 중단을 막는다.

## **10\. 비용 효율성 및 보안 (Cost & Security)**

* **REQ-COST-01 (Spot Instance Recovery):**  
  * 저렴한 Spot Instance 사용 시, 중단 시그널(SIGTERM)을 감지하여 실행 중인 Temporal 작업을 안전하게 **Checkpoint** 저장 후 다른 노드로 이관해야 한다.  
* **REQ-SEC-01 (PII Redaction):**  
  * 성능 저하 없이 비동기적으로 개인정보(PII) 마스킹을 수행하여 로그 및 학습 데이터 오염을 방지한다.