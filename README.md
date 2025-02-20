# 기본과제- 프론트엔드 배포 CI/CD 파이프라인 구축하기

## 🤷‍♂️ 목적

- 변경된 코드를 GitHub에 올리면 자동으로 지정된 아래 작업을 완료하고 배포하도록 만듦

## 📦 구축된 CI/CD 의 빌드 과정

![image](https://github.com/user-attachments/assets/b7e0b25a-5496-4389-a3f5-2e7bb1e12c08)

1. **Checkout 액션을 사용해 코드 내려받기** 
→ GitHub Actions에서 제공하는 `actions/checkout`을 사용해 최신 코드를 가져옴 
2. **의존성 설치**
    - `npm ci` 명령어로 프로젝트 의존성 설치 (`npm install`보다 빠르고 일관적)
3. **Next.js프로젝트 빌드**
    - `npm run build`
4. **AWS 자격 증명 구성**
    - GitHub Actions에 저장된 계정 인증 정보를 사용해 구성
5. **빌드된 파일을 S3 버킷에 동기화**
    - AWS CLI 명령어를 사용하여 빌드된 정적 파일을 S3 버킷에 업로드.
6. **CloudFront 캐시 무효화**
    - 최신 컨텐츠로 갱신

## 🌐 주요 링크

- S3 버킷 웹사이트 엔드포인트: http://zenna-bucket.s3-website-ap-southeast-2.amazonaws.com/
- CloudFrount 배포 도메인 이름: https://d2a4btkl9b5lcr.cloudfront.net/

## 📙 주요 개념

- CI/CD 도구:
    - **CI** (Continuous Integration) - 지속적 통합 : 
    개발자가 변경한 코드가 정기적으로 빌드, 테스트되어 코드 품질을 유지하고 문제를 빠르게 감지할 수 있도록 하는 프로세스
    - **CD** (Continuous Deployment/Delivery) - 지속적 배포/제공: 
    자동으로 or 승인 후 배포해서 운영환경에 적용하도록 하는 프로세스
- **GitHub Actions** : GitHub 저장소 내에서 자동화된 워크플로우를 구축할 수 있는 도구
    - `.github/workflows/deployment.yml` 의 설정대로 진행됨
- **S3**와 스토리지: (Simple Storage Service) = S3
    - AWS에서 제공하는 객체 스토리지 서비스. 대규모 데이터를 안전하게 저장
    - CloudFront와 연결하여 정적 웹 애플리케이션 배포
    - 버킷(Bucket)과 객체(Object)로 구성
- **CloudFront** = AWS의 CDN(Content Delivery Network) 서비스.
- **CDN** (Content Delivery Network) :
    - 가까운 Edge Location에서 캐싱된 서비스를 제공해 빠름
- **캐시 무효화(Cache Invalidation):**
  - CDN을 통해 캐싱된 파일을 제공하면, 변경 내용이 있어도 자동으로 캐싱이 되는 것은 아니기 때문에 바로 서비스에 반영되지 않을 수 있음!
    ![image](https://github.com/user-attachments/assets/f2e9a08a-87ec-417c-bc14-ce17b13e444d)
  - 이 문제를 해결하기 위해 캐시 무효화를 실행해서 변경 내용을 반영
  - CloudFront Invalidation을 실행해 캐시 무효화하는 명령
    `aws cloudfront create-invalidation --distribution-id <배포ID> --paths "/*"` - 배포 ID : CloudFront 배포를 생성할 때 자동 생성된 ID
  - 배포 후 변경 사항이 즉시 사용자에게 적용되도록 보장.
  - CloudFront는 콘텐츠가 변경되어도 캐시 만료 시간(TTL) 전까지는 기존 캐시 제공.
  - CloudFront의 캐시 무효화는 월 1,000번만 무료 서비스…!
  - 전체 경로("/\*") 무효화는 비용이 많이 발생할 수 있으니, 변경된 파일만 무효화하기
    - 위 명령어에서 `".*"` 는 전체 경로 무효화
- Repository secret과 환경변수:
  - GitHub Actions에 민감한 정보(Access Key, API Key 등)를 환경 변수로 저장해두고 사용
  - GitHub Settings → Secrets and variables → Actions
  - 워크플로우에서 `${{ secrets.<변수명> }}`로 참조.
 
# #심화과제 - CDN도입 성능 개선 보고서

### 준비

- CDN도입 전 : `S3`
- CDN도입 후 : `CloudFront`

### 측정 환경 세팅

- 측정 환경 : 동일한 PC / 윈도우 / Chrome
- 측정 도구 : Chrome 개발자 도구 - Network, Performance, Lighthouse

### 기대 효과

- TTFB 감소: 요청이 지역 엣지 서버로 라우팅.
- 대역폭 최적화: 압축 및 캐싱으로 전송 비용 감소.
- 성능 일관성: 사용자 위치에 따른 성능 차이 감소.

## #성능 측정

|  | CDN 미적용 | CDN 적용 | 비고 |
| --- | --- | --- | --- |
| 요청 건수 (건) | 17 | 17 |  |
| 전송(kB) | 486 | 187 | 61.5% 개선 |
| 리소스 (kB) | 479 | 479 |  |
| 완료시간 (ms) | 1940 | 461 | 76.2% 개선 |

![image.png](attachment:8db2694b-8a5f-4ca7-8226-9fd3e0baefa5:image.png)

### Queueing (대기열)

요청이 브라우저의 네트워크 큐에 대기한 시간

|  | 미적용(S3) | 적용(CloudFront) | 비고 |
| --- | --- | --- | --- |
| Queueing (ms) | 4.21 | 2.27 | 46.1% 개선 |

### Stalled (중단됨)

요청이 실행되기 전 네트워크 자원 확보를 기다린 시간

|  | 미적용(S3) | 적용(CloudFront) | 비고 |
| --- | --- | --- | --- |
| Stalled (ms) | 0.95 | 6.92 | **86.3% 저하** |
- 저하 원인 분석 & 개선
    1. DNS 조회 및 TCP 연결 증가 → AWS CloudFront에서 Keep-Alive 설정 _ 하려고 했으나 CloudFront는 원칙적으로 keep-alive가 적용된 HTTP/2를 이미 지원
    2. HTTP/2 또는 HTTP/3 설정 누락 → 이미 설정됨
    3. 캐시 히트율(Cache Hit Ratio) 저조 → 이미 CacheOptimized 적용됨
    4. TTL(Time-to-Live) 연장 → 이미 권장 설정만큼 되어 있음
- 두 페이지 모두 다시 강력 새로고침 실행

|  | 미적용(S3) | 적용(CloudFront) | 비고 |
| --- | --- | --- | --- |
| Stalled (ms) | 53.34 | 5.67 | 89.4% 개선 |

### DNS Lookup Time (DNS 조회)

도메인 이름을 IP로 변환하는 시간

| CDN | 미적용(S3) | 적용(CloudFront) | 비고 |
| --- | --- | --- | --- |
| DNS Lookup (µs) | 42 | 0 | - |

### Initial Connection

TCP/SSL 연결을 설정하는 데 걸린 시간

| CDN | 미적용(S3) | 적용(CloudFront) | 비고 |
| --- | --- | --- | --- |
| Initial Connection (ms) | 197.58 | 0 | - |

### Request sent(요청 전송됨)

클라이언트(브라우저)가 서버로 요청을 전송하는 데 걸린 시간

| CDN | 미적용(S3) | 적용(CloudFront) | 비고 |
| --- | --- | --- | --- |
| Request sent (ms) | 0.87 | 0.30 | 65.5 |

### TTFB (Time to First Byte)

Waiting for server response

서버가 요청을 처리하고 첫 바이트를 반환할 때까지의 시간 

- ≠ Latency
    - `TTFB =  Latency + 서버의 첫 응답 생성 시간`
    - Latency는 서버와 클라이언트 간 데이터 전송 지연시간

| CDN | 미적용(S3) | 적용(CloudFront) | 비고 |
| --- | --- | --- | --- |
| TTFB (ms) | 168.80 | 16.55 | 90.2% 개선 |

### Download Time

콘텐츠 다운로드 시간.

| CDN | 미적용(S3) | 적용(CloudFront) | 비고 |
| --- | --- | --- | --- |
| Download Time(ms) | 2.82 | 1.63 | 42.2% 개선 |

### Content-Encoding

CDN은 압축 및 최적화를 통해 전송 데이터 크기를 줄임

![image.png](attachment:04cc2824-39c0-4ae6-a377-f829463884bc:image.png)

### LCP (Largest Contentful Paint)

웹페이지의 주요 콘텐츠가 화면에 표시되는 시간

페이지 로딩이 **사용자에게 체감되는 속도**

- 🟢 2.5초 이내 /🟡 4초 이내 / 🔴 4초 이상
- **개선 팁**
    - **이미지 최적화**(WebP, AVIF 포맷 사용)
    - **렌더링 차단 리소스 최소화**(CSS, JS)
    - **CDN을 통한 콘텐츠 전송 가속**

| CDN | 미적용(S3) | 적용(CloudFront) | 비고 |
| --- | --- | --- | --- |
| LCP(s) | 0.64🟢 | 0.40🟢 | 37.5% 개선 |

### **CLS (Cumulative Layout Shift)**

페이지 콘텐츠의 레이아웃이 예상치 못하게 변경되는 정도

사용자 경험(UX)을 직접적으로 저하

광고, 이미지, 폰트 로딩으로 인한 레이아웃 변경 방지

- 🟢 0.1 이하 / 🟡 0.25 이하 / 🔴 0.25 초과
- 개선 팁
    - 이미지, 광고에 크기 속성(width/height) 명시
    - 웹 폰트 FOIT/FOUT 현상 방지(font-display: swap 사용)
    - 동적 콘텐츠 추가 시 레이아웃 예약 공간 확보

| CDN | 미적용(S3) | 적용(CloudFront) | 비고 |
| --- | --- | --- | --- |
| **CLS** | 0 | 0 | 동일 |

### INP (Interaction to Next Paint)

사용자의 입력(클릭, 탭, 키 입력)에 대한 반응성 측정

사용자가 버튼을 눌렀을 때 페이지가 얼마나 빠르게 반응하는지 확인

UI 반응성이 사용자의 만족도에 직접적인 영향을 미침

- 🟢 200ms 이내 / 🟡 500ms 이내 / 🔴 500ms 이상
- 개선 팁
    - 메인 스레드 작업 최적화(long task 줄이기)
    - 필요할 때만 JS 로드(코드 스플리팅)
    - RequestAnimationFrame, requestIdleCallback 적절히 활용

| CDN | 미적용(S3) | 적용(CloudFront) | 비고 |
| --- | --- | --- | --- |
| INP  | - | - |  |

### Lighthouse

![image.png](attachment:ff5697b7-302e-4902-8c16-c32428e884d2:image.png)

CDN적용 전은 Performance (성능)면에서 뒤쳐짐. 웹 페이지의 로딩 속도와 사용자 체감 성능이 비교적 나쁘다는 것을 의미.

| **지표** | **설명** | **권장 기준** |
| --- | --- | --- |
| **FCP (First Contentful Paint)** | 사용자가 **처음 콘텐츠**를 볼 수 있는 시점 | **1.8초 이하** |
| **LCP (Largest Contentful Paint)** | 가장 큰 콘텐츠(이미지/텍스트)가 로드되는 시점 | **2.5초 이하** |
| **TTI (Time to Interactive)** | 페이지가 **완전히 인터랙티브**해지는 시점 | **3.8초 이하** |
| **TBT (Total Blocking Time)** | 사용자 입력을 **차단한 총 시간** (메인 스레드의 블로킹 시간) | **200ms 이하** |
| **CLS (Cumulative Layout Shift)** | 예상치 못한 **레이아웃 변경의 누적 점수** | **0.1 이하** |
| **INP (Interaction to Next Paint)** *(2024 도입)* | 사용자 입력에 **페이지가 응답**하는 시간 | **200ms 이하** |

특히 TBT에서 좋지 않은 모습을 보였다

BestPractices는 Http설정 등과 연관되어있어 다루지 않음.

---

# 과제 회고

### 과제에서 좋았던 부분

- 다른 사람들의 이력서에서만 보았던 ‘성능 개선’ 수치를 어떻게 확인할 수 있는지 알 수 있었다.
- 어떻게 진행할지 막연하게 던져지지 않고 설명이 잘 되어있었고, 그동안의 AWS실습과 다르게 모든 설정의 의미가 설명이 되어 있어 이해하기 쉬웠다.

### 새롭게 알게된 점

- S3리전을 아시아로 설정하고 진행했어서, CDN을 적용해도 크게 달라지지 않을 줄 알았는데 성능 차이가 나서 신기했다. 단순히 가까운 edge location으로 먼저 연결해주는 게 아니라는 것을 알게되었다.
- 생각보다 어렵지 않게 성능 개선하는 방법이 있다는 것을 알게 되었다.

### 어렵거나 아쉬운 점, 궁금한 점

- 네트워킹 탭의 Stalled가 CDN도입 후 오히려 늘어나서, 이 부분을 해결하려고 여러 시도를 해보았는데 이미 잘 설정되어 있었습니다.. 의문을 가지고 새로고침을 했더니 전 날 저녁까지도 일관적이던 비교 결과가 뒤바뀌어 있었는데 원인을 잘 모르겠습니다.
