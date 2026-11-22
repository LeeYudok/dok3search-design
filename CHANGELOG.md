# 변경 이력

## v1.8 — 2026-11-22
- `dok3search` `main` `57bf0bd` 기준으로 설계 사본 동기화(#66) 및 P3-1~P3-3 완료 상태 반영:
  - `v1.7` 설계 사본 37개·`provenance.json`·`AGENTS.md`/`AGENTS.yaml` 동기화 ([PR #66](https://github.com/LeeYudok/dok3search/pull/66))
  - P3-1 주기 수집 워커 스케줄러·작업 API 완료: `DOK3_REFRESH_INTERVAL_SECS`, `DOK3_ENABLE_ADMIN` IPv4 루프백 가드, `IndexJobPort`, `index_jobs.json`, `GET/POST /api/v1/index/jobs`, `GET /api/v1/index/jobs/{job_id}` ([PR #65](https://github.com/LeeYudok/dok3search/pull/65))
  - P3-2 규칙 기반 자동 분류·원문 불변 분류 수정 API/UI 완료: 오프라인 `RuleClassifier`, `ClassificationStore`(`classifications.json`, `classification_audit.jsonl`, `pending_index_sync`), `FileDocumentStore::put_immutable`·`update_metadata` 원문 불변 보장, `ClassificationPort`, `GET/PATCH /api/v1/documents/{id}/classification`, `/docs/:id` 분류 편집 UI ([PR #68](https://github.com/LeeYudok/dok3search/pull/68))
  - P3-3 다중 Source 레지스트리·관리 UI 완료: `SourceRegistry`(`sources.json`, `local-markdown` `managed_by_env: true` 보호, 비심볼릭 실재 디렉터리 검증, 2단계 삭제/롤백, 소스별 `last_error` 격리, 디스크 원본 파일 비삭제), `SourceRegistryPort`, `GET/POST /api/v1/sources`, `DELETE /api/v1/sources/{id}`, `/admin`·`/admin/sources`·`/admin/jobs` 관리 UI ([PR #70](https://github.com/LeeYudok/dok3search/pull/70))
- `index.yaml`, `index.md`, `map.yaml`, `map.md`, `README.md`, `contracts/index-job.example.yaml`, `dok3search-implementation-roadmap.md`, `RUST_WORKSPACE_AND_CONTRACTS.md` 현행화(영속 모드 전제, `GET` ACL vs `PATCH`/관리자 가드 분리, P3 REST/P4 DB 작업 계약 구분, P2-1 Tree-sitter 잔여 및 후속 마일스톤 `P4~P8` 반영)

## v1.7 — 2026-11-22
- `dok3search` `main` `e896c47` 기준으로 P2-2·P2-4·P3-1 핵심 완료 상태 반영:
  - P2-2 한국어 정규화 완료: 오프라인 국문 조사 제거·금융/개발 복합어 정규화, 한국어 골든 40건, 한국어 10쿼리 `Recall@5`/`MRR@5` `0.20 → 1.00`, ACL 위반 0건 ([PR #61](https://github.com/LeeYudok/dok3search/pull/61))
  - P2-4 자체 운영 매뉴얼 완료: `docs/manual/dok3search-manual.md`(15개 H2 절) Knowledge Pack 자동 적재·UI/MCP self-search 검증 ([PR #62](https://github.com/LeeYudok/dok3search/pull/62))
  - P3-1 영속 런타임 증분 재수집 핵심 완료: `PersistentRuntime::refresh()`·`RefreshReport`·`ReaderLease` 보호·비호환 세대 격리·저널 정합화·크래시 복구 통합 테스트 3종 ([PR #60](https://github.com/LeeYudok/dok3search/pull/60))
- `index.yaml`, `index.md`, `README.md`, `dok3search-implementation-roadmap.md`, `RUST_WORKSPACE_AND_CONTRACTS.md` 현행화 및 후속 과제(P3-1 워커·P3-2·P3-3) 반영

## v1.6 — 2026-11-22
- P2 런타임 배선 완료 반영 (`dok3search` `main` `d26bc1f`):
  - 설계 사본 `v1.5` 37개 파일 및 `provenance.json` 동기화 ([PR #53](https://github.com/LeeYudok/dok3search/pull/53))
  - `dok3-index` + `dok3-analyzer` `SCHEMA_VERSION = 2` 통합 및 `dok3-benchmark` 비교 (`Recall@5`·`MRR@5` `0.7143 → 1.0000`, [PR #56](https://github.com/LeeYudok/dok3search/pull/56))
  - `DOK3_DATA_DIR` 영속 스토어·변경 저널·`dok3-index-manager` 세대 전환(`tantivy-persistent`, [PR #54](https://github.com/LeeYudok/dok3search/pull/54))
  - `dok3-agent` `POST /mcp` Streamable HTTP·`--mcp-stdio` 및 `dok3-api::build_router_with_extra` 배선 ([PR #55](https://github.com/LeeYudok/dok3search/pull/55))
- `index.yaml` (`merged_phase: 'P2-wiring'`, `index_schema_version: '2'`, `DOK3_DATA_DIR`·`mcp_transports`), `index.md`, `dok3search-implementation-roadmap.md`, `RUST_WORKSPACE_AND_CONTRACTS.md`, `README.md` 현행화 및 후속 스프린트(`P2-2`·`P2-4`·`P3-1`) 반영

## v1.5 — 2026-11-22
- P1·P2 크레이트(`dok3-storage` PR #22·#27, `dok3-parser` PR #24, `dok3-index` PR #26, `dok3-connectors` PR #29, `dok3-analyzer` PR #31, `dok3-agent` PR #34, `dok3-index-manager` PR #36, `dok3-benchmark` PR #37) 및 `dok3-server` `DOK3_SOURCE_DIR` 기반 로컬 Markdown `tantivy-local` 색인 경로(PR #43, 컴파일 수정 PR #46), GitHub Actions 필수 체크(`rust-fmt`, `rust-clippy`, `rust-test`, `frontend-build`, PR #48) `main` 머지 상태 반영
- `index.yaml`(`implementation.merged_phase: 'P1-4'`, `document_tree`에 `AGENTS.md` 등록), `index.md`, `dok3search-implementation-roadmap.md`, `RUST_WORKSPACE_AND_CONTRACTS.md`, `README.md` 통합 현행화
- 후속: 본 PR 머지 직후 구현 저장소 `frontend/src/samples/design/` 및 `provenance.json` 사본 동기화(`scripts/check-design-copy.py --source ../dok3search-design`) 예정

## v1.4 — 2026-11-22
- Rust·Go를 별도 상위 분류에서 개발 가이드 세부 주제로 이동
- Rust Reference와 Cargo Book을 포함한 공식 참고 자료 범위와 로컬 검색 경로 반영
- 프런트 정적 자료와 REST 검색 API의 범위를 구분
- 문서 본문의 외부 URL을 클릭 링크가 아닌 외부 출처 표기로 제한하는 에어갭 열람 규칙 추가

## v1.3 — 2026-11-20
- 실제 오픈소스 참고 사례 및 도입 경계 명시 (`REFERENCE_ARCHITECTURES.md`)
- Rust workspace/crate/Port/테스트 골격 추가 (`RUST_WORKSPACE_AND_CONTRACTS.md`)
- Blue-Green 원자성 한계, 전환 게이트, 변경 저널, 롤백 및 Windows 파일 핸들 테스트 상세화 (`BLUE_GREEN_OPERATIONAL_PROTOCOL.md`)
- YAML 계약 3개 추가 (`workspace`, `reference-patterns`, `search-snapshot`)
- README 버전과 실제 패키지 구성 정합성 정리

## v1.2 — 2026-11-20
- Blue-Green 세대 색인, 변경분 재생, 검증, 활성화, 복구 계약 추가

## v1.1 — 2026-11-20
- P0/P1 설계 문제 검토, 인덱스 정합성·ACL·증분 처리 보완
