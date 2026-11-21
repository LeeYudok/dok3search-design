# dok3search 최종 설계 패키지 v1.3

**2026-11-20 설계 패키지 / Markdown + YAML**

금융권 Windows 폐쇄망에서 사람과 AI 에이전트가 교재·매뉴얼·코드·사내 문서를 검색하고 읽는 플랫폼의 설계다. Rust·Tantivy·React를 사용하고, 운영 배포는 HTTPS 8443을 기준으로 설계한다.

설계 개요와 읽는 순서는 [index.md](index.md)에, 문서 트리는 [index.yaml](index.yaml)에 있다. 화면과 구성요소의 연결은 [map.md](map.md)와 [map.yaml](map.yaml)에서 확인한다.

## 현재 구현

[구현 저장소의 P0-3](https://github.com/LeeYudok/dok3search/pull/12)은 Axum 인메모리 검색 API와 React 검색·원문 화면을 연결한다. 로컬 서버 기본 주소는 `127.0.0.1:8080`이다. [설계 인덱스](index.md#현재-구현)에 Rust·Go 참고 자료의 검색 경로를 기록한다.

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
