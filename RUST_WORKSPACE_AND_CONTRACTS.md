# Rust Workspace 골격과 인터페이스 우선 개발

## 원칙
전체 crate 구조·핵심 도메인 타입·포트(Trait)·계약 테스트를 먼저 고정한다. **수백 개의 빈 함수나 실행 경로의 `todo!()`는 만들지 않는다.** 기능별 Vertical Slice로 RED → GREEN → REFACTOR를 반복한다.

아래 경로는 [구현 저장소](https://github.com/LeeYudok/dok3search)의 구성이다. P0 구현은 `dok3-core`·`dok3-domain`의 계약 타입과 `dok3-search`·`dok3-api`·`dok3-server`의 인메모리 HTTP 검색을 제공한다([PR #12](https://github.com/LeeYudok/dok3search/pull/12), [PR #14](https://github.com/LeeYudok/dok3search/pull/14)).

### P1·P2 구현 현황 (`main` 머지 기준, 2026-11-22)
- **`dok3-storage` (P1-2, `main` 머지됨 [PR #22](https://github.com/LeeYudok/dok3search/pull/22)·[PR #27](https://github.com/LeeYudok/dok3search/pull/27))**: `FileDocumentStore`(SHA-256 `objects/<sha256>` + 원자 `documents.json`), `FileChangeJournal`(`fsync` `journal.jsonl` + torn-write 복구), 단일 writer 락·저널 append 롤백·`dir_sync` 재시도·손상 복구 및 SQLite 전환 ADR(`docs/adr/0001-p1-file-storage-and-sqlite-migration.md`)을 반영했다.
- **`dok3-parser` (P1-1, `main` 머지됨 [PR #24](https://github.com/LeeYudok/dok3search/pull/24))**: 외부 크레이트 0개 순수 Rust `MarkdownParser`(`DocumentParserPort`), H1~H3 평탄 타일링 섹션·코드 펜스·유니코드/한글 중복 방지 앵커 및 골든 테스트 7종을 반영했다.
- **`dok3-index` (P1-3, `main` 머지됨 [PR #26](https://github.com/LeeYudok/dok3search/pull/26))**: Tantivy 0.27 기반 `TantivyIndex`(`SearchPort`), `dok3_text` + 2~3 n-gram `dok3_ngram` 토크나이저, 단일 `Mutex<IndexWriter>`, `ReloadPolicy::Manual` 및 `reload()` 발행 잠금 일원화, 히트·카테고리 집계 선 ACL 적용을 반영했다.
- **`dok3-connectors` (P1-4a, `main` 머지됨 [PR #29](https://github.com/LeeYudok/dok3search/pull/29))**: 로컬 디렉터리 결정적 순회·숨김/심볼릭 링크 차단·크기 상한 skip 처리 및 337건 오프라인 Markdown 샘플 수집·파싱 통합 테스트를 반영했다.
- **`dok3-server` 로컬 Markdown 색인 (P1-4a 런타임, `main` 머지됨 [PR #43](https://github.com/LeeYudok/dok3search/pull/43)·[PR #46](https://github.com/LeeYudok/dok3search/pull/46))**: `DOK3_SOURCE_DIR` 설정 시(IPv4 루프백 `DOK3_BIND`, 기본 `127.0.0.1`) 기동 전 `LocalFileConnector` + `MarkdownParser` + `TantivyIndex::create_in_ram`으로 `tantivy-local` 스냅샷을 구성하며(초기 색인 실패 시 기동 중단), 미설정 시 `in-memory-fake`로 동작한다.
- **`dok3-index-manager` (P1-4b, `main` 머지됨 [PR #36](https://github.com/LeeYudok/dok3search/pull/36))**: Blue-Green 세대 전환 상태 머신·단일 writer 파일 제어 포인터(`generation_state.json`)·크래시 복구·롤백 저널 catch-up 가드·`ReaderLease` 참조 보호·8종 실패 주입 테스트(11/11)를 반영했다.
- **`dok3-analyzer` (P2-1, `main` 머지됨 [PR #31](https://github.com/LeeYudok/dok3search/pull/31))**: 외부 크레이트 0개 순수 Rust `camelCase`/`PascalCase`/`snake_case`/`::`/오류코드 분해 및 문자 단위 2~3 n-gram 분석기(6/6)를 반영했다.
- **`dok3-agent` (P2-5, `main` 머지됨 [PR #34](https://github.com/LeeYudok/dok3search/pull/34))**: MCP JSON-RPC 2.0 stdio + Streamable HTTP(`POST /mcp`) 서버 어댑터(`2025-11-25` `initialize` 및 `2026-07-28` `server/discover`/`Mcp-Method`/`Mcp-Name`/`-32020`/`-32022` 동시 호환), SSOT 7개 `kb_*` 도구, 요청별 `PrincipalResolver` ACL 강제를 반영했다.
- **`dok3-benchmark` (P2-6, `main` 머지됨 [PR #37](https://github.com/LeeYudok/dok3search/pull/37))**: PoC 검색 품질(Recall@K·MRR@K·ACL 누출 검사) 및 동시 검색 부하(P50/P95/P99·QPS) 측정 하네스·CLI(1 unit + 7 contract)를 반영했다.
- **다음 런타임 배선 과제**: `dok3-analyzer`·`dok3-index-manager`·`dok3-agent`·`dok3-storage` 영속 스토어는 현재 `dok3-server`/`dok3-api`/`dok3-index` 소스 경로에 아직 연결되지 않았으며(`Cargo.toml` 워크스페이스 의존만 선언), 후속 통합 PR에서 배선한다.

```text
dok3search/
  backend/rust/
    Cargo.toml               # workspace members, resolver = "3"
    Cargo.lock
    rust-toolchain.toml
    crates/
      dok3-core/             # 오류(Dok3Error), 설정(DOK3_SOURCE_DIR 등), 공통 타입
      dok3-domain/           # DocumentId, RevisionId, SectionId, GenerationId, Port Trait
      dok3-storage/          # 파일 원문 SHA-256 저장소·JSONL 변경 저널(P1-2 완료)
      dok3-connectors/       # 로컬 파일 Markdown 커넥터(P1-4a 완료), 향후 Git, SVN, DB 소스 확장
      dok3-parser/           # MarkdownParser(P1-1 완료), 향후 HTML/PDF/DOCX/Tree-sitter
      dok3-analyzer/         # 코드 식별자·오류코드·문자 n-gram 분석기(P2-1 크레이트 완료)
      dok3-index/            # TantivyIndex(SearchPort), schema/tokenizer, 단일 writer, 수동 reader reload(P1-3 완료)
      dok3-index-manager/    # Blue-Green 세대 빌드·검증·활성화·복구 상태 머신(P1-4b 크레이트 완료)
      dok3-search/           # 검색, ACL 필터, 원문 접근 응용 서비스
      dok3-agent/            # MCP JSON-RPC stdio/Streamable HTTP 어댑터(P2-5 크레이트 완료)
      dok3-api/              # Axum 라우터와 검색·문서 조회
      dok3-benchmark/        # PoC 검색 품질·동시 부하 하네스(P2-6 완료)
      dok3-server/           # HTTP 서버 진입점(`tantivy-local` / `in-memory-fake`)
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
4. 후속 런타임 배선: `dok3-server`/`dok3-api`/`dok3-index`에 영속 `FileDocumentStore`·`dok3-index-manager` 세대 전환·`dok3-agent` `/mcp`·`dok3-analyzer` 연결 (예정)
5. 지속 변경 이벤트, ACL, 변경/삭제 동기화, 국문 형태소·Tree-sitter 보강
6. DB 어댑터별 계약 테스트, 50만 건 벤치마크, Windows 서비스 및 폐쇄망 설치 번들

## 의존성 방향
`dok3-domain`은 외부 크레이트 구현(Tantivy, Axum, SQLite 드라이버 등)에 의존하지 않는다. `dok3-parser`는 `dok3-index`를 직접 의존하거나 수정하지 않는다. `dok3-index-manager`는 `dok3-storage`와 `dok3-index`를 조합해 세대 빌드·전환·복구를 담당하며, `dok3-api`와 `dok3-agent`는 `dok3-search` 응용 서비스를 호출하고 도메인 로직을 중복 구현하지 않는다.
