# dok3search 최종 설계 패키지 v2.2

**2026-11-24 현행화 / Markdown + YAML**

금융권 Windows 폐쇄망에서 사람과 AI 에이전트가 교재·매뉴얼·코드·사내 문서를 검색하고 읽는 플랫폼의 설계다. Rust·Tantivy·React를 사용하고, 운영 배포는 HTTPS 8443을 기준으로 설계한다.

설계 개요와 읽는 순서는 [index.md](index.md)에, 문서 트리는 [index.yaml](index.yaml)에 있다. 화면과 구성요소의 연결은 [map.md](map.md)와 [map.yaml](map.yaml)에서 확인한다. 협업·SSOT 규칙은 [AGENTS.md](AGENTS.md)에 있다.

## 현재 구현

[구현 저장소](https://github.com/LeeYudok/dok3search) `main`(`249a66f` + `ParserRegistry` 배선 PR `118`)은 Axum REST·`/mcp` Streamable HTTP·`--mcp-stdio`와 React 검색·원문·분류 편집·`/admin` 관리 화면을 연결하고(`127.0.0.1:8080`), `DOK3_SOURCE_DIR`+`DOK3_DATA_DIR` 기반 영속 세대 색인(`tantivy-persistent`, 스키마 `v2`)과 `DOK3_SOURCE_DIR` 단독 인메모리 스냅샷(`tantivy-local`), 내장 샘플(`in-memory-fake`) 모드를 지원한다. 검색 상위 분류는 개발 가이드, 운영 매뉴얼, 코드 레퍼런스, 내부 규정, 설계 문서다. Rust·Go 참고 자료는 개발 가이드 안의 `topic` 필터로 검색한다. [PR #61](https://github.com/LeeYudok/dok3search/pull/61)로 한국어 10쿼리 `Recall@5`·`MRR@5`가 `0.20 → 1.00`이 됐고, [PR #62](https://github.com/LeeYudok/dok3search/pull/62)로 자체 운영 매뉴얼이 자동 색인되며 UI·MCP self-search를 통과했다. P3 단계에서는 [PR #60](https://github.com/LeeYudok/dok3search/pull/60)(`PersistentRuntime::refresh()`·`ReaderLease`·크래시 복구)에 이어 [PR #65](https://github.com/LeeYudok/dok3search/pull/65)(주기 수집 워커 `DOK3_REFRESH_INTERVAL_SECS`·`/api/v1/index/jobs*`·`DOK3_ENABLE_ADMIN` 루프백 가드), [PR #68](https://github.com/LeeYudok/dok3search/pull/68)(오프라인 `RuleClassifier`·원문 불변 `classifications.json`·`/api/v1/documents/{id}/classification`·`/docs/:id` 분류 편집 UI), [PR #70](https://github.com/LeeYudok/dok3search/pull/70)(다중 Source `sources.json`·`/api/v1/sources*`·`/admin` 관리 UI)을 모두 완료했다. P3 작업·분류·소스 관리 API와 `/admin` 관리 기능은 `DOK3_SOURCE_DIR`+`DOK3_DATA_DIR` 영속 모드(`tantivy-persistent`)를 전제하며(비영속 모드에서는 `400 INVALID_INPUT`), 분류 조회(`GET`)는 문서 ACL, 분류 변경(`PATCH`) 및 작업·소스 관리 API는 `DOK3_ENABLE_ADMIN=true` 루프백 가드를 따른다. [설계 인덱스](index.md#현재-구현)에 최신 머지 범위와 후속 과제를 기록한다. P4~P8은 크레이트·라우트·스크립트 초안 수준이며, 서버 배선·실제 DB 드라이버·실기 검증은 미완이다(v2.1 정정).

**v2.2 현황(2026-11-24)**: P4-1 SQLite 는 `SqliteSourceAdapter`+`dok3-server` `SourceRegistry`·`source_cursors.json` 증분·전체 수집+`/admin` UI 배선까지 완료했다(#125). P4-2(MySQL/MariaDB)·P4-3(PostgreSQL)은 순수 Rust 드라이버(`PostgresRowReader`·`MysqlRowReader`)·읽기 전용 트랜잭션·TCP/적색화 계약 테스트까지만 완료했고(#124) 서버 Source UI 배선과 실DB 통합 테스트가 남았다. P5-1 Oracle 은 방언 쿼리 추상화만 구현했고 실제 드라이버·통합 테스트는 없다. P7-1 Windows 는 네이티브 SCM 서비스(`--windows-service`)·설치 스크립트까지 완료했고(#126) Windows 실기 재부팅·Event Log 검증이 남았다. 최종 머지 SHA: `425b4af`.

## 우선 읽을 문서
1. `DESIGN_REVIEW_AND_FIXES.md` — 기존 P0/P1 검토
2. `BLUE_GREEN_DESIGN_REVIEW.md` — 세대 전환 설계 검토
3. `BLUE_GREEN_OPERATIONAL_PROTOCOL.md` — **신규: 스왑/정합성/롤백/장애 주입 상세 프로토콜**
4. `RUST_WORKSPACE_AND_CONTRACTS.md` — **신규: Rust 폴더·crate·Trait·단위 테스트 골격**
5. `REFERENCE_ARCHITECTURES.md` — **신규: Quickwit, Meilisearch, Zoekt, Elasticsearch/OpenSearch 참조 패턴과 도입 경계**
6. `dok3search-final-architecture.md` — 전체 아키텍처
7. `dok3search-implementation-roadmap.md` — 탑다운·TDD 단계별 구현
8. `dok3search-network-deployment.md` — 포트/TLS/폐쇄망 네트워크
9. `dok3search-offline-supplychain.md` — 오프라인 빌드/내재화
10. `dok3search-subagents.md`, `dok3search-product-manual-draft.md` — 에이전트/제품 매뉴얼

## 설정과 계약
- `config/*.example.yaml`: 서버, 소스, 색인, 세대, 검색 프로파일, 워크스페이스, 참조 패턴 예시
- `contracts/*.example.yaml`: 문서, 검색, 작업, 섹션 읽기, 세대 전환, 스냅샷 계약

YAML은 운영 구성과 데이터 형식을 설명하는 설계 예시다. 현재 로컬 서버의 실행 명령과 환경 변수는 [구현 저장소 README](https://github.com/LeeYudok/dok3search#로컬-실행-단일-바이너리-또는-개발-프록시)에 있다.

## 구현 우선순위
1. Rust Workspace + 계약 테스트 → 2. Markdown Vertical Slice → 3. 세대 전환/장애 복구 → 4. ACL/증분 동기화 → 5. React/MCP → 6. 다중 DB/파서 → 7. 50만 건 부하 검증.

## 변경 이력
`CHANGELOG.md` 참조. ZIP 파일명은 이후 수정에도 `dok3search-final-md-yaml.zip` 유지.
