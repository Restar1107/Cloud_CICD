# 🛍️ services/ - 마이크로서비스 소스 코드 디렉터리

이 디렉터리는 **Online Boutique(microservices-demo)** 데모의 핵심 결제 흐름(Secure Payment Flow)을 구성하는 개별 마이크로서비스들의 독립된 소스 코드와 컨테이너 빌드를 위한 `Dockerfile`들을 모아둔 핵심 소스 공간입니다.

이 가이드는 마이크로서비스들이 실제 사용자 시나리오 기반으로 어떻게 유기적으로 동작하고 다른 폴더들과 연결되는지 설명합니다.

---

## 🗺️ 핵심 시나리오 기반 유기적 서비스 통신 흐름

사용자가 웹 브라우저에서 상점을 방문하여 상품을 담고 결제를 완료하기까지, 이 폴더의 소스 코드들은 아래 시나리오에 따라 실시간으로 연결됩니다.

```
[사용자 브라우저] 
       │ (HTTP 8080)
       ▼
   frontend ──(상품조회)──> productcatalogservice
       │
       ├──(장바구니 담기)──> cartservice ──(세션 캐시 저장)──> redis
       │
       └──(주문 완료 요청)─> checkoutservice (주문 오케스트레이터)
                                  │
                                  ├──(1. 장바구니 품목 조회)──> cartservice
                                  ├──(2. 상품 원가 검증)──> productcatalogservice
                                  ├──(3. 다국적 통화 변환)──> currencyservice
                                  ├──(4. 배송비 산출)─────> shippingservice
                                  ├──(5. 카드 결제 승인)───> paymentservice [보안 핵심]
                                  ├──(6. 장바구니 비우기)──> cartservice
                                  └──(7. 구매 영수증 메일)──> emailservice
```

### 🏃 시나리오 단계별 동작과 소스 코드의 역할

1.  **웹 브라우저 접속 및 상품 탐색 (Browse)**
    *   사용자가 `frontend` 서비스에 접속하면, [frontend](file:///c:/Users/lucky/Univ/SECURIOUS/Cloud_Security/services/frontend) (Go) 소스는 [productcatalogservice](file:///c:/Users/lucky/Univ/SECURIOUS/Cloud_Security/services/productcatalogservice) (Go)에 gRPC 요청을 보내 상품 정보(`products.json`)를 읽어와 화면에 뿌려줍니다.
    *   글로벌 다국적 가격 렌더링을 위해 [currencyservice](file:///c:/Users/lucky/Univ/SECURIOUS/Cloud_Security/services/currencyservice) (Node.js)에 통화 변환을 요청합니다.

2.  **장바구니에 담기 (Add to Cart)**
    *   사용자가 상품 상세 페이지에서 "장바구니 담기"를 누르면, `frontend`는 [cartservice](file:///c:/Users/lucky/Univ/SECURIOUS/Cloud_Security/services/cartservice) (C#)에 담기 요청을 보냅니다.
    *   `cartservice`는 세션 상태를 유지하기 위해 로컬 캐시인 **`redis`** 컨테이너에 사용자 세션 ID를 키값으로 장바구니 정보를 기록합니다.

3.  **주문서 작성 및 최종 결제 승인 (Checkout - 핵심 보안 영역)**
    *   사용자가 "주문 완료(Place Order)" 버튼을 누르면, 주문 오케스트레이터인 [checkoutservice](file:///c:/Users/lucky/Univ/SECURIOUS/Cloud_Security/services/checkoutservice) (Go)가 기동되어 다음 순서로 트랜잭션을 엄격히 조율합니다.
        *   **장바구니 데이터 조회**: `cartservice`를 호출해 장바구니 내 품목들을 가져옵니다.
        *   **상품 유효성 검증**: `productcatalogservice`를 통해 실제 재고 목록과 일치하는지 대조합니다.
        *   **환율 및 배송료 환산**: `currencyservice` 및 `shippingservice` (Go)를 호출해 배송 비용을 책정하고 금액을 최종 원화/달러로 환산합니다.
        *   **[결제 트랜잭션]**: 최종 산출된 총액과 카드 정보를 **`paymentservice` (Node.js)**에 gRPC로 전달하여 모의 결제 승인 처리를 수행합니다.
        *   **장바구니 비우기 & 메일 발송**: 결제가 성공하면 `cartservice`를 통해 Redis 내 장바구니를 비우고, `emailservice` (Python)를 호출해 구매 영수증 이메일을 발송합니다.

---

## 🔗 타 폴더/파일과의 유기적 연결 고리

이 `services/` 폴더 내의 파일들은 프로젝트 내 다른 구성 요소들과 다음과 같이 강력하게 결합되어 유기적으로 맞물려 돌아갑니다.

*   **[docker-compose.yaml](file:///c:/Users/lucky/Univ/SECURIOUS/Cloud_Security/docker-compose.yaml) (로컬 가상화 환경)**:
    *   `services/` 내 각 폴더의 `Dockerfile` 경로를 읽어 로컬에서 컨테이너 이미지를 컴파일하고 환경 변수와 포트를 주입하여 통합 작동을 테스트합니다.
*   **[jenkins-pipeline/](file:///c:/Users/lucky/Univ/SECURIOUS/Cloud_Security/jenkins-pipeline) (CI 파이프라인)**:
    *   젠킨스는 이 `services/` 폴더 내 소스 코드를 대상으로 **Snyk Code(SAST)** 정적 분석 및 **Snyk Open Source(SCA)** 라이브러리 검사를 실시하여 소스 레벨의 보안 취약성을 빌드 전에 차단합니다.
    *   `services/` 내부의 각 `Dockerfile`을 빌드한 후, 최종 컨테이너 이미지를 **Harbor 사설 레지스트리**로 안전하게 전송합니다.
*   **[k8s-manifests/](file:///c:/Users/lucky/Univ/SECURIOUS/Cloud_Security/k8s-manifests) (GitOps 배포)**:
    *   `services/`에 기재된 포트(`EXPOSE`), 필요한 연동 환경 변수 스키마 규격이 `k8s-manifests/` 내의 Deployment YAML 설정들과 완전히 1:1로 매핑됩니다.
    *   `services/` Dockerfile에서 지정한 비특권 유저(`USER appuser`, UID `10001` 또는 `1000`) 런타임 보안 정책이 쿠버네티스 YAML의 `securityContext`와 일치하여 하드닝 샌드박스를 구성합니다.
