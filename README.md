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
  - CloudFront와 연결하여 정적 웹 애플리케이션 배포
  - 버킷(Bucket)과 객체(Object)로 구성
- CloudFront **/** CDN:
  - **CloudFront** = AWS의 CDN(Content Delivery Network) 서비스.
  - **CDN** (Content Delivery Network) :
    - 가까운 Edge Location에서 캐싱된 서비스를 제공해 빠름
- **캐시 무효화(Cache Invalidation):**
  - CDN을 통해 캐싱된 파일을 제공하면, 변경 내용이 있어도 자동으로 캐싱이 되는 것은 아니기 때문에 바로 서비스에 반영되지 않을 수 있음!
    ![image.png](attachment:dd475950-3d3d-4d12-888d-871025d07121:image.png)
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
