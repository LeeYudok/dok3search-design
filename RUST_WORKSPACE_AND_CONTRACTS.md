# Rust Workspace 골격과 인터페이스 우선 개발

## 원칙
전체 crate 구조·핵심 도메인 타입·포트(Trait)·계약 테스트를 먼저 고정한다. **수백 개의 빈 함수나 실행 경로의 `todo!()`는 만들지 않는다.** 기능별 Vertical Slice로 RED → GREEN → REFACTOR를 반복한다.

아래 경로는 [구현 저장소](https://github.com/LeeYudok/dok3search)의 구성이다. P0 구현은 `dok3-core`·`dok3-domain`의 계약 타입과 `dok3-search`·`dok3-api`·`dok3-server`의 인메모리 HTTP 검색을 제공한다([PR #12](https://github.com/LeeYudok/dok3search/pull/12), [PR #14](https://github.com/LeeYudok/dok3search/pull/14)).

### P1 구현 현황 (2026-11-22 기준)
- **`dok3-storage` (P1-2, `main` 머지됨 [PR #22](https://github.com/LeeYudok/dok3search/pull/22))**: `FileDocumentStore`(SHA-256 `objects/<sha256>` + 원자 `documents.json`)와 `FileChangeJournal`(`fsync` `journal.jsonl` + torn-write 복구)을 구현했다. 교차 리뷰에서 도출된 단일 writer 락·저널 append 보강·SQLite 메타데이터 전환은 후속 이슈 [#23](https://github.com/LeeYudok/dok3search/issues/23)으로 추적한다.
- **`dok3-parser` (P1-1, [PR #24](https://github.com/LeeYudok/dok3search/pull/24) 리뷰 완료/승인 대기)**: 외부 크레이트 0개 순수 Rust `MarkdownParser`(`DocumentParserPort`), H1~H3 평탄 타일링 섹션·코드 펜스·유니코드/한글 중복 방지 앵커 및 골든 테스트 7종을 구현했다.
- **`dok3-index` (P1-3, [PR #26](https://github.com/LeeYudok/dok3search/pull/26) 리뷰/승인 대기)**: Tantivy 0.27 기반 `TantivyIndex`(`SearchPort`), `dok3_text` + 2~3 n-gram `dok3_ngram` 토크나이저, 단일 `Mutex<IndexWriter>`, `ReloadPolicy::Manual`, 히트·카테고리 집계 선 ACL 적용을 구현했다.

```text
dok3search/
  backend/rust/
    Cargo.toml               # workspace members, resolver = "3"
    Cargo.lock
    rust-toolchain.toml
    crates/
      dok3-core/             # 오류(Dok3Error), 설정, 공통 타입
      dok3-domain/           # DocumentId, RevisionId, SectionId, GenerationId, Port Trait
      dok3-storage/          # 파일 원문 SHA-256 저장소·JSONL 변경 저널(P1-2); SQLite 메타·락 보강(#23)
      dok3-connectors/       # 파일, Git, SVN, DB 소스 포트
      dok3-parser/           # MarkdownParser(P1-1 완료), 향후 HTML/PDF/DOCX/Tree-sitter
      dok3-analyzer/         # 한국어/영어/코드 분석기
      dok3-index/            # TantivyIndex(SearchPort), schema/tokenizer, 단일 writer, 수동 reader reload(P1-3)
      dok3-index-manager/    # 세대 빌드/검증/활성화/복구
      dok3-search/           # 검색, ACL 필터, 원문 접근 응용 서비스
      dok3-agent/            # MCP 우선, A2A/ACP 후속
      dok3-api/              # Axum 라우터와 검색·문서 조회
      dok3-benchmark/        # 재현 가능한 부하 생성/측정
      dok3-server/           # HTTP 서버 진입점
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
2. `dok3-storage` + `dok3-parser` + `dok3-index` → P1-4 Markdown 100건 색인·검색·절 조회 end-to-end 연결 (진행 중)
3. Generation Manager(`dok3-index-manager`): 신규 세대 빌드 → 검증 → 교체 → 복구
4. 지속 변경 이벤트, ACL, 변경/삭제 동기화
5. MCP 도구, 코드(Tree-sitter)·한국어 분석기·품질 측정셋
6. DB 어댑터별 계약 테스트, 50만 건 벤치마크, Windows 서비스 및 폐쇄망 설치 번들

## 의존성 방향
`dok3-domain`은 외부 크레이트 구현(Tantivy, Axum, SQLite 드라이버 등)에 의존하지 않는다. `dok3-parser`는 `dok3-index`를 직접 의존하거나 수정하지 않는다. `dok3-index-manager`는 `dok3-storage`와 `dok3-index`를 조합해 세대 빌드·전환·복구를 담당하며, `dok3-api`와 `dok3-agent`는 `dok3-search` 응용 서비스를 호출하고 도메인 로직을 중복 구현하지 않는다.
