# 📊 동영상 광고 & 매체별 인벤토리 관리 시스템

<div align="center">

[![Django](https://img.shields.io/badge/Django-5.1-092E20?style=flat&logo=django)](https://www.djangoproject.com/)
[![Python](https://img.shields.io/badge/Python-3.12-3776AB?style=flat&logo=python)](https://www.python.org/)
[![Docker](https://img.shields.io/badge/Docker-Ready-2496ED?style=flat&logo=docker)](https://www.docker.com/)
[![License](https://img.shields.io/badge/License-Proprietary-red.svg)](LICENSE)

</div>

## 👨‍💼 Project Leadership & Development

**Role**: Project Lead (PL) & Lead Developer  
**Responsibilities**: 
- 전체 시스템 아키텍처 설계 및 기술 스택 선정
- 핵심 비즈니스 로직 설계 및 구현 (코드 기여도 70%+)
- 팀 리딩 및 코드 리뷰

---

## 🎯 Project Overview

여러 매체에 동영상 광고 집행을 위한, 캠페인의 **기획 → 집행 → 모니터링** , 전체 생애주기를 관리하는 엔터프라이즈급 광고 운영 플랫폼

### Core Capabilities
- 📊 **Campaign Management** - 다중 플랫폼 캠페인 통합 관리 및 실시간 모니터링
- 📦 **Inventory System** - 일별/매체별 인벤토리 자동 분배 및 차감
- 🎯 **Targeting Engine** - 프로그램/채널/키워드 기반 정교한 타겟팅
- 📈 **Excel Integration** - 광고 제안서 자동 생성 (MediaMix, 타겟팅 시트)
- 🔄 **Purchase Workflow** - 트랜잭션 기반 안전한 광고 구매 프로세스

---

## 🛠 Tech Stack

### Backend
```
Django 5.1         - Modern Web Framework
Python 3.12        - Core Language
MySQL              - Primary Database
Redis              - Caching & Session Store
Gunicorn           - WSGI Application Server
```

### Frontend
```
JavaScript (ES6+)  - Client-side Logic
jQuery             - DOM Manipulation & AJAX
TailwindCSS        - Utility-first CSS Framework
```

### DevOps
```
Docker             - Containerization
Nginx              - Reverse Proxy & Static Files
Private Registry   - Container Image Management
Cron               - Scheduled Task Automation
```

### Key Libraries
```python
openpyxl           # Excel 처리 (복잡한 포맷팅 지원)
django-mysql       # MySQL 고급 기능
boto3              # AWS S3 Integration
mysqlclient        # Native MySQL Driver
django-cors-headers # CORS 처리
```

### Infra

```
AWS                - EC2, S3, Route53
```

---

## 🏗 System Architecture

### 1. Layered Architecture
```
┌─────────────────────────────────────┐
│         Presentation Layer          │
│   (Django Templates + JavaScript)   │
├─────────────────────────────────────┤
│             View Layer              │
│   - Dashboard, Campaign, Inventory  │
├─────────────────────────────────────┤
│            Service Layer            │
│   - CampaignPurchaseService         │
│   - CampaignUnitStatusAggregator    │
│   - ExcelService                    │
│   - MakeInventoryFromForecast       │
├─────────────────────────────────────┤
│          Model Layer (ORM)          │
│   - Campaign, CampaignUnit, Ad      │
│   - UnitInventoryData               │
├─────────────────────────────────────┤
│         Data Layer (MySQL)          │
└─────────────────────────────────────┘
```

### 2. Key Services

| Service | Size | Description |
|---------|------|-------------|
| `make_inventory_from_forecast_data.py` | 83KB | 매체별 일별 프로그램별 인벤토리 예측치, 판매가능 상품 인벤토리 변환 |
| `campaign_purchase_service.py` | 123KB | 캠페인 구매 및 차감 프로세스 전체 관리 |
| `excel_service.py` | 35KB | 복잡한 Excel 제안서 생성 |
| `campaign_unit_status_aggregator.py` | 8KB | 광고 상태 집계 및 동기화 |

---

## 💡 Technical Highlights

### 1. 복잡한 인벤토리 분배 알고리즘

**3단계 분배 시스템**
```python
구매금액 (100%)
  └─ 플랫폼별 분배 (균등 비율)
      ├─ Platform A: 33.33%
      ├─ Platform B: 33.33%
      └─ Platform C: 33.34% (나머지 할당)
          ├─ 타겟팅그룹별 분배 (가중치 비율)
          ├─ Group 1: 40%
          ├─ Group 2: 35%
          └─ Group 3: 25%
              └─ CO별 분배 (비율 설정)
              ├─ CO A: 60%
              └─ CO B: 40%
```

**핵심 로직**
- 사용자의 구매 인벤토리를 구매상품 특성에 따라 세부적으로 분리하여 정교한 인벤토리 관리 및 타겟팅 운영
- 나머지 처리를 통한 정확한 100% 분배
- 소수점 오차 방지 알고리즘

### 2. Excel 제안서 자동 생성 시스템

**복잡한 Excel 구조 처리**
```
MediaMix 시트 (16개 컬럼)
├─ 캠페인 정보 (병합 셀)
├─ 상품 정보 (다중 행)
├─ 자동 계산식
│   ├─ 예상노출수 = 구매금액 / ECPM * 1000
│   ├─ 예상클릭수 = 예상노출수 * CTR
│   └─ 예상조회수 = 예상노출수 * VTR
└─ TOTAL 합계 (수식 기반)

큐레이션타겟팅 시트
├─ 프로그램 타겟팅 (방송사별)
├─ 유튜브 채널 타겟팅
└─ 맞춤 키워드
```

**기술적 난이도**
- openpyxl을 활용한 병합 셀, 조건부 서식, 계산식 처리
- 동적 데이터 기반 자동 레이아웃 생성
- 전문 제안서 수준의 스타일링

### 3. 트랜잭션 기반 안전한 구매 프로세스

```python
@transaction.atomic
def process_campaign_purchase(self):
    """
    All-or-Nothing 보장
    - 인벤토리 차감 실패 → 전체 롤백
    - 광고 생성 실패 → 인벤토리 복구
    - 상세한 에러 로깅
    """
```

**안전장치**
- 인벤토리 선차감 후 검증
- Savepoint 활용한 부분 롤백
- 계층적 결과 출력으로 디버깅 용이

### 4. 인벤토리 데이터 변환 파이프라인

**외부 데이터 → 판매 인벤토리 자동 전환**

```
ForecastInventoryData (외부 제공)
  ↓
[process_standard_inventory_load]
  ├─ 파트별 타입 매핑 (SMR_PROGRAM → PROGRAM)
  ├─ meta_id별 그룹화 및 합산
  ├─ 변화율 자동 계산
  └─ Bulk Create/Update (2000건 청크)
  ↓
StandardInventoryData (표준 인벤토리 전환 -> 판매 전, 수동 보정 및 확인을 위한 중간데이터)
  ↓
[process_unit_inventory_load]
  ├─ Forecast 비율 기반 일별 분배
  ├─ CONTENTS/RANDOM/GROUP 타입별 처리
  ├─ 플랫폼별 인벤토리 생성
  └─ Raw SQL Upsert (최적화)
  ↓
UnitInventoryData (운영자가 시스템내에서 직접 만든 광고상품에 인벤토리가 부여됨)
```

**핵심 알고리즘**
```python
# 1. Forecast 비율 기반 일별 분배
- 외부 데이터에서 meta_id별, 일별 시간대별 합산
- 일별 비율 계산 (일별 인벤토리 / 총 인벤토리)
- 가중평균으로 정확한 분배 (소수점 오차 누적 방지)

# 2. 그룹 타겟팅 복합 비율 계산
- meta_id별 비중 * 일별 forecast 비율
- 정규화를 통한 합 = 1 보장
- 반올림 차이를 첫날에 보정
```

**성능 최적화**
- Django Management Command로 백그라운드 처리
- Raw SQL Upsert로 대량 데이터 처리
- 청크 단위 Iterator 사용 (메모리 효율)
- 상태 플래그로 중복 실행 방지

**처리 규모**
- Standard Inventory: ~수천 건/월
- Unit Inventory: ~수만 건/월 (일별 분배)
- 처리 시간: 평균 10-30초

---

## 📊 Database Design

### Core Tables
```sql
Campaign (캠페인)
  └─ CampaignUnit (상품별 유닛)
       ├─ Ad (플랫폼별 광고)
       │    └─ ChildAd (타겟팅그룹별 하위 광고)
       │
       └─ UnitInventoryData (인벤토리 정보)
            └─ UnitInventoryCampaignUnitData (차감 내역)
```

### Key Models (56개 Python 파일)

**Campaign Management**
- `campaign.py` - 캠페인 기본 정보
- `campaign_unit.py` - 상품별 실행 유닛
- `ad.py` - 플랫폼별 광고
- `child_ad.py` - 타겟팅그룹별 하위 광고

**Inventory System**
- `unit_inventory_data.py` - 유닛별 인벤토리
- `forecast_inventory_data.py` - 예측 인벤토리
- `unit_inventory_campaign_unit_data.py` - 차감 내역

**Targeting**
- `targeting_group.py` - 타겟팅 그룹
- `unit_platform_targeting_group.py` - 플랫폼별 타겟팅

### Index Strategy
```sql
-- 복합 인덱스 예시
CREATE INDEX idx_inventory_lookup 
ON unit_inventory_data(metaid, platform, date);

CREATE INDEX idx_campaign_unit_status 
ON campaign_unit(publish_request_status, updated_at);
```

---

## 🚀 Key Features Implementation

### 1. 캠페인 구매 플로우
```
사용자 입력
  ↓
[Dashboard View] - AJAX POST
  ↓
[CampaignPurchaseService]
  ├─ 인벤토리 검증
  ├─ 인벤토리 차감
  ├─ CampaignUnit 생성/수정
  ├─ Ad 생성 (플랫폼별)
  ├─ ChildAd 생성 (타겟팅그룹별)
  └─ 결과 계층적 출력
  ↓
JSON Response (성공/실패)
```

### 2. 인벤토리 변환 및 관리
```
[외부 업체 데이터 수신]
  ↓
ForecastInventoryData 저장
  ↓
[Management Command: process_standard_inventory_load]
  ├─ 파트별 타입 매핑 (6가지 → 4가지 타입)
  ├─ Meta_id별 그룹화 및 합산
  ├─ 변화율 계산 (prophet vs actual)
  └─ Bulk Upsert (2000건/청크)
  ↓
StandardInventoryData (표준 인벤토리)
  ↓
[Management Command: process_unit_inventory_load]
  ├─ Forecast 비율 계산 (일별 가중치)
  ├─ CONTENTS 타겟팅 인벤토리 생성
  ├─ RANDOM 인벤토리 집계 및 생성
  ├─ GROUP 타겟팅 복합 비율 계산
  └─ Raw SQL Upsert (일별 데이터)
  ↓
UnitInventoryData (판매 가능 인벤토리)
  ↓
[캠페인 구매 시]
  ├─ 일별/플랫폼별 잔여 확인
  ├─ 실시간 차감
  └─ UnitInventoryCampaignUnitData 기록
```

### 3. 상태 동기화
```
하위 광고 상태 변경
  ↓
[CampaignUnitStatusAggregator]
  ↓
우선순위 기반 상태 결정
  ↓
CampaignUnit 상태 업데이트
  ↓
대시보드 반영
```

---

## 📈 Performance Optimizations

### 1. Database Query Optimization
```python
# Before: N+1 쿼리 문제
for unit in units:
    unit.campaign  # 쿼리 발생
    
# After: select_related로 최적화
units = CampaignUnit.objects.select_related('campaign')
```

### 2. Inventory Calculation Optimization
```python
# Before: 2번의 쿼리 (월별 총량, 일별 데이터)
monthly_total = get_monthly_total()
daily_data = get_daily_data()

# After: 1번의 쿼리로 통합
data = get_data_with_grouping()  # 50% 응답 시간 단축
```

### 3. Memory Optimization
```python
# 딕셔너리 컴프리헨션으로 간소화
platform_map = {p.id: p.name for p in platforms}

# 제너레이터 패턴 활용
def iter_large_dataset():
    for batch in queryset.iterator(chunk_size=1000):
        yield process(batch)
```

---

## 🔐 Security & Reliability

### Security Measures
- ✅ SSH 포트 변경 (2222)
- ✅ Docker exec 차단으로 무단 접근 방지
- ✅ 환경변수 기반 설정 관리 (`.env`)
- ✅ CORS 정책 적용

### Reliability
- ✅ 트랜잭션 기반 데이터 일관성
- ✅ 상세한 에러 로깅 및 추적
- ✅ Rollback 메커니즘
- ✅ Health check 구현

---

## 🐳 Docker Deployment

### Multi-Stage Build
```dockerfile
# Python 3.12 Slim
FROM python:3.12-slim-bullseye

# Nginx + Gunicorn
# Cron for scheduled tasks
# Korean locale & timezone
```

### CI/CD Pipeline
```bash
# 1. Build & Push (AMD64 for production)
docker buildx build --platform linux/amd64 \
  -t registry.netinsight.co.kr/smap-manager:latest --push .

# 2. Deploy to staging
docker-compose pull
docker-compose down
docker-compose up -d

# 3. Health check
curl http://localhost/health
```

---

## 🎓 Technical Challenges Overcome

### 1. 복잡한 분배 알고리즘 설계
**문제**: 3단계 분배 시 소수점 오차로 인한 합계 불일치  
**해결**: `_distribute_with_remainder()` 함수로 나머지를 마지막 항목에 할당

### 2. 대용량 Excel 처리
**문제**: 16개 컬럼, 다중 시트, 복잡한 포맷팅  
**해결**: openpyxl의 고급 기능 활용 + 메모리 최적화

### 3. 트랜잭션 롤백 시나리오
**문제**: 부분 실패 시 데이터 정합성 문제  
**해결**: Django의 `transaction.atomic`과 Savepoint 활용

### 4. 실시간 상태 동기화
**문제**: 수천 개 광고의 상태를 실시간 집계  
**해결**: 우선순위 기반 알고리즘 + 배치 처리 조합

### 5. 외부 데이터의 판매 인벤토리 변환
**문제**: 
- 외부 업체에서 받은 예측 데이터(Forecast)를 일별 판매 인벤토리로 자동 변환
- 6가지 파트 타입을 4가지 플랫폼 타입으로 매핑 및 합산
- meta_id별 일별 비율 계산 시 소수점 오차 누적
- 대용량 데이터(수만 건) 처리 시 성능 및 메모리 문제

**해결**: 
- **2단계 변환 파이프라인 구축**
  - Step 1: Forecast → Standard (파트 매핑, 그룹화, 변화율 계산)
  - Step 2: Standard → Unit (일별 분배, 타입별 처리, Raw SQL Upsert)
- **정교한 비율 계산 알고리즘**
  - 일별 forecast 비율을 가중평균으로 계산
  - 누적 오차를 다음 날짜에 반영하여 정확도 향상
  - 최종 나머지는 비율이 가장 높은 날에 할당
- **성능 최적화**
  - Django ORM Bulk Create/Update (1000건 배치)
  - Raw SQL INSERT ON DUPLICATE KEY UPDATE
  - Iterator를 통한 청크 단위 처리 (2000건/청크)
  - 상태 플래그로 동시 실행 방지

**성과**: 수만 건의 데이터를 10-30초 내 안정적으로 처리

---

## 🎯 Skills Demonstrated

### Backend Development
- ✅ Django ORM 고급 활용 (select_related, prefetch_related)
- ✅ 복잡한 비즈니스 로직 설계 및 구현
- ✅ Service Layer Pattern 적용
- ✅ 트랜잭션 처리 및 에러 핸들링

### System Design
- ✅ Layered Architecture 설계
- ✅ Database 스키마 설계 및 최적화
- ✅ 복잡한 알고리즘 구현 (분배, 집계, 비율 계산)
- ✅ 확장 가능한 구조 설계

### Data Pipeline & ETL
- ✅ 외부 데이터 자동 변환 파이프라인 구축
- ✅ Django Management Command 활용
- ✅ 대용량 데이터 Bulk 처리 (Raw SQL Upsert)
- ✅ 청크 단위 Iterator 패턴으로 메모리 최적화
- ✅ 정교한 비율 계산 알고리즘 (가중평균, 오차 누적 방지)

### DevOps
- ✅ Docker 컨테이너화
- ✅ CI/CD 파이프라인 구축
- ✅ Nginx + Gunicorn 설정
- ✅ Private Registry 운영

### Code Quality
- ✅ DRY 원칙 적용
- ✅ 관심사 분리 (Separation of Concerns)
- ✅ 재사용 가능한 컴포넌트 설계
- ✅ 상세한 로깅 및 디버깅 시스템

---

## 📚 Key Learnings

1. **Enterprise-level 시스템 설계 경험**
   - 복잡한 도메인 로직을 명확한 레이어로 분리
   - 확장성과 유지보수성을 고려한 아키텍처

2. **Performance Optimization**
   - Database 쿼리 최적화로 50% 성능 향상
   - 메모리 효율적인 데이터 처리
   - Raw SQL과 ORM의 적절한 조합

3. **Data Pipeline Engineering**
   - 외부 시스템 데이터의 안정적인 변환 및 적재
   - 수학적 정확도를 요구하는 비율 계산 알고리즘
   - 대용량 배치 처리 최적화 기법

4. **Team Leadership**
   - 코드 리뷰 및 기술 가이드
   - 개발 표준 및 컨벤션 수립

5. **Problem Solving**
   - 복잡한 비즈니스 요구사항의 기술적 구현
   - 다양한 엣지 케이스 처리

---

## 📝 Notes

- 이 프로젝트는 실제 운영 중인 상용 시스템입니다
- 민감한 정보는 제외하고 기술적 내용만 공개합니다

---

