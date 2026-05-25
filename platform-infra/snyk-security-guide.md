# DevSecOps: Snyk 보안 취약점 진단 및 컨테이너 하드닝 가이드

이 가이드는 프로젝트의 보안 취약성을 정기적으로 스캔하고 빌드 전 단계에서 탐지하기 위해 **Snyk(SAST, SCA, Container Scan)**을 도입하고, 로컬 및 Jenkins CI 파이프라인에 통합하는 구체적인 아키텍처 가이드라인입니다.

---

## 🔒 1. Snyk 보안 통합 개요

DevSecOps 파이프라인에서 Snyk는 세 가지 주요 영역에서 보안 위험을 차단합니다.

1.  **Snyk Code (SAST)**:
    *   마이크로서비스 소스 코드(Go, Node.js, Python, C#)의 비보안 코딩 패턴, 하드코딩된 API 키/비밀번호, 주입 공격(Injection) 등의 취약점을 정적 분석으로 탐지합니다.
2.  **Snyk Open Source (SCA)**:
    *   `package.json`, `go.mod`, `requirements.txt` 등 라이브러리 선언 파일을 검사하여 악성코드나 알려진 취약점(CVE)이 포함된 라이브러리 사용을 차단합니다.
3.  **Snyk Container**:
    *   `Dockerfile` 빌드를 통해 최종 생성된 Docker 이미지 내의 OS 레이어 취약점을 탐지하고, `Base Image`를 더 안전한 버전으로 변경하는 권장안을 제시합니다.

---

## 🛠️ 2. 개발자 로컬 환경 Snyk 검증 가이드 (Local Scan)

Jenkins 파이프라인에 코드를 올리기 전, 개발자는 로컬 PC에서 즉시 보안 진단을 수행하여 조기에 취약점을 조치(Shift-Left Security)할 수 있습니다.

### 2.1. Snyk CLI 설치 및 인증
1.  **Snyk CLI 설치** (Node.js 환경이 설치되어 있는 경우):
    ```bash
    npm install -g snyk
    ```
2.  **Snyk 가입 및 인증**:
    ```bash
    snyk auth
    ```
    *   브라우저가 열리면 Snyk 계정(GitHub 또는 Google 연동)으로 로그인하여 로컬 CLI 토큰 인증을 완료합니다.

### 2.2. 로컬 마이크로서비스 스캔 명령어
각 서비스 폴더로 이동하여 진단을 수행합니다.

*   **소스 코드 취약점 분석 (SAST)**:
    ```bash
    # frontend Go 코드 검사
    cd services/frontend
    snyk code test
    ```
*   **오픈소스 종속성 취약점 분석 (SCA)**:
    ```bash
    # paymentservice npm 패키지 검사
    cd services/paymentservice
    snyk test
    ```
*   **Docker 컨테이너 이미지 분석**:
    ```bash
    # 로컬에서 이미지를 빌드한 후 OS 취약점 분석
    docker build -t local/paymentservice:latest .
    snyk container test local/paymentservice:latest --file=Dockerfile
    ```

---

## 🏗️ 3. 컨테이너 보안 강화 (Non-Root Hardening)

Snyk 컨테이너 스캔을 통과하고 컨테이너 런타임의 탈옥 위험을 원천 차단하기 위해, 각 마이크로서비스의 Dockerfile에 **Non-Root 권한 기동** 패치를 적용합니다. 

기본적으로 `frontend` 및 `checkoutservice`가 사용하는 `gcr.io/distroless/static` 이미지는 이미 non-root(nobody)로 안전하게 설정되어 있습니다. 
하지만 Alpine 기반의 Node.js, Python 등의 서비스는 기본적으로 `root` 권한으로 실행되므로 아래와 같이 Dockerfile 하드닝을 진행해야 합니다.

### 🐍 Python (recommendationservice) Hardening 예시
```dockerfile
# ... (Base/Builder 빌드 생략) ...
FROM base

ENV PYTHONDONTWRITEBYTECODE=1
ENV PYTHONUNBUFFERED=1

RUN apk update \
    && apk add --no-cache libstdc++ \
    && rm -rf /var/cache/apk/*

WORKDIR /recommendationservice
COPY --from=builder /usr/local/lib/python3.14/ /usr/local/lib/python3.14/

# [보안패치]: UID/GID 10001의 권한이 축소된 비특권 사용자 생성 및 소유권 변경
RUN addgroup -g 10001 -S appgroup && adduser -u 10001 -S appuser -G appgroup
COPY . .
RUN chown -R appuser:appgroup /recommendationservice

ENV PORT="8080"
EXPOSE 8080

# [보안패치]: Non-Root 사용자로 실행 컨텍스트 전환
USER appuser
ENTRYPOINT ["python", "recommendation_server.py"]
```

### 🟢 Node.js (paymentservice) Hardening 예시
```dockerfile
# ... (Builder 생략) ...
FROM alpine:3.23.4@sha256:5b10f432ef3da1b8d4c7eb6c487f2f5a8f096bc91145e68878dd4a5019afde11

RUN apk add --no-cache nodejs

# [보안패치]: 비특권 사용자 및 그룹 생성
RUN addgroup -g 10001 -S appgroup && adduser -u 10001 -S appuser -G appgroup

WORKDIR /usr/src/app
COPY --from=builder /usr/src/app/node_modules ./node_modules
COPY . .

# [보안패치]: 디렉터리 소유권 할당
RUN chown -R appuser:appgroup /usr/src/app

EXPOSE 50051

# [보안패치]: Non-Root 사용자로 기동
USER appuser
ENTRYPOINT [ "node", "index.js" ]
```

---

## ⚓ 4. Jenkins CI Pipeline Snyk 자동화 연동

실제 상용 환경과 동일한 구조의 Kubernetes 내부 Jenkins 파이프라인에서 Snyk를 자동화하여 구동하기 위해 아래와 같이 `Jenkinsfile`에 스캔 단계를 구성합니다.

### 4.1. Jenkins 준비사항 (Secret 등록)
*   Jenkins 관리 ➡️ Credentials에 **Snyk API Token**을 `Secret Text` 형식의 Credential ID `snyk-api-token`으로 등록합니다.

### 4.2. Jenkinsfile 내 Snyk 파이프라인 스니펫
```groovy
pipeline {
    agent {
        kubernetes {
            yaml """
apiVersion: v1
kind: Pod
metadata:
  labels:
    some-label: jenkins-agent
spec:
  containers:
  - name: snyk-cli
    image: snyk/snyk:alpine
    command: ['cat']
    tty: true
  - name: docker
    image: docker:dind
    securityContext:
      privileged: true
"""
        }
    }
    
    environment {
        SNYK_TOKEN = credentials('snyk-api-token')
        REGISTRY = "harbor.harbor.svc.cluster.local/online-boutique"
    }

    stages {
        stage('Checkout') {
            steps {
                git branch: 'main', url: 'https://github.com/your-org/Cloud_Security.git'
            }
        }

        stage('Snyk Security Scan (SAST & SCA)') {
            steps {
                container('snyk-cli') {
                    sh 'echo "=== 1. SAST 코드 취약점 스캔 ==="'
                    // --severity-threshold=high 옵션을 주면 심각도 High 이상일 때 빌드를 실패(Exit code > 0) 처리합니다.
                    sh 'snyk code test --severity-threshold=high || true'
                    
                    sh 'echo "=== 2. SCA 오픈소스 라이브러리 스캔 ==="'
                    sh 'snyk test --severity-threshold=high || true'
                }
            }
        }

        stage('Docker Build & Container Scan') {
            steps {
                container('docker') {
                    // 도커 이미지 빌드
                    sh 'docker build -t ${REGISTRY}/paymentservice:${BUILD_NUMBER} ./services/paymentservice'
                }
                
                container('snyk-cli') {
                    sh 'echo "=== 3. 빌드 완료 이미지 OS 취약점 스캔 ==="'
                    sh 'snyk container test ${REGISTRY}/paymentservice:${BUILD_NUMBER} --file=./services/paymentservice/Dockerfile --severity-threshold=high || true'
                }
            }
        }
    }
}
```

*   **`|| true` 전략**: 초기 연동 단계에서는 빌드가 바로 중단되는 것을 막기 위해 `|| true`를 붙여 취약점이 발견되어도 리포트만 출력하고 다음 단계로 진행하게 한 뒤, 취약점 조치가 누적되면 `|| true`를 제거해 상용 보안 게이트(Security Gate)를 견고하게 통제합니다.
