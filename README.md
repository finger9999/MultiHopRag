# Multi-Hop RAG 프로젝트

## 개요

본 프로젝트는 복잡한 질의에 답변하기 위해 여러 문서와 정보 소스를 연결하여 추론하는 **Multi-Hop RAG(Retrieval-Augmented Generation)** 시스템을 구현합니다. Multi-hop 질의응답은 단일 문서에서 답을 찾을 수 없고, 여러 문서에서 정보를 집계하고 종합해야 하는 어려운 작업입니다.

## Multi-Hop RAG란?

Multi-hop 추론은 다음과 같은 질문에 답변하기 위한 능력을 의미합니다:
- **다단계 검색**: 여러 문서에서 관련 정보 찾기
- **문서 간 추론**: 서로 다른 소스의 사실들을 연결
- **복잡한 질문 유형**: 비교, 브리지, 추론, 복합 질문 등

기존의 단일 홉 RAG 시스템은 "대부의 감독의 배우자는 누구인가?"와 같은 질의에 어려움을 겪습니다. 이러한 질문은:
1. 먼저 감독을 식별하고 (프란시스 포드 코폴라)
2. 그 다음 감독의 배우자 정보를 찾아야 하기 때문입니다

본 프로젝트는 이러한 Multi-hop 추론 작업을 처리하기 위한 고급 RAG 파이프라인을 구현합니다.

## Multi-Hop RAG 파이프라인 아키텍처

파이프라인은 여러 핵심 단계로 구성됩니다:

```
Query → 질문 유형 분류 → BM25 검색 → 하이브리드 Dense 재순위화
    → 증거 추출 (LLM) → 질문 유형 인식 추론 (LLM)
```

### 파이프라인 단계:

1. **질문 유형 분류**: 질의를 분류하여 특화된 검색 전략 적용
2. **BM25 검색**: 초기 희소 검색으로 후보 문서 탐색
3. **하이브리드 Dense 재순위화**: 의미적 임베딩을 사용하여 결과 재순위화 및 정제
4. **증거 추출**: LLM이 검색된 문단에서 관련 사실 추출
5. **질문 유형 인식 추론**: 증거를 기반으로 LLM이 최종 답변 생성

## 기술 스택

### Vector Database
- **Chroma**: 임베딩 저장 및 유사도 검색을 위한 고성능 벡터 데이터베이스

### Embedding Model
- **모델**: BAAI/bge-m3
- **목적**: 텍스트 청크를 의미적 검색을 위한 밀집 벡터 표현으로 변환

### Text Splitting
- **방법**: RecursiveCharacterTextSplitter
- **파라미터**:
  - `chunk_size`: 900자
  - `overlap`: 100자

### Document Metadata
각 문서 청크는 다음 메타데이터 필드를 포함합니다:
- `author`: 문서 작성자
- `category`: 콘텐츠 카테고리
- `title`: 문서 제목
- `published_at`: 발행 날짜
- `source`: 소스 식별자
- `url`: 문서 URL

### Language Model
- **모델**: OpenAI GPT-4o-mini
- **파라미터**:
  - `seed`: 2026
  - `temperature`: 0 (결정적 출력)
  - `retrieval k`: 5 (상위 5개 문서 검색)

## 데이터셋

**데이터셋**: [yixuantt/MultiHopRAG](https://huggingface.co/datasets/yixuantt/MultiHopRAG)

- **Subset**: MultiHopRAG
- **Split**: train
- **설정**:
  - `train_size`: 50개 샘플
  - `stratify`: `question_type` 필드 기준 계층화
  - `random_state`: 42

MultiHopRAG 데이터셋은 여러 문서에 걸친 추론이 필요한 복잡한 Multi-hop 질문을 포함하며, 비교, 브리지, 추론 질문 등 다양한 질문 유형이 있습니다.

## 평가 지표

시스템은 4가지 핵심 지표로 평가됩니다:

### 1. **Hit@5**
- 상위 5개 검색 문서에 정답이 포함되어 있는지 측정
- 이진 지표 (상위 5개에 답변이 있으면 1, 없으면 0)

### 2. **MRR@5 (Mean Reciprocal Rank)**
- 첫 번째 관련 문서의 순위 위치 평가
- 공식: 첫 번째 관련 결과의 `1 / 순위`
- 높은 값은 더 나은 순위 품질을 나타냄

### 3. **MAP@5 (Mean Average Precision)**
- 각 관련 문서 위치에서의 정밀도를 고려
- 순위 품질에 대한 보다 포괄적인 관점 제공
- 관련 결과의 순서와 완전성을 고려

### 4. **Exact Match (Answer Accuracy)**
- 생성된 답변이 정답과 정확히 일치하는지 측정
- 최종 추론 출력에 대한 엄격한 평가

## 실험 결과

<img width="1200" height="600" alt="Code_Generated_Image_2" src="https://github.com/user-attachments/assets/18bdece4-f9b2-4574-8767-627a11ef3616" />

### 파이프라인 비교:

| 파이프라인 | Hit@5 | MRR@5 | MAP@5 | Exact Match |
|----------|-------|-------|-------|-------------|
| **Pipeline NY** | **0.800** | **0.730** | **0.488** | **0.720** |
| Pipeline MJ | 0.680 | 0.592 | 0.313 | 0.480 |
| Pipeline EB | 0.640 | 0.600 | 0.379 | 0.640 |
| Pipeline MJ Basic | 0.420 | 0.400 | 0.273 | 0.480 |

**주요 발견**:
- **Pipeline NY**가 모든 지표에서 최고의 성능을 달성
- 베이스라인(Pipeline MJ Basic) 대비 상당한 개선 보임
- Hit@5 0.800은 강력한 검색 품질을 나타냄
- Exact Match 정확도 0.720은 효과적인 추론 능력을 보여줌

## 프로젝트 구조

```
multi-hop/
├── README.md
├── multi_hop_RAG_basic.ipynb       # 기본 RAG 구현
├── multi_hop_RAG_TSSS.ipynb        # 고급 Multi-hop RAG 파이프라인
└── Code_Generated_Image (2).png    # 평가 결과 시각화
```

## 사용 방법

### 필수 요구사항

```bash
pip install langchain chromadb openai datasets transformers sentence-transformers
```

### 환경 설정

OpenAI API 키 설정:

```python
import os
os.environ["OPENAI_API_KEY"] = "your-api-key-here"
```

### 파이프라인 실행

1. MultiHopRAG 데이터셋 로드
2. Chroma 벡터 데이터베이스 초기화
3. BAAI/bge-m3를 사용하여 문서 임베딩
4. Multi-hop RAG 파이프라인 실행
5. 4가지 지표로 평가

자세한 구현은 Jupyter 노트북을 참조하세요.

## 주요 기능

- **Multi-hop 추론**: 여러 검색 단계가 필요한 복잡한 질의 처리
- **질문 유형 인식**: 질의 분류에 따라 전략 적응
- **하이브리드 검색**: BM25 희소 검색과 밀집 의미 검색 결합
- **증거 기반 추론**: 여러 소스에서 정보 추출 및 종합
- **종합적 평가**: 검색 및 생성 품질을 평가하기 위한 다중 지표 사용

## 향후 개선 방향

- 다양한 임베딩 모델 실험
- 반복적 검색 전략 구현
- 복잡한 질문에 대한 쿼리 분해 추가
- 그래프 기반 추론 접근 방식 탐색
- 도메인별 데이터에 대한 재순위화 모델 미세 조정

## 참고 자료

- 데이터셋: [yixuantt/MultiHopRAG on Hugging Face](https://huggingface.co/datasets/yixuantt/MultiHopRAG)
- 임베딩 모델: [BAAI/bge-m3](https://huggingface.co/BAAI/bge-m3)
- Vector Database: [Chroma](https://www.trychroma.com/)
- LLM: OpenAI GPT-4o-mini

## 라이선스

본 프로젝트는 교육 및 연구 목적으로 사용됩니다.
