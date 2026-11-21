# dok3search 최종 설계 패키지 v1.3

**2026-11-20 / Markdown + YAML / 구현 전 설계 산출물**

금융권 Windows 폐쇄망에서 사람과 AI 에이전트가 교재·매뉴얼·코드·사내 문서를 검색/읽기 위한 Rust + Tantivy + React 플랫폼. 벡터 DB/RAG 필수 아님. 기본 인바운드 포트는 HTTPS 8443 하나.

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

**주의:** YAML은 구문 검증된 **설계 예시**이며 실제 서버에서 즉시 실행되는 설정은 아니다. 현재 실제 Rust 바이너리/라이브러리/책 데이터는 포함하지 않는다.

## 구현 우선순위
1. Rust Workspace + 계약 테스트 → 2. Markdown Vertical Slice → 3. 세대 전환/장애 복구 → 4. ACL/증분 동기화 → 5. React/MCP → 6. 다중 DB/파서 → 7. 50만 건 부하 검증.

## 변경 이력
`CHANGELOG.md` 참조. ZIP 파일명은 이후 수정에도 `dok3search-final-md-yaml.zip` 유지.
