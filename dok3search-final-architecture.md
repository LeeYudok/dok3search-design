# dok3search — 금융권 폐쇄망 개발 지식 파운데이션 최종 설계서

> 문서 버전: 1.0 (MVP 우선 설계)  
> 작성 기준일: 2026-11-20  
> 개발 방법론: Superpowers의 설계 → 계획 → TDD → 검증을 반영한 단계적 탑다운 개발  
> 주요 환경: Windows 온프레미스 / Rust / Tantivy / React / TypeScript  
> 원칙: 벡터·임베딩·외부 웹 의존성 없음. 사람과 AI가 동일한 원문 지식을 검색하고 읽는다.

릴리스별로 실제 사용할 수 있는 기능은 [설계 인덱스의 현재 구현](index.md#현재-구현)에 기록한다.

## 1. 비전과 범위

dok3search는 금융권 폐쇄망에서 개발 문서·기술 교재·매뉴얼·API 레퍼런스·코드 예제·사내 소스·SR/업무 문서를 수집·구조화·색인·검색·열람하는 **개발 지식 파운데이션**이다. 검색 결과를 짧게 보여주는 것으로 끝내지 않고 **문서의 제목→요약→장/절→본문→예제→출처→이전/다음 문맥**으로 탐색할 수 있어야 한다. 사람은 React 블로그형 웹 UI로, AI 코딩 에이전트는 MCP/REST로 동일한 원문을 읽는다.

대표 시나리오: Rust 비동기 코딩 경험이 부족한 에이전트가 `async`, `await`, Tokio 공식 API, 로컬 반입된 Rust 교재의 장을 검색 → 목차와 관련 절을 원문으로 읽기 → 코드 생성 → `cargo test` → 실패 시 컴파일 오류로 재검색한다. 검색으로 얻은 문서의 버전과 출처를 항상 표시하며, 검색 성공이 코드 정답을 보증하지는 않는다.

### 1.1 필수 목표

- Windows 단일 서버/PC에서 **Rust 네이티브 프로세스**로 실행하며 Windows Service 등록이 가능하다.
- Rust + Tantivy 기반 역색인 검색; BM25, 정확 일치, 형태소 분석, bi-gram, n-gram, 코드 심볼 검색을 제공한다.
- Markdown, HTML, 텍스트 및 주요 코드 언어를 파싱한다. PDF, DOCX, JSP 등은 단계적으로 추가한다.
- 블로그형 React 검색 UI, 문서 원문 뷰어, 목차·코드 하이라이트·요약·태그·키워드·카테고리를 제공한다.
- 파일·Git/SVN·SQLite·MariaDB·MySQL·PostgreSQL·Oracle 소스를 모듈화된 Connector로 색인한다.
- REST + MCP를 먼저 제공하고, A2A·ACP는 독립 어댑터로 확장한다.
- 승인 워크플로 없이 자동 수집/갱신한다. 기존 시스템 권한·접근 제어·비밀정보 보호는 유지한다.
- 독립 모듈별 **단위 테스트 우선**, 인터페이스 계약 테스트 및 단계별 통합 테스트로 빅뱅 개발을 방지한다.

### 1.2 명시적 비범위(초기)

- 벡터 DB·임베딩·시맨틱 벡터 검색·외부 SaaS 의존성.
- 다중 노드 분산 검색·자동 HA·무제한 대규모 크롤링.
- AI가 원문을 ‘학습’했다고 보장하거나 임의 라이브러리의 동작을 보증하는 기능.
- 사내 지식의 사람 승인 단계(추후 플러그인으로 확장 가능).
- 모든 DB에서 동일 수준의 CDC를 초기 버전에 동시에 완성하는 것.

## 2. 아키텍처 원칙 — 탑다운·모듈형 모놀리스

전체 시스템은 **상위 요구사항 → 도메인 유스케이스 → Port(인터페이스) → Adapter(구현) → 라이브러리/인프라** 순으로 설계한다. 단일 `dok3search.exe`로 서버를 구동할 수 있으나 모듈 간 직접 구현 의존은 금지한다.

```text
[React Web UI]        [Codex/Claude Code/AI-Coder]      [A2A/ACP optional]
      | REST                    | MCP/REST                       | adapter
      +-------------------------+-----------------------------------+
                                |
                        [Axum API Gateway]
                   Auth / ACL / Rate limit / Audit
                                |
                         [Application Layer]
          SearchUseCase | DocumentUseCase | IndexJobUseCase
                  |                  |                  |
          [SearchPort]        [DocumentPort]      [SourcePort]
                  |                  |                  |
       [Tantivy Adapter]    [SQLite+File]    [File/DB/Git Adapters]
                  ^                  ^                  |
                  +---- [Parser → Chunker → Enricher → Indexer] --+
```

**의존성 방향**: `domain`은 Tantivy·Axum·DB·React를 알지 못한다. `application`은 도메인의 Port만 참조한다. `adapters`가 구현체를 제공한다. HTTP/MCP/A2A/ACP는 동일한 UseCase에 접근하는 별도 진입점이다. 서비스가 커지더라도 먼저 모듈 경계를 유지하고, 분산 서비스는 실측 병목이 확인될 때만 검토한다.

### 2.1 Rust Workspace

```text
dok3search/
├── backend/rust/
│   ├── Cargo.toml
│   ├── Cargo.lock
│   ├── rust-toolchain.toml
│   └── crates/                 # 도메인, 소스, 파서, 색인, 검색, API, 에이전트, 서버
├── frontend/                   # React + TypeScript + Vite
├── config/                     # 설정 예시와 스키마
├── contracts/                  # REST·MCP·도메인 계약
├── tests/                      # 계약·통합·장애 주입 테스트
├── docs/                       # 설계와 운영 문서
└── scripts/                    # 저장소 검증과 유틸리티
```

현재 crate 이름과 역할은 [Rust 워크스페이스·계약](RUST_WORKSPACE_AND_CONTRACTS.md)에 있다.

### 2.2 모듈 설계 수칙

1. 모듈 하나 = 한 가지 책임. 형식별 파서와 DB별 Connector는 플러그인 레지스트리로 교체 가능하게 한다.
2. 내부 API는 DTO/trait로 명시한다. 다른 모듈의 DB 테이블이나 Tantivy 자료구조에 직접 의존하지 않는다.
3. Error 타입을 `ParseError`, `IndexError`, `SourceError`, `AuthError` 등으로 구조화하며 오류 체인과 복구 가능성을 기록한다.
4. 입출력·SQL·시간·파일시스템은 Port 뒤로 숨겨 `Mock`/`Fake`를 주입할 수 있게 한다.
5. 각 단계는 완성된 한 개의 수직 기능(Vertical Slice)을 제공한다. `검색 → 결과 → 원문 열람`이 가능한 최소 제품부터 개발한다.
6. `cargo test --workspace`와 프런트엔드 단위 테스트가 실패하면 병합하지 않는다.

## 3. 핵심 도메인 및 데이터 계약

서로 다른 소스도 공통 문서 모델로 정규화한다. **문서(Document)는 논리적 원본**, **버전(Revision)은 특정 시점**, **섹션(Section)은 목차 항목**, **청크(Chunk)는 검색 단위**이다. 원문 파일과 렌더링된 HTML은 색인과 독립적으로 저장한다.

```json
{
  "document_id": "public:rust-book:ch17",
  "revision_id": "sha256:...",
  "source_id": "rust-official-docs",
  "source_kind": "official_docs",
  "title": "Async Programming",
  "summary": "비동기 작업과 await의 원리를 설명하는 장",
  "tags": ["rust", "async", "runtime"],
  "keywords": ["async", "await", "future", "tokio"],
  "category_path": ["개발언어", "Rust", "비동기"],
  "language": "rust",
  "library": "tokio",
  "library_version": "1.x",
  "doc_type": "manual",
  "original_uri": "kb://public/rust-book/ch17",
  "original_format": "html",
  "license": "source-verified-license",
  "visibility": "public",
  "acl_scope": "public",
  "parser_version": "parser-rustbook-v1",
  "enrichment_status": "rule_generated",
  "sections": [
    {"section_id":"s17-1","heading":"Futures", "order":1}
  ]
}
```

요약·태그·키워드·카테고리는 **필수 저장 필드**다. 초기에는 제목·목차·규칙 사전에서 생성하고 사용자가 직접 편집할 수도 있다. 나중에 서브에이전트가 자동 생성하면 `generated_by`, `model_version`, `confidence`, `updated_at`, `provenance`를 별도 보관한다. 원문은 생성 결과로 덮어쓰지 않는다. 미분류는 `uncategorized`로 유지하며 수집을 막지 않는다.

**ID 정책**: `source_id + native_primary_key`의 안정적 ID, `revision_id`는 내용 해시/리비전, 청크 ID는 `document_id + section key + logical ordinal`로 정한다. 변경된 문서만 새 리비전으로 업데이트하고, 이전 청크 삭제와 신규 청크 반영을 하나의 인덱싱 작업으로 추적한다.

## 4. 파서·문서 구조화

### 4.1 ParserPort

```rust
pub trait DocumentParser: Send + Sync {
    fn supports(&self, media_type: &str, filename: &str) -> bool;
    fn parse(&self, input: &SourceBlob) -> Result<ParsedDocument, ParseError>;
}

pub struct ParsedDocument {
    pub title: String,
    pub sections: Vec<Section>,
    pub normalized_text: String,
    pub code_blocks: Vec<CodeBlock>,
    pub links: Vec<SourceLink>,
    pub parse_warnings: Vec<String>,
}
```

포맷 우선순위: **MVP: Markdown·HTML·TXT·JSON → 확장: Rust/Java/Go/Python/JS·TS/SQL/XML → 후속: PDF·DOCX·JSP/특수 소스**. 코드 구조 분석은 Tree-sitter로 노드·심볼·행 범위를 추출한다. JSP는 HTML + Java 혼합 구문용 어댑터로 다룬다. PDF는 추출 손실을 기록하고, 표와 코드 블록이 복구되지 않으면 검토 표식을 남긴다.

### 4.2 Chunking

- 일반 매뉴얼: H1/H2/H3, 장/절 경계 우선. 400~1,000 토큰을 시작점으로 튜닝한다.
- API: 하나의 함수/클래스/설정 항목과 관련 예제를 함께 둔다.
- 코드: 심볼(함수/메서드/클래스) 단위. 파일 레벨 요약·경로 검색용 문서를 별도로 둔다.
- 오류: 오류 식별자 + 원인 + 재현 조건 + 해결책을 한 단위로 둔다.
- 모든 청크에 `document_id`, `section_id`, `start_line/end_line`, `heading_path`, `content_hash`, `acl_scope`가 필요하다.
- AI 원문 탐색용 `get_document`/`get_section`은 **청크 일부가 아닌 원문 전체 또는 지정 절**을 반환한다.

## 5. Tantivy 색인 및 검색 설계

### 5.1 물리 인덱스

- `public_docs`: 기술 교재, 공식 매뉴얼, 외부 라이브러리 API.
- `public_examples`: 검증된 코드 예제, 오류 지식.
- `internal_docs_<scope>`: 사내 표준, SR·업무 문서 등 보호 범위별 색인.
- `internal_code_<scope>`: 소스 파일·심볼·함수 본문. ACL이 다른 자료는 같은 물리 색인에 무조건 혼합하지 않는다.

공통 논리 스키마는 `title`, `summary`, `tags`, `keywords`, `category`, `body_morph`, `body_bigram`, `body_ngram`, `body_plain`, `symbol_exact`, `symbol_parts`, `error_code`, `language`, `version`, `source_id`, `document_id`, `section_id`, `acl_scope`, `revision_id` 등으로 구성한다. 각 필드는 분석 방식에 맞게 별도로 정의한다.

### 5.2 Analyzer 구성

| 분석 필드 | 색인 전략 | 검색 역할 |
|---|---|---|
| `title_morph`, `body_morph` | 한국어 형태소 분석(사전 고정, 품질 테스트) | 업무용어·한글 자연어 |
| `body_bigram` | 2글자 문자 bi-gram | 부분 검색·띄어쓰기 보완 |
| `body_ngram` | 문자 2~4-gram | 오타/불완전 문자열 보조 검색 |
| `body_plain` | 일반 토큰화 | 영어 설명서, 코딩 매뉴얼 |
| `symbol_exact` | 원문 식별자 보존 | `tokio::spawn`, `getAccountList` |
| `symbol_parts` | camelCase·snake_case·점·콜론 분리 | `account list`, 패키지 탐색 |
| `error_code` | 정확 일치 | `ORA-00904`, `E0382` |
| `tags`, `keywords` | 정규화 키워드 | 필터·부스팅·분류 |

형태소·n-gram은 **별도 필드**로 관리한다. 부정확한 n-gram이 상위 순위를 점유하지 않도록 낮은 가중치로 후보를 보완한다. 사전/분석기를 변경하면 해당 필드의 재색인이 필요하다. Tantivy의 BM25를 기본 후보 점수로 사용한다.

### 5.3 검색 파이프라인

1. Query Analyzer: 오류 코드, 심볼, 자연어, 버전, 언어, 사용자 권한 분석.
2. Query Planner: 정확 일치/구문/일반 BM25/형태소/bi-gram/n-gram 검색 계획 수립.
3. ACL Enforcement: **검색 전에** 허용된 인덱스·범위를 제한. 원문 조회에서도 재확인.
4. Tantivy 실행: 다중 필드 검색을 별도 실행하고 Top K 획득.
5. Merge/Rank: RRF 기반 병합 또는 정규화 점수 결합; 정확 심볼·일치 버전·공식 출처·승인된 내부 표준 우대.
6. Snippet: 원문 기준 하이라이트를 생성. 각 결과에 출처, 장/절, 라인, 라이브러리 버전, ACL 포함.
7. AI Context: `get_section`으로 특정 절을 추가 조회하도록 유도. 검색 결과를 실행 지시로 취급하지 않는다.

### 5.4 측정 지표

- 코딩 질의셋(최소 100개, 이후 확장)으로 `Recall@10`, `MRR@10`, API 정확 일치율을 측정.
- 첫 목표: 정답이 존재하는 질의의 Recall@10 ≥ 0.90, P95 검색 지연 ≤ 300ms (벤치마크 장비/데이터 규모를 명시한 **목표**, 보장 수치 아님).
- 한글·오류·심볼·버전·사내업무 질의로 테스트셋 분할. n-gram 적용 전후 결과를 비교.

## 6. 블로그형 React 검색/원문 열람 UI

**페이지 구조**: `/` 홈(카테고리/최신 문서), `/search` 통합 검색(필터·정렬·하이라이트), `/docs/:id` 문서 상세(블로그형), `/sources` 출처 브라우징, `/admin/sources` 커넥터 관리, `/admin/jobs` 작업/증분 색인 상태, `/admin/analyzers` 검색어 분석 실험.

**원문 상세**: 문서 제목·요약·태그·키워드·카테고리, 출처·원본 버전·수집일, 왼쪽 탐색 트리, 중앙 본문(표·코드·인용), 오른쪽 장/절 목차, 앞/뒤 문서 이동, 코드 복사, 코드 행 번호, 검색어 강조, 원본 파일 보기. AI가 참조한 문서 ID와 해당 절에 링크를 걸 수 있어야 한다.

**구현**: React + TypeScript + Vite 빌드 산출물을 Axum 정적 자원으로 제공한다. 운영 중 Node.js 서버는 불필요하다. Markdown/HTML 렌더는 신뢰할 수 없는 HTML·스크립트를 제거하고 CSP를 적용한다. 기술 교재나 책을 반입할 경우 저작권·배포/이용 권한을 확인한다. 문서 본문은 임의 원격 스크립트나 네트워크 자원을 불러오지 않는다.

**접근성**: 키보드 탐색, 검색 결과 제목/요약 구조, 모바일/모니터 대응, 코드를 복사할 때 행 번호 미포함, 다크/라이트 모드(후속).

### 화면 색상 기준

| 색상 | 쓰임 |
|---|---|
| 화이트 | 화면과 문서 본문의 기본 배경 |
| 블랙·먹색 | 본문, 제목, 주요 버튼, 읽기 진행선 |
| 그레이 | 출처·날짜·요약, 구분선, 선택 상태의 연한 배경 |
| 레드 | 덕³ 도장. 상태 표시는 오류나 위험처럼 의미가 필요한 곳에 한정 |

검색 결과와 본문에서 색만으로 상태를 구분하지 않는다. 현재 목차는 굵은 글씨와 왼쪽 표시선을 함께 사용한다. 색은 시맨틱 토큰으로 적용한다.

## 7. 수집 파이프라인과 증분 색인

### 7.1 표준 단계

`Source Discovery → Fetch → Normalize → Parse → Chunk → Enrich(rule) → ACL Tag → Tantivy Upsert/Delete → Verify → Checkpoint`. 작업 이력과 원본은 SQLite에 기록하고 인덱스는 재생성 가능해야 한다. 소스 읽기는 기본 읽기 전용, 배치 크기/재시도/중단·재개 기능을 제공한다.

### 7.2 외부 지식 반입

인터넷 연결망에서 공식 문서와 합법적인 기술 문서·예제를 수집하여 `manifest.json`(출처·버전·라이선스·해시), 문서 원본, 체크섬을 포함한 Knowledge Pack 생성 → 악성코드 및 반입 정책 확인 → 폐쇄망 전달 → 변경된 문서만 색인. 무단 책 복제·라이선스 위반 자료는 대상에서 제외한다.

### 7.3 파일 / Git / SVN

파일: 루트 경로·확장자 화이트리스트, 크기 상한, SHA-256 및 수정 시각 비교, 삭제 감지. Git/SVN: 리비전 ID 및 변경 파일 목록, Renamed/Deleted 고려. 기존 데이터와 새 스냅샷의 키 집합 차이로 삭제를 확정한다.

### 7.4 이기종 DB Connector

| DB | 기본 기술 | 증분 색인 | 삭제 처리 |
|---|---|---|---|
| SQLite | SeaORM/SQLx 또는 rusqlite | `updated_at + PK` / 변경 버전 | 삭제 로그 또는 주기 대조 |
| MySQL | SeaORM(SQLx MySQL) | `updated_at + PK` / 변경 로그 | soft-delete·tombstone·키 대조 |
| MariaDB | SeaORM(SQLx MySQL) | 위와 동일 | 위와 동일 |
| PostgreSQL | SeaORM(SQLx Postgres) | `updated_at + PK`; 추후 logical decoding | tombstone/키 대조 또는 CDC |
| Oracle | Oracle 전용 드라이버 + Repository Port | 변경 번호/업데이트 시각 + 안정적 PK | 변경 테이블·키 대조·별도 CDC |

**ORM 경계**: 검색엔진의 내부 운영 메타데이터는 SeaORM을 사용. 이기종 외부 DB 커넥터는 공통 `SourceRepository`/`DbConnector` Port를 두고, ORM이 지원되는 DB는 SeaORM, Oracle은 전용 바인드 쿼리 어댑터로 구현한다. Oracle을 SeaORM에서 직접 지원한다고 가정하지 않는다. 이기종 SELECT를 임의 문자열로 직접 연결하지 않고 **관리자가 등록한 테이블·컬럼 매핑과 파라미터 바인딩된 쿼리**를 사용한다.

**폴링 방식의 조건**: 변경 추적 컬럼이 신뢰 가능하고 충분한 정밀도를 가져야 한다. `(updated_at, primary_key)` 복합 커서를 사용하고 동일 시각 데이터 누락을 막는다. 수집 도중 값이 바뀌는 경우를 고려해 overlap/replay window와 멱등 문서 ID를 적용한다. `updated_at`이 없는 테이블은 안전한 제한 범위 내 주기 전체 대조 또는 변경 테이블이 필요하다. 물리 삭제를 워터마크만으로 탐지할 수 없으므로 tombstone/키 대조/CDC 중 하나가 필수다. Oracle `ORA_ROWSCN` 단독 사용으로 정확한 행별 변경/삭제를 보장한다고 설계하지 않는다.

**정합성**: 배치 추출 시 커서·범위의 일관성을 확인하고, 인덱싱 성공 후에만 Checkpoint를 전진시킨다. 중단 시 동일 배치를 재시도해도 중복 문서가 생기지 않도록 한다. DB 유형별 스냅샷 격리/트랜잭션 동작은 각 커넥터에서 검증한다.

### 7.5 Connector 계약

```rust
pub trait SourceConnector: Send + Sync {
    fn discover(&self) -> Result<Vec<SourceDescriptor>, SourceError>;
    fn fetch_batch(&self, cursor: Option<&Cursor>, limit: usize)
        -> Result<SourceBatch, SourceError>;
}

pub struct SourceBatch {
    pub items: Vec<SourceRecord>,
    pub tombstones: Vec<SourceKey>,
    pub next_cursor: Option<Cursor>,
    pub has_more: bool,
}
```

비동기/동기 드라이버 차이는 Adapter에서 흡수한다. 위 코드는 의도를 설명하는 계약 초안이며 실제 구현 시 async 처리, 취소·타임아웃·백프레셔가 포함된다.

## 8. 외부 연동: REST, MCP, A2A, ACP

### 8.1 REST(필수)

| API | 설명 |
|---|---|
| `POST /api/v1/search` | 검색어·필터·scope·pagination·rank 옵션 |
| `GET /api/v1/documents/{id}` | 문서 메타데이터와 원문 링크 |
| `GET /api/v1/documents/{id}/sections/{sid}` | 특정 장/절 전체 원문 |
| `GET /api/v1/documents/{id}/original` | 접근 허용된 원본 전송 |
| `POST /api/v1/sources` | 데이터 소스 등록/관리 |
| `POST /api/v1/index/jobs` | 전체/증분 색인 시작 |
| `GET /api/v1/index/jobs/{job_id}` | 작업 상태·오류·체크포인트 |
| `POST /api/v1/search/analyze` | 토큰화/형태소/n-gram 결과 확인 |
| `GET /health/ready` | 인덱스·메타데이터·서비스 준비 상태 |

### 8.2 MCP(필수, 초기)

`kb_search`, `kb_get_document`, `kb_get_section`, `kb_search_api`, `kb_search_error`, `kb_search_code`, `kb_list_categories`를 지원한다. **stdio**와 **Streamable HTTP** 전송을 구분 구현한다. 2026-07-28 버전은 Streamable HTTP 전송에서 단일 POST 엔드포인트 중심이며 이전 2025-11-25 규격과 GET/세션 관련 차이가 있으므로 버전 네고시에이션과 호환성 매트릭스를 별도로 검증한다. 최초에는 REST 공용 UseCase를 이용한 어댑터로 구현해 중복 비즈니스 로직을 만들지 않는다. Codex/Claude Code의 실제 클라이언트 설정은 설치 버전별로 검증한다.

### 8.3 A2A(후속)

에이전트 간 **작업 위임/결과 반환**을 위한 별도 서버 어댑터. Agent Card, 작업 상태·메시지·아티팩트 계약과 인증/인가를 사용한다. MCP가 제공하는 검색 도구 기능을 A2A에서 일관되게 노출하되 도메인 로직을 재구현하지 않는다.

### 8.4 ACP(후속)

에디터↔코딩 에이전트 통신용 Agent Client Protocol. dok3search 자체는 기본적으로 **검색 도구 서버**이지 코딩 에이전트가 아니다. 따라서 ACP는 에이전트 클라이언트/브리지에서 MCP 검색 도구를 호출하도록 돕는 선택적 어댑터로 제공한다. ACP 지원이 Codex/Claude CLI의 MCP 지원을 대체하지는 않는다.

## 9. 자동 분류 서브에이전트와 인덱싱 정책

초기에는 규칙 기반 요약·태그·카테고리 생성기로 시작하고, 후속에 **오프라인 LLM 서브에이전트**를 추가한다. 외부 LLM API 없이 실행하도록 설정한다. 서브에이전트는 `document_id, section text, taxonomy version`을 입력받아 `summary, tags, keywords, category candidates, evidence`를 반환한다. 신뢰도/오류를 기록하며 검증을 통과한 필드만 재색인한다. **사람 승인 단계는 초기 제품에 없음**. 원문과 출처는 자동 분류 결과로 변경할 수 없다.

서브에이전트별 계약·테스트 및 개발 작업 분담은 별도 `dok3search-subagents.md`에 상세 정의한다.

## 10. 운영·보안·Windows 배포

- Windows 11/Windows Server 지원 목표. 운영 모드는 Windows Service, 개발 모드는 콘솔 실행.
- Rust 빌드 단일 실행 파일에 React 정적 빌드 자산을 내장하거나 동일 배포 폴더에서 제공. 실행 서버에 Node.js/JVM 설치 불필요.
- `ProgramData/dok3search/` 아래 `config`, `data/originals`, `data/indexes`, `data/meta.db`, `logs`, `backups` 구분. 실행 계정 최소 권한.
- 내부 전용 바인딩, HTTPS(필요 시 내부 리버스 프록시), 사내 인증과 ACL, 사용자 단위 감사, 쿼리/원문 제공 양쪽에서 권한 확인.
- DB 연결 비밀번호는 OS 보호 비밀 저장소 또는 승인된 비밀 관리 시스템에서 주입. 설정 파일 평문 저장 금지.
- 문서 HTML/Markdown은 렌더 전에 sanitize하며, 외부 이미지·JS 로딩과 프롬프트 인젝션을 통제한다. Markdown의 외부 URL은 클릭 가능한 링크로 만들지 않고 출처 표기로 남기며, 로컬 문서·절·번들 라이선스 파일만 이동을 허용한다.
- Tantivy Writer는 인덱스별 단일 소유자, 읽기 Reader는 병렬. 커밋 및 Reader reload 시점을 명시.
- 검색용 인덱스를 원본과 분리하여 **재생성 가능**하게 유지. 인덱스 백업은 일관된 스냅샷/중지 백업 방식으로 검증.
- 폐쇄망 빌드: Cargo.lock, Cargo vendor/내부 레지스트리, npm lockfile+내부 패키지 미러, 서명/해시/SBOM/라이선스 목록을 반입 패키지에 포함.
- 콘텐츠 권한: 기술서적·유료 매뉴얼 등은 내부 검색/전송 권한을 별도 확인하며, 원문 보기 권한은 콘텐츠 라이선스와 사내 정책을 따른다.

## 11. 비기능 요구사항 및 장애 복구

| 구분 | 1차 목표 및 검증 방법 |
|---|---|
| 성능 | 측정 데이터셋·장비 고정 후 P95 검색 ≤ 300ms 목표; 실제 결과 보고 |
| 정확도 | 언어·오류·심볼·한글 질의셋 Recall@10 ≥ 0.90 목표 |
| 메모리 | 데이터 규모별 RSS/peak 측정, 메모리 상한은 PoC 후 결정 |
| 정합성 | 재시작/중단 후 Checkpoint 복구, 중복/유실 0 건 테스트 |
| 안전성 | 비인가 문서 검색·원문 조회 0 건, ACL 테스트 필수 |
| 가용성 | 단일 노드에서 서비스 복구, 건강 점검, 장애 로그, 재색인 절차 |
| 관측성 | 색인 속도, 실패율, 큐 길이, 검색 P95, 인덱스 크기, 최신 수집 시각 |

장애 시: 작업 상태 `pending/running/failed/completed`를 저장 → 재기동 시 미완료 작업 재평가 → Checkpoint 이후부터 안전 재실행 → 인덱스/메타데이터 일치 확인 → 실패 데이터는 dead-letter 목록에서 재처리. 자동 복구 실패는 명확히 알린다.

## 12. Top-down 비빅뱅 구현 계획 — Superpowers 기반

**Superpowers 적용**: `brainstorming`을 본 문서의 요구사항·범위 정리에 적용, `writing-plans`로 파일 경로와 검증 명령이 있는 세부 작업 계획 작성, `using-git-worktrees`로 작업별 격리, `test-driven-development`의 RED→GREEN→REFACTOR, `subagent-driven-development`의 요구사항/코드품질 2단계 리뷰, `verification-before-completion`으로 증거 확인. 사람의 사내 지식 승인과 혼동하지 않는다.

**톱다운 순서**: 외부 계약(검색 결과·원문 API·UI 동선) 정의 → UseCase 인터페이스 → Fake 구현으로 연결 → Tantivy/Parser/DB 실제 Adapter로 교체 → 기능별 테스트 → 수직 기능 릴리스. 인프라를 한꺼번에 만든 뒤 마지막에 통합하지 않는다.

| 단계 | 배포 가능한 수직 기능 | 구현 모듈 | 통과 게이트 |
|---|---|---|---|
| P0 | 앱 기동, `/health/ready`, 인메모리 REST 검색과 React 화면 | domain/search/api/server/frontend | 계약 테스트·UI 스모크 |
| P1 | Markdown/HTML 파일 반입 → 검색 → 블로그 원문 보기 | parser/store/tantivy/indexer/web | 문서 100건 골든 테스트, API E2E |
| P2a | MCP stdio + Streamable HTTP로 AI-Coder 원문 검색 | mcp | 실제 클라이언트 연결 E2E |
| P2 | 형태소·bi-gram/n-gram·BM25·코드 심볼 검색 | analyzer/search | 검색 평가셋, 토큰 골든 테스트 |
| P3 | 메타데이터(요약·태그·키워드) + 관리 UI + 자동 증분 파일 색인 | enrichment/worker/ui | 변경·삭제·재시작 테스트 |
| P4 | SQLite/PG/MySQL/MariaDB DB 소스 색인 | connector + ORM adapter | DB별 fixture/커서/삭제 테스트 |
| P5 | Oracle Connector, Git/SVN·추가 파서(PDF/DOCX/JSP) | adapter/parser | Oracle 환경 계약 테스트, 파서 품질 |
| P7 | Windows Service·보안·운영 복구 / A2A·ACP 선택 어댑터 | windows/security/adapters | 재부팅/복구/ACL/연동 테스트 |
| P8 | 로컬 서브에이전트 분류와 검색 품질 고도화 | enrichment agents | JSON 계약·회귀 테스트 |

단계 순서는 현장 우선순위에 따라 P4~P5를 재배열할 수 있으나, **P1에서 사람·AI의 원문 검색을 끝까지 동작시킨 후 확장**한다. A2A/ACP는 설계 계약만 먼저 고정하고 단계적 구현을 권장한다.

### 12.1 작업 단위 정의

각 작업은 범위와 파일을 적고, **1개의 독립 검증 산출물**만 만든다. 예: `docs/architecture/ADR-0003-tokenizers.md` → `crates/dok3-analyzer/tests/korean_golden.rs` (실패 확인) → `korean.rs` 최소 구현 → `cargo test -p dok3-analyzer` 성공 → 리팩터 → 리뷰 → 커밋. 서로 무관한 모듈을 한 번에 변경하는 PR은 금지한다.

### 12.2 테스트 계층

- **Unit**: parser, chunker, tokenizer, ranking, ACL, cursor 계산, 정책. Fake Port·고정 clock/fixtures 사용.
- **Contract**: SourceConnector, ParserPort, SearchPort, DocumentPort, MCP tool schema, REST response schema.
- **Integration**: Tantivy 임시 인덱스/SQLite 실제 파일, MySQL·PG·MariaDB·Oracle 별도 테스트 DB, DB restart.
- **E2E**: 브라우저 `반입→검색→원문 열람`, MCP `검색→문서 전체 읽기`, DB `수정→삭제→재시작→증분 색인`.
- **Security**: 인가 거부, HTML XSS, SQL 인젝션 방지, 경로 순회, 스니펫 민감정보.
- **Regression**: 한국어·영어·코드 식별자·오류코드 골든 질의 100개 이상과 인덱스 스키마 버전 테스트.

**릴리스 게이트**: `cargo fmt --check`, `cargo clippy --workspace -- -D warnings`, `cargo test --workspace`, UI lint/typecheck/test/build, REST/MCP 계약 테스트, Windows 실행 스모크. 실제 실행 결과 없이는 '검증 완료'로 표기하지 않는다.

## 13. 단위 테스트 예시와 계약 테스트

```rust
#[test]
fn camel_case_identifier_is_split_without_losing_exact_symbol() {
    let tokens = tokenize_symbol("getAccountList");
    assert!(tokens.contains(&"getAccountList".to_string()));
    assert!(tokens.contains(&"account".to_string()));
    assert!(tokens.contains(&"list".to_string()));
}

#[test]
fn second_run_with_same_revision_does_not_duplicate_document() {
    let mut index = FakeIndex::new();
    sync_once(&mut index, fixture_source("rev-1"));
    sync_once(&mut index, fixture_source("rev-1"));
    assert_eq!(index.count_by_id("doc:42"), 1);
}
```

테스트 예시는 **계약 의도**를 보여주며, 타입·함수는 구현 단계에서 만든다. `cargo test`가 성공할 실제 테스트 코드는 해당 기능을 구현하는 스프린트에 작성한다.

### 13.1 개발 완료 정의(DoD)

1. 정의된 UseCase를 API/UI/MCP 중 해당 경로에서 실행할 수 있다.
2. Unit/Contract/Integration 중 해당 범위의 테스트를 **실제 실행하고 로그**를 남긴다.
3. 이전 기능 회귀 테스트가 모두 통과한다.
4. 입력 검증, 오류 코드, 권한 검사, 취소/재시도 경로가 있다.
5. 소스·버전·원문 링크가 검색 결과에서 재현된다.
6. 구조·품질 2단계 리뷰를 거쳐 사용자 시나리오에서 정상 동작한다.

## 14. 아키텍처 결정 기록(ADR)과 미결정 사항

| ADR | 결정 | 비고 |
|---|---|---|
| ADR-001 | Rust/Tantivy 검색 엔진 | Java/JVM 운영 의존 제거 |
| ADR-002 | 모듈형 모놀리스 | Windows 단일 서버/데스크톱 시작 |
| ADR-003 | React 정적 빌드 + Axum 제공 | Node 운영 불필요 |
| ADR-004 | 벡터/임베딩 제외 | 어휘 검색 품질 강화 |
| ADR-005 | Domain Port / DB별 Adapter | Oracle은 SeaORM 직접 미지원 |
| ADR-006 | 최초 파일 문서 검색을 수직 완성 | 비빅뱅 개발 |
| ADR-007 | 승인 워크플로 초기 제외 | 검색·조회 권한 보호는 유지 |
| ADR-008 | MCP 우선, A2A/ACP는 어댑터 | 프로토콜 역할 구분 |

구현 착수 시 결정할 값: Windows 지원 최소 버전, 한글 형태소 분석기 및 사전 라이선스, 공식 문서 반입 범위, 사내 인증 제공 방식, DB별 접속/네트워크, 테스트용 데이터 크기, 외부 기술 문서 라이선스, API 프로토콜 버전 핀. 이 값들은 기능 구현을 멈추게 하는 거대한 선행 단계가 아니라 해당 모듈의 첫 구현 작업에서 고정한다.

## 15. 폐쇄망 완전 내재화와 검색엔진 자체 매뉴얼

**필수 원칙: "인터넷 없이 실행"보다 "인터넷 없이 개발·빌드·테스트·배포·복구할 수 있다"가 더 엄격한 요구사항이다.** Windows에서 필요한 Rust 도구체인, MSVC 링커/Windows SDK, Cargo crates 및 Git 의존성, Node/npm 툴체인과 npm tarball 캐시, React 정적 리소스, Tree-sitter grammar, 한국어 형태소 사전, 네이티브 DLL과 Oracle Client, 테스트 fixture까지 반입 범위로 정의한다.

### 15.1 완전 내재화 인수 기준

1. **Rust**: `Cargo.lock` + `cargo vendor` 산출물 + `.cargo/config.toml`의 로컬 소스 대체 설정을 포함하고 Windows 클린 환경에서 `cargo build --frozen`과 `cargo test --frozen` 성공. 단, `cargo vendor`만으로 rustc, Windows SDK, MSVC 링커, C/C++ 빌드 도구가 포함되지는 않는다.
2. **React**: `package-lock.json`과 모든 npm tarball 또는 사내 npm 미러 포함. `npm ci --offline` 후 UI 빌드 성공. `--prefer-offline`은 캐시 누락 시 네트워크를 시도할 수 있으므로 오프라인 보증으로 사용하지 않는다.
3. **리소스**: CSS/폰트/아이콘/코드 하이라이터/Worker가 외부 CDN을 참조하지 않는다. 모든 파서 grammar, 형태소 분석기 사전, 사용자 정의 용어 사전, 매핑·설정 및 네이티브 라이브러리는 버전·해시를 관리한다.
4. **원문 데이터**: Rust Book, Rust 표준 라이브러리, Cargo Book, Tantivy API, dok3search 사용자·운영·API 매뉴얼을 출처·버전·라이선스와 함께 Knowledge Pack으로 등록한다. 타인의 상업 교재/유료 PDF는 권한 확보 후 반입한다.
5. **자체 매뉴얼 검색**: `dok3search` 사용자·관리자·검색 분석기·REST/MCP·DB 증분 색인·복구 매뉴얼을 기본 제공 문서로 자체 색인한다. 사람은 UI에서 블로그형으로 읽고 AI는 `kb_get_document` 및 `kb_get_section`으로 조회한다.
6. **재현성**: 모든 반입 산출물에 SHA-256, SBOM, 라이선스 목록, 문서 출처, 타깃 플랫폼 정보를 부착한다. 외부 DNS/HTTP가 차단된 서버·빌드 PC에서 설치/색인/검색/복구가 성공해야 한다.

### 15.2 번들 예시

```text
dok3search-airgap-bundle/
  manifest.json | checksums.sha256 | sbom/
  src/ | rust/toolchain/ | rust/vendor/ | rust/build-tools/
  web/node-distribution/ | web/npm-cache/ | web/dist/
  native/oracle-client/ | grammars/ | analyzers/korean-dictionaries/
  knowledge-packs/rust-official/ | knowledge-packs/dok3search-manual/
  tests/ | runbooks/
```

Oracle Client와 MSVC/SDK의 실제 재배포 가능 여부는 각 공급자 라이선스 및 은행 반입 정책에 따른다. 위 디렉터리 구성은 설계 목록이며 상용 라이선스 재배포 허가를 의미하지 않는다. 상세한 설치 명령과 인수 시험은 별도 `dok3search-offline-supplychain.md` 참조.

## 16. 공식 기술 출처와 설계 검증 근거

- Superpowers 개발 방법론: https://github.com/obra/superpowers
- Superpowers `writing-plans`: https://github.com/obra/superpowers/blob/main/skills/writing-plans/SKILL.md
- Tantivy 라이브러리와 Schema/Tokenizer: https://docs.rs/tantivy/latest/tantivy/
- Tantivy NgramTokenizer: https://docs.rs/tantivy/latest/tantivy/tokenizer/struct.NgramTokenizer.html
- SeaORM 지원 DB: https://www.sea-ql.org/SeaORM/docs/install-and-config/database-and-async-runtime/
- Oracle Rust 드라이버 및 클라이언트 요건: https://docs.rs/oracle/latest/oracle/
- Cargo vendor: https://doc.rust-lang.org/cargo/commands/cargo-vendor.html
- Cargo 소스 대체: https://doc.rust-lang.org/cargo/reference/source-replacement.html
- npm ci 및 오프라인 설정: https://docs.npmjs.com/cli/npm-ci/ , https://docs.npmjs.com/cli/v11/using-npm/config/
- Rust Book 및 소스 라이선스: https://github.com/rust-lang/book
- Windows Rust toolchain: https://github.com/rust-lang/rustup/blob/main/doc/user-guide/src/installation/windows.md
- MCP 2026-07-28 전송: https://github.com/modelcontextprotocol/modelcontextprotocol/blob/main/docs/specification/2026-07-28/basic/transports/index.mdx
- MCP Streamable HTTP 호환 차이: https://github.com/modelcontextprotocol/modelcontextprotocol/blob/main/docs/specification/2026-07-28/basic/transports/streamable-http.mdx
- A2A AgentCard: https://a2a-protocol.org/latest/specification/agent-card/
- ACP: https://zed.dev/acp

> 구현 시 주의: 라이브러리 버전·라이선스·Windows ABI·Oracle 클라이언트·한국어 사전 및 MCP 클라이언트 호환성은 실제 설치 환경에서 검증한다. 이 문서는 **설계 및 검증 계획**이지 구현/시험 완료 보고서가 아니다.


## 17. 설계 검토 보완 적용 (필독)

운영 정합성·권한·증분 수집·AI 문서 읽기 관련 필수 수정은 `DESIGN_REVIEW_AND_FIXES.md`를 우선한다. 기존 예시에서 충돌하는 표현이 있으면 이 문서의 보완 결정을 기준으로 구현한다.

## v1.2 색인 세대 전환 아키텍처
전체 재색인은 BUILDING 세대에서 수행하고, 변경 이벤트 재생/정합성/ACL/성능 검증 후 영속 포인터와 런타임 Reader를 전환한다. 자세한 상태 전이, 재시작, 롤백과 한계는 `BLUE_GREEN_DESIGN_REVIEW.md`가 우선한다.

## v1.3 추가 설계 연결

- 실제 구현 참고 사례와 도입 경계: `REFERENCE_ARCHITECTURES.md`
- Rust Workspace와 핵심 Trait: `RUST_WORKSPACE_AND_CONTRACTS.md`
- Blue-Green 스왑/롤백/장애 복구의 상세 계약: `BLUE_GREEN_OPERATIONAL_PROTOCOL.md`

