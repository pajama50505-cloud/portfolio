# 📊 예측 수요 풀 기반 판매 데이터 자동화 관리 시스템

<div align="center">

[![Django](https://img.shields.io/badge/Django-5.1-092E20?style=flat&logo=django)](https://www.djangoproject.com/)
[![Python](https://img.shields.io/badge/Python-3.12-3776AB?style=flat&logo=python)](https://www.python.org/)
[![Docker](https://img.shields.io/badge/Docker-Ready-2496ED?style=flat&logo=docker)](https://www.docker.com/)
[![License](https://img.shields.io/badge/License-Proprietary-red.svg)](LICENSE)

</div>

## 👨‍💼 Project Leadership & Development

**Role**: Project Lead (PL) & Lead Developer  
**Responsibilities**:
```
- 전체 시스템 아키텍처 설계 및 기술 스택 선정
- 핵심 비즈니스 로직 설계 및 구현 (코드 기여도 70%+)
   ㄴ 매체별 예측 인벤토리 관리 및 판매인벤토리 전환
   ㄴ 캠페인 구매 및 실제 광고 운영 데이터 세팅
   ㄴ 외부연동 API 개발
- 팀 리딩 및 코드 리뷰
```
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
│   - 구매 서비스                        │
│   - 상태관리 서비스                     │
│   - 엑셀 관리 서비스                    │
│   - 구매 인벤토리 관리 서비스             │
├─────────────────────────────────────┤
│          Model Layer (ORM)          │
├─────────────────────────────────────┤
│         Data Layer (MySQL)          │
└─────────────────────────────────────┘
```

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

**안전장치**
- 인벤토리 선차감 후 검증
- Savepoint 활용한 부분 롤백
- 계층적 결과 출력으로 디버깅 용이

### 4. 인벤토리 데이터 변환 파이프라인

**외부 데이터 → 판매 인벤토리 자동 전환**

```
예측 인벤토리 데이터
  ↓
  ├─ 파트별 타입 매핑
  ├─ meta_id별 그룹화 및 합산
  ├─ 변화율 자동 계산
  └─ Bulk Create/Update (2000건 청크)
  ↓
표준 인벤토리 전환 -> 판매 전, 수동 보정 및 확인을 위한 중간데이터
  ↓
  ├─ Forecast 비율 기반 일별 분배
  ├─ 타입별 처리
  ├─ 플랫폼별 인벤토리 생성
  └─ Raw SQL Upsert (최적화)
  ↓
판매 가능한 인벤토리
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

**UI 구현**
<img width="2312" height="1204" alt="image" src="https://github.com/user-attachments/assets/6878f4fc-20ed-4dcc-ab5a-4ced32baff9c" />
<img width="2305" height="1180" alt="image" src="https://github.com/user-attachments/assets/3cb85585-80ec-4ee8-80b3-5e2ea11c3cef" />

---

## 🎓 Technical Challenges Overcome

### 1. 복잡한 분배 알고리즘 설계
**문제**: 3단계 분배 시 소수점 오차로 인한 합계 불일치  
**해결**: 소수점 나머지를 가장 점수가 높은 항목에 할당

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

