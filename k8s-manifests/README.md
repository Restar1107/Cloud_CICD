# ☸️ k8s-manifests/ - Kubernetes Deployment 매니페스트 디렉터리

이 디렉터리는 프로젝트의 최종 런타임 환경인 **Kubernetes**에서 마이크로서비스들이 어떻게 구동되고 서로 연결되는지를 정의하는 **선언적 매니페스트(YAML) 저장소**입니다. 

GitOps의 원칙에 따라 이 폴더는 클러스터의 "이상적인 상태(Desired State)"를 나타내며, **ArgoCD**가 이 폴더를 지속적으로 감시하여 실제 인프라 상태와 동기화시킵니다.

---

## 🔒 컨테이너 보안 하드닝 (Container Hardening) 연동 시나리오

이 폴더의 매니페스트들은 소스 코드 폴더(`services/`)에서 작업한 보안 강화 항목들을 실제 쿠버네티스 커널 수준에서 아래와 같이 보증하고 제어합니다.

### 🛡️ Non-Root 비특권 실행 연동 (Security Context)
*   [paymentservice.yaml](file:///c:/Users/lucky/Univ/SECURIOUS/Cloud_Security/k8s-manifests/paymentservice.yaml), [currencyservice.yaml](file:///c:/Users/lucky/Univ/SECURIOUS/Cloud_Security/k8s-manifests/currencyservice.yaml), [recommendationservice.yaml](file:///c:/Users/lucky/Univ/SECURIOUS/Cloud_Security/k8s-manifests/recommendationservice.yaml) 등 Alpine 기반의 서비스들은 컨테이너 침투 시 호스트 커널 탈옥을 방지하기 위해 다음 보안 설정을 완비하고 있습니다.
    ```yaml
    securityContext:
      runAsNonRoot: true   # Root 권한으로 실행되는 것을 전면 차단
      runAsUser: 10001     # Dockerfile에서 지정한 비특권 UID (appuser) 강제
      runAsGroup: 10001    # 비특권 GID 강제
      fsGroup: 10001
    ```
*   [cartservice.yaml](file:///c:/Users/lucky/Univ/SECURIOUS/Cloud_Security/k8s-manifests/cartservice.yaml)은 .NET 컨테이너 표준에 맞게 비특권 UID `1000`을 명시하여 구동하도록 동기화되어 있습니다.

---

## ⚙️ GitOps 배포 및 동기화 작동 시나리오 (ArgoCD & Harbor)

새로운 보안 패치가 적용된 코드가 실제 인프라에 배포되기까지, 이 폴더는 다음과 같은 자동화 파이프라인 시나리오를 통해 타 폴더와 유기적으로 작동합니다.

```
 [서비스 소스 수정] ──> [Jenkins 빌드 & Snyk 검사] ──> [Harbor로 이미지 Push]
                                                              │
   ┌──────────────────────────────────────────────────────────┘
   ▼
1. Jenkins가 이 폴더(k8s-manifests/) 내 Deployment YAML의 image 태그를 새 버전으로 변경
   │
   ▼
2. GitHub/GitLab 원격 저장소에 수정된 YAML이 Commit & Push 됨
   │
   ▼
3. ArgoCD가 k8s-manifests/ 폴더의 변경 내역을 주기적(또는 Webhook)으로 감시 및 감지
   │
   ▼
4. ArgoCD가 변경된 YAML을 Kubernetes 클러스터에 배포 (kubectl apply 자동화)
   │
   ▼
5. Kubernetes가 Harbor 사설 레지스트리로부터 최신 하드닝 이미지를 다운로드받아 안전하게 Pod 기동
```

---

## 🔗 타 폴더/파일과의 유기적 연결 고리

*   **[services/](file:///c:/Users/lucky/Univ/SECURIOUS/Cloud_Security/services) (소스 코드 디렉터리)**:
    *   소스 코드의 gRPC 포트 설정과 `k8s-manifests/` 내 Service YAML의 `targetPort` 설정이 완벽히 맞물립니다.
    *   Dockerfile 내의 `USER appuser` 정책이 YAML의 `securityContext`와 1:1로 대응되어 보안 정책의 불일치를 막아줍니다.
*   **[jenkins-pipeline/](file:///c:/Users/lucky/Univ/SECURIOUS/Cloud_Security/jenkins-pipeline) (CI 파이프라인)**:
    *   젠킨스는 빌드를 완수한 후 Git에 위치한 이 폴더의 Deployment YAML 파일들을 직접 수정하여 새 이미지 버전을 반영하는 **GitOps 배포 트리거** 역할을 합니다.
*   **[platform-infra/](file:///c:/Users/lucky/Univ/SECURIOUS/Cloud_Security/platform-infra) (인프라 플랫폼)**:
    *   이 매니페스트들이 성공적으로 기동되려면 `platform-infra/`를 통해 설치된 **Harbor**(컨테이너 저장소)가 이미지를 공급해주어야 하며, **ArgoCD**가 가동되어 이 폴더를 활발하게 배포 동기화 컨트롤러로 다스려야 합니다.
