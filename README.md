# 기본과제와 심화과제는 아래에 있습니다.


## 1️⃣ 기본과제

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
- 코드가 변경되어도 캐시가 남아 있으면 구버전이 사용자에게 제공될 수 있어요.
- 따라서 배포 후 "/*" 경로를 무효화해서, 최신 콘텐츠를 엣지에서 다시 받아오도록 강제하는 합니다.
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
