# 9주차 과제

## 1. 시스템 개요

GitHub Applications의 ASW앱을 연동하여 코드 저장소에 푸시하게 되면 아래와 같은 워크플로우로 빌드 및 배포가 진행되게 됩니다.

## 2. 아키텍처 구성

### 2.1 전체 시스템 구조

![system](https://github.com/user-attachments/assets/41987b3c-dd6a-40d9-9ff5-2fb27ebb3f4e)

```
코드 수정 및 푸시 → GitHub Application → AWS CodeBuild → S3 → CloudFront → 최종 사용자
```

### 2.2 핵심 구성 요소

**GitHub Repository**
- 소스 코드 저장
- GitHub Application을 통한 AWS CodeBuild와의 연동
- 코드 변경사항 감지 및 트리거 역할

**AWS CodeBuild**
- 자동화된 빌드 서비스
- buildspec.yml 파일 기반 빌드 프로세스 실행
- 의존성 설치, 빌드를 통한 정적 리소스 생성, S3 업로드, CloudFront Invalidate 실행

**Amazon Route53**
- S3,CloudFront 배포 도메인과 연결되도록 설정
- TLS 인증서와 연동 하여 HTTPS 접속 가능 (CloudFront)

**Amazon S3**
- 정적 웹사이트 호스팅
- 빌드된 파일들의 저장소
- S3배포 주소: [http://s3.전도명.site](http://s3.전도명.site)

**Amazon CloudFront**
- 글로벌 CDN(Content Delivery Network)
- 전 세계 사용자에게 빠른 콘텐츠 전달
- CloudFront배포 주소: [https://cf.전도명.site](https://cf.전도명.site)

## 3. 배포 프로세스

### 3.1 단계별 배포 과정

1. **코드 변경 및 푸시**
    - 개발자가 GitHub 저장소에 코드 변경사항을 푸시

2. **빌드 트리거**
    - GitHub Application이 변경사항을 감지
    - AWS CodeBuild 프로젝트가 자동으로 시작

3. **빌드 실행**
    - CodeBuild가 buildspec.yml 파일에 정의된 명령어 실행
    - 패키지 의존성 설치 및 프론트엔드 빌드 수행

4. **파일 배포**
    - 빌드된 정적 파일들이 S3 버킷에 자동 업로드

5. **캐시 갱신**
    - CloudFront 배포의 캐시 무효화(invalidation) 실행
    - 엣지 로케이션에서 최신 콘텐츠 제공

6. **서비스 제공**
    - 사용자가 CloudFront를 통해 최신 버전에 접근

## 4. 보안 및 권한 관리

### 4.1 IAM 역할 설정

CodeBuild 서비스가 필요한 AWS 리소스에 접근할 수 있도록 최소 권한 원칙에 따라 IAM 역할을 구성했습니다:

- S3 버킷에 대한 읽기/쓰기 권한
- CloudFront 캐시 무효화 쓰기 권한

### 4.2 환경변수 및 보안 정보 관리

AWS Application 사용으로 민감한 정보를 외부로 노출 최소화:

- API 키
- 환경별 설정값
- 빌드 시 필요한 인증 정보

## 5. 성능 최적화

### 5.1 CDN 활용

![cloudfront netwrok](https://github.com/user-attachments/assets/2b806287-f64d-4da1-b915-467996e43692)
![cloudfront netwrok](https://github.com/user-attachments/assets/57195950-6f83-41fb-921b-c3e2905464f9)

| 항목                      | S3 웹 호스팅 | CloudFront (CDN) |
| ----------------------- | -------- | ---------------- |
| **총 전송량 (Transferred)** | 483 kB   | 201 kB           |
| **전체 완료 시간 (Finish)**   | 381ms    | 145ms            |
| **DOMContentLoaded**    | 171ms    | 77ms             |
| **Load 완료 시간**          | 325ms    | 124ms            |

CloudFront를 통한 성능 최적화 효과:
- 네트워크 요청 최적화 - 압축 전송으로 네트워크에 사용되는 비용이 감소되어 빠른 데이터 전송 가능
- 엣지 로케이션 캐싱 - 클라이언트와 물리적으로 가까운 위치의 엣지 로케이션에서 데이터 캐싱 및 전송 

![cloudfront edge](https://docs.aws.amazon.com/ko_kr/AmazonCloudFront/latest/DeveloperGuide/images/how-cloudfront-delivers-content.png)

엣지 로케이션에 컨텐츠를 캐싱하고 있기 때문에, S3에 반영을 하더라도, CloudFront 주소로 접속하면 이전 컨텐츠를 보게될 수 있다. 
이러한 이유로 배포 후에는 모든 엣지 로케이션에 존재하는 파일에 대해 캐시를 무효화하여 새로운 파일을 받아오게 설정해주어야 새로운 컨텐츠를 가져올 수 있다.

### 5.2 캐시 전략

효율적인 캐시 관리를 통한 성능 향상:
- 정적 자산의 적절한 캐시 정책 설정
- 배포 시 자동 캐시 무효화로 최신 콘텐츠 보장
