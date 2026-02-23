# README

# 🖼️ 이미지 유사도 검색 PoC

## 대량의 이미지 색인 데이터를 기반, 유사한 이미지를 검색하는 서비스

## 📌 목차

1. [시작하기](about:blank#-%EC%8B%9C%EC%9E%91%ED%95%98%EA%B8%B0)
2. [프로젝트 개요](about:blank#-%ED%94%84%EB%A1%9C%EC%A0%9D%ED%8A%B8-%EA%B0%9C%EC%9A%94)
3. [시스템 아키텍처](about:blank#-%EC%8B%9C%EC%8A%A4%ED%85%9C-%EC%95%84%ED%82%A4%ED%85%8D%EC%B2%98)
4. [기술적 의사결정](about:blank#-%EA%B8%B0%EC%88%A0%EC%A0%81-%EC%9D%98%EC%82%AC%EA%B2%B0%EC%A0%95)
5. [검색 알고리즘 진화 과정](about:blank#-%EA%B2%80%EC%83%89-%EC%95%8C%EA%B3%A0%EB%A6%AC%EC%A6%98-%EC%A7%84%ED%99%94-%EA%B3%BC%EC%A0%95)
6. [색인 파이프라인](about:blank#-%EC%83%89%EC%9D%B8-%ED%8C%8C%EC%9D%B4%ED%94%84%EB%9D%BC%EC%9D%B8)
7. [API 안정성 (Circuit Breaker)](about:blank#-api-%EC%95%88%EC%A0%95%EC%84%B1-circuit-breaker)
8. [성능 최적화 및 모니터링](about:blank#-%EC%84%B1%EB%8A%A5-%EC%B5%9C%EC%A0%81%ED%99%94-%EB%B0%8F-%EB%AA%A8%EB%8B%88%ED%84%B0%EB%A7%81)
9. [어려웠던 부분 및 개선 방향](about:blank#-%EC%96%B4%EB%A0%A4%EC%9B%A0%EB%8D%98-%EB%B6%80%EB%B6%84-%EB%B0%8F-%EA%B0%9C%EC%84%A0-%EB%B0%A9%ED%96%A5)
10. [참고 레퍼런스](about:blank#-%EC%B0%B8%EA%B3%A0-%EB%A0%88%ED%8D%BC%EB%9F%B0%EC%8A%A4)

---

## 🚀 시작하기

### 실행 방법

```bash
# 1. 프로젝트 클론
git clone <repository-url>
cd vector-vision-search

# 2. Docker Compose로 전체 서비스 실행
docker-compose up --build

# 3. 검색 웹 페이지 접속
http://localhost:58080/web

# 4. API 문서 확인 Swagger UI
http://localhost:58080/docs
```

### 동작 방식

1. `docker-compose up --build` 실행
2. 컨테이너 시작 시 Parquet 파일 자동 감지 (entrypoint.sh)
    - 미리 세팅된 Parquet 파일 사용하여 Elasticsearch에 색인
    - 파일이 없으면 색인 프로세스를 실행하여 Elasticsearch에 색인 (+Parquet 파일 생성)
3. Elasticsearch에 벡터 색인 (약 10초)
4. 웹 검색 즉시 가능

### 데이터 파일

| 항목 | 값 |
| --- | --- |
| 갯수 | 약 10,000 여개 |
| 파일 크기 | 약 1.2MB |

DINOv2 모델로 추출한 384차원 특징 벡터를 Parquet 파일로 백업하고, 재색인 시 Parquet 파일을 읽어, feature 추출 과정 없이 Elasticsearch에 색인합니다.

| 항목 | 값 |
| --- | --- |
| 갯수 | 약 10,000 여개 |
| 파일 크기 | 약 17MB |

### 재색인 (필요시)

```bash
# 전체 파이프라인 실행 (CSV → 특징 추출 → ES 색인)
docker exec -it vector-vision-search-app python indexing/image_pipeline.py --save-to-file parquet

# Parquet 파일에서 ES로 직접 로드
docker exec -it vector-vision-search-app python indexing/load_indexed.py
```

**왜 Parquet인가?** : 384차원의 고차원 벡터 데이터를 텍스트(JSON)로 저장할 경우 파일 크기가 비대해지고 I/O 속도가 저하됩니다. 열 지향 저장 방식인 Parquet을 사용하여 데이터 크기를 약 60% 압축하고, 색인 파이프라인의 로딩 속도를 비약적으로 향상시켰습니다. 백업파일로 사용하여, 저장공간을 절약하고 재색인이 필요한 상황에서 빠르게 색인이 가능합니다.

---

## 📋 프로젝트 개요

### 주요 기능

- **이미지 유사도 검색**: 검색 이미지와 유사한 상품을 코사인 유사도 기반으로 검색
- **간단한 웹페이지 구성**: 필터링, 집계 정보 등과 함께, 검색 이미지와 유사한 이미지를 리스팅
- **캐싱**: Redis 기반 1분 TTL 캐싱
- **장애 대응**: Circuit Breaker 패턴 적용
- **이미지 색인 파이프라인**: 대량의 이미지 특징 벡터 추출 후 Elasticsearch에 색인

### 기술 스택

| 구분 | 기술 |
| --- | --- |
| **Backend** | FastAPI 0.109.0 (Python 3.11) |
| **Search Engine** | Elasticsearch 9.2.1 (kNN + int8_hnsw) |
| **Cache** | Redis 5.0.1 |
| **ML Model** | DINOv2 (transformers 4.57.3) |
| **Frontend** | Vanilla JS + CSS |
| **Infra** | Docker Compose |

---

## 🏗 시스템 아키텍처

### Layered Architecture

```
┌─────────────────────────────────────────┐
│        Presentation Layer (Web)         │
└─────────────────────────────────────────┘
                   ↓
┌─────────────────────────────────────────┐
│           API Layer (FastAPI)           │
└─────────────────────────────────────────┘
                   ↓
┌─────────────────────────────────────────┐
│          Service Layer (Python)         │
└─────────────────────────────────────────┘
                   ↓
┌─────────────────────────────────────────┐
│            Core Layer (Python)          │
└─────────────────────────────────────────┘
```

### 아키텍처 선택 이유

1. **명확한 책임 분리**: 각 레이어가 독립적인 역할 수행
2. **테스트 용이성**: 레이어별 단위 테스트 및 모킹 가능
3. **유지보수성**: 변경 영향 범위 최소화
4. **확장성**: 새로운 기능 추가 시 해당 레이어만 수정

## 

## 🧠 기술적 의사결정

### DINOv2 + Elasticsearch int8_hnsw 선택 이유

정확도와 속도 사이의 최적의 균형점을 찾기 위해 깊이 고민했습니다. 서비스의 특성상 ’검색 정확도’가 사용자 만족도와 직결된다고 판단했습니다.

이에 따라 모델 선정 시 ’정확도’에 가장 큰 가치를 두었습니다. 연산량이 다소 무겁더라도 정밀한 시각적 특징 추출이 가능한 DINOv2 모델을 채택하였고, 이 모델의 성능을 뒷받침할 수 있도록 검색 속도 최적화에 집중했습니다.

한편, 원본 float32 벡터를 그대로 사용하면 정밀도는 높지만 메모리 비용이 커지는 문제가 있었습니다. 이를 해결하기 위해 해싱은 필수적이라고 생각했고, 검색 속도와 정확성을 모두 만족시키는 Elasticsearch의 양자화(Quantization) 기술을 도입했습니다.

다양한 방식 중 Binary(1-bit)보다 정확도가 높은 int8 양자화를 선택하여, 정보 손실을 최소화하면서도 메모리 점유율을 75% 절감했습니다. 결과적으로 고성능 DINOv2와 효율적인 int8_hnsw 방식을 조합하여, 비용 효율성과 검색 품질이라는 두 마리 토끼를 모두 잡을 수 있었습니다.

### DINOv2

**1. 객체 중심적 특징 추출 (Object-Centric)**
- 텍스트-이미지 연관성(CLIP)을 넘어, 이미지 자체의 기하학적 구조와 디테일을 이해합니다.
- 운동화 밑창 패턴, 셔츠 칼라 형태 등 미세한 차이를 정교하게 벡터화합니다.

**2. 강력한 Zero-shot 성능 (범용성)**
- 패션, 가전, 가구 등 방대한 카테고리에 대해 별도의 추가 학습(Fine-tuning) 없이도 즉시 높은 수준의 유사도 계산이 가능합니다.

**3. 배경 분리 및 상품 집중 (Focusing)**
- 스튜디오 소품이나 배경 노이즈를 배제하고 실제 상품 객체에만 집중하여 임베딩을 생성하므로 검색 정확도가 대폭 향상됩니다.

### Elasticsearch int8_hnsw

**1. 공간 효율적 벡터 해싱 (int8 Quantization)**
- 고차원 float32 데이터를 256개의 버킷으로 매핑하는 8-bit 해싱 기법을 적용하여 메모리 점유율을 75% 절감했습니다.
- 이는 단순한 압축을 넘어, 대규모 인덱스를 RAM 내에 유지할 수 있게 함으로써 디스크 I/O 병목 현상을 원천 차단합니다.

**2. 해시 기반 그래프 탐색 (HNSW)**
- 양자화된 해시값을 활용해 수백만 개의 노드 사이를 빠르게 이동하는 계층적 그래프 알고리즘을 사용합니다.
- 숫자 간 연산이 단순화되어 CPU의 연산 처리량을 높이고, 검색 응답 시간(Latency)을 밀리초(ms) 단위로 단축했습니다.

**3. 해시 오차 보완을 위한 재랭킹 (Rescoring)**
- **1단계**: 양자화된 해시 데이터로 빠르게 후보군을 추리고(Filtering)
- **2단계**: 상위 후보에 대해서만 원본 벡터로 정밀 거리 재계산을 수행하여 해싱 과정에서 발생하는 정보 손실을 완벽히 보완했습니다.

### 최종 선택

- **DINOv2**: Vision Transformer 기반, 이미지 유사도 검색에 최적화
- **int8 quantization**: 벡터 크기 75% 절감
- **HNSW 알고리즘**: 실시간 검색 보장

---

## 🔄 검색 알고리즘 진화 과정

### 1차: ResNet50 + LSH 해싱

### LSH 해싱값을 ES에 저장하고, 해싱값끼리만 비교

```
이미지 → ResNet50 (CNN) → 2048차원 벡터 → LSH 해싱 → 10개 해시값 → ES 저장
```

**문제점**: 해시 충돌로 인한 낮은 정확도, 유사하지 않은 상품이 검색됨

### 2차: LSH + 코사인 유사도 재정렬 (Rescoring)

### LSH 해싱값과 벡터 원본을 ES에 함께 저장, 해싱값끼리 비교하여 후보군을 추린 후, 후보군에 대해서는 원본 벡터로 해밍 거리 재계산하여 랭킹 재정렬

```
1단계: LSH 해시로 후보군 선정 (1000개)
2단계: 원본 벡터로 코사인 유사도 계산 → 재정렬
```

**문제점**: 2단계 로직으로 시스템 복잡도 증가, 여전히 모델 한계 존재

### 3차: ResNet50 + HNSW (kNN)

### ResNet50으로 추출된 벡터를 ES에 저장하고, HNSW 알고리즘을 사용하여 kNN 검색

```
이미지 → ResNet50 → 2048차원 벡터 → ES kNN 검색 (HNSW 인덱스)
```

**문제점**: 고차원 벡터로 인한 메모리 폭증, 검색 속도 저하

### 4차: DINOv2 + int8_hnsw (최종 선택)

### DINOv2로 추출된 벡터를 ES에 int8_hnsw 인덱스 + 벡터로 저장하고, HNSW 알고리즘을 사용하여 kNN 검색

```
이미지 → DINOv2 (ViT) → 384차원 벡터 → ES int8_hnsw → kNN 검색
```

### 모델 및 알고리즘 비교

| 비교 항목 | ResNet50 + LSH | LSH + Rescoring | ResNet50 + KNN | **DINOv2 + int8_hnsw** |
| --- | --- | --- | --- | --- |
| 핵심 기술 | 고정 해싱 | 해싱 + 재정렬 | HNSW | ViT + int8 양자화 |
| 추출 모델 | ResNet50 | ResNet50 | ResNet50 | **DINOv2** |
| 색인 속도 | ~10s | ~12s | ~14s | ~34s |
| 검색 정확도 | 최하 | 낮음 | 우수 | **최상** |
| 메모리 효율 | 좋음 | 좋음 | 나쁨 | **우수** |
| 시스템 복잡도 | 낮음 | 높음 | 낮음 | **매우 낮음** |

> 💡 핵심 인사이트: 디스크는 저렴하지만 고사양 RAM은 비쌉니다. int8 양자화로 TCO(총 소유 비용)를 크게 절감할 수 있습니다.
> 

**결과**:
- ✅ 정확도 향상 (Vision Transformer의 맥락/질감 파악 능력)
- ✅ 메모리 75% 절감 (int8 양자화)
- ✅ 실시간 검색 (HNSW 알고리즘)

---

## 💿 색인 파이프라인

### 파이프라인 단계별 상세 프로세스

### 1단계: CSV 검증 및 데이터 준비 (`validate_and_prepare`)

```python
# products.csv 로드 → 데이터 검증 → Redis PENDING 상태 저장
```

**주요 동작:**
- `products.csv` 파일을 읽어 각 상품 데이터를 검증합니다
- 필수 필드 체크
- 검증 통과 시 Redis에 `PENDING` 상태로 저장
- 이미 처리 완료(`COMPLETE`)되었거나 처리 중(`PROCESSING`)인 항목은 스킵
- 검증 실패 시 `FAILED` 상태로 저장하여 추후 확인 가능

**중단 및 재시작 안전성:**
- Redis 기반 상태 관리로 프로세스 중단 시에도 진행 상황 보존
- 재시작 시 이미 검증된 항목은 건너뛰고 이어서 처리

### 2단계: 이미지 특징 추출 (`extract_features`)

```python
# PENDING 상품 조회 → 배치 단위로 이미지 다운로드 → DINOv2 특징 추출 → COMPLETE 상태 저장
```

**주요 동작:**
- Redis에서 `PENDING` 상태의 상품들을 조회
- 설정된 배치 크기(기본 64개)만큼 묶어서 처리
- 각 배치에 대해:
1. 상태를 `PROCESSING`으로 변경
2. 이미지 URL에서 이미지 다운로드 (병렬 처리)
3. DINOv2 모델로 384차원 벡터 추출
4. 성공 시 `COMPLETE` 상태로 변경하고 벡터 데이터 저장
5. 실패 시 `FAILED` 상태로 변경하고 에러 정보 기록

**성능 최적화:**
- **배치 추론**: 이미지를 배치로 묶어서 CPU 활용도 극대화
- **병렬 다운로드**: 16개 워커 스레드로 이미지 동시 다운로드
- **큐 기반 처리**: 다운로드 완료된 이미지부터 즉시 추론 시작

### 3단계: Elasticsearch 색인 (`index_to_elasticsearch`)

```python
# COMPLETE 상품 조회 → Bulk API로 ES 색인 → Parquet 파일 저장 (옵션)
```

**주요 동작:**
- Redis에서 `COMPLETE` 상태의 상품들을 조회
- Elasticsearch Bulk API용 액션 생성:
- 기본 필드: `id`, `name`, …
- 벡터 필드: `image_features` (384차원)
- 모든 데이터를 한 번에 Bulk API로 색인 (성능 최적화)
- 색인 성공 시 Redis에서 해당 상품 데이터 삭제 (메모리 절약)

**백업 파일 저장 (옵션):**
- **Parquet 형식** (권장): 압축률 높고 빠른 I/O, 재색인 시 유용
- **JSON 형식**: 사람이 읽기 쉬운 형태
- 백업 파일이 있으면 전체 파이프라인 재실행 없이 빠르게 재색인 가능

**명령어 예시:**

```bash
# 전체 파이프라인 실행 (Parquet 백업 포함)
docker exec -it vector-vision-search-app python indexing/image_pipeline.py --save-to-file parquet

# 백업 파일에서 직접 색인 (이미지 다운로드/추론 생략)
docker exec -it vector-vision-search-app python indexing/load_indexed.py
```

### StateManager (상태 관리)

Redis 기반 파이프라인 상태 관리로 중단 시 재시작 가능:

| 상태 | 설명 |
| --- | --- |
| `PENDING` | 처리 대기 중 |
| `PROCESSING` | 처리 중 |
| `COMPLETE` | 완료 |
| `FAILED` | 실패 |

```python
# 이미 완료된 항목은 건너뛰고 실패한 항목만 재처리
pending_items = state_manager.get_all_by_status(state_manager.PENDING)
```

---

## 🛡 API 안정성 (Circuit Breaker)

### Circuit Breaker 패턴

Elasticsearch 장애 시 연쇄 장애 방지를 위한 Circuit Breaker 구현:

| 상태 | 동작 |
| --- | --- |
| **CLOSED** | 정상 동작, ES 호출 |
| **OPEN** | ES 호출 차단, 캐시 또는 빈 결과 반환 |
| **HALF_OPEN** | 테스트 요청 1개 허용, 성공 시 CLOSED로 전환 |

---

## ⚡ 성능 최적화 및 모니터링

### 1. 배치 추론 파이프라인

```python
# 다운로드와 추론을 병렬 처리
다운로드 스레드 → 큐 → 배치 추론 (CPU)
```

- 다운로드 완료된 이미지를 큐에 적재
- 배치 크기(32개)만큼 모이면 즉시 DINOv2 추론
- I/O 대기 시간 최소화

### 2. Elasticsearch Bulk API

```python
# 개별 색인 대신 Bulk API로 한 번에 색인
bulk(es, actions, raise_on_error=False)
```

### 3. Redis 캐싱

```python
# 동일한 검색 요청 1분간 캐싱
cache_key = f"search:{image_url}:{site}:{date}"
```

### 4. int8 Quantization

```python
# ES 인덱스 설정
"index_options": {
    "type": "int8_hnsw",  # float32 → int8 변환
    "m": 16,
    "ef_construction": 100
}
```

---

## 📚 참고 레퍼런스

### 공식 문서

- [Elasticsearch Dense Vector](https://www.elastic.co/guide/en/elasticsearch/reference/current/dense-vector.html)
- [Elasticsearch kNN Search](https://www.elastic.co/guide/en/elasticsearch/reference/current/knn-search.html)
- [DINOv2 (Meta AI)](https://github.com/facebookresearch/dinov2)
- [FastAPI Documentation](https://fastapi.tiangolo.com/)

### 기술 블로그

- [Elasticsearch Binary Quantization](https://www.elastic.co/search-labs/blog/scalar-quantization-101)
- [HNSW Algorithm Explained](https://www.pinecone.io/learn/hnsw/)
- [Circuit Breaker Pattern](https://martinfowler.com/bliki/CircuitBreaker.html)
