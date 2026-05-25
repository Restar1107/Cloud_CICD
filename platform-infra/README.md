# 🛠️ platform-infra/ - 플랫폼 인프라 디렉터리

이 디렉터리는 프로젝트의 DevSecOps 플랫폼을 이루는 세 가지 기둥인 **Jenkins(CI)**, **Harbor(사설 Registry)**, **ArgoCD(GitOps)** 등을 Kubernetes 내부에 안정적으로 띄우기 위한 **Helm 차트 설정 및 보안 통합 가이드**를 관리하는 디렉터리입니다.

이 폴더는 보안 진단 기법 문서인 [snyk-security-guide.md](file:///c:/Users/lucky/Univ/SECURIOUS/Cloud_Security/platform-infra/snyk-security-guide.md)와 함께 DevSecOps 파이프라인의 **인프라 뼈대** 역할을 담당합니다.

---

## 🛠️ 플랫폼 인프라 설치 및 구성 시나리오

이 폴더에 기재될 헬름 설정들과 보안 가이드는 실제 쿠버네티스 기동 단계에서 다음과 같은 순서로 유기적으로 작동하여 플랫폼 환경을 완성시킵니다.

### 1단계: 로컬/상용 Kubernetes 및 Helm 기동
*   개발용 Minikube나 AWS EKS 등 클러스터가 동작하면, 이 폴더의 Helm 매니페스트 설정을 기점으로 네임스페이스를 개별 격리하여 플랫폼 도구들을 구축하기 시작합니다.

### 2단계: Harbor & ArgoCD & Jenkins 설치
*   **Harbor (사설 저장소)**: 클러스터 내부의 격리된 영구 볼륨을 할당받아 쿠버네티스 전용 Docker 이미지 저장소로 구동됩니다. 외부 해커로부터 이미지를 철저히 보호하고 서명 검증을 준비합니다.
*   **ArgoCD (GitOps 엔진)**: 쿠버네티스 오퍼레이터 패턴으로 상주하며 Git 변경 사항을 클러스터와 맞추는 싱크 엔진 역할을 개시합니다.
*   **Jenkins (CI 오케스트레이터)**: 쿠버네티스 에이전트 플러그인이 사전 내장되어, 동적으로 빌드 에이전트 Pod를 생성하여 무거운 빌드 수행 후 자원을 바로 클러스터에 돌려주는 탄력적 빌드 서버로 구동됩니다.

### 3단계: 보안 비밀키 연동 (Secrets & Credentials)
*   **Snyk API Token**: Jenkins 자격 증명 관리 시스템에 `snyk-api-token`이라는 이름으로 Snyk 가입 시 획득한 API Key를 보관하여 [Jenkinsfile](file:///c:/Users/lucky/Univ/SECURIOUS/Cloud_Security/jenkins-pipeline/Jenkinsfile)이 안전하게 활용하도록 연동합니다.
*   **Harbor Credential**: Jenkins가 Harbor 저장소에 이미지를 밀어넣을 수 있도록 `harbor-registry-credentials` 기밀 데이터를 Jenkins Credentials에 바인딩합니다.

---

## 🔗 타 폴더/파일과의 유기적 연결 고리

*   **[services/](file:///c:/Users/lucky/Univ/SECURIOUS/Cloud_Security/services) (소스 코드 디렉터리)**:
    *   `services/`에 하드코딩된 비특권 계정 설정(`USER appuser`)과 `platform-infra/` 내 Snyk 가이드의 **컨테이너 하드닝 지침(Container Hardening)**이 일치하도록 안내합니다.
*   **[jenkins-pipeline/](file:///c:/Users/lucky/Univ/SECURIOUS/Cloud_Security/jenkins-pipeline) (CI 파이프라인)**:
    *   `jenkins-pipeline/` 내의 파이프라인 스크립트가 실행되는 **실제 물리적/가상적 백엔드 플랫폼 엔진**이 바로 이 `platform-infra/`를 통해 설치된 Jenkins 및 Kubernetes 환경입니다.
*   **[k8s-manifests/](file:///c:/Users/lucky/Univ/SECURIOUS/Cloud_Security/k8s-manifests) (GitOps 배포)**:
    *   이 폴더에서 설치한 **ArgoCD** 플랫폼 툴이 실제로 지켜보고 조작하는 대상 매니페스트 저장소가 바로 `k8s-manifests/`입니다.
