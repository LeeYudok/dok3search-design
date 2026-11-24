# dok3search — Superpowers 기반 탑다운 구현 계획

> 작업 단위 = 계약 + 파일 + 실패 테스트(RED) + 최소 구현(GREEN) + 리팩터 + 리뷰 + 검증 로그. 빅뱅 금지.

## 1. 전체 의존 순서

`제품 유스케이스/UX → REST/MCP 계약 → Domain Port → Fake Adapter → File Parser/Indexer 실제 연결 → MCP 기본 검색 → 검색 품질 → 증분·DB → 운영 → 확장 에이전트`.

단계 간 API 변경은 ADR로 기록하고 회귀 테스트를 수행한다. Windows 서비스와 폐쇄망 설치 검증은 P7에서 수행한다.

P0-1~P0-3([PR #12](https://github.com/LeeYudok/dok3search/pull/12)) 이후 P1·P2 크레이트([#22](https://github.com/LeeYudok/dok3search/pull/22)~[#48](https://github.com/LeeYudok/dok3search/pull/48)), 설계 사본 동기화([PR #53](https://github.com/LeeYudok/dok3search/pull/53)·[#66](https://github.com/LeeYudok/dok3search/pull/66)), **P2 런타임 배선 3건**(`dok3-index`+`dok3-analyzer` `SCHEMA_VERSION = 2` 통합 [PR #56](https://github.com/LeeYudok/dok3search/pull/56), `DOK3_DATA_DIR` 영속 스토어·`dok3-index-manager` 세대 전환 [PR #54](https://github.com/LeeYudok/dok3search/pull/54), `dok3-agent` `POST /mcp`·`--mcp-stdio` 라우팅 [PR #55](https://github.com/LeeYudok/dok3search/pull/55)), **P2-2 한국어 정규화**([PR #61](https://github.com/LeeYudok/dok3search/pull/61)), **P2-4 자체 운영 매뉴얼 자동 색인**([PR #62](https://github.com/LeeYudok/dok3search/pull/62)), 그리고 **P3-1~P3-3 전체**(`PersistentRuntime::refresh()`·`ReaderLease`·크래시 복구 [PR #60](https://github.com/LeeYudok/dok3search/pull/60), 주기 수집 워커·작업 상태 API [PR #65](https://github.com/LeeYudok/dok3search/pull/65), 규칙 기반 자동 분류·원문 불변 분류 편집 API/UI [PR #68](https://github.com/LeeYudok/dok3search/pull/68), 다중 Source 레지스트리·`/admin` 관리 UI [PR #70](https://github.com/LeeYudok/dok3search/pull/70))가 [구현 저장소 main](https://github.com/LeeYudok/dok3search) `main`(`249a66f` + `ParserRegistry` 배선 PR `118`)에 모두 반영됐다. Rust·Go 참고 자료([PR #38](https://github.com/LeeYudok/dok3search/pull/38)·[#40](https://github.com/LeeYudok/dok3search/pull/40)·[#42](https://github.com/LeeYudok/dok3search/pull/42))는 개발 가이드의 `topic=rust`와 `topic=go`로 고르며([구현 이슈 #21](https://github.com/LeeYudok/dok3search/issues/21)), 프런트 정적 자료로 분리한다. P4~P8은 크레이트·라우트·스크립트 초안까지만 구현됐고 서버 배선·실기 검증은 끝나지 않았다(v2.1 정정). 현재 후속 과제: P1-2 SQLite 이관(ADR-0001: 현재 `documents.json`/`journal.jsonl`), P2-1 Tree-sitter, P4-1 `SqliteSourceAdapter` 서버·`SourceRegistry` 배선, P4-2·P4-3·P5-1 실제 DB 드라이버(sea-orm/sqlx/oracle)·통합 테스트, P5-2 PDF·DOCX·Git·SVN, P6-1 원문 다운로드(`/original`)·분석 API(`/search/analyze`)·`/admin/analyzers`, P7-1 네이티브 SCM(windows-service)·Event Log·실기 검증, P7-2 툴체인·SBOM·클린 빌드 실증, P8 `AcpAdapter` 서버 배선·오프라인 분류 에이전트·인증.

**v2.2 현황(2026-11-24)**: P4-1 SQLite 는 `SqliteSourceAdapter`+`dok3-server` `SourceRegistry`·`source_cursors.json` 증분·전체 수집+`/admin` UI 배선까지 완료했다(#125). P4-2(MySQL/MariaDB)·P4-3(PostgreSQL)은 순수 Rust 드라이버(`PostgresRowReader`·`MysqlRowReader`)·읽기 전용 트랜잭션·TCP/적색화 계약 테스트까지만 완료했고(#124) 서버 Source UI 배선과 실DB 통합 테스트가 남았다. P5-1 Oracle 은 방언 쿼리 추상화만 구현했고 실제 드라이버·통합 테스트는 없다. P7-1 Windows 는 네이티브 SCM 서비스(`--windows-service`)·설치 스크립트까지 완료했고(#126) Windows 실기 재부팅·Event Log 검증이 남았다. 최종 머지 SHA: `ffc2a5f`.

## 2. 에픽과 작업 분해

| 순서 | 에픽 | 구체 작업 | 테스트·인수 근거 |
|---|---|---|---|
| P0-1 | 뼈대 [완료] | `backend/rust/` Cargo 워크스페이스, React·Vite, 설정 디렉터리 ([PR #2](https://github.com/LeeYudok/dok3search/pull/2)) | Rust·React 빌드 (`ci.yml` [PR #48](https://github.com/LeeYudok/dok3search/pull/48)) |
| P0-2 | Domain [완료] | 문서·절·검색·ACL·변경 DTO와 포트 ([PR #12](https://github.com/LeeYudok/dok3search/pull/12)) | JSON 직렬화, ACL, 요청 정규화 계약 테스트 |
| P0-3 | 수직 UI [완료] | 인메모리 검색 → `POST /api/v1/search` → React 검색·문서 상세 ([PR #12](https://github.com/LeeYudok/dok3search/pull/12)·[#14](https://github.com/LeeYudok/dok3search/pull/14)) | HTTP 검색·절 읽기·오류 응답과 React 연결 검증 |
| P1-1 | Markdown [완료] | `ParserPort`+Markdown 섹션/코드/링크 추출 (`dok3-parser` [PR #24](https://github.com/LeeYudok/dok3search/pull/24)) | `markdown_golden` RED/GREEN |
| P1-2 | Original Store [파일 저장소 완료 / SQLite 이관 잔여] | 파일 원문+SHA-256/`documents.json`+변경 저널 (`dok3-storage` [PR #22](https://github.com/LeeYudok/dok3search/pull/22)·[#27](https://github.com/LeeYudok/dok3search/pull/27)) | 원문 해시·revision·롤백·복구 테스트 |
| P1-3 | Tantivy [완료] | schema, single writer, upsert/delete, reader reload (`dok3-index` [PR #26](https://github.com/LeeYudok/dok3search/pull/26)) | in-memory index tests |
| P1-4 | end-to-end [완료] | 커넥터 337건([PR #29](https://github.com/LeeYudok/dok3search/pull/29))·`DOK3_SOURCE_DIR` `tantivy-local`([PR #43](https://github.com/LeeYudok/dok3search/pull/43))·`DOK3_DATA_DIR` `tantivy-persistent`·세대 전환([PR #36](https://github.com/LeeYudok/dok3search/pull/36)·[#54](https://github.com/LeeYudok/dok3search/pull/54)) 완료 | 커넥터 337건·세대 11/11·콜드/웜/증분 영속 E2E |
| P2-1 | 코드 [식별자·줄 단위 심볼 파서 배선 완료 / Tree-sitter 잔여] | `dok3-analyzer`([PR #31](https://github.com/LeeYudok/dok3search/pull/31))·`dok3-index` v2 통합([PR #56](https://github.com/LeeYudok/dok3search/pull/56))·`dok3-parser` `CodeSymbolParser`(Rust/Java/Go/SQL/JSP) & `ParserRegistry`([PR #108](https://github.com/LeeYudok/dok3search/pull/108)) | 분석기 6/6·심볼/행범위/레지스트리 테스트·Recall/MRR 1.0000 |
| P2-2 | 국문 [완료] | 오프라인 국문 조사 제거·복합어 정규화([PR #61](https://github.com/LeeYudok/dok3search/pull/61)) 및 순수 Rust `mecab-ko-dic` 형태소 코어(`dok3-analyzer::kodic`, `builtin`/`lindera`/`nori-mixed`/`nori-discard`, [PR #100](https://github.com/LeeYudok/dok3search/pull/100)) | Korean golden 40건·4종 프로파일·`Recall@5`/`MRR@5` `0.20 → 1.00`·ACL 위반 0건 |
| P2-3 | n-gram [완료] | `dok3-index` v2(`dok3_text_v2`/`dok3_ngram_v2`) + `dok3-analyzer`([PR #56](https://github.com/LeeYudok/dok3search/pull/56))·`dok3-benchmark`([PR #37](https://github.com/LeeYudok/dok3search/pull/37)) | Recall@5/MRR@5 `0.7143 → 1.0000`, 0/14,000 오류 |
| P2-4 | 운영 매뉴얼 [완료] | `docs/manual/dok3search-manual.md`을 기본 Knowledge Pack(`dok3search-manual`)으로 자동 색인 ([이슈 #58](https://github.com/LeeYudok/dok3search/issues/58)·[PR #62](https://github.com/LeeYudok/dok3search/pull/62)) | UI/MCP manual self-search |
| P2-5 | MCP [런타임 배선 완료] | `dok3-agent`([PR #34](https://github.com/LeeYudok/dok3search/pull/34)) + `dok3-api` `build_router_with_extra` + `dok3-server` `/mcp`·`--mcp-stdio`([PR #55](https://github.com/LeeYudok/dok3search/pull/55)) | HTTP `/mcp`·stdio fail-closed 통합 테스트 6종 |
| P3-1 | 주기 수집 [완료] | `PersistentRuntime::refresh()`·`ReaderLease` 보호·비호환 세대 격리·저널 정합화 ([이슈 #59](https://github.com/LeeYudok/dok3search/issues/59)·[PR #60](https://github.com/LeeYudok/dok3search/pull/60)); `DOK3_REFRESH_INTERVAL_SECS` 워커·`index_jobs.json`·`/api/v1/index/jobs*`·`DOK3_ENABLE_ADMIN` 가드 ([이슈 #63](https://github.com/LeeYudok/dok3search/issues/63)·[PR #65](https://github.com/LeeYudok/dok3search/pull/65)) | 중단/재시작 idempotence·리더 보호·크래시 복구·작업 상태·루프백 가드 |
| P3-2 | 분류 [완료] | 오프라인 `RuleClassifier`·`classifications.json`/`classification_audit.jsonl`·`GET/PATCH /api/v1/documents/{id}/classification`·`/docs/:id` 분류 편집 UI ([이슈 #67](https://github.com/LeeYudok/dok3search/issues/67)·[PR #68](https://github.com/LeeYudok/dok3search/pull/68)) | 분류 JSON/원문(`objects/<sha256>`·본문·`revision_id`) 불변·재색인 반영 |
| P3-3 | 관리 UI [완료] | 다중 Source 레지스트리(`sources.json`)·`GET/POST /api/v1/sources`·`DELETE /api/v1/sources/{id}`·`/admin`(`/admin/sources`, `/admin/jobs`) 관리 UI ([이슈 #69](https://github.com/LeeYudok/dok3search/issues/69)·[PR #70](https://github.com/LeeYudok/dok3search/pull/70)) | `local-markdown` 보호·경로 검증·2단계 삭제/롤백·오류 격리·권한 가드 |
| P4-1 | SQLite [완료: `SqliteSourceAdapter`+`SourceRegistry`·`source_cursors.json` 서버 배선, #125] | `SqliteSourceAdapter` + `dok3-server` `SourceRegistry`·`source_cursors.json`·`/admin` UI 배선(#125) 커서 증분 수집·`sql_guard` SQL-aware SELECT/CTE 검증·`batch_limit`/`has_more` hardening ([PR #110](https://github.com/LeeYudok/dok3search/pull/110)) | DDL fixture, CTE/리터럴 세미콜론, 페이지네이션 회귀 테스트 |
| P4-2 | MySQL/MariaDB [순수 Rust 드라이버·읽기 전용 트랜잭션 완료(#124) / 서버 Source UI 배선·실DB 통합 테스트 잔여] | `RelationalSourceAdapter` + `MysqlRowReader`(`mysql` `minimal-rust`, #124) + `DbDialect::MySql`/`MariaDb` + `TombstoneStrategy` ([PR #110](https://github.com/LeeYudok/dok3search/pull/110)) | 방언별 바인딩·커서·tombstone 계약 테스트 |
| P4-3 | PostgreSQL [순수 Rust 드라이버·읽기 전용 트랜잭션 완료(#124) / 서버 Source UI 배선·실DB 통합 테스트 잔여] | `RelationalSourceAdapter` + `DbDialect::PostgreSql` (`$1/$2`) + `TombstoneStrategy` ([PR #110](https://github.com/LeeYudok/dok3search/pull/110)) | 방언별 바인딩·커서·tombstone 계약 테스트 |
| P5-1 | Oracle [방언 쿼리 추상화만 구현 / 실제 드라이버·통합 테스트 미구현] | `RelationalSourceAdapter` + `DbDialect::Oracle` (`:cursor/:limit`, `FETCH FIRST`) ([PR #110](https://github.com/LeeYudok/dok3search/pull/110)) | Oracle 방언 바인딩·읽기 전용 SELECT 계약 테스트 |
| P5-2 | 문서 [HTML·코드·TXT 런타임 배선 완료 / PDF·DOCX·Git·SVN 미구현] | `HtmlParser`·`CodeSymbolParser`(SQL/JSP 경고)·`PlainTextParser`·`ParserRegistry` ([PR #108](https://github.com/LeeYudok/dok3search/pull/108)) | HTML 골든 17종 + 코드/JSP/레지스트리 테스트 |
| P6-1 | REST [커서·메트릭 완료 / 원문 다운로드·분석 API·/admin/analyzers 잔여] | cursor pagination(`next_cursor`), ACL, 루프백 `/metrics` Prometheus 출력 ([PR #80](https://github.com/LeeYudok/dok3search/pull/80)) | 액세스 차단·커서·메트릭 테스트 |
| P7-1 | Windows [네이티브 SCM 서비스(`--windows-service`)·설치 스크립트 완료(#126) / Windows 실기 재부팅·Event Log 검증 잔여] | `deploy/windows/` Service 등록·제거·자동 복구·smoke 스크립트 ([PR #112](https://github.com/LeeYudok/dok3search/pull/112)) | Windows Service 등록/재시작 복구 절차 |
| P7-2 | 폐쇄망 [Cargo·npm·사전 번들 스크립트 완료 / 툴체인·SBOM·클린 빌드 실증 잔여] | `scripts/offline-bundle.sh` vendor/npm 캐시·`--dict ko-dic.bin` 번들·Windows 체크리스트 ([PR #83](https://github.com/LeeYudok/dok3search/pull/83)·[#112](https://github.com/LeeYudok/dok3search/pull/112)) | 네트워크 0건+frozen build |
| P8 | 확장 [A2A·/llms.txt 라우트 완료 / AcpAdapter 서버 배선·오프라인 분류 에이전트·인증 미구현] | A2A(`/.well-known/agent.json`, `/a2a`)·`AcpAdapter`·`/llms.txt`·Google AX `Workspace` 매니페스트 ([PR #114](https://github.com/LeeYudok/dok3search/pull/114)) | A2A/ACP/ACL 계약 테스트 |

## 3. 구현 작업 템플릿

```markdown
### TASK-ID: [P1-1] Markdown parser
목표: H1/H2/코드블록/링크를 구조적으로 추출한다.
생성/변경 파일:
  crates/dok3-parser/src/markdown.rs
  crates/dok3-parser/tests/markdown_golden.rs
  fixtures/docs/md/*.md
RED: cargo test -p dok3-parser markdown_golden  # 실패를 직접 확인
GREEN: cargo test -p dok3-parser                # 통과 확인
회귀: cargo test --workspace
리뷰1: 요구사항·라인/섹션/코드블록 충족
리뷰2: 에러 처리·경계·성능·불필요 의존 확인
완료 증거: 명령, 실행 로그, 변경 요약, commit hash
```

## 4. 정적 품질·개발 게이트

```powershell
cargo fmt --manifest-path backend/rust/Cargo.toml --all -- --check
cargo clippy --manifest-path backend/rust/Cargo.toml --workspace --all-targets --locked -- -D warnings
cargo test --manifest-path backend/rust/Cargo.toml --workspace --locked
npm --prefix frontend run build
```

위 명령은 [구현 저장소](https://github.com/LeeYudok/dok3search) 루트에서 실행한다. 프런트 빌드는 타입 검사와 Vite 빌드를 함께 실행한다. P0 REST 동작은 `dok3-api`의 HTTP 계약 테스트로 확인한다. 폐쇄망 빌드에는 고정 버전 의존성을 먼저 반입한다.

## 5. PoC 검색 검증셋

- Rust `async/await`, `Result`, `Send/Sync`, `borrow checker`, `tokio::spawn` 등 20개.
- Java/Spring API, `ORA-00904`, SQL 트랜잭션, 로그/설정 등 20개.
- 한글 은행 업무용어·띄어쓰기·부분문자/오타·2gram 등 20개.
- 코드 식별자 `getAccountList`·`snake_case`·`X.Y::Z`·패키지 경로 등 20개.
- 사내 ACL/버전 정합성/문서 원문/원문 인접 절 검색 등 20개.

정답 문서가 있는 100개 질의로 Recall@10, MRR@10을 측정하고, `BM25 only`, `+morph`, `+bi/tri/4gram` 옵션별 비교표를 생성한다. 성능 수치는 CPU/메모리/색인 문서 수와 함께 기록한다.

## 6. 릴리스 인수 기준

1. P1 MVP: Windows에서 문서 업로드/색인 → 검색 → 블로그형 원문·목차 보기 가능.
2. P2: 한글/코드/오류 검색 시 골든 질의 회귀 테스트 통과.
3. P3: 파일 변경·삭제·강제 중지·재기동에도 문서 중복/누락 없음.
4. P4/P5: DB별 읽기 전용 증분·물리 삭제 검증 전략과 실제 테스트 로그.
5. P2: MCP를 통해 실제 클라이언트에서 문서 검색·섹션 조회 가능.
6. P7: 외부 네트워크 없는 Windows 클린 설치·빌드·운영, 제품 매뉴얼 검색 가능.

후속 기능이 미완성이어도 이전 릴리스의 검색·원문 열람은 계속 정상 제공한다.

## v1.2 선행 작업
증분 색인 구현 전 `dok3-index-manager`의 generation state machine/전환 장애 주입 테스트를 작성한다. 1만→10만→50만 문서 단계별 성능 및 검색 동시 부하 게이트를 통과한 뒤 운영 규모를 확정한다.

## v1.3 구현 게이트 추가

- Workspace 골격과 포트 계약을 먼저 만들되 미구현 함수를 실행 경로에 두지 않는다.
- Generation Manager는 스왑 직전 변경분 경계, crash injection, Windows 파일 핸들 테스트를 통과해야 한다.
- 50만 원본/최대 500만 섹션 벤치마크를 색인 중 검색 부하와 함께 측정한다.
- 세부 기준: `RUST_WORKSPACE_AND_CONTRACTS.md`, `BLUE_GREEN_OPERATIONAL_PROTOCOL.md`.

