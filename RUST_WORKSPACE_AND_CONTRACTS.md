# Rust Workspace 골격과 인터페이스 우선 개발

## 원칙
전체 crate 구조·핵심 도메인 타입·포트(Trait)·계약 테스트를 먼저 고정한다. **수백 개의 빈 함수나 실행 경로의 `todo!()`는 만들지 않는다.** 기능별 Vertical Slice로 RED → GREEN → REFACTOR를 반복한다.

아래 경로는 [구현 저장소](https://github.com/LeeYudok/dok3search)의 구성이다. P0 구현은 `dok3-core`·`dok3-domain`의 계약 타입과 `dok3-search`·`dok3-api`·`dok3-server`의 인메모리 HTTP 검색을 제공한다([PR #12](https://github.com/LeeYudok/dok3search/pull/12), [PR #14](https://github.com/LeeYudok/dok3search/pull/14)).

### P1·P2·P3 구현·런타임 배선 현황 (`main` `249a66f` + `ParserRegistry` 배선 PR `118` 기준, 2026-11-24)
- **`dok3-storage` (P1-2·영속 배선·원문 불변 강화 완료, [PR #22](https://github.com/LeeYudok/dok3search/pull/22)·[PR #27](https://github.com/LeeYudok/dok3search/pull/27)·[PR #54](https://github.com/LeeYudok/dok3search/pull/54)·[PR #68](https://github.com/LeeYudok/dok3search/pull/68))**: `FileDocumentStore`·`FileChangeJournal`·단일 writer 락·손상 복구를 구현했고, `DOK3_DATA_DIR` 설정 시 `dok3-server`에 영속 연결됐다(`document_ids`/`list_documents`/`last_sequence`). P3-2에서는 `put_immutable`·`update_metadata`로 기존 `objects/<sha256>`·`content_hash`·`revision_id`·`body`·`markdown`·`source_uri` 불변성을 보장한다.
- **`dok3-parser` (P1-1 완료, [PR #24](https://github.com/LeeYudok/dok3search/pull/24))**: 외부 크레이트 0개 순수 Rust `MarkdownParser`(`DocumentParserPort`), H1~H3 섹션·코드 펜스·유니코드/한글 중복 방지 앵커 및 골든 테스트 7종을 반영했다.
- **`dok3-analyzer` + `dok3-index` 통합 (P1-3·P2-1 코드 식별자 통합·P2-2 완료; Tree-sitter Rust/Java 잔여, [PR #26](https://github.com/LeeYudok/dok3search/pull/26)·[PR #31](https://github.com/LeeYudok/dok3search/pull/31)·[PR #56](https://github.com/LeeYudok/dok3search/pull/56)·[PR #61](https://github.com/LeeYudok/dok3search/pull/61))**: `dok3_analyzer::analyze_text`를 `TantivyIndex` 스키마 `v2`(`SCHEMA_VERSION = 2`, `dok3_text_v2`·`dok3_ngram_v2`)의 색인·질의 경로에 연결했다(기존 v1 디스크 색인은 새 세대 재색인 필요). [PR #61](https://github.com/LeeYudok/dok3search/pull/61)은 오프라인 국문 조사를 제거하고 금융/개발 복합어를 정규화했으며, 한국어 골든 40건과 한국어 10쿼리 `Recall@5`·`MRR@5` `0.20 → 1.00`, ACL 위반 0건을 확인했다.
- **`dok3-benchmark` 비교 하네스 (P2-6 완료, [PR #37](https://github.com/LeeYudok/dok3search/pull/37)·[PR #56](https://github.com/LeeYudok/dok3search/pull/56))**: v1 대비 v2 비교 실측에서 `Recall@5` `0.7143 → 1.0000`, `MRR@5` `0.7143 → 1.0000`, ACL 누출 `0건`, 부하 요청 오류 `0/14,000`, 최대 질의 p95 `0.3054 ms → 0.3191 ms`를 확인했다.
- **`dok3-connectors` + `dok3-index-manager` + `dok3-server` (P1-4a/b·영속 세대 완료, [PR #29](https://github.com/LeeYudok/dok3search/pull/29)·[PR #36](https://github.com/LeeYudok/dok3search/pull/36)·[PR #43](https://github.com/LeeYudok/dok3search/pull/43)·[PR #54](https://github.com/LeeYudok/dok3search/pull/54))**: `DOK3_SOURCE_DIR`+`DOK3_DATA_DIR` 지정 시 `tantivy-persistent`(콜드 색인·웜 재사용·증분 세대 교체 및 퇴역 GC), `DOK3_SOURCE_DIR`만 지정 시 `tantivy-local`, 미지정 시 `in-memory-fake`로 동작한다(`DOK3_BIND` 기본 `127.0.0.1`, 비루프백 거부).
- **`dok3-agent` + `dok3-api` MCP 런타임 배선 (P2-5 완료, [PR #34](https://github.com/LeeYudok/dok3search/pull/34)·[PR #55](https://github.com/LeeYudok/dok3search/pull/55))**: `build_router_with_extra`로 `POST /mcp` Streamable HTTP를 SPA fallback 이전·보안 헤더 안쪽에 병합하고, `--mcp-stdio` 플래그와 `stderr` 진단 분리(`print_build_report` `eprintln!`)를 적용했다(`Principal::anonymous_public()` 고정, fail-closed).
- **`dok3-search` + `dok3-server` 자체 운영 매뉴얼 (P2-4 완료, [PR #62](https://github.com/LeeYudok/dok3search/pull/62))**: `docs/manual/dok3search-manual.md`(15개 H2 절)를 Knowledge Pack(`dok3search-manual`, `category: manual`)으로 자동 적재하고 `frontend/src/documents.ts`와 `vite.config.ts` `server.fs.allow`를 보완했다. UI·MCP `kb_search`/`kb_get_document`/`kb_get_section` self-search를 검증했다.
- **`dok3-server` 영속 런타임·주기 수집·작업 API (P3-1 완료, [PR #60](https://github.com/LeeYudok/dok3search/pull/60)·[PR #65](https://github.com/LeeYudok/dok3search/pull/65))**: `PersistentRuntime::refresh()`·`RefreshReport`·`ReaderLease` 보호·비호환 세대 격리(`quarantine_incompatible_generations`)·저널 정합화(`reconcile_store_journal`)에 더해 `DOK3_REFRESH_INTERVAL_SECS` 주기 수집 워커 스케줄러, `index_jobs.json`, `GET/POST /api/v1/index/jobs`, `GET /api/v1/index/jobs/{job_id}`, `DOK3_ENABLE_ADMIN` IPv4 루프백 가드를 구현했다. `IndexJobPort`·`ClassificationPort`·`SourceRegistryPort`는 `DOK3_SOURCE_DIR`+`DOK3_DATA_DIR` 영속 모드(`tantivy-persistent`)에서만 연결되며 비영속 모드에서는 `400 INVALID_INPUT`을 반환한다.
- **규칙 기반 자동 분류·원문 불변 편집 (`P3-2` 완료, [PR #68](https://github.com/LeeYudok/dok3search/pull/68))**: `dok3-server` 오프라인 `RuleClassifier`와 분리 저장소 `ClassificationStore`(`classifications.json`·`classification_audit.jsonl`, `pending_index_sync`), `GET/PATCH /api/v1/documents/{id}/classification`(`GET`은 문서 ACL 적용, `PATCH`만 `require_admin_for_mutations_middleware` 관리자+CSRF+JSON 가드 적용), `/docs/:id` 분류 편집·초기화 UI를 구현했다.
- **다중 Source 레지스트리·관리 UI (`P3-3` 완료, [PR #70](https://github.com/LeeYudok/dok3search/pull/70))**: `SourceRegistry`(`sources.json`, `local-markdown` `managed_by_env: true` 보호, 비심볼릭 디렉터리 검증, 2단계 삭제/롤백, 소스별 `last_error` 격리, 디스크 원본 비삭제), `GET/POST /api/v1/sources`, `DELETE /api/v1/sources/{id}` (`require_admin_middleware` 전체 메서드 `DOK3_ENABLE_ADMIN=true` 루프백 가드 적용), `/admin`·`/admin/sources`·`/admin/jobs` 관리 UI 및 `AGENTS.md`/`AGENTS.yaml` 현행화를 완료했다.
- **다음 구현 과제 (`P4` DB 어댑터 · `P2-1` Tree-sitter · `P5~P8`)**: SQLite·MySQL/MariaDB·PostgreSQL 증분 커넥터 및 계약 테스트, Tree-sitter Rust/Java 보강, 50만 건 부하·Windows 서비스·폐쇄망 설치 번들을 순차 진행한다.

```text
dok3search/
  backend/rust/
    Cargo.toml               # workspace members, resolver = "3"
    Cargo.lock
    rust-toolchain.toml
    crates/
      dok3-core/             # 오류(Dok3Error), 설정(DOK3_SOURCE_DIR·DOK3_DATA_DIR·DOK3_REFRESH_INTERVAL_SECS·DOK3_ENABLE_ADMIN·DOK3_ANALYZER_PROFILE·DOK3_DICT_PATH·DOK3_USER_DICT_PATH 등), 공통 타입
      dok3-domain/           # DocumentId, RevisionId, SectionId, GenerationId, Port Trait(검색·저장·작업·분류·소스)
      dok3-storage/          # 파일 원문 SHA-256 저장소·JSONL 변경 저널(P1-2 파일 저장소·영속 배선·원문 불변 강화 완료; SQLite 이관 잔여, ADR-0001)
      dok3-connectors/       # 로컬 파일, SQLite(`sql_guard`+`batch_limit`), SQLite(`dok3-server` `SourceRegistry` 배선 완료), MySQL/MariaDB·PostgreSQL 순수 Rust `DbRowReader`(`MysqlRowReader`·`PostgresRowReader`, 읽기 전용 트랜잭션; 서버 Source UI 배선·실DB 통합 테스트 잔여), Oracle 방언 쿼리 추상화만 구현(실제 드라이버 없음; P5-1 잔여)
      dok3-parser/           # Markdown, HTML, CodeSymbolParser(Rust/Java/Go/SQL/JSP), PlainTextParser, ParserRegistry(dok3-server 다중 확장자 런타임 배선 완료; Tree-sitter·PDF·DOCX·Git·SVN 잔여)
      dok3-analyzer/         # 코드 식별자·오류코드·문자 n-gram·순수 Rust mecab-ko-dic 형태소 분석기(`kodic`, 프로파일: `builtin`/`lindera`/`nori-mixed`/`nori-discard`) 및 사용자 사전 지원
      dok3-index/            # TantivyIndex(SearchPort), schema v2(dok3-analyzer 통합), 수동 reader reload
      dok3-index-manager/    # Blue-Green 세대 빌드·검증·활성화·복구 상태 머신(DOK3_DATA_DIR 배선 완료)
      dok3-search/           # 검색, ACL 필터, 원문 접근 응용 서비스
      dok3-agent/            # MCP JSON-RPC stdio/HTTP, A2A(`/.well-known/agent.json`, `/a2a`), ACP, `/llms.txt` 어댑터(A2A·`/llms.txt` 라우트만 완료; `AcpAdapter` 서버 배선·인증 미구현)
      dok3-api/              # Axum 라우터(`build_router_with_extra`)·검색·문서 조회·작업/분류/소스 API·관리자 가드
      dok3-benchmark/        # PoC 검색 품질(v1 vs v2)·동시 부하 하네스(P2-6 완료)
      dok3-server/           # HTTP(`tantivy-persistent`/`tantivy-local`/`in-memory-fake`)·워커·분류·소스 레지스트리·`/mcp`·`--mcp-stdio` 진입점
  frontend/                  # React + TypeScript
  config/                    # 예시 YAML
  contracts/                 # YAML 계약 및 버전
  tests/                     # 통합/장애 주입/성능
  docs/                      # 설계/운영/자체 매뉴얼
  scripts/                   # 오프라인 빌드/설계 사본/AGENTS.yaml 검증
```

## 핵심 Port 계약 (`dok3-domain/src/ports.rs` 구현 기준)
```rust
pub trait SearchPort: Send + Sync {
    fn search(&self, principal: &Principal, req: &SearchRequest) -> Result<SearchResponse, Dok3Error>;
    fn snapshot(&self) -> SearchSnapshotMeta;
}

pub trait DocumentStorePort: Send + Sync {
    fn get_document(&self, principal: &Principal, id: &DocumentId) -> Result<DocumentDetail, Dok3Error>;
    fn get_section(&self, principal: &Principal, req: &SectionReadRequest) -> Result<SectionReadResponse, Dok3Error>;
    fn document_count(&self) -> usize;
}

pub trait DocumentParserPort: Send + Sync {
    fn supports(&self, media_type: &str, filename: &str) -> bool;
    fn parse(&self, filename: &str, content: &[u8]) -> Result<ParsedDocument, Dok3Error>;
}

pub trait ChangeJournalPort: Send + Sync {
    fn append(&self, change: ChangeEvent) -> Result<Sequence, Dok3Error>;
    fn read_after(&self, seq: Sequence, limit: usize) -> Result<Vec<ChangeEvent>, Dok3Error>;
}

pub trait IndexJobPort: Send + Sync {
    fn list_jobs(&self) -> Result<IndexJobListResponse, Dok3Error>;
    fn get_job(&self, job_id: &str) -> Result<IndexJobRecord, Dok3Error>;
    fn trigger_job(&self, req: &TriggerIndexJobRequest) -> Result<IndexJobRecord, Dok3Error>;
}

pub trait ClassificationPort: Send + Sync {
    fn get_classification(
        &self,
        principal: &Principal,
        id: &DocumentId,
    ) -> Result<DocumentClassification, Dok3Error>;
    fn update_classification(
        &self,
        principal: &Principal,
        id: &DocumentId,
        req: &UpdateClassificationRequest,
    ) -> Result<DocumentClassification, Dok3Error>;
}

pub trait SourceRegistryPort: Send + Sync {
    fn list_sources(&self) -> Result<SourceListResponse, Dok3Error>;
    fn register_source(&self, req: &CreateSourceRequest) -> Result<RegisteredSource, Dok3Error>;
    fn delete_source(&self, source_id: &str) -> Result<DeleteSourceResponse, Dok3Error>;
}
```

### 후속 `dok3-index-manager` 예정 Port 스케치 (미구현)
```rust
pub trait IndexBuilder: Send + Sync {
    fn build(&self, generation: GenerationId, snapshot: SnapshotToken) -> Result<BuildReport, IndexError>;
    fn replay_until(&self, generation: GenerationId, seq: Sequence) -> Result<Sequence, IndexError>;
    fn validate(&self, generation: GenerationId) -> Result<ValidationReport, IndexError>;
}
```
> 예정 인터페이스는 설계 스케치이며 실행 경로에 `todo!()` 를 두지 않는다. 세대 전환 동시성·복구 세부 계약은 `BLUE_GREEN_OPERATIONAL_PROTOCOL.md` 와 통합 테스트로 확정한다.

## 오류 코드 계약 (`dok3-core::Dok3Error`)
API 응답과 감사 로그는 아래 안정적인 대문자 코드를 사용한다.

| Variant | Code | HTTP/용도 의미 |
|---|---|---|
| `InvalidInput` | `INVALID_INPUT` | 요청 형식·질의·파라미터 오류 (HTTP 400) |
| `NotFound` | `NOT_FOUND` | 문서·절 미존재 (HTTP 404) |
| `Forbidden` | `FORBIDDEN` | ACL 권한 부족 (HTTP 403) |
| `Parse` | `PARSE_ERROR` | UTF-8 디코딩·문서 구문 파싱 실패 |
| `Index` | `INDEX_ERROR` | Tantivy 색인·리더·검색 엔진 오류 |
| `Storage` | `STORAGE_ERROR` | 원문 저장소·변경 저널 I/O 및 무결성 오류 |
| `Internal` | `INTERNAL_ERROR` | 예기치 못한 내부 상태 오류 (예: 락 오염) |

## 단계별 산출물
1. Workspace, 오프라인 빌드, 계약 타입, P0-3 인메모리 HTTP 검색·React UI (`main` 머지 완료)
2. P1 크레이트(`dok3-storage`, `dok3-parser`, `dok3-index`, `dok3-connectors`, `dok3-index-manager`) 및 `DOK3_SOURCE_DIR` 로컬 Markdown `tantivy-local` 색인 경로 (`main` 머지 완료)
3. P2 크레이트(`dok3-analyzer`, `dok3-agent`, `dok3-benchmark`) 및 CI 필수 체크 도입 (`main` 머지 완료)
4. P2 런타임 배선: `dok3-index`+`dok3-analyzer` `SCHEMA_VERSION = 2` 통합([PR #56](https://github.com/LeeYudok/dok3search/pull/56)), `DOK3_DATA_DIR` 영속 세대 전환([PR #54](https://github.com/LeeYudok/dok3search/pull/54)), `POST /mcp`·`--mcp-stdio`([PR #55](https://github.com/LeeYudok/dok3search/pull/55)) (`main` 머지 완료)
5. 국문 조사/복합어 정규화(P2-2), 자체 운영 매뉴얼 자동 색인(P2-4), 주기 수집·분류·관리 UI(P3-1~P3-3) `main` 머지 완료; Tree-sitter 보강(P2-1) 잔여
6. 잔여: 실제 DB 드라이버·통합 테스트와 서버 배선(P4~P5), 50만 건 벤치마크, 네이티브 Windows 서비스·실기 검증 및 폐쇄망 클린 빌드 실증(P7), `AcpAdapter` 배선·인증(P8)

## 의존성 방향
`dok3-domain`은 외부 크레이트 구현(Tantivy, Axum, SQLite 드라이버 등)에 의존하지 않는다. `dok3-parser`는 `dok3-index`를 직접 의존하거나 수정하지 않는다. `dok3-index-manager`는 `dok3-storage`와 `dok3-index`를 조합해 세대 빌드·전환·복구를 담당하며, `dok3-api`와 `dok3-agent`는 `dok3-search` 응용 서비스를 호출하고 도메인 로직을 중복 구현하지 않는다.
