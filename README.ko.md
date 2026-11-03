<p align="center">
  <img src="images/logo.png" alt="MOVEBRAIN 로고" width="220">
</p>

<h1 align="center">M-Pilot</h1>

<p align="center">
  <b>근거 기반 설비 장애 대응 RAG 서비스</b><br>
  데이터 레이어 · 이재서 (<a href="https://github.com/8113jsl-ui">@8113jsl-ui</a>)
</p>

<p align="center">
  <a href="README.md">English</a> · 한국어
</p>

<p align="center">
  <img alt="Python" src="https://img.shields.io/badge/Python-3.12-3776AB?logo=python&logoColor=white">
  <img alt="MySQL" src="https://img.shields.io/badge/MySQL-8-4479A1?logo=mysql&logoColor=white">
  <img alt="SQLAlchemy" src="https://img.shields.io/badge/SQLAlchemy-Core-D71F00?logo=sqlalchemy&logoColor=white">
  <img alt="Chroma" src="https://img.shields.io/badge/Vector%20DB-Chroma-FF6446">
  <img alt="LangChain" src="https://img.shields.io/badge/LangChain-1C3C3C?logo=langchain&logoColor=white">
  <img alt="pytest" src="https://img.shields.io/badge/tested%20with-pytest-0A9EDC?logo=pytest&logoColor=white">
</p>

> 원본 팀 저장소 [movebrain/m-pilot](https://github.com/movebrain/m-pilot)의 개인 포트폴리오용 포크. 저장소 내 매뉴얼·에러코드·서비스 이력은 모두 가상 데이터.

## 요약

설비 에러코드 신고를 셀프 가이드 또는 엔지니어 접수로 분기하고, 엔지니어 질문에는 매뉴얼·과거 확정 조치 근거와 출처를 붙여 답변하는 서비스. 근거가 없으면 답변 대신 근거 없음 안내.

**데이터 레이어** 핵심 기능 담당: 매뉴얼 PDF 표의 출처 있는 청크 변환, 동시 처리에 안전한 대화·조치 이력 저장, 확정 조치의 즉시 검색 색인 반영.

| | |
|---|---|
| **기간** | 2026.10.21 ~ 2026.11.02 (2주, 현대AI캠퍼스 1차 프로젝트) |
| **인원** | 7명 (UI·총괄 1, 데이터 3, RAG 3) |
| **역할** | 데이터 레이어 개발 |
| **기여** | 커밋 23건 · PR 7건 머지 · 테스트 모듈 5건 |
| **기술** | Python 3.12, pdfplumber, MySQL, SQLAlchemy, Chroma, LangChain, pytest, GitHub Actions |

## 핵심 성과

- **매뉴얼 PDF 표 추출 파이프라인 개발**: 행·열 구조 유지, 쪽 넘김 표 자동 연결, 인쇄 쪽 번호 기준 출처 표기로 답변 출처와 실제 매뉴얼 쪽 일치
- **동시 처리 안전한 이력 저장 설계**: 행 잠금·조건부 UPDATE로 중복 완료 및 완료 후 저장 차단
- **조치 결과 피드백 루프 구현**: 엔지니어가 확정한 조치를 즉시 색인에 반영, 다음 답변 근거로 활용
- **색인 재생성 안정성 확보**: 신규 문서 전체 검증 후 기존 색인 교체, 원자료 오류 시 기존 색인 보존
- **RDB 모듈 구조 개선**: 단일 `rdb.py`를 기능별 패키지로 분리, 타 파트 호출 코드 변경 없음

## 아키텍처

노란색 박스: 담당 영역

```mermaid
flowchart LR
    PDF["매뉴얼 PDF<br/>(설비 4종)"] --> T["표 추출·청킹"]
    PDF --> X["텍스트 청킹"]
    T --> IDX["검색 색인 생성"]
    X --> IDX
    CODE["에러코드·<br/>시드 이력"] --> IDX
    IDX --> RAG["RAG 검색·답변"]
    RAG --> CHAT["대화 이력 저장"]
    CHAT --> RES["조치 결과 확정"]
    RES --> HIS["신규 이력<br/>색인 반영"]
    HIS -.->|"다음 답변<br/>근거로 활용"| RAG

    classDef mine fill:#FFE066,stroke:#B8860B,stroke-width:3px,color:#000;
    class T,IDX,CHAT,RES,HIS mine;
```

## 주요 구현

### 1. 매뉴얼 표 출처화

[`mpilot/db/parser/tables.py`](mpilot/db/parser/tables.py) · PR [#57](https://github.com/movebrain/m-pilot/pull/57), [#103](https://github.com/movebrain/m-pilot/pull/103)

**문제**
- 에러코드·점검 절차·사양 등 핵심 정보 대부분이 표 형태
- 일반 텍스트 추출 시 행·열이 섞여 맥락 없는 조각 검색
- PDF 순번과 매뉴얼 인쇄 쪽 번호 불일치

**해결**
- pdfplumber 기반 쪽별 표 추출, 캡션·위치(bbox) 부여
- 꼬리글 인쇄 쪽 번호 사용
- 캡션 없이 앞 쪽 마지막 표와 머리글이 같은 표를 이어진 표로 판단, 캡션 승계
- 병합 셀·빈 셀·누락 칸 검출
- 마크다운 표 변환, 긴 표는 행 단위 분할 후 조각마다 캡션·머리글 재부착

**결과**
- 표 조각 단독 검색 시에도 표 맥락 유지
- 답변 출처가 실제 매뉴얼 쪽과 일치

| 함수 | 기능 |
|---|---|
| `_extract_tables` | 쪽별 표 추출, 쪽 넘김 표 연결 |
| `_validate_table` | 빈 셀·병합 셀 좌표 검출 |
| `_table_to_drafts` | 마크다운 청크 생성, 분할 시 머리글 반복 |

이미지 페이지 OCR 경로([`ocr.py`](mpilot/db/parser/ocr.py))도 구현 후 리뷰 합의로 보류 (현재 매뉴얼에 이미지 페이지 없음, 재활성화 절차 모듈 내 문서화).

### 2. 동시 처리 안전한 대화·조치 이력 저장

[`mpilot/db/rdb/chat.py`](mpilot/db/rdb/chat.py) · PR [#98](https://github.com/movebrain/m-pilot/pull/98)

**문제**
- 엔지니어 2명 동시 작업 또는 중복 클릭 시 같은 건에 대한 동시 처리 발생
- 동일 건 중복 완료, 완료 건·이관 전 셀프 가이드 건에 대한 저장 차단 필요

**해결**
- `SELECT ... FOR UPDATE`로 신고 행 잠금 후 상태 확인
- 조치 기록 저장과 `completed` 전환을 단일 트랜잭션으로 처리
- 미완료 건에만 적용되는 조건부 `UPDATE`로 완료 처리 보호
- 첫 질문 저장 시 `received` → `in_progress` 자동 전환
- 입력 길이 검증 (질문 1~2,000자, 조치 항목별 1~5,000자), 오류 유형별 예외 분리 (`ValidationError`, `NotFoundError`, `InvalidStateError`, `DependencyError`)

**결과**
- 상태 전환 규칙을 DB 단에서 보장, UI·서비스 계층 단순화

| 함수 | 기능 |
|---|---|
| `fetch_chat_turns` | 신고 건별 대화 이력 조회 |
| `save_chat_turn` | 질문·출처 포함 답변 저장, 상태 전환 |
| `save_confirmed_resolution` | 조치 결과 저장 및 완료 확정 |

### 3. 확정 조치 기반 검색 색인

[`mpilot/db/vectordb.py`](mpilot/db/vectordb.py) · PR [#56](https://github.com/movebrain/m-pilot/pull/56), [#133](https://github.com/movebrain/m-pilot/pull/133)

**문제**
- 매뉴얼과 실제 현장 조치 이력을 함께 검색해야 함
- 방금 확정된 조치도 앱 재시작 없이 바로 활용 필요

**해결**
- 앱 시작 시 Chroma 컬렉션 2개 재생성: `manual`(에러코드 행·매뉴얼 청크), `history`(확정 조치)
- 조치 확정 즉시 해당 건 `history`에 upsert
- 문서 ID 규칙 검증: `error:`·`manual:`·`history:` 접두어, ID 구성 요소 내 `:` 금지, 중복 금지
- 전체 문서 생성·검증 후에만 기존 컬렉션 초기화
- 미확정 건·셀프 가이드 건 이력 등록 차단

**결과**
- 조치 완료 건이 쌓일수록 답변 근거 확대
- 잘못된 원자료 적재 시 빈 색인 대신 명확한 오류로 중단

| 함수 | 기능 |
|---|---|
| `rebuild_search_index` | 검증 후 두 컬렉션 재생성 |
| `index_history` | 확정 조치 1건 upsert |

### 4. RDB 모듈 구조 개선

[`mpilot/db/rdb/`](mpilot/db/rdb/) · PR [#152](https://github.com/movebrain/m-pilot/pull/152)

- `rdb.py`를 `setup`·`cases`·`chat`·`_common` 모듈로 분리
- 공개 함수 재노출로 기존 `rdb.<함수>` 호출 방식 유지
- RDB 테스트 모듈 7건 신규 구조 대응
- 스키마 생성·DB 연결 기능 공동 개발 (데이터 파트 3인)

### 5. 협업 환경 구성

PR [#25](https://github.com/movebrain/m-pilot/pull/25)

- 팀 AI 코딩 규칙 첫 버전 작성 (`AI_RULES.md`는 이후 정리되어 삭제, `CLAUDE.md`는 팀 공용 규칙 문서로 유지)
- 프로젝트 첫날 `.gitignore` 구성 (비밀 정보·벡터DB·캐시 업로드 방지)

## 테스트

| 모듈 | 검증 항목 |
|---|---|
| `tests/test_parser_tables_ocr.py` | 표 추출, 쪽 넘김 표 연결, 셀 검증, 행 단위 분할 |
| `tests/test_parser_types.py` | 공유 타입, bbox 좌표 순서 |
| `tests/test_rdb_chat_resolution.py` | 대화 저장, 조치 확정, 완료·이관 전 건 차단 |
| `tests/test_rdb_schema.py` | 테이블 생성·초기화 |
| `tests/test_vectordb_indexing.py` | 색인 재생성, 이력 upsert, 검증 실패 시 기존 색인 보존 |
| `tests/conftest.py` | MySQL 공용 fixture, CI에서 테스트 DB 연결 실패 시 건너뛰지 않고 실패 처리 |

MySQL 테스트는 GitHub Actions의 MySQL 서비스 컨테이너에서 실행.

```bash
pytest tests/test_parser_tables_ocr.py tests/test_rdb_chat_resolution.py tests/test_vectordb_indexing.py
```

## 빠른 실행

```bash
python3.12 -m venv venv && source venv/bin/activate
pip install -r requirements.txt
cp .env.example .env

# 화면만 확인 (MySQL·API 키 불필요)
python app_customer.py --fake
```

MySQL·OpenAI 포함 전체 실행: [docs/RUN.md](docs/RUN.md)

## 기여 PR

| PR | 내용 |
|---|---|
| [#25](https://github.com/movebrain/m-pilot/pull/25) | 팀 AI 코딩 규칙, `.gitignore` |
| [#56](https://github.com/movebrain/m-pilot/pull/56) | 검색 색인 재생성, 확정 조치 색인 반영 |
| [#57](https://github.com/movebrain/m-pilot/pull/57) | 파서 공유 타입, bbox 검증, 인쇄 쪽 번호 기준 |
| [#98](https://github.com/movebrain/m-pilot/pull/98) | DB 스키마·연결, 대화·조치 저장 |
| [#103](https://github.com/movebrain/m-pilot/pull/103) | 표 추출·검증·청킹, 이미지 OCR(보류) |
| [#133](https://github.com/movebrain/m-pilot/pull/133) | 색인 저장 코드 단순화 |
| [#152](https://github.com/movebrain/m-pilot/pull/152) | `rdb.py` 기능별 패키지 분리 |

## 회고

- **검색 품질은 수집 단계에서 결정**: 인쇄 쪽 번호, 표 머리글 반복이 프롬프트 조정보다 답변 신뢰도에 직접적인 영향
- **불변 조건은 DB에서 보장**: 행 잠금·조건부 UPDATE로 중복 제출 문제 해소, UI 코드 단순화
- **삭제 전 검증**: 색인 초기화처럼 되돌리기 어려운 작업은 대체 데이터 생성·검증 완료 후 실행
- **읽기 쉬운 코드도 팀 기능**: 줄 단위 한글 주석, 단순 문법 위주 작성으로 리뷰 속도 개선

## 팀

| 이름 | 파트 | GitHub |
|:---:|:---:|:---:|
| 노동균 | UI·총괄 | [@rohdonggyun-lab](https://github.com/rohdonggyun-lab) |
| 김성일 | 데이터 | [@tjddlf112015-jpg](https://github.com/tjddlf112015-jpg) |
| 손재호 | 데이터 | [@thswogh](https://github.com/thswogh) |
| **이재서 (본인)** | **데이터** | [@8113jsl-ui](https://github.com/8113jsl-ui) |
| 도예진 | RAG | [@yejjini](https://github.com/yejjini) |
| 정혜윤 | RAG | [@hyey00n](https://github.com/hyey00n) |
| 한시온 | RAG | [@zionhann](https://github.com/zionhann) |

원본 팀 README: [docs/TEAM_README.md](docs/TEAM_README.md)
