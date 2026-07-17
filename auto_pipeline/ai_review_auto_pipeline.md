# AI 코드리뷰 및 CI/CD 배포 자동화 파이프라인

TV/OTT 광고 캠페인 관리 플랫폼(SMAP)의 배포 자동화 시스템.

---

## 배경

3인 개발팀, 주 1~2회 배포. 기존 방식은 환경별 배포 스크립트를 머지 후 수동 실행했다.
문제는 스크립트 자체가 아니라 사람이 개입하는 구간에 있었다.

- 배포할 환경과 스크립트를 개발자가 직접 매칭해야 함 → dev용 스크립트를 prod에 돌리거나 반대의 경우
- "지금 나간 게 뭐야?"를 팀이 공유하지 않으면 모름 → 사고가 나도 원인 추적이 늦음
- 롤백 역시 수동 → 장애 상황에서 다시 스크립트 실행, 버전 확인, 확인에 10분 이상

주 1~2회지만 한 번 잘못 나가면 광고 송출에 직접 영향을 미치는 시스템이라 배포 자체에 불필요한 긴장이 있었다.

---

## 목표

1. 환경 혼동 제거 — dev/prod 배포가 구조적으로 분리되어야 함
2. 팀 전체가 배포 시점과 내용을 자동으로 공유받아야 함
3. 롤백을 버튼 하나로 — 장애 상황에서 판단할 여력만 남기기
4. 추가 인프라 없이 — 배포 빈도(주 1~2회)에 항상 켜져 있는 서버를 두는 건 낭비

---

## 아키텍처

```
━━━━━━━━━━━━━━━━━━ ① 리뷰 ━━━━━━━━━━━━━━━━━━

PR 등록 / 업데이트
        │ Bitbucket Webhook
        ▼
[ LLM 리뷰 서버 ] Claude CLI
  PR diff 분석 → AWS SES → 리뷰 결과 이메일
        │
        │ 팀원 확인 후 머지

━━━━━━━━━━━━━━━━━━ ② 빌드 ━━━━━━━━━━━━━━━━━━

develop push / master merge
        │
        ▼
[ Bitbucket Pipelines ]
  pytest 게이트 — 실패 시 여기서 중단
  Docker 빌드 & 레지스트리 Push
  태그: {env}-{build_number}-{YYYYMMDDHH}
        │
        ▼
[ AWS SES ] 빌드 완료 이메일
  변경된 PR 제목 + 커밋 목록
  → [배포하기] 버튼 (HMAC 서명, 2시간 유효)

━━━━━━━━━━━━━━━━━━ ③ 배포 ━━━━━━━━━━━━━━━━━━

        │ 버튼 클릭
        ▼
[ AWS Lambda ] — Function URL
  HMAC-SHA256 서명 검증
  중복 실행 방지 (is_pipeline_running 체크)
        │ Bitbucket Pipeline REST API
        ▼
[ deploy-dev / deploy-prod ]
  SSH → docker pull → docker-compose up
        │
        ├── /health 헬스체크 5회 재시도
        │   실패 → Lambda auto-rollback 트리거
        │
        ▼
[ AWS SES ] 배포 완료 이메일
  배포된 이미지 태그 + [롤백하기] 버튼
```

---

## 설계 결정

### 이메일 버튼 승인 방식

자동 배포(push → 즉시 배포)가 아닌 이메일 승인을 선택한 이유:
팀이 작을수록 "지금 배포하면 되나?"의 판단이 중요하다. 인프라 배포와 달리 광고 송출 시스템은 캠페인 일정에 따라 배포 타이밍을 고려해야 하는 경우가 있다. 자동화하되 최종 트리거는 사람이 누르는 구조를 유지했다.

### Lambda로 트리거 서버를 대체

주 1~2회 트리거를 위해 상시 서버를 운영하는 건 낭비다. Lambda Function URL은 별도 서버 없이 HTTP 엔드포인트를 제공하고, 실행 시간만 과금된다. 배포 빈도가 낮은 프로젝트에 적합한 선택이다.

### Docker 태그에 빌드 번호 + 타임스탬프

롤백 시 이전 버전으로 돌아가려면 어떤 이미지가 어느 시점 것인지 식별할 수 있어야 한다. `{env}-{N}-{YYYYMMDDHH}` 형식은 순서(N)와 시각 둘 다 담는다. 별도 배포 이력 DB 없이 이미지 태그만으로 롤백 대상을 특정할 수 있다.

### HMAC 서명 URL

이메일 버튼에 단순 토큰을 쓰면 링크 재사용이 가능하다. HMAC-SHA256 서명 + 2시간 만료로 위변조와 재사용을 차단했다. Lambda에서 서명 검증만 하면 되어 DB 없이 stateless하게 구현할 수 있다.

### LLM 코드리뷰 자동화

코드리뷰는 CI/CD의 첫 번째 관문이다. 기존에는 팀원 리뷰에만 의존했는데, 3인 팀에서 모든 PR을 꼼꼼하게 보는 건 현실적으로 어렵다. PR 등록 시 Claude CLI가 diff를 분석해 1차 리뷰 결과를 이메일로 발송하고, 팀원은 LLM 의견을 참고해 최종 판단한다.

Anthropic API를 별도 계약하지 않고 기존 구독(Claude Team Plan)의 CLI를 활용했다. Claude CLI는 OAuth 인증 방식으로 Lambda 실행 불가 — Bitbucket Webhook 수신 겸 전용 서버를 별도 운영한다. diff 규모에 따라 전체 or 파일별 분할 처리로 컨텍스트 한계를 우회한다.

---

## 결과

| 항목 | 이전 | 이후 |
|------|------|------|
| 배포 소요 시간 (개발자 개입) | ~15분 | ~1분 (이메일 확인 + 버튼 클릭) |
| 환경 혼동 가능성 | 수동 스크립트 선택 의존 | 브랜치로 구조적 분리 |
| 롤백 | 버전 확인 후 수동 재배포 | 이메일 내 버튼 1회 클릭 |
| 배포 내역 공유 | 배포자가 직접 공유해야 함 | 이메일로 전원 자동 수신 |
| 코드리뷰 | PR당 팀원 리뷰에만 의존 | LLM 1차 리뷰 + 팀원 확인 |

---

## 기술 스택

| 영역 | 기술 |
|------|------|
| CI/CD | Bitbucket Pipelines |
| 컨테이너 | Docker, docker-compose |
| 서버리스 트리거 | AWS Lambda (Python 3.12, Function URL) |
| 이메일 | AWS SES (SMTP) |
| 보안 | HMAC-SHA256, Bearer Token |
| 테스트 | pytest, pytest-django |
| 모니터링 | AWS CloudWatch |
| 코드리뷰 | Claude CLI (Team Plan) |

---

## 파일 구조

```
smap-manager/
├── bitbucket-pipelines.yml     # 파이프라인 정의 (테스트·빌드·배포·롤백)
└── scripts/
    ├── lambda_handler.py       # 배포/롤백 트리거, HMAC 검증, 중복 방지
    └── notify.py               # 이메일 발송 (build/deploy/rollback × dev/prod)
```

---

## 환경변수

**Bitbucket Repository Variables**

| 변수 | 설명 |
|------|------|
| `BB_API_TOKEN` | Bitbucket Repository Access Token (Pipelines R/W) |
| `DEPLOY_HOST` / `PROD_DEPLOY_HOST` | Staging / Production 서버 호스트 |
| `DEPLOY_PATH` / `PROD_DEPLOY_PATH` | Staging / Production 배포 경로 |
| `DEPLOY_TOKEN_SECRET` | HMAC 서명 키 (Lambda와 동일값) |
| `LAMBDA_BASE_URL` | Lambda Function URL |
| `REGISTRY_USERNAME` / `REGISTRY_PASSWORD` | Docker 레지스트리 인증 |
| `SMTP_USERNAME` / `SMTP_PASSWORD` | AWS SES SMTP 인증 |

**Lambda Environment Variables**

| 변수 | 설명 |
|------|------|
| `DEPLOY_TOKEN_SECRET` | HMAC 서명 키 |
| `BB_API_TOKEN` | Bitbucket Repository Access Token |
| `BB_WORKSPACE` / `BB_REPO` | Bitbucket workspace / repo slug |
