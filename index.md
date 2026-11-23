# dok3search 설계 인덱스

dok3search는 금융권 Windows 폐쇄망에서 쓰는 검색 도구다. 사람과 AI 에이전트가 개발 문서, 매뉴얼, 코드, 사내 자료를 검색하고 읽는다. 이 문서에서 설계의 핵심과 읽는 순서를 확인한다. 전체 문서 트리는 [index.yaml](index.yaml)에 있다.

## 시스템 구조

목표 구조는 Rust 서버에서 자료를 원문과 절 단위로 정리하고 Tantivy로 색인한다. React 화면은 검색 결과와 원문을 보여 준다. AI 에이전트의 자료 조회 경로는 REST와 MCP로 설계한다. 저장소 경계와 검색 흐름은 [최종 아키텍처 설계서](dok3search-final-architecture.md)에서 확인한다.

## 현재 구현

[구현 저장소](https://github.com/LeeYudok/dok3search) `main`(`b298c60`)에는 **P0~P8 로드맵 전체**가 반영됐다. P0~P3([#12](https://github.com/LeeYudok/dok3search/pull/12)~[#70](https://github.com/LeeYudok/dok3search/pull/70))에 더해, P6-1 커서 페이지네이션·`/metrics`([PR #80](https://github.com/LeeYudok/dok3search/pull/80)), `dok3-analyzer::kodic` 형태소 프로파일(`builtin`/`lindera`/`nori-mixed`/`nori-discard`, [PR #100](https://github.com/LeeYudok/dok3search/pull/100)), `blog.doksam.com` 83편 코퍼스([PR #104](https://github.com/LeeYudok/dok3search/pull/104)), P2-1/P5-2 코드 심볼·PlainText·`ParserRegistry`([PR #108](https://github.com/LeeYudok/dok3search/pull/108)), P4/P5-1 SQL-aware 검증 및 MySQL/MariaDB·PostgreSQL·Oracle 공통 관계형 어댑터([PR #110](https://github.com/LeeYudok/dok3search/pull/110)), P7-1/P7-2 Windows Service 및 오프라인 사전 번들([PR #83](https://github.com/LeeYudok/dok3search/pull/83)·[#112](https://github.com/LeeYudok/dok3search/pull/112)), P8 A2A·ACP·`/llms.txt`·Google AX `Workspace` 매니페스트([PR #114](https://github.com/LeeYudok/dok3search/pull/114))가 모두 완료됐다.

REST 요청 본문이나 조회 인자의 형식이 잘못되면 HTTP 400과 `INVALID_INPUT` 코드를 담은 JSON을 반환한다. [API 입력 오류 계약](https://github.com/LeeYudok/dok3search/pull/14)

검색의 상위 분류는 개발 가이드(`guide`), 운영 매뉴얼(`manual`), 코드 레퍼런스(`code`), 내부 규정(`policy`), 설계 문서(`design`)다. Rust와 Go는 별도 상위 분류로 만들지 않고 개발 가이드의 주제로 나눈다. 표준 검색 주소는 `/search?category=guide&topic=rust`와 `/search?category=guide&topic=go`다. 예전 `/search?category=rust`와 `/search?category=go` 주소는 해당 주제로 이어진다.

[Rust·Go 참고 자료](https://github.com/LeeYudok/dok3search/issues/21)는 프런트 정적 자료로 검색한다. Rust 자료는 Comprehensive Rust 한국어판 337장, Rust 공식 책 111장, Rust Reference 120장, Cargo Book 98장으로 총 666장이다. Go 자료는 공식 문서 51장, 한국어 실무 가이드 5편, Learn Go with Tests 영문 실습 3편으로 총 59편이다. 실습은 요청 취소, HTTP 핸들러, 시간 테스트를 다룬다. `testing/synctest` 실습은 Go 1.25 이상이 필요하다. 자료 본문·예제 코드·그림은 로컬에서 열 수 있고, 각 자료의 출처와 라이선스를 확인할 수 있다.

문서 본문의 외부 URL은 에어갭 환경에서 열 수 없으므로 클릭 가능한 링크로 렌더하지 않는다. 출처 텍스트와 `외부` 표기로 남기고, 로컬 문서·절·번들 라이선스 파일만 이동을 허용한다. 이 규칙은 문서 열람이 외부 DNS나 HTTP 요청을 시도하지 않게 한다.

Rust Reference와 Cargo Book은 [Rust 1.100.0 공식 문서](https://doc.rust-lang.org/1.100.0/) 설치본에서 가져온다. Go 언어 명세는 [Go 1.26.2 SDK](https://go.dev/ref/spec)에서 가져온다. Comprehensive Rust 문서에는 CC BY 4.0, 예제 코드에는 Apache 2.0을 적용한다. Rust 문서는 MIT 또는 Apache 2.0, Go 문서는 CC BY 4.0, 예제 코드는 BSD 3-Clause를 따른다. 각 코퍼스의 README와 `provenance.json`에 입력 버전, 출처, 라이선스, 해시를 기록한다. 원본과 라이선스 전문은 [구현 저장소의 참고 자료 안내](https://github.com/LeeYudok/dok3search#폐쇄망-개발-참고-자료)에서 확인한다.

이 주제 자료는 현재 REST 검색·문서 API에 적재하지 않는다. `topic`은 프런트에서 정적 자료를 고르는 URL 필터다. 백엔드 API에는 상위 분류인 `guide`만 요청한다.

## 화면 구조

| 화면 | 경로 | 주요 과업 |
|---|---|---|
| 메인 | `/` | 검색어를 입력하거나 카테고리로 들어간다. |
| 검색 결과 | `/search` | 검색 결과를 카테고리로 좁힌다. 결과는 15건씩 표시하고 요약은 두 줄까지 보여 준다. |
| 개발 가이드 | `/search?category=guide` | 검색어로 개발 문서를 찾고 Rust·Go 주제를 선택한다. |
| Rust 주제 | `/search?category=guide&topic=rust` | Rust 공식 참고 자료 666장을 검색·열람한다. |
| Go 주제 | `/search?category=guide&topic=go` | Go 참고 자료와 실습 59편을 검색·열람한다. |
| 문서 상세 | `/docs/:id` | 제목, 출처, 요약, 본문을 읽는다. 영속 모드(`DOK3_SOURCE_DIR`+`DOK3_DATA_DIR`)에서 분류(카테고리·요약·태그·키워드)를 조회(문서 ACL)하고 `DOK3_ENABLE_ADMIN=true` 추가 시 편집·초기화한다. 읽기 진행선과 현재 절을 표시한다. |
| 설계 문서 | `/search?category=design` | 인덱스와 맵을 한 건씩 표시한다. 상세에서 Markdown과 YAML을 전환한다. |
| 운영 관리 | `/admin` (`/admin/sources`, `/admin/jobs`) | 영속 모드(`DOK3_SOURCE_DIR`+`DOK3_DATA_DIR`) + `DOK3_ENABLE_ADMIN=true`에서 다중 Source 등록·삭제·재스캔과 영속 색인 작업 트리거·이력 조회를 관리한다. |

검색어, 카테고리, 주제, 페이지는 결과 URL에 남긴다. 검색 결과에서 문서를 연 뒤 돌아갈 때 같은 조건을 복원한다.

## 색상과 서체

| 색 | 역할 |
|---|---|
| 화이트 | 화면과 문서의 기본 배경 |
| 블랙·먹색 | 본문과 제목, 주요 버튼, 읽기 진행선 |
| 그레이 | 출처와 날짜, 구분선, 선택 상태의 배경 |
| 레드 | 덕³ 도장과 오류·위험 표시 |

서체는 IBM Plex Sans KR을 사용한다. 현재 읽는 절은 강조색과 함께 굵은 글씨와 표시선으로 구별한다.

## 설계 관계

화면과 구성요소, 계약, 운영 절차의 연결은 [map.md](map.md)에서 확인한다. 도구에서 읽을 수 있는 항목 ID와 연결 관계는 [map.yaml](map.yaml)에 있다.

## 문서 읽는 순서

1. [최종 아키텍처 설계서](dok3search-final-architecture.md)에서 전체 흐름을 확인한다.
2. [구현 로드맵](dok3search-implementation-roadmap.md)과 [Rust 워크스페이스·계약](RUST_WORKSPACE_AND_CONTRACTS.md)에서 구현 순서와 모듈 경계를 확인한다.
3. [인덱스 세대 전환 절차](BLUE_GREEN_OPERATIONAL_PROTOCOL.md), [네트워크 배포](dok3search-network-deployment.md), [폐쇄망 공급망](dok3search-offline-supplychain.md)에서 운영 절차를 확인한다.
4. 설정 예시는 `config/`에서, 요청·응답과 작업 계약은 `contracts/`에서 찾는다. [index.yaml](index.yaml)에 두 디렉터리의 전체 파일 목록과 설계 검토 자료를 기록한다.

파일을 추가하거나 옮길 때는 `index.yaml`의 경로를 같이 고친다. 사용자 흐름이나 색상 기준이 바뀌면 이 문서와 관련 설계서를 갱신한다. 설계 관계를 바꿀 때는 `map.md`와 `map.yaml`도 같이 고친다.
