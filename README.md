# Cloud Security DevSecOps Project: Secure Payment Flow

이 프로젝트는 Google의 `microservices-demo` (Online Boutique) 오픈소스 마이크로서비스 중 결제 흐름에 필요한 핵심 서비스를 식별하고, 로컬 Kubernetes(minikube 등) 환경 위에 안전하고 자동화된 DevSecOps(CI/CD 및 보안 스캔) 파이프라인을 구축하는 프로젝트입니다.

---

## 🏗️ 전체 아키텍처 및 도구 구성

본 프로젝트는 상용 클라우드 아키텍처와 동일한 구조적 설계를 따릅니다.

*   **Version Control (VCS)**: GitHub (코드 변경 이력 및 매니페스트 추적)
*   **Containerization**: Docker (Multi-stage 빌드를 적용한 이미지 최적화)
*   **Static Application Security Testing (SAST/SCA)**: Snyk (오픈소스 취약점 분석 및 빌드 차단)
*   **Private Registry**: Harbor (쿠버네티스 내부 격리 레지스트리 및 이미지 서명)
*   **Continuous Integration (CI)**: Jenkins (컨테이너화된 빌드 에이전트 구동 및 보안 파이프라인 실행)
*   **Continuous Delivery (CD)**: ArgoCD (GitOps 기반의 선언적 자동 동기화 배포)
*   **Container Orchestration**: Kubernetes (로컬 격리 네임스페이스 환경)

---

## 🛍️ 마이크로서비스 아키텍처 맵 (핵심 결제 흐름)

*   `frontend` (Go): 사용자 입출력 및 게이트웨이
*   `productcatalogservice` (Go): 상품 조회 및 검색 gRPC 서비스
*   `cartservice` (C#): 사용자 장바구니 관리
*   `redis` (Redis): 장바구니 세션 캐시 스토어
*   `recommendationservice` (Python): 연관 상품 분석 및 추천 제공 gRPC 서비스
*   `checkoutservice` (Go): 장바구니 ➡️ 결제 오케스트레이터
*   `paymentservice` (Node.js): 가상 카드사 트랜잭션 처리 결제 모듈

---

## 📂 폴더 구조 가이드

*   [services/](file:///c:/Users/lucky/Univ/SECURIOUS/Cloud_Security/services/README.md): 각 마이크로서비스별 독립 소스 코드 및 Dockerfile (사용자 결제 트랜잭션 흐름 소스)
*   [k8s-manifests/](file:///c:/Users/lucky/Univ/SECURIOUS/Cloud_Security/k8s-manifests/README.md): ArgoCD가 감시할 Kubernetes YAML 매니페스트 저장소 (보안 하드닝 정책 및 GitOps 정의)
*   [platform-infra/](file:///c:/Users/lucky/Univ/SECURIOUS/Cloud_Security/platform-infra/README.md): Jenkins, Harbor, ArgoCD 등 k8s 내부에 플랫폼 툴을 설치하기 위한 Helm 차트 및 설정 (인프라 및 Snyk 가이드)
*   [jenkins-pipeline/](file:///c:/Users/lucky/Univ/SECURIOUS/Cloud_Security/jenkins-pipeline/README.md): Jenkins CI 파이프라인 스크립트 (보안 검문 및 자동 빌드 파이프라인)

---

## 🚀 빠른 시작 가이드 (로컬 개발 단계)

### Phase 1: 로컬 개발 환경 코드 배치
1. Google microservices-demo 저장소로부터 핵심 결제 서비스 추출 (`services/` 폴더 하위)
2. 로컬 도커 가상화를 통해 결제 시나리오 gRPC 정합성 검증

*(각 단계별 세부 가이드는 진행 과정에 맞춰 지속적으로 업데이트될 예정입니다.)*
