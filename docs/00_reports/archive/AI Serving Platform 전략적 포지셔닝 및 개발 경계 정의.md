# **AI Serving Platform: 전략적 포지셔닝 및 개발 경계 정의**

Date: 2025-05-22  
Target Audience: 개발팀, 경영진, 이해관계자  
Context: 중소기업 소규모 팀(Resource Constraint)을 위한 효율적 개발 전략 수립

## **1\. 비교 분석: 왜 '이 프로젝트'가 필요한가?**

우리가 구축하려는 시스템은 \*\*"Bare Metal Engine"\*\*과 **"Heavy MLOps Platform"** 사이의 \*\*Missing Link(연결 고리)\*\*를 채우는 \*\*'경량화된 미들웨어(Lightweight Middleware)'\*\*이다.

### **1.1 vs. 특화 라이브러리 (vLLM, TGI, Triton) 단독 사용 시**

vLLM이나 Triton은 훌륭한 \*\*'엔진(Engine)'\*\*이지만, \*\*'자동차(Car)'\*\*는 아니다. 엔진만으로는 승객(사용자)을 태울 수 없다.

| 비교 항목 | 특화 라이브러리 (vLLM/Triton) | 우리의 프로젝트 (AI Foundation) | 팀장 코멘트 (Why We Need This) |
| :---- | :---- | :---- | :---- |
| **핵심 역할** | 빠른 추론 연산 (GPU 가속) | 비즈니스 로직, 상태 관리, 안정성 보장 | 엔진은 빠르지만, 요청이 실패하면 그냥 죽어버림. |
| **요청 대기열** | 단순 FIFO (메모리 큐) | **Priority Queue, 영속적 상태 저장** | 서버 재시작 시 대기 중인 요청이 날아가지 않게 하려면 Temporal이 필수. |
| **보안/인증** | 거의 없음 (Basic Auth 수준) | **RBAC, PII 마스킹, Rate Limiting** | vLLM을 그대로 인터넷에 노출하면 보안 사고 직행. |
| **확장성** | 수동 복제 필요 | **오토스케일링 정책 및 라우팅** | 트래픽 몰릴 때 vLLM 컨테이너를 누가 띄우고 누가 연결해 줄 것인가? |

**결론:** 특화 라이브러리는 \*\*'가져다 쓰는 대상'\*\*이지 경쟁 대상이 아니다. 우리는 이 엔진들을 **'감싸는(Wrap)'** 안정적인 껍질을 만드는 것이다.

### **1.2 vs. 대규모 서빙 플랫폼 (KServe, AWS SageMaker)**

KServe나 SageMaker는 훌륭하지만, 소규모 팀이 유지보수하기엔 \*\*'오버엔지니어링(Over-engineering)'\*\*의 위험이 크다.

| 비교 항목 | 대규모 플랫폼 (KServe/SageMaker) | 우리의 프로젝트 (AI Foundation) | 팀장 코멘트 (Why Not This) |
| :---- | :---- | :---- | :---- |
| **구축 난이도** | 최상 (Istio, Knative, DNS 등 학습 곡선 높음) | **중 (FastAPI \+ Temporal)** | 우리 팀 인원으로 쿠버네티스 네트워크(Istio)까지 관리하다간 비즈니스 로직 못 짠다. |
| **비용** | 높음 (Managed 비용 또는 리소스 오버헤드) | **최적화 가능 (Spot Instance 등)** | AWS SageMaker 비용은 상상을 초월함. 직접 구축이 훨씬 싸다. |
| **유연성** | 프레임워크가 정한 규칙을 따라야 함 | **코드 레벨에서 자유롭게 수정 가능** | "RAG 파이프라인 중간에 외부 API 호출하고 분기 처리해" \-\> KServe로는 복잡하지만 Temporal로는 코드 몇 줄임. |
| **디버깅** | 복잡함 (YAML 지옥, 로그 분산) | **직관적 (Python 코드 \+ Temporal UI)** | 에러 났을 때 어디서 멈췄는지 Temporal UI에서 바로 보임. |

**결론:** KServe는 '인프라'로 문제를 풀려 하고, 우리는 \*\*'코드(Temporal)'\*\*로 문제를 풀려 한다. 인원이 적을수록 인프라 복잡도를 낮추고 코드 레벨 제어권을 가져가는 것이 유리하다.

## **2\. 개발의 경계 (The Boundary): 무엇을 하고, 무엇을 안 할 것인가?**

소규모 팀의 리소스는 한정적이다. \*\*"모든 것을 커스텀 제작"\*\*하는 것이 아니라, \*\*"통합(Integration)과 접착(Glue)"\*\*에 집중해야 한다.

### **2.1 우리가 반드시 만들어야 하는 영역 (Core Competency)**

*이 부분은 남이 해결해 주지 않는다. 우리 비즈니스에 맞춰 직접 짜야 한다.*

1. **Workflow Orchestration (Temporal):**  
   * "전처리 \-\> 모델 A \-\> 모델 B \-\> 후처리"로 이어지는 흐름 제어.  
   * 오류 발생 시 재시도 전략, 보상 트랜잭션(결제 취소 등).  
   * *이건 라이브러리가 대신 못 해준다. 우리의 비즈니스 로직이기 때문.*  
2. **Universal Interface (Adapter):**  
   * 프론트엔드 팀이 모델 바뀔 때마다 API를 수정하지 않게 막아주는 방어막.  
   * OpenAI Spec 호환 레이어 구현.  
3. **Gatekeeping (Security & Billing):**  
   * 누가 얼마나 썼는지 카운팅(Billing).  
   * 개인정보가 들어오면 마스킹 처리.

### **2.2 우리가 절대 건드리지 말아야 할 영역 (Out of Scope)**

*이 부분은 오픈소스를 100% 활용하고, 커스텀하지 않는다.*

1. **Model Inference Engine (추론 엔진 내부):**  
   * Pytorch로 직접 model.forward() 짜지 마라.  
   * **무조건 Triton, vLLM, TGI를 쓴다.** 성능 최적화(PagedAttention 등)는 그들의 몫이다.  
2. **Low-level GPU Management:**  
   * GPU 드라이버, CUDA 버전을 직접 코드로 제어하려 하지 마라.  
   * \*\*Docker와 Kubernetes(NVIDIA Device Plugin)\*\*에게 위임한다.  
3. **Complex Network Mesh:**  
   * Istio 같은 서비스 메시를 섣불리 도입하지 마라.  
   * **Nginx** 하나로 충분하다.

## **3\. 실질적 구현 전략: 'Thin Wrapper' 패턴**

우리의 시스템은 거대한 플랫폼이 아니라, 강력한 엔진들에 **표준화된 손잡이**를 달아주는 \*\*'얇은 래퍼(Thin Wrapper)'\*\*여야 한다.

### **3.1 아키텍처 원칙**

* **Engine Agnostic:** 백엔드가 vLLM이든 Triton이든, FastAPI 코드는 InferenceAdapter.predict() 한 줄로 끝나야 한다.  
* **Fail Fast, Retry Smart:** 엔진이 죽으면 즉시 에러를 뱉고(Fail Fast), Temporal이 이를 감지해 다른 워커로 넘기거나 잠시 후 재시도(Retry Smart)한다.  
* **Configuration over Coding:** 모델 추가는 '코드 수정'이 아니라 'YAML 설정 추가'로 끝나야 한다.

### **3.2 의사결정 매트릭스 (팀원 가이드용)**

| 상황 | 행동 지침 |
| :---- | :---- |
| **새로운 모델이 나왔다.** | 그 모델을 지원하는 \*\*Docker 이미지(vLLM 등)\*\*가 있는지 찾는다. 직접 Python 코드를 짜는 건 최후의 수단이다. |
| **추론 속도가 느리다.** | 파이썬 코드를 튜닝하지 말고, **TensorRT 변환**이나 \*\*엔진 설정(Batch Size)\*\*을 튜닝한다. |
| **서버가 자꾸 죽는다.** | 서버 코드를 고치기 전에, **Temporal의 Retry 정책**과 **K8s의 Liveness Probe**를 먼저 확인한다. |

## **4\. 결론**

우리는 \*\*"AI 모델 개발사"\*\*가 아니라 \*\*"AI 서비스 제공사"\*\*이다.

* 우리의 경쟁력은 "얼마나 최신 모델을 빨리 튜닝하느냐"가 아니라, \*\*"어떤 모델이든 얼마나 안정적으로 서비스에 녹여내느냐"\*\*에 있다.  
* 따라서 이 프로젝트는 \*\*'기술적 깊이(Deep Tech)'\*\*보다는 \*\*'운영적 넓이(Operational Width)'\*\*를 커버하는 **Foundation** 구축에 집중한다.