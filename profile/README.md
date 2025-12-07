# NanoGrid (ナノグリッド)

## 😁 팀원 소개

| **고동현** | **소부승** | **이여재** | **나영민** | **정윤주** | **이상무** |
| :-----------------------------------------------------------------------------------------------------------------------------------------: | :-----------------------------------------------------------------------------------------------------------------------------------: | :-------------------------------------------------------------------------------------------------------------------------------------: | :-------------------------------------------------------------------------------------------------------------------------------------: | :-------------------------------------------------------------------------------------------------------------------------------------: | :-------------------------------------------------------------------------------------------------------------------------------------: |
| [<img src="https://github.com/Gosorasora.png" height=150 width=150> <br/> @Gosorasora](https://github.com/Gosorasora) <br/> **Infra, Security** | [<img src="https://github.com/bootkorea.png" height=150 width=150> <br/> @bootkorea](https://github.com/bootkorea) <br/> **Web, Fullstack** | [<img src="https://github.com/iyeojae.png" height=150 width=150> <br/> @iyeojae](https://github.com/iyeojae) <br/> **Infra, Backend** | [<img src="https://github.com/skdudals99.png" height=150 width=150> <br/> @skdudals99](https://github.com/skdudals99) <br/> **Monitoring, Infra** | [<img src="https://github.com/yun1270.png" height=150 width=150> <br/> @yun1270](https://github.com/yun1270) <br/> **Monitoring, QA** | [<img src="https://github.com/sangmu1126.png" height=150 width=150> <br/> @sangmu1126](https://github.com/sangmu1126) <br/> **Infra, Backend** |


# Lambda에 의존하지 않고, EC2 위에 직접 구축한 셀프 호스트형 FaaS 플랫폼

## 1. 프로덕트 개요 (Product Overview)

**「VMware에서 탈피」와 「클라우드 네이티브화」를 실현한 차세대 서버리스 엔진**

- **무제한 실행 시간**: 타임아웃 제한(Lambda의 29초)을 제거하고, 긴 시간 실행이 필요한 작업이나 비동기 처리에 대응.
- **멀티 클라우드 영속성**: AWS S3뿐만 아니라 GCP Cloud Storage에 이중 저장하여 데이터를 영구적으로 보호.
- **다양한 언어 지원**: Node.js, Python, C++, Go 등 Lambda보다 유연한 런타임 환경을 제공.
- **Private AI 추론**: 외부 API에 의존하지 않고 VPC 내부에서 완료되는 안전한 AI 추론 환경.

## 2. 개발 배경과 진화의 과정 (Background & Evolution)

**문제: 기존 FaaS의 제약 (타임아웃, 벤더 락인)과 VMware 의존에서 벗어나는 문제**

| **버전** | **해결책 (NanoGrid Solution)** | **기술적 특징** |
| --- | --- | --- |
| **V1** | **VMware 탈피 및 독자 엔진 설계** | 경량 컨테이너 기반의 독자적인 클라우드 엔진 구축, 확장성과 비동기 처리를 보장. |
| **V2** | **29초 타임아웃(Lambda 제한) 돌파** | Lambda를 폐기하고 **EC2 Controller** 도입. API Gateway → ALB로 변경하여 긴 실행 시간을 실현. |
| **V3** | **고가용성(High Availability)** | Controller와 Worker를 **Multi-AZ**로 배치하여 단일 장애점(SPOF)을 제거. |
| **V4** | **멀티 클라우드 영속성** | 실행 성공 코드를 **GCP Cloud Storage**에 영구 저장. 벤더 락인 위험을 감소시킴. |
| **V5** | **엔터프라이즈 보안** | **AWS WAF** 도입 (SQLi, XSS, DoS 방어). 모든 데이터를 VPC 내부에서만 처리하는 폐쇄망 설계. |
| **V6** | **Private AI 내재화** | 내부 **AI EC2** 노드를 구축하여 데이터 유출을 방지하는 안전한 추론 환경을 구현. |

## 3. **아키텍처 및 주요 기능 (Architecture & Key Features)**

<img width="1077" height="1100" alt="image" src="https://github.com/user-attachments/assets/3efe2a73-de0e-42fc-91f5-3593243c4a3c" />

### 3-1. Lambda-less 독자 제어 평면 (Control Plane)

- **구성**: EC2 Controller 2대, **Multi-AZ** 구성 (ap-northeast-2a, 2c).
- **보안**: **ALB + WAF**를 이용한 트래픽 분산 및 공격 방어 (SQLi, XSS, Rate Limit).
- **기능**: 라우터 역할을 하며, 타임아웃 없는 긴 실행 작업을 관리합니다.

### 3-2. VPC 내부 Private AI 노드

- **완전 격리**: Private Subnet에 배치된 AI 추론 서버 (IP: 10.0.20.100).
- **데이터 주권**: 외부 인터넷과의 접촉 없이 데이터를 유출을 차단.
- **접근 제어**: Security Group을 통해 Worker만 접근 가능.

### 3-3. SQS 기반 Auto Scaling (핵심 로직)

- **스케일 아웃**: SQS 메시지 **10개 이상** → 최대 5대까지 자동 증가.
- **스케일 인**: 메시지 **0개** (3분 유지) → 최소 2대까지 축소.
- **안정성**: Multi-AZ 분산 배치와 300초 쿨다운 타임으로 급격한 변동을 방지.

### 3-4. 멀티 클라우드 데이터 영속성 (Persistence)

- **AWS S3**: 소스 코드 원본을 저장.
- **GCP Cloud Storage**: 실행 성공한 코드와 결과를 영구 저장.
- **보안 전송**: Worker가 NAT Gateway를 통해 **AWS Secrets Manager**에서 관리된 인증 키로 GCP로 직접 전송.

### 3-5. Redis Pub/Sub 실시간 상태 관리

- **비동기 통신**: Controller ↔ Worker 간 빠른 상태 동기화.
- **상태 추적**: PENDING → PROCESSING → SUCCESS/FAILED 상태를 실시간으로 시각화.
- **자동 정리**: TTL 24시간 설정으로 메모리 관리.

---

## 4. **기술 스택 (Tech Stack)**

| **분류** | **기술 (Technology)** |
| --- | --- |
| **언어** | Node.js/Express (Controller), Python (Worker) |
| **AWS 인프라** | EC2, VPC, ALB, SQS, S3, ElastiCache Redis, Secrets Manager |
| **멀티 클라우드** | **GCP Cloud Storage** (Backup & Persistence) |
| **IaC** | Terraform |
| **컨테이너** | Docker (Isolation & Runtime) |
| **보안** | **AWS WAF** (SQLi, XSS, Log4j, Rate Limit), Private Subnet, NAT GW |
| **모니터링** | CloudWatch Alarms (Auto Scaling Trigger) |

---

## 5. **중요 지표 (Key Metrics)**

| **항목 (Item)** | **수치/사양 (Value/Spec)** |
| --- | --- |
| **Controller** | **2대** (Multi-AZ 고가용성 구성) |
| **Worker** | **2~5대** (SQS 기반 Auto Scaling) |
| **Scale Out 임계값** | SQS 메시지 **10개** 이상 |
| **Scale In 조건** | SQS 메시지 **0개** (3분 간 유지) |
| **Cooldown** | **300초** (5분) |
| **AI Node** | Private Subnet **완전 격리** (인터넷 접속 차단) |
| **데이터 저장** | **AWS S3 + GCP Cloud Storage** (멀티 클라우드) |

