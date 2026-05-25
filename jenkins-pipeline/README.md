# ⚓ jenkins-pipeline/ - CI/CD DevSecOps 파이프라인 디렉터리

이 디렉터리는 프로젝트의 **보안 중심 지속적 통합(CI/CD) 및 DevSecOps 자동화**를 구현하는 [Jenkinsfile](file:///c:/Users/lucky/Univ/SECURIOUS/Cloud_Security/jenkins-pipeline/Jenkinsfile)이 위치한 곳입니다. 

젠킨스는 소스 코드 변경 시점부터 빌드, 보안 검사, 그리고 사설 이미지 레지스트리 적재에 이르는 전 과정을 코드 정의 파이프라인(Pipeline-as-Code) 형태로 제어합니다.

---

## ⚓ DevSecOps 파이프라인 시나리오 기반 동작 원리

개발자가 코드를 업로드한 뒤, 젠킨스 파이프라인은 아래와 같이 여러 도구들을 조율하여 유기적인 보안 검문을 수행합니다.

### 1. Dynamic Kubernetes Pod Agent 생성
*   젠킨스는 고정 머신을 쓰지 않고, 빌드 시작과 동시에 쿠버네티스 내부에 **임시 에이전트 Pod**를 스케줄링하여 띄웁니다.
*   이 Pod는 보안 도구가 담긴 **`snyk-cli` 컨테이너**와 안전하게 격리된 컴파일을 수행할 수 있는 **`docker:dind` (Docker-in-Docker) 컨테이너**로 동적 결합되어 있습니다.

### 2. Shift-Left 보안 검문 (SAST & SCA)
*   **Snyk Code (SAST)**: 빌드 전 단계에서 `services/` 소스 코드 내부에 비밀번호나 하드코딩된 API 토큰이 노출되었는지, 또는 SQL 주입 공격 등에 노출되었는지 컴파일 전에 정적 분석합니다.
*   **Snyk Open Source (SCA)**: Node.js, Go, Python, C# 의존성 설정 파일들을 직접 스캔해 알려진 취약점(CVE)이 포함된 악성 라이브러리의 빌드 유입을 막아냅니다.

### 3. 병렬 컴파일 & 컨테이너 강화 스캔 (Container Scan)
*   프로젝트의 핵심 결제 시나리오 서비스인 5개 주요 서비스(`paymentservice`, `currencyservice`, `recommendationservice`, `checkoutservice`, `frontend`)에 대해 빌드 시간을 대폭 줄이기 위해 **병렬(parallel) 스테이지**로 동시에 컴파일을 돌립니다.
*   이미지 생성이 끝나면, 최종 컨테이너 패키징 내부의 Linux OS 취약성을 검증하기 위해 **Snyk Container Scan**을 즉시 가동하고 결과를 리포팅합니다.
*   모든 보안이 승인된 이미지는 Kubernetes 사설망 내부의 **Harbor 사설 레지스트리**로 `docker push` 됩니다.

### 4. GitOps 배포 연동 (ArgoCD 트리거)
*   파이프라인이 성공하면 젠킨스는 [k8s-manifests/](file:///c:/Users/lucky/Univ/SECURIOUS/Cloud_Security/k8s-manifests) 디렉터리의 Deployment YAML 내 `image:` 속성에 이번에 새로 생성된 Harbor 이미지 태그 버전을 주입하여 Git에 커밋합니다. 이 커밋이 완료되면 **ArgoCD**가 동기화를 개시하게 됩니다.

---

## 🔗 타 폴더/파일과의 유기적 연결 고리

*   **[services/](file:///c:/Users/lucky/Univ/SECURIOUS/Cloud_Security/services) (소스 코드 디렉터리)**:
    *   젠킨스 파이프라인이 스캔하고 컨테이너 이미지로 빌드하는 대상이 바로 `services/` 폴더입니다.
    *   `services/` 내의 각 `Dockerfile`을 빌드 레퍼런스로 활용해 Snyk 컨테이너 검사를 수행합니다.
*   **[k8s-manifests/](file:///c:/Users/lucky/Univ/SECURIOUS/Cloud_Security/k8s-manifests) (GitOps 배포)**:
    *   파이프라인의 최종 단계에서 젠킨스가 새로 생성한 이미지 태그를 주입하여 수정하는 대상 폴더입니다. 이 수정을 기점으로 실제 쿠버네티스 클러스터 배포가 확정됩니다.
*   **[platform-infra/](file:///c:/Users/lucky/Univ/SECURIOUS/Cloud_Security/platform-infra) (인프라 플랫폼)**:
    *   젠킨스 파이프라인 내부에서 외부 자격 증명을 바인딩(`credentials()`)하여 로그인하는 대상인 **Harbor 사설 저장소**가 이 인프라 정의를 통해 가동됩니다.
