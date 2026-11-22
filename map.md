# dok3search 전체 설계 맵

화면, 시스템 구성, 데이터 계약, 운영 절차가 어떻게 이어지는지 기록한다. 도구에서 관계를 읽을 때는 [map.yaml](map.yaml)을 사용한다. 자료의 전체 목록은 [index.md](index.md)와 [index.yaml](index.yaml)에 있다.

검색 상위 분류는 개발 가이드, 운영 매뉴얼, 코드 레퍼런스, 내부 규정, 설계 문서다. Rust·Go 자료는 개발 가이드 안의 주제로 나눈다. 정적 자료는 프런트에서 검색하며 REST API 계약에는 `topic`을 보내지 않는다.

## 설계 맵 읽는 법

작업 대상의 ID부터 찾는다. 연결 관계에서 관련 항목을 고르고, 각 항목의 근거 문서나 절을 읽는다.

예를 들어 검색 결과 페이징은 `search_results`에서 시작한다. `search_engine`, `search_request`, `search_snapshot`과 `roadmap`을 따라가면 검색 동작과 계약, 구현 순서를 확인할 수 있다.

## 사용자 화면

| 항목 | ID | 경로 | 상태 | 근거 |
|---|---|---|---|---|
| 메인 | `home_screen` | `/` | 구현 | [화면 구조](index.md#화면-구조) |
| 검색 결과 | `search_results` | `/search` | REST 연결(영속/로컬/샘플) | [화면 구조](index.md#화면-구조) |
| 문서 상세 | `document_reader` | `/docs/:id` | REST 연결·영속 모드(`tantivy-persistent`) 분류 조회(ACL)·편집(`DOK3_ENABLE_ADMIN=true`, `#68`) | [6. 블로그형 React 검색/원문 열람 UI](dok3search-final-architecture.md#6-블로그형-react-검색원문-열람-ui) |
| 설계 문서 목록 | `design_catalog` | `/search?category=design` | 형식별 원문 열람 | [화면 구조](index.md#화면-구조) |
| 개발 가이드 | `guide_topics` | `/search?category=guide` | Rust·Go 세부 주제를 고른다 | [화면 구조](index.md#화면-구조) |
| Rust 주제 | `rust_topic` | `/search?category=guide&topic=rust` | 정적 자료 로컬 검색·열람, 666장 | [현재 구현](index.md#현재-구현) |
| Go 주제 | `go_topic` | `/search?category=guide&topic=go` | 정적 자료 로컬 검색·열람, 59편 | [현재 구현](index.md#현재-구현) |
| 출처 브라우징 | `sources_screen` | `/sources` | 계획 | [6. 블로그형 React 검색/원문 열람 UI](dok3search-final-architecture.md#6-블로그형-react-검색원문-열람-ui) |
| 커넥터 관리 | `connector_admin` | `/admin/sources` | 구현(`/admin`·영속 모드+`DOK3_ENABLE_ADMIN=true`·로컬 Source 등록·삭제·재스캔, `#70`) | [6. 블로그형 React 검색/원문 열람 UI](dok3search-final-architecture.md#6-블로그형-react-검색원문-열람-ui) |
| 색인 작업 관리 | `jobs_screen` | `/admin/jobs` | 구현(영속 모드+`DOK3_ENABLE_ADMIN=true`·작업 이력·수동 트리거·오류 상세, `#65`·`#70`) | [6. 블로그형 React 검색/원문 열람 UI](dok3search-final-architecture.md#6-블로그형-react-검색원문-열람-ui) |
| 검색어 분석 실험 | `analyzer_screen` | `/admin/analyzers` | 계획 | [6. 블로그형 React 검색/원문 열람 UI](dok3search-final-architecture.md#6-블로그형-react-검색원문-열람-ui) |

## 시스템 구성

| 항목 | ID | 근거 |
|---|---|---|
| 수집·증분 색인 | `ingestion_pipeline` | [7. 수집 파이프라인과 증분 색인](dok3search-final-architecture.md#7-수집-파이프라인과-증분-색인) |
| Tantivy 검색 | `search_engine` | [5. Tantivy 색인 및 검색 설계](dok3search-final-architecture.md#5-tantivy-색인-및-검색-설계) |
| REST·MCP | `agent_api` | [8. 외부 연동: REST, MCP, A2A, ACP](dok3search-final-architecture.md#8-외부-연동-rest-mcp-a2a-acp) |
| 자동 분류 | `classification` | [9. 자동 분류 서브에이전트와 인덱싱 정책](dok3search-final-architecture.md#9-자동-분류-서브에이전트와-인덱싱-정책) |

## 데이터 계약

| 항목 | ID | 근거 |
|---|---|---|
| 검색 요청 | `search_request` | [`contracts/search-request.example.yaml`](contracts/search-request.example.yaml) |
| 검색 스냅샷 | `search_snapshot` | [`contracts/search-snapshot.example.yaml`](contracts/search-snapshot.example.yaml) |
| 원문 문서 | `document_contract` | [`contracts/document.example.yaml`](contracts/document.example.yaml) |
| 문서 절 조회 | `section_read` | [`contracts/section-read.example.yaml`](contracts/section-read.example.yaml) |
| 색인 작업 | `index_job` | [`contracts/index-job.example.yaml`](contracts/index-job.example.yaml) (P3 REST 계약 + P4 DB 증분 예시) |
| 색인 세대 전환 | `generation_transition` | [`contracts/generation-transition.example.yaml`](contracts/generation-transition.example.yaml) |

## 구현과 운영

| 항목 | ID | 근거 |
|---|---|---|
| 구현 로드맵 | `roadmap` | [2. 에픽과 작업 분해](dok3search-implementation-roadmap.md#2-에픽과-작업-분해) |
| 워크스페이스·계약 | `workspace` | [`RUST_WORKSPACE_AND_CONTRACTS.md`](RUST_WORKSPACE_AND_CONTRACTS.md) |
| 전환과 롤백 | `index_operations` | [롤백](BLUE_GREEN_OPERATIONAL_PROTOCOL.md#롤백) |
| 폐쇄망 배포 | `deployment` | [서비스/배포](dok3search-network-deployment.md#서비스배포) |
| 검색 프로파일 | `search_profiles` | [`config/search-profiles.example.yaml`](config/search-profiles.example.yaml) |

## 연결 관계

| 출발 항목 | 관계 | 연결 항목 |
|---|---|---|
| 메인 | 이어지는 흐름 | 검색 결과 |
| 메인 | 이어지는 흐름 | 설계 문서 목록 |
| 메인 | 이어지는 흐름 | 개발 가이드 |
| 개발 가이드 | 이어지는 흐름 | Rust 주제 |
| 개발 가이드 | 이어지는 흐름 | Go 주제 |
| 검색 결과 | 이어지는 흐름 | 문서 상세 |
| Rust 주제 | 이어지는 흐름 | 문서 상세 |
| Go 주제 | 이어지는 흐름 | 문서 상세 |
| 검색 결과 | 설계 근거 | Tantivy 검색 |
| 검색 결과 | 데이터 계약 | 검색 요청 |
| 검색 결과 | 데이터 계약 | 검색 스냅샷 |
| 검색 결과 | 구현 계획 | 구현 로드맵 |
| 문서 상세 | 데이터 계약 | 원문 문서 |
| 문서 상세 | 데이터 계약 | 문서 절 조회 |
| 출처 브라우징 | 설계 근거 | 수집·증분 색인 |
| 커넥터 관리 | 사용하는 구성요소·설정 | 수집·증분 색인 |
| 색인 작업 관리 | 데이터 계약 | 색인 작업 |
| 검색어 분석 실험 | 설계 근거 | Tantivy 검색 |
| 수집·증분 색인 | 사용하는 구성요소·설정 | 자동 분류 |
| 수집·증분 색인 | 이어지는 흐름 | Tantivy 검색 |
| 수집·증분 색인 | 데이터 계약 | 색인 작업 |
| 수집·증분 색인 | 데이터 계약 | 원문 문서 |
| Tantivy 검색 | 사용하는 구성요소·설정 | 검색 프로파일 |
| Tantivy 검색 | 운영 절차 | 전환과 롤백 |
| REST·MCP | 사용하는 구성요소·설정 | Tantivy 검색 |
| REST·MCP | 데이터 계약 | 검색 요청 |
| REST·MCP | 데이터 계약 | 문서 절 조회 |
| 전환과 롤백 | 데이터 계약 | 색인 세대 전환 |
| 전환과 롤백 | 운영 절차 | 폐쇄망 배포 |
| 설계 문서 목록 | 구현 계획 | 구현 로드맵 |
| 구현 로드맵 | 설계 근거 | 워크스페이스·계약 |

항목이나 연결을 바꾸면 [map.yaml](map.yaml)과 이 문서를 함께 갱신한다. 근거 파일을 옮기거나 절 제목을 바꾸면 맵의 링크도 고친다.
