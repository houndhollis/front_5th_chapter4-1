# 기본과제와 심화과제는 아래에 있습니다.


## 1️⃣ 기본과제

### 주요 링크

- S3 버킷 웹사이트 엔드포인트: http://kywbucket1.s3-website.ap-northeast-2.amazonaws.com
- CloudFrount 배포 도메인 이름: https://d29keg2xwnzrdd.cloudfront.net

### 주요 개념

- ### GitHub Actions과 CI/CD 도구 <br/> 
자동으로 코드를 빌드하고 배포해주는 도구입니다. **GitHub Actions**는 GitHub에 내장된 CI/CD(Continuous Integration / Continuous Deployment) 플랫폼입니다. <br/><br/>
예: main 브랜치에 push → 빌드 → S3에 업로드 → CloudFront 캐시 무효화까지 자동 실행 <br/>
CI는 테스트/빌드 자동화, CD는 배포 자동화

- ### S3와 스토리지 <br/>
정적 웹사이트 파일(html, js, css 등)을 저장하는 AWS 서비스입니다.<br/>
Next.js 빌드 결과물 (out/) 폴더의 내용을 올리는 공간으로 일반적인 웹 호스팅 서버처럼 HTML/CSS/JS 정적 파일 제공합니다. CloudFront와 함께 쓰면 훨씬 빠르고 안전하게 배포 가능합니다. 
- ### CloudFront와 CDN <br/>
사용자에게 가장 가까운 위치에서 빠르게 콘텐츠를 제공하는 CDN (Content Delivery Network)입니다.
S3에서 직접 가져오는 것보다 빠르고 안정적으로 서비스 가능하며 전 세계 엣지 로케이션에 캐시되어 응답 속도를 향상시킵니다.

커스텀 도메인 연결, HTTPS 제공, 캐시 설정 등 가능
- ### 캐시 무효화(Cache Invalidation) <br/>
CloudFront에 남아 있는 옛날 파일을 강제로 새 파일로 바꿔주는 명령어입니다. <br/>

예: 빌드된 JS/CSS 파일은 이름이 같아서 바뀌어도 CloudFront가 예전 파일을 계속 줄 수 있습니다. <br/>
-> 캐시 무효화 필요 !
🧠 자세한 내용은 아래에 있습니다.

- ### Repository secret과 환경변수 <br/>
보안이 필요한 민감한 값을 GitHub에 안전하게 저장하는 방법입니다. <br/>

예: AWS_ACCESS_KEY_ID, S3_BUCKET_NAME, CLOUDFRONT_DISTRIBUTION_ID 등... <br/>
GitHub Secrets에 저장하고, GitHub Actions에서 ${{ secrets.XXX }} 로 사용되며, 직접 코드에 노출되면 보안상 매우 위험합니다.

### 배포 파이브라인 시각화
<img width="1142" alt="스크린샷 2025-05-28 오후 8 00 26" src="https://github.com/user-attachments/assets/a36eb312-6397-4835-93e8-899885df5746" />
1. 개발자 → GitHub (push)<br/>
2. GitHub Actions → AWS S3 (정적 파일 업로드))<br/>
3. GitHub Actions → CloudFront (캐시 무효화))<br/>
4. 클라이언트 → CloudFront → S3)<br/>
방식으로 진행됩니다. 

### 캐시 무효화
deployment.yml 에서 ```aws cloudfront create-invalidation``` 부분이 바로 **캐시 무효화(invalidation)** 단계입니다.
- CloudFront는 성능 향상을 위해 정적 콘텐츠를 전 세계 엣지 로케이션에 캐싱합니다.
- 코드가 변경되어도 캐시가 남아 있으면 구버전이 사용자에게 제공될 수 있습니다.
- 따라서 배포 후 "/*" 경로를 무효화해서, 최신 콘텐츠를 엣지에서 다시 받아오도록 강제 합니다.
<br/>

> ### 추가 리서치 Time To Live
TTL은 CloudFront가 원본(S3 등) 에서 가져온 콘텐츠를 얼마나 오랫동안 캐시로 유지할지 결정하는 시간 값입니다. 이 콘텐츠는 TTL 동안 동일한 파일로 유지되고, TTL이 만료되기 전까지는 원본 서버(S3 등)에 재요청하지 않습니다.

- Default TTL :	기본 캐시 유지 시간 (초 단위)
- Minimum TTL :	반드시 이 시간 이상은 캐싱함
- Maximum TTL : 이 시간을 넘겨서는 절대 캐싱하지 않음

```
예시 상황

Default TTL: 86400 (1일)
Minimum TTL: 60
Maximum TTL: 604800 (7일)
```
CloudFront는 기본적으로 콘텐츠를 1일 동안 캐시합니다. 요청 후 1일이 지나야 다시 원본(S3)으로부터 새로 받아올 수 있습니다. 하지만 만약 원본에서 Cache-Control 헤더가 짧게 설정되어 있다면, 그에 따라 캐시 갱신 시점이 앞당겨질 수도 있습니다.
<br/><br/>
### 🧠 TTL의 중요성과 고려사항
- 상황	TTL 설정 전략 <br/>

<span>콘텐츠가 자주 바뀜 -> 짧은 TTL (예: HTML, JS: 60초 ~ 10분)</span> <br/>
<span>콘텐츠가 거의 안 바뀜 ->	긴 TTL (예: 이미지, 폰트: 1일 ~ 1년)</span> <br/>
<span>민감한 데이터, 실시간성 요구 -> TTL 매우 짧게 + 캐시 무효화 필요</span> <br/>
<br/>
### 🎯 실무 적용 전략
1. 빌드된 JS/CSS 파일은 해시 붙이기
- main.abcd123.js
이렇게 하면 파일이 바뀔 때마다 이름도 바뀌므로, TTL을 길게 해도 안전합니다.

2. HTML 같은 루트 파일은 TTL 짧게 설정 + 캐시 무효화
- 사용자에게 항상 최신 페이지가 보이도록 하기 위함

3. 헤더로 직접 TTL 제어 가능
- Cache-Control 또는 Expires 헤더를 S3에 설정하면 CloudFront가 이를 따릅니다.
> Cache-Control: max-age=600 → 10분간 캐시 유지

<br/>

### 🔧 TTL 설정 방법 (CloudFront 콘솔에서)
1. CloudFront 콘솔 접속 <br/>
2. 배포 선택 → Behaviors 탭 <br/>
3. 해당 경로(예: Default (*)) 편집 <br/>
4. TTL 설정 <br/>

### 📝 요약
- TTL이란?	콘텐츠를 엣지에 얼마나 오래 보관할지 결정하는 시간
- 설정 위치 :	CloudFront Behavior 또는 원본의 Cache-Control
- 무효화와 차이 : TTL은 자동 만료, 무효화는 강제 삭제
- 전략	자주 바뀌는 콘텐츠는 짧은 TTL, 안 바뀌는 건 긴 TTL

### Route53 
AWS의 DNS 서비스로, 도메인 이름을 웹 서버, S3 버킷, CloudFront 배포 등과 연결할 수 있게 해주는 서비스입니다.
```
사용자 브라우저
      ↓
www.domain.com (도메인 입력)
      ↓
Route 53 (DNS 해석)
      ↓
CloudFront (글로벌 CDN)
      ↓
S3 (정적 파일 서비스)
```
## 2️⃣ 심화과제
| CDN 도입전 | CDN 도입후 |
|----------|-----------|
| <img width="780" alt="스크린샷 2025-05-28 오후 9 08 50" src="https://github.com/user-attachments/assets/e0fdeb80-bc92-4355-b2da-3a463ef45634" /> | <img width="780" alt="스크린샷 2025-05-28 오후 9 09 07" src="https://github.com/user-attachments/assets/08b45c17-9bfe-44ed-84c1-852b765ecdc5" /> |
### 성능
CDN 적용 후, 네트워크 요청 수와 리소스 로딩 시간이 현저히 감소한 것을 확인할 수 있습니다. 이는 정적 리소스가 S3가 아닌 CloudFront 엣지 서버에서 캐싱되어 빠르게 전달되기 때문입니다.

### 📌 AS-IS: S3 버킷 단독 정적 파일 제공
- 정적 파일을 S3 버킷에서 직접 제공
- 단일 리전(예: 서울 리전)에서 파일 응답 처리
- 대한민국에서 거리가 먼 지역(예: 해외 사용자)에서 높은 레이턴시(Latency) 경험
- 브라우저 네트워크 탭에서:
  -  TTFB(Time to First Byte) 값이 상대적으로 높음
  -  병렬 요청 처리 한계 및 일정량의 지연 발생

### 📌 TO-BE: CloudFront CDN을 이용한 정적 파일 캐싱 및 배포
- AWS CloudFront를 사용하여 정적 리소스를 전 세계 엣지 서버에 캐싱
- 사용자 요청 시 가까운 엣지 로케이션에서 빠르게 응답
- S3는 CloudFront의 오리진(Origin) 역할만 수행
- 브라우저 네트워크 탭에서:
  - TTFB 및 콘텐츠 다운로드 시간 급감
  - 캐시 히트 시, 응답 시간이 수 밀리초(ms) 단위로 짧아짐

### ✅ Result
- 평균 TTFB 300~800ms -> 50~150ms
- 전체 페이지 로딩 시간 약 2.5~4초 -> 약 0.8~1.5초
- 평균 감소율 약 60~80% 감소


 
