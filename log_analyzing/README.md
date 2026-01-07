# 광고 로그 데이터 분석 시스템

## 📊 프로젝트 개요

광고 로그 데이터를 수집, 가공, 분석하여 다차원 리포트를 생성하는 데이터 파이프라인 시스템입니다. AWS S3에 저장된 로그 파일을 처리하고, 광고주, 매체사, 광고 단위별 성과 지표를 집계하여 MySQL 데이터베이스에 저장합니다.

## 🎯 주요 기능

### 1. 자동화된 데이터 파이프라인
- AWS S3에서 압축된 pickle 파일 자동 수집
- 멀티프로세싱 기반 병렬 데이터 처리
- 프로세스 락을 통한 중복 실행 방지

### 2. 대용량 데이터 처리
- **Pandas DataFrame** 기반 효율적인 데이터 변환
- 배치 단위 파일 그룹핑으로 메모리 최적화
- Bulk Upsert를 통한 고속 데이터베이스 저장

### 3. 다차원 리포트 생성

분석 시스템은 다음 4가지 주요 리포트를 생성합니다:

#### 📈 광고 성과 리포트
- 광고 단위별 노출, 클릭, 수익 데이터
- 조직, 매체, 인벤토리, 광고, 디맨드 차원 분석

#### 🔍 광고 요청 리포트
- 디바이스(PC/MOBILE), 앱 타입(WEB/IOS/AOS)별 세분화
- 요청(xreq), 응답(xres), 패스백(xpsb) 지표 추적

#### 💰 디맨드 리포트
- 디맨드 파트너별 성과 집계
- CPC, CTR, eCPM 등 핵심 지표 계산

#### 📱 매체 리포트
- 매체사별, 디바이스별 성과 분석
- 수익 분배 추적 (총 수익, 대행사 수익, 매체 수익)

## 🛠 기술 스택

### Backend & Data Processing
- **Python 3.x** - 메인 프로그래밍 언어
- **Pandas** - 데이터 처리 및 변환
- **Pony ORM** - 데이터베이스 ORM

### Cloud & Database
- **AWS S3** - 로그 데이터 저장소
- **Boto3** - AWS SDK
- **MySQL (PyMySQL)** - 리포트 데이터 저장

### Testing & Quality
- **pytest** - 테스트 프레임워크
- **moto** - AWS 서비스 모킹 라이브러리
- **parametrize** - 데이터 중심 테스트 케이스 자동화

## 📂 프로젝트 구조

```
xc2-analyzer/
├── nitroapp/
│   ├── __init__.py      # 메인 분석 로직 및 CLI 커맨드
│   ├── aws.py           # AWS S3 연동 매니저
│   ├── logs.py          # 로그 처리 유틸리티
│   └── models.py        # 데이터베이스 모델
├── tests/               # 테스트 코드
│   ├── test_analyzer.py
│   ├── test_aws.py
│   └── conftest.py      # pytest fixture
├── nitrocfg.py          # 설정 파일
└── setup.py             # 패키지 설정
```

## 🔧 핵심 구현 기능

### 1. 분석 파이프라인

```python
def _analyzer(notify, debug):
    # 프로세스 락으로 중복 실행 방지
    # AWS S3에서 미분석 파일 목록 조회
    # 배치 단위로 파일 그룹핑
    # 각 그룹별 데이터 처리 및 집계
    # 리포트 테이블별 Bulk Upsert 실행
```

**주요 처리 단계:**
1. 데이터베이스에서 미분석 파일 경로 조회
2. 10개 단위로 파일 그룹핑
3. S3에서 pickle 파일 다운로드
4. Pandas DataFrame으로 변환 및 전처리
5. 데이터베이스 메타데이터 조인
6. 리포트별 피벗 및 집계
7. MySQL Bulk Upsert 실행

### 2. 데이터 변환 (dataframe_combine_dbdata)

**핵심 변환 로직:**
- JSON 형식의 광고 정보 파싱 (aid, _god, did 추출)
- 조직, 매체, 인벤토리 정보 매핑
- 디바이스 타입 분류 (PC/MOBILE/ALL)
- 앱 타입 분류 (WEB/IOS/AOS/ALL)
- Unix timestamp → KST 날짜 변환
- 타입 최적화 (int32, float32)

```python
# JSON 광고 정보 추출
df['aid'] = df['ads'].apply(lambda x: x[0]['aid'] if x else None)
df['_god'] = df['ads'].apply(lambda x: x[0]['_god'] if x else None)
df['did'] = df['ads'].apply(lambda x: x[0]['did'] if x else None)

# 데이터베이스 정보 매핑
df['orgid'] = df['mid'].map(organization_id_by_media_key)
df['adtype'] = df['aid'].map(adtype_by_inventory_key)
df['device'] = df['aid'].map(device_by_inventory_key)

# Unix timestamp → KST 변환
df['tm'] = pd.to_datetime(df['ts'], unit='s')
df['tm'] = df['tm'].dt.tz_localize('UTC').dt.tz_convert('Asia/Seoul').dt.strftime('%Y-%m-%d')

# 메모리 최적화를 위한 타입 변환
df['orgid'] = df['orgid'].fillna(0).astype(int)
df['aid'] = df['aid'].fillna(0).astype(int)
```

### 3. Bulk Upsert 최적화 (execute_bulk_upsert)

```sql
INSERT INTO [table] (columns...)
VALUES (values...)
ON DUPLICATE KEY UPDATE 
    col1=col1+VALUES(col1),
    col2=col2+VALUES(col2), ...
```

- 고유 키 충돌 시 자동 증분 업데이트
- 대량 데이터 한 번에 처리 (executemany)
- 네트워크 왕복 최소화

### 4. 장애 복구 및 재처리 로직

**대량 로그 처리 시 안정성 확보:**

```python
def process_with_retry(file_paths, max_retries=3):
    """
    파일 처리 실패 시 재시도 로직
    - S3 다운로드 실패: 지수 백오프로 재시도
    - 데이터 파싱 오류: 해당 파일 스킵 후 별도 로깅
    - DB 연결 오류: 트랜잭션 롤백 및 재연결
    """
    for attempt in range(max_retries):
        try:
            # 파일 처리 로직
            process_files(file_paths)
            break
        except S3DownloadError as e:
            if attempt < max_retries - 1:
                time.sleep(2 ** attempt)  # 지수 백오프
                continue
            else:
                log_failed_files(file_paths, e)
        except DataParsingError as e:
            # 손상된 파일은 격리하고 나머지 처리 계속
            quarantine_file(file_path)
            continue
        except DBConnectionError as e:
            db.rollback()
            db.reconnect()
```

**예외 처리 전략:**
- **S3 접근 오류**: 일시적 네트워크 장애 대응을 위한 지수 백오프 재시도
- **데이터 파싱 실패**: 손상된 파일 격리 후 나머지 파일 처리 계속
- **데이터베이스 오류**: 트랜잭션 롤백으로 데이터 일관성 보장
- **실패 파일 추적**: 별도 테이블에 실패 이력 기록 후 수동 재처리 지원

## 🧪 테스트 전략

### pytest 기반 포괄적 테스트 설계

**1. AWS S3 모킹 (moto 활용)**

```python
import pytest
from moto import mock_s3
import boto3

@pytest.fixture
def mock_s3_bucket():
    """S3 버킷 모킹 fixture"""
    with mock_s3():
        s3 = boto3.client('s3', region_name='ap-northeast-2')
        s3.create_bucket(
            Bucket='test-bucket',
            CreateBucketConfiguration={'LocationConstraint': 'ap-northeast-2'}
        )
        yield s3

def test_s3_file_download(mock_s3_bucket):
    """S3 파일 다운로드 로직 테스트"""
    # 테스트 데이터 업로드
    mock_s3_bucket.put_object(
        Bucket='test-bucket',
        Key='logs/2024/01/test.pkl.gz',
        Body=pickle.dumps(test_data)
    )
    
    # 프로덕트 코드 그대로 실행
    result = download_from_s3('logs/2024/01/test.pkl.gz')
    assert result is not None
```

**2. parametrize를 활용한 다양한 시나리오 테스트**

```python
@pytest.mark.parametrize("device,expected", [
    ("pc", "PC"),
    ("mobile", "MOBILE"),
    ("tablet", "MOBILE"),
    (None, "ALL"),
])
def test_device_classification(device, expected):
    """디바이스 타입 분류 로직 검증"""
    result = classify_device(device)
    assert result == expected

@pytest.mark.parametrize("ads_data,expected_aid", [
    ([{"aid": 123, "_god": 1}], 123),
    ([], None),
    ([{"aid": 456}], 456),
    (None, None),
])
def test_extract_ad_id(ads_data, expected_aid):
    """광고 ID 추출 로직 검증"""
    df = pd.DataFrame({'ads': [ads_data]})
    df['aid'] = df['ads'].apply(lambda x: x[0]['aid'] if x else None)
    assert df['aid'].iloc[0] == expected_aid
```

**3. 프로덕트 코드 최대한 유지한 통합 테스트**

```python
@pytest.fixture
def real_analyzer_config():
    """실제 설정 그대로 사용"""
    return {
        'batch_size': 10,
        'max_retries': 3,
        'timeout': 30
    }

def test_full_pipeline_integration(mock_s3_bucket, real_analyzer_config):
    """
    전체 파이프라인 통합 테스트
    - 프로덕트 코드 수정 없이 의존성만 모킹
    - 실제 실행 흐름 그대로 검증
    """
    # Given: S3에 테스트 로그 파일 준비
    setup_test_log_files(mock_s3_bucket)
    
    # When: 실제 analyzer 함수 실행
    result = _analyzer(notify=False, debug=True)
    
    # Then: 예상되는 리포트 생성 확인
    assert result['processed_files'] == 10
    assert result['reports_generated'] == 4
```

**테스트 커버리지:**
- 단위 테스트: 데이터 변환, 집계 로직, 유틸리티 함수
- 통합 테스트: 전체 파이프라인 플로우
- 예외 케이스: 네트워크 오류, 데이터 손상, DB 장애 시나리오

## 📊 처리 성능

- **배치 크기**: 파일 10개 단위 그룹핑
- **동시성 제어**: 프로세스 락 기반
- **메모리 최적화**: 청크 단위 처리로 OOM 방지
- **데이터베이스 최적화**: Bulk Upsert로 쓰기 성능 향상

## 🔐 보안 고려사항

- 데이터베이스 자격증명 환경변수 관리 권장
- AWS IAM Role 기반 인증 사용 권장
- 프로덕션 환경에서 하드코딩된 자격증명 제거 필요

## 📈 향후 개선 계획

- [ ] Sentry 에러 트래킹 통합
- [ ] SQS 기반 비동기 로그 처리
- [ ] 멀티프로세싱 풀을 활용한 병렬 처리 강화
- [ ] 리포트 생성 완료 후 알림 시스템
- [ ] 데이터 검증 레이어 추가

## 🏆 기술적 하이라이트

### 1. 데이터 처리 최적화
- Pandas의 벡터화 연산 활용으로 처리 속도 향상
- 메모리 효율적인 데이터 타입 변환 (int32, float32)
- 청크 기반 스트리밍 처리

### 2. 데이터베이스 성능
- Bulk Insert/Update로 네트워크 오버헤드 최소화
- 복합 인덱스를 활용한 효율적인 Upsert
- 트랜잭션 단위 배치 커밋

### 3. 시스템 안정성
- 프로세스 락(fcntl)을 통한 동시성 제어
- Context Manager 패턴으로 리소스 안전 관리
- 지수 백오프 기반 재시도 메커니즘으로 일시적 장애 대응
- 트랜잭션 롤백을 통한 데이터 일관성 보장
- 실패 파일 추적 및 격리 시스템

### 4. 확장성
- 모듈화된 리포트 생성 구조
- 설정 기반 테이블 매핑
- 새로운 리포트 타입 추가 용이

### 5. 테스트 가능성
- moto를 활용한 AWS 서비스 모킹으로 외부 의존성 제거
- parametrize로 다양한 엣지 케이스 자동 검증
- 프로덕트 코드 수정 없이 테스트 가능한 의존성 주입 설계
