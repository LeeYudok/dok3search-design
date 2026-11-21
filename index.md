# dok3search 설계 인덱스

dok3search는 금융권 Windows 폐쇄망에서 쓰는 검색 도구다. 사람과 AI 에이전트가 개발 문서, 매뉴얼, 코드, 사내 자료를 검색하고 읽는다. 이 문서에서 설계의 핵심과 읽는 순서를 확인한다. 전체 문서 트리는 [index.yaml](index.yaml)에 있다.

## 시스템 구조

목표 구조는 Rust 서버에서 자료를 원문과 절 단위로 정리하고 Tantivy로 색인한다. React 화면은 검색 결과와 원문을 보여 준다. AI 에이전트의 자료 조회 경로는 REST와 MCP로 설계한다. 저장소 경계와 검색 흐름은 [최종 아키텍처 설계서](dok3search-final-architecture.md)에서 확인한다.

## 현재 구현

[구현 저장소의 P0-3](https://github.com/LeeYudok/dok3search/pull/12)은 Axum 인메모리 저장소로 검색, 문서 상세, 절 조회를 제공한다. React는 이 API에 연결하고, 개발 서버에서 API에 연결할 수 없으면 내장 샘플을 사용한다. Tantivy 색인과 MCP는 [구현 로드맵](dok3search-implementation-roadmap.md)의 후속 단계에 있다.

[Rust·Go 참고 자료](https://github.com/LeeYudok/dok3search/issues/8)는 프런트에 포함한다. 두 카테고리에서 검색·열람을 로컬로 처리하며, 각 문서는 Markdown 원문과 예제 코드, 출처와 라이선스를 포함한다.

## 화면 구조

| 화면 | 경로 | 주요 과업 |
|---|---|---|
| 메인 | `/` | 검색어를 입력하거나 카테고리로 들어간다. |
| 검색 결과 | `/search` | 검색 결과를 카테고리로 좁힌다. 결과는 15건씩 표시하고 요약은 두 줄까지 보여 준다. |
| 문서 상세 | `/docs/:id` | 제목, 출처, 요약, 본문을 읽는다. 읽기 진행선과 현재 절을 표시한다. |
| 설계 문서 | `/search?category=design` | 인덱스와 맵을 한 건씩 표시한다. 상세에서 Markdown과 YAML을 전환한다. |
| Rust 자료 | `/search?category=rust` | Google Comprehensive Rust 한국어판과 공식 Rust Book을 찾는다. |
| Go 자료 | `/search?category=go` | Go 언어 명세, Go Modules Reference, Effective Go를 찾는다. |

검색어, 카테고리, 페이지는 결과 URL에 남긴다. 검색 결과에서 문서를 연 뒤 돌아갈 때 같은 조건을 복원한다.

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
