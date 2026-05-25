# 🛡️ Secure Payment Flow DevSecOps Project (Online Boutique)

이 프로젝트는 Google의 `microservices-demo` (Online Boutique) 오픈소스 마이크로서비스 중 **결제 흐름(Secure Payment Flow)에 필요한 핵심 서비스**들을 식별하고, 로컬 및 클러스터 환경에서 안전하고 자동화된 **DevSecOps(CI/CD, SAST, SCA, Container Hardening 및 GitOps) 파이프라인**을 구축하여 학습할 수 있도록 설계된 최첨단 보안 실습 리포지토리입니다.

처음 이 프로젝트를 클론(Clone)받으신 분들이 손쉽게 로컬 환경부터 완전 자동화 배포 파이프라인까지 기동하고 실습할 수 있도록 단계별 친절한 한국어 가이드를 제공합니다.

---

## 🏗️ 1. 전체 아키텍처 및 폴더 구성

이 프로젝트는 최신 상용 클라우드 아키텍처의 설계 패턴을 완벽히 따릅니다.

*   **Version Control (VCS)**: GitHub (코드 변경 이력 및 매니페스트 추적)
*   **Containerization**: Docker (Multi-stage 빌드와 Non-Root 사용자 기동을 적용한 이미지 최적화 및 보호)
*   **Static Application Security Testing (SAST/SCA)**: Snyk (소스 코드 약점 정적 분석 및 취약 라이브러리 검출)
*   **Private Registry**: Harbor (쿠버네티스 내부 사설 이미지 저장소 및 이미지 관리)
*   **Continuous Integration (CI)**: Jenkins (쿠버네티스 dynamic pod agent를 통한 보안 검사 및 자동 빌드)
*   **Continuous Delivery (CD)**: ArgoCD (GitOps 기반 선언적 자동 동기화 배포 엔진)
*   **Container Orchestration**: Kubernetes (로컬 격리 및 리소스 제한이 적용된 네임스페이스 환경)

### 📂 폴더 구조 가이드 (각 폴더명을 클릭해 상세 가이드를 공부해보세요!)

*   [services/](file:///c:/Users/lucky/Univ/SECURIOUS/Cloud_Security/services/README.md): 결제 시나리오의 주역인 9개 핵심 마이크로서비스 소스 코드 및 Dockerfile
*   [k8s-manifests/](file:///c:/Users/lucky/Univ/SECURIOUS/Cloud_Security/k8s-manifests/README.md): ArgoCD가 감시하여 실제 쿠버네티스 클러스터와 1:1 자동 동기화할 Deployment 및 Service YAML 정의서 (Non-Root 보안 컨텍스트 탑재)
*   [jenkins-pipeline/](file:///c:/Users/lucky/Univ/SECURIOUS/Cloud_Security/jenkins-pipeline/README.md): 젠킨스가 dynamic pod agent를 생성하여 Snyk 취약점 분석 및 컴파일을 동적으로 수행하고 Harbor 레지스트리에 푸시하도록 선언된 `Jenkinsfile`
*   [platform-infra/](file:///c:/Users/lucky/Univ/SECURIOUS/Cloud_Security/platform-infra/README.md): Jenkins, Harbor, ArgoCD 등 k8s 내부에 플랫폼 툴을 설치하기 위한 Helm 차트 구성 요령 및 세부 [Snyk 보안 진단 가이드](file:///c:/Users/lucky/Univ/SECURIOUS/Cloud_Security/platform-infra/snyk-security-guide.md)

---

## 🚀 2. [가장 빠른 방법] 로컬 개발 환경 1초 퀵스타트 (Docker Compose)

복잡한 쿠버네티스나 플랫폼 도구 없이, 여러분의 컴퓨터에 **Docker**와 **Docker Compose**만 설치되어 있다면 1초 만에 10개의 핵심 마이크로서비스 전체를 기동하고 직접 웹 서비스 동작을 검증할 수 있습니다.

### 기동 방법 (터미널)
1.  프로젝트의 루트 디렉터리로 이동합니다.
2.  아래 명령어를 실행하여 컨테이너 이미지를 컴파일하고 실행합니다.
    ```bash
    docker-compose up -d --build
    ```
3.  모든 서비스가 성공적으로 빌드되고 시작되면 브라우저를 열고 다음 주소에 접속합니다:
    *   **접속 주소**: `http://localhost:8080` (웹 상점 프론트엔드가 즉시 정상 가동됩니다)
4.  **로컬 종료 방법**:
    ```bash
    docker-compose down
    ```

---

## ☸️ 3. [완전 자동화] Kubernetes 및 DevSecOps 파이프라인 기동 가이드

코드 수정을 감지하여 보안 취약점을 검사하고 인프라에 자동으로 무중단 배포를 단행하는 **DevSecOps GitOps 파이프라인**을 실구동하는 정밀 시나리오 가이드입니다.

### 📋 3.1. 필수 요구사항
실습 진행을 위해 아래의 도구들이 PC에 설치되어 있어야 합니다.
*   **Docker Desktop** (WSL2 또는 Hyper-V 활성화)
*   **Kubectl** (쿠버네티스 제어 CLI)
*   **Helm** (쿠버네티스 패키지 매니저)
*   **Minikube** 또는 로컬 쿠버네티스 가상 환경

---

### 🚀 3.2. 실습 가동 단계 (Step-by-Step)

#### 1단계: 로컬 쿠버네티스 환경 가동
터미널을 열고 Minikube를 구동하여 격리된 쿠버네티스 클러스터를 켭니다.
```bash
minikube start --cpus=4 --memory=8192
```

#### 2단계: Helm을 이용해 플랫폼 인프라 도구 설치
`platform-infra/` 에 맞춰 쿠버네티스 내부에 자동화 엔진 삼총사(Jenkins, Harbor, ArgoCD)를 네임스페이스별로 각각 배포합니다.
```bash
# Helm 레포지토리 추가 및 플랫폼 툴 배포
helm repo add jenkins https://charts.jenkins.io
helm repo add harbor https://helm.goharbor.io
helm repo add argo https://argoproj.github.io/argo-helm
helm repo update

# 네임스페이스 생성 및 설치
kubectl create namespace jenkins
kubectl create namespace harbor
kubectl create namespace argocd

helm install jenkins jenkins/jenkins -n jenkins
helm install harbor harbor/harbor -n harbor
helm install argocd argo/argo-cd -n argocd
```

#### 3단계: 젠킨스(Jenkins) 보안 및 레지스트리 자격 증명 등록
Jenkins 관리 콘솔에 웹 브라우저로 진입한 후, **Credentials (자격 증명)**에 아래의 두 핵심 암호 데이터를 등록합니다.
1.  **Snyk API Token** (`Secret Text` 형식)
    *   ID: `snyk-api-token`
    *   Secret: Snyk 홈페이지에서 회원 가입 후 획득한 API Key 문자열 대입
2.  **Harbor Registry Credentials** (`Username with password` 형식)
    *   ID: `harbor-registry-credentials`
    *   Username / Password: Harbor 관리자 계정 ID/PW 대입

#### 4단계: ArgoCD에 GitOps 동기화 애플리케이션 생성
ArgoCD 콘솔에 로그인한 뒤 **New App** 버튼을 누르고 다음과 같이 설정하여 배포 선언부를 깃허브와 연동합니다.
*   **Application Name**: `online-boutique-secure-flow`
*   **Repository URL**: `https://github.com/Restar1107/Cloud_CICD.git` (현재 저장소)
*   **Revision**: `main`
*   **Path**: `k8s-manifests` (매니페스트 선언부 폴더 경로)
*   **Cluster URL**: `https://kubernetes.default.svc`
*   **Namespace**: `default` (혹은 격리할 커스텀 네임스페이스)
*   **Sync Policy**: `Automatic` (체크 시 Git의 변화를 즉시 추적해 무중단 배포 시작)

---

## 🏃 4. 실습생들이 꼭 체험해야 할 학습용 보안 가상 시나리오

이 리포지토리는 보안 분석 및 파이프라인 학습을 극대화하기 위해 다음과 같은 보안 시나리오 체험을 제공합니다.

### 🔍 시나리오 A: "소스 코드에 몰래 하드코딩된 API Key 노출시키기" (SAST 검증)
1.  `services/frontend/main.go` 파일 내부에 임의로 `const API_KEY = "xoxb-12345678-abcde"`와 같이 가짜 보안 토큰을 하드코딩해 봅니다.
2.  소스를 `git push` 하여 원격 저장소에 반영합니다.
3.  **결과 관찰**: Jenkins가 빌드를 감지하여 실행한 뒤 **`Snyk Code Scan (SAST)`** 단계에서 민감 정보 노출 경고 리포트를 콘솔 창에 선명하게 출력하는 것을 확인하며 Shift-Left 보안의 가치를 몸소 학습합니다.

### 🛡️ 시나리오 B: "컨테이너 내 루트(Root) 권한 탈출 시도 차단" (Kubernetes Hardening)
1.  웹페이지에서 "상품 구매 완료(Checkout)" 버튼을 누르면 내부적으로 `paymentservice` Pod와 `checkoutservice` Pod가 gRPC 통신을 일으킵니다.
2.  이때, `paymentservice` Pod 안으로 침투하여 호스트 OS 커널 탈옥을 유발하려고 해도, [k8s-manifests/paymentservice.yaml](file:///c:/Users/lucky/Univ/SECURIOUS/Cloud_Security/k8s-manifests/paymentservice.yaml)의 `runAsNonRoot: true` 및 `runAsUser: 10001` 정책에 의해 사용자의 권한이 제한되어 침투 공격자가 어떤 고권한 행위도 할 수 없는 것을 직접 매니페스트 설정을 통해 분석하고 이해합니다.
