# dok3search — Superpowers 기반 탑다운 구현 계획

> 작업 단위 = 계약 + 파일 + 실패 테스트(RED) + 최소 구현(GREEN) + 리팩터 + 리뷰 + 검증 로그. 빅뱅 금지.

## 1. 전체 의존 순서

`제품 유스케이스/UX → REST/MCP 계약 → Domain Port → Fake Adapter → File Parser/Indexer 실제 연결 → MCP 기본 검색 → 검색 품질 → 증분·DB → 운영 → 확장 에이전트`.

단계 간 API 변경은 ADR로 기록하고 회귀 테스트를 수행한다. Windows 서비스와 폐쇄망 설치 검증은 P7에서 수행한다.

P0-1~P0-3([PR #12](https://github.com/LeeYudok/dok3search/pull/12)) 이후 P1 크레이트(`dok3-storage` [PR #22](https://github.com/LeeYudok/dok3search/pull/22)·[#27](https://github.com/LeeYudok/dok3search/pull/27), `dok3-parser` [#24](https://github.com/LeeYudok/dok3search/pull/24), `dok3-index` [#26](https://github.com/LeeYudok/dok3search/pull/26), `dok3-connectors` [#29](https://github.com/LeeYudok/dok3search/pull/29), `dok3-index-manager` [#36](https://github.com/LeeYudok/dok3search/pull/36)), P2 크레이트(`dok3-analyzer` [#31](https://github.com/LeeYudok/dok3search/pull/31), `dok3-agent` [#34](https://github.com/LeeYudok/dok3search/pull/34), `dok3-benchmark` [#37](https://github.com/LeeYudok/dok3search/pull/37)), `DOK3_SOURCE_DIR` 기반 로컬 Markdown `tantivy-local` 색인 서버 경로([PR #43](https://github.com/LeeYudok/dok3search/pull/43), [#46](https://github.com/LeeYudok/dok3search/pull/46) 컴파일 오류 수정), 그리고 PR 필수 체크([PR #48](https://github.com/LeeYudok/dok3search/pull/48))가 [구현 저장소 main](https://github.com/LeeYudok/dok3search)에 반영됐다. Rust·Go 참고 자료([PR #38](https://github.com/LeeYudok/dok3search/pull/38)·[#40](https://github.com/LeeYudok/dok3search/pull/40)·[#42](https://github.com/LeeYudok/dok3search/pull/42))는 개발 가이드의 `topic=rust`와 `topic=go`로 고르며([구현 이슈 #21](https://github.com/LeeYudok/dok3search/issues/21)), 프런트 정적 자료로 분리한다. 다음 통합 슬라이스에서는 `dok3-server`·`dok3-api` 실행 경로에 영속 `FileDocumentStore`·`FileChangeJournal`, `dok3-index-manager` 세대 전환, `dok3-agent` MCP, `dok3-analyzer`를 배선한다.

## 2. 에픽과 작업 분해

| 순서 | 에픽 | 구체 작업 | 테스트·인수 근거 |
|---|---|---|---|
| P0-1 | 뼈대 [완료] | `backend/rust/` Cargo 워크스페이스, React·Vite, 설정 디렉터리 ([PR #2](https://github.com/LeeYudok/dok3search/pull/2)) | Rust·React 빌드 (`ci.yml` [PR #48](https://github.com/LeeYudok/dok3search/pull/48)) |
| P0-2 | Domain [완료] | 문서·절·검색·ACL·변경 DTO와 포트 ([PR #12](https://github.com/LeeYudok/dok3search/pull/12)) | JSON 직렬화, ACL, 요청 정규화 계약 테스트 |
| P0-3 | 수직 UI [완료] | 인메모리 검색 → `POST /api/v1/search` → React 검색·문서 상세 ([PR #12](https://github.com/LeeYudok/dok3search/pull/12)·[#14](https://github.com/LeeYudok/dok3search/pull/14)) | HTTP 검색·절 읽기·오류 응답과 React 연결 검증 |
| P1-1 | Markdown [완료] | `ParserPort`+Markdown 섹션/코드/링크 추출 (`dok3-parser` [PR #24](https://github.com/LeeYudok/dok3search/pull/24)) | `markdown_golden` RED/GREEN |
| P1-2 | Original Store [완료] | 파일 원문+SHA-256/`documents.json`+변경 저널 (`dok3-storage` [PR #22](https://github.com/LeeYudok/dok3search/pull/22)·[#27](https://github.com/LeeYudok/dok3search/pull/27)) | 원문 해시·revision·롤백·복구 테스트 |
| P1-3 | Tantivy [완료] | schema, single writer, upsert/delete, reader reload (`dok3-index` [PR #26](https://github.com/LeeYudok/dok3search/pull/26)) | in-memory index tests |
| P1-4 | end-to-end [부분 완료] | 커넥터 337건([PR #29](https://github.com/LeeYudok/dok3search/pull/29))·`DOK3_SOURCE_DIR` `tantivy-local`([PR #43](https://github.com/LeeYudok/dok3search/pull/43))·세대 머신([PR #36](https://github.com/LeeYudok/dok3search/pull/36)) `main` 반영; 서버 영속/세대 배선·UI E2E 잔여 | 커넥터 337건·세대 11/11·로컬 스냅샷 테스트 |
| P2-1 | 코드 [크레이트 완료] | `dok3-analyzer` 식별자·오류코드 분해([PR #31](https://github.com/LeeYudok/dok3search/pull/31)); Tree-sitter Rust/Java 및 서버 배선 잔여 | 분석기 계약 테스트 6/6 (`dok3-analyzer`) |
| P2-2 | 국문 | 형태소 사전 오프라인 로딩, 원형/조사 검증 | Korean golden |
| P2-3 | n-gram [토크나이저 완료] | `dok3-index` 2~3gram 토크나이저([PR #26](https://github.com/LeeYudok/dok3search/pull/26))·`dok3-analyzer`([PR #31](https://github.com/LeeYudok/dok3search/pull/31))·`dok3-benchmark`([PR #37](https://github.com/LeeYudok/dok3search/pull/37)) `main` 반영 | Recall/MRR/부하 측정 하네스 (`dok3-benchmark`) |
| P2-4 | 운영 매뉴얼 | 자체 매뉴얼을 기본 Knowledge Pack으로 자동 색인 | UI/MCP manual self-search |
| P2-5 | MCP [크레이트 완료] | `dok3-agent` stdio/Streamable HTTP, 7개 `kb_*` 도구([PR #34](https://github.com/LeeYudok/dok3search/pull/34)); `dok3-server` 라우팅·실제 Codex/Claude 연결 잔여 | `dok3-agent` 단위·계약 테스트 |
| P3-1 | 주기 수집 | 파일 워터마크/hash, batch, checkpoint, 재시도 | 중단/재시작 idempotence |
| P3-2 | 분류 | Summary/tag/keyword/category 규칙 생성+수정 UI | 분류 JSON/원문 불변 |
| P3-3 | 관리 UI | Source 등록/삭제, 작업 오류/재수집 | UI 권한/상태 E2E |
| P4-1 | SQLite | SeaORM SourceAdapter, 커서/삭제 조정 | DDL fixture update/delete |
| P4-2 | MySQL/MariaDB | 연결풀/조인 제한/커서/복구 | 두 DB 별도 계약 테스트 |
| P4-3 | PostgreSQL | cursor, isolation, tombstone | DB contract suite |
| P5-1 | Oracle | Oracle client + adapter + 바인딩된 SELECT | Oracle 통합망에서 테스트 |
| P5-2 | 문서 | HTML/PDF/DOCX/SQL/JSP 보강 | 손실 경고+파서 fixture |
| P6-1 | REST | pagination/ACL/log/metrics | 액세스 차단·로드 테스트 |
| P7-1 | Windows | Service, event log, health, stop/restart, recovery | 재부팅 시 smoke test |
| P7-2 | 폐쇄망 | vendor/npm 캐시/매뉴얼/설치 번들 | 네트워크 0건+frozen build |
| P8 | 확장 | 분류 agent, A2A/ACP bridge | 프로토콜 계약 테스트 |

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

