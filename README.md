## 프론트엔드 배포 파이프라인
---
###  배포 파이프라인

![스크린샷 2025-05-29 오후 11 22 51](https://github.com/user-attachments/assets/22817b29-8a5b-4cee-ba07-3fa00d84f850)

이러한 배포 파이프라인을 구성하기 전 아래 항목들을 수행:
1. AWS S3 Bucket 생성 및 정책 등록
2. Local에서 Next.js 프로젝트를 생성 및 빌드
3. S3에 out 디렉토리 업로드 및 정적 웹사이트 호스팅 활성화
	- 엔드포인트 주소 생성
4. CloudFront로 CDN 구성 및 S3 정적 페이지 연결
5. CloudFront로 배포 후 배포 URL 확인
6. IAM 사용자 생성 및 권한 정책 부여
7. 엑세스 키 발급
	-  access key, secret key를 통해 외부 서비스에서 AWS 제어 가능하도록

### Github Actions Workflow

1. 코드 변경 & Github Push
```yaml
on:
  push:
    branches:
      - main
```

- `main` 브랜치에 코드를 수정 후 push 하면
- GitHub Actions가 실행됨 (자동 트리거)

2. GitHub Actions가 실행됨 (자동 트리거)
```yaml
jobs:
  deploy:
    runs-on: ubuntu-latest
```
- `.github/workflows/deployment.yml` 파일의 내용에 따라 순서대로 작업 실행
- 서버 환경으로는 `ubuntu-latest` 사용

3. 저장소 코드 다운로드
```yaml
- name: Checkout repository
  uses: actions/checkout@v4
```
- actions/checkout 사용해서 저장소의 최신 코드를 CI 환경에 가져옴

4. 프로젝트 의존성 설치
```yaml
- name: Install dependencies
  run: npm ci
```
- `npm ci`를 통해 정확하고 빠르게 패키지 설치

5. 정적 빌드 실행
```yaml
- name: Build
  run: npm run build
```
- `npm run build` 명령어로 `out/` 디렉토리 생성 (Next.js export)

6. AWS 자격 증명 설정
```yaml
- name: Configure AWS credentials
  uses: aws-actions/configure-aws-credentials@v1
  with:
    aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY_ID }}
    aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
    aws-region: ${{ secrets.AWS_REGION }}
```
- GitHub Secrets에 저장된 인증키로 AWS CLI 인증

7. S3 버킷에 배포
```yaml
- name: Deploy to S3
  run: |
    aws s3 sync out/ s3://${{ secrets.S3_BUCKET_NAME }} --delete
```
- `aws s3 sync` 명령어로 `out/` 안의 파일을 S3에 업로드
- `--delete` 옵션으로 기존 S3에 있던 삭제된 파일도 반영됨

8. CloudFront 캐시 무효화
```yaml
- name: Invalidate CloudFront cache
  run: |
    aws cloudfront create-invalidation --distribution-id ${{ secrets.CLOUDFRONT_DISTRIBUTION_ID }} --paths "/*"
```
- 새로 배포한 파일이 곧바로 반영되도록 CloudFront의 캐시를 비워줌 (invalidate)

>캐시 무효화를 하는 이유

- CloudFront에서 “오리진”으로 S3의 **정적 웹사이트 엔드포인트**를 등록했기 때문에, S3에 새로운 파일을 업로드 하면 CloudFront가 그걸 가져와서 배포
- 하지만 CloudFront는 성능을 위해 **한번 본 파일을 캐싱**해서 재사용
- 그래서 캐시 무효화를 통해 이전 CloudFront 캐시를 비워서 새로운 파일로 갱신되도록 함

### 주요 링크
- S3 버킷 웹사이트 엔드포인트: http://neo622-awsbucket.s3-website-ap-southeast-2.amazonaws.com
- CloudFrount 배포 도메인 이름: https://d12ekenpjitxux.cloudfront.net/

### 주요 개념
- GitHub Actions과 CI/CD 도구
	- GitHub Actions는 GitHub에서 제공하는 자동화 도구로, 코드 변경 이벤트에 따라 워크플로우를 실행할 수 있다.
	- CI(지속적 통합)는 코드를 자주 병합하고 테스트하여 품질을 유지하는 개발 방식이다.
	- CD(지속적 배포)는 테스트가 완료된 코드를 자동으로 배포 환경에 전달해준다.
	- Actions를 활용하면 빌드, 테스트, 배포까지의 과정을 자동화하여 개발 효율성을 높일 수 있다.

 - S3와 스토리지
	- Amazon S3는 정적 파일을 저장하고 웹에서 접근할 수 있는 객체 스토리지 서비스이다.
	- 이미지, JS, HTML 등의 정적 자산을 저장하기 적합하며, 버킷 단위로 관리된다.
	- 퍼블릭 접근 정책을 설정하면 외부에서 누구나 파일에 접근할 수 있도록 할 수 있다.
	- 정적 웹사이트 호스팅 기능을 사용하면 S3 버킷을 웹 서버처럼 사용할 수 있다.

 - CloudFront와 CDN
	- Amazon CloudFront는 글로벌 CDN(콘텐츠 전송 네트워크)으로 S3와 연결하여 콘텐츠를 빠르게 전달할 수 있다.
	- CDN은 사용자의 지리적 위치에 가까운 엣지 서버에서 콘텐츠를 전달해 지연 시간을 줄인다.
	- 오리진(Origin)으로 S3 정적 사이트 엔드포인트를 지정해 연동할 수 있다.
	- HTTPS 연결, 요청 리다이렉션, 캐싱 전략 등 다양한 최적화 설정을 제공한다.

 - 캐시 무효화 (Cache Invalidation)
	- CDN은 성능 향상을 위해 리소스를 일정 시간 동안 캐시해두고 재사용한다.
	- 새 버전의 파일을 배포했을 경우, 기존 캐시를 무효화해야 사용자에게 최신 버전이 보인다.
	- CloudFront에서는 `create-invalidation` 명령어로 캐시된 경로를 무효화할 수 있다.
	- 캐시 무효화는 비용과 속도에 영향을 줄 수 있으므로 효율적인 경로 지정이 중요하다.

 - Repository secret과 환경변수
	- 민감한 정보(AWS 키, 배포 ID 등)는 GitHub의 Repository Secrets 기능을 통해 안전하게 저장한다.
	- Secrets는 워크플로우 내에서 `${{ secrets.변수명 }}` 형태로 불러올 수 있다.
	- 환경변수는 배포 환경에 따라 동적으로 설정될 수 있어 코드와 배포의 유연성을 높인다.
	- 민감 데이터의 노출을 막고, 팀 협업 시 안전하게 배포 자동화를 구성할 수 있다.

## CDN과 성능 최적화
---

![image](https://github.com/user-attachments/assets/52e8b493-5335-47df-bf44-73bd8d7884ee)
