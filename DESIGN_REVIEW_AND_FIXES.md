# dok3search 설계 검토 및 보완 결과 (2026-11-20)

## 검토 범위와 결론
기존 Markdown 7개 및 YAML 8개를 검토했다. 구조적 모놀리스, Rust/Tantivy, React, 오프라인 우선, 증분 색인, 문서/코드 파서의 방향은 유지한다. 다만 기존 자료는 **구현 계약 예시**이지 검증된 동작 구현이나 완성된 규격이 아니다. 아래 P0 항목을 해결하기 전 금융권 운영 배포를 승인하지 않는다.

## P0: 데이터 정합성·보안

### F-01 Tantivy와 SQLite 간 원자적 트랜잭션 부재
- 문제: Tantivy commit과 SQLite checkpoint를 하나의 DB 트랜잭션으로 묶을 수 없다. 중간 장애 시 누락·중복·유령 문서가 발생할 수 있다.
- 결정: SQLite에 `index_jobs`, `index_operations`, `source_checkpoints`, `index_generations`를 둔다. 소스 배치를 durable journal에 먼저 기록하고, deterministic document/section ID를 이용하여 delete+add를 반복 가능하게 만든다. Tantivy commit 성공과 reader reload 확인 뒤에만 checkpoint를 완료한다. 재시작 시 `prepared/applied/visible/committed` 상태를 재조사해 재실행한다. 검색 일관성은 generation ID로 검증한다.
- 주의: Tantivy commit과 SQLite commit 사이의 분산 원자성은 제공하지 않는다. **최종 일관성 + 복구 가능성**을 보장 목표로 하고 fault-injection으로 입증한다.

### F-02 변경 시각 워터마크와 삭제 누락
- 문제: `updated_at`과 PK를 커서로 써도 동시 업데이트/늦은 커밋/동일 시각/타임존/삭제를 자동 해결하지 않는다. 조회 중 최신값으로 변경된 레코드가 이미 지나간 범위로 이동할 수 있다.
- 결정: 소스별 `snapshot`, `watermark`, `change_table`, `cdc`, `full_reconcile` 중 지원 모드를 명시한다. watermark는 DB 시각 기반 안전 상한(lookback window), 안정적 정렬, 재조회, tombstone 또는 전체 PK 대조가 필수다. 정확한 변경 이력이 필요한 경우 DB 트리거 기반 변경 테이블 또는 DB별 CDC를 채택한다. CDC는 초기 필수 기능이 아니다.
- 삭제: `soft_delete`, `tombstone`, `key_reconciliation` 중 적어도 하나가 검증되지 않으면 자동 삭제 반영을 보장하지 않는다. 커넥터별 DB 실기기 테스트를 통과해야 한다.

### F-03 접근 제어의 색인·문서·캐시 불일치
- 문제: 검색 시 ACL만 필터링해도 문서 ID 직접 조회, 캐시, 자동완성, 통계, MCP, 다운로드에서 정보가 누출될 수 있다.
- 결정: 인증된 주체에서 서버가 scope를 계산한다. 요청자의 `scope` 값은 권한 상승에 사용하지 않는다. 검색 후보 선정 이전에 허용 범위 적용, 결과·원문·섹션·첨부·하이라이트·facet·suggestion·cache 모두 정책 적용. 문서 ACL 변경 시 해당 문서/섹션 색인과 캐시를 무효화한다. 기본 거부와 부정 테스트를 필수화한다.

### F-04 HTML 및 도구 출력의 신뢰 경계
- 문제: 원문 HTML의 XSS, 외부 자원 호출, 문서 안의 에이전트 명령 주입.
- 결정: HTML sanitize, CSP, 외부 URL/이미지 기본 차단, 다운로드 콘텐츠 분리, 검색 결과를 데이터로 취급. MCP/A2A 출력에도 출처·권한·신뢰 레이블을 포함한다. 소스 SQL 매핑은 allowlist된 테이블/컬럼 식별자만 조합하고 값은 bind parameter로 전달한다.

## P1: 제품 완성도·품질

### F-05 검색 점수와 n-gram 과잉 일치
- `exact symbol > error exact > phrase > morphology/BM25 > ngram fallback` 순으로 query intent별 프로파일을 둔다. ngram은 기본 랭킹을 대체하지 않는다. analyzer 이름/사전 버전/schema 버전을 색인 manifest에 저장하고 변경 시 blue-green 재색인 및 원자적 alias 전환을 검증한다.
- 한글 형태소 사전과 Tantivy 어댑터의 버전 호환성은 실제 빌드로 검증한다. 특정 라이브러리의 최신 버전을 근거 없이 고정하지 않는다.

### F-06 교재 원문 보존과 AI 읽기 경로
- 책/매뉴얼은 라이선스와 원본 바이트를 보존하고 `book -> chapter -> section -> code example` 계층으로 모델링한다. `get_document`는 기본 메타데이터/목차만, `get_section`은 크기 제한 및 다음 페이지 커서를 제공한다. 대용량 전체 원문을 단일 MCP 응답으로 보내지 않는다.
- 문서 판본/라이브러리 버전, 인용 가능한 장·절/페이지/라인, 코드 블록 언어를 보존한다. 검색용 추출 텍스트와 사람용 원문은 별도 보관한다. OCR이 필요한 스캔 PDF는 품질 경고 및 후속 단계로 분리한다.

### F-07 프로토콜 범위 혼동
- Codex/Claude CLI의 기본 통합 경로는 MCP이다. A2A는 에이전트 간 작업 위임이 필요할 때만, ACP는 에디터-에이전트 브리지 요구가 확정된 뒤 구현한다. ACP를 검색 서버의 직접 필수 기능으로 오해하지 않는다. 규격·클라이언트 버전은 릴리스별 호환성 시험으로 확정한다.

### F-08 ORM 경계
- 내부 SQLite 메타데이터에 SeaORM 적용 가능. 외부 DB의 임의 스키마/증분 스캔에는 ORM 모델 강제 대신 `SourceConnector`와 DB별 query adapter를 둔다. Oracle은 별도 드라이버, 인증/네이티브 의존성/트랜잭션 시험을 필수화한다. MariaDB와 MySQL도 SQL/버전 차이를 계약 테스트한다.

### F-09 폐쇄망 공급망·운영
- 소스 저장소 + lockfile + vendored crates + npm tarballs/cache + Rust toolchain/Windows linker + native DB runtime + Tree-sitter grammars + 한국어 사전 + React 정적 자산 + Tantivy/자체 운영 매뉴얼 + 라이선스 + SBOM + SHA-256을 **반입 manifest**로 추적한다.
- `cargo --offline`만으로 완전한 오프라인 검증이 아니다. 빈 캐시의 격리 Windows 빌드 머신에서 재현 빌드 및 무인터넷 실행 시험을 한다. 외부 라이브러리 문서는 해당 라이선스에 맞춰 반입한다.

### F-10 Windows 서비스·네트워크
- TCP 8443 한 개는 기본 인바운드 목표. DB·AD/LDAP·DNS·NTP·CRL/OCSP 등 아웃바운드는 별도 매트릭스와 보안 정책을 따른다. Windows 서비스 제어, graceful shutdown, writer lock, TLS 인증서 갱신, 복구/롤백, 백업·복원 테스트를 릴리스 게이트로 둔다.

## 개발 순서(탑다운, 기능 수직 슬라이스)
1. M0 계약: DTO, ACL policy, 오류 모델, 테스트 fixtures, index job state machine.
2. M1 Markdown 1개 수집 → 파싱 → 색인 → 검색 → React 원문 열람, 단위/통합 테스트.
3. M2 장·절·코드 블록, Rust 공식 매뉴얼 지식팩, MCP search/get_section.
4. M3 형태소·코드 심볼·ngram 분석기, 검색 품질 회귀셋.
5. M4 파일 증분·삭제·복구 저널, 장애 주입 시험.
6. M5 SQLite/PostgreSQL/MySQL/MariaDB/Oracle 커넥터를 각각 독립 계약 시험으로 추가.
7. M6 Windows 서비스·인증·권한·폐쇄망 공급망/복원 검증.
8. M7 자동 분류, A2A/ACP 등 선택적 확장.

## 릴리스 게이트
- P0 보완 항목 F-01~F-04 통합/보안 테스트 통과.
- 색인 작업 강제 종료를 commit 전/후, checkpoint 전/후에 주입해 재시작 후 문서 수/버전/삭제 결과가 기대값과 일치.
- 다중 사용자 ACL 검색·원문·MCP·facet·cache 권한 우회 0건.
- 5종 DB를 지원한다고 표시하기 전에 **각 DB 실기기**에서 최초/증분/삭제/재시작 시험 통과.
- 한글/영문/심볼/오류 검색 회귀셋, 대형 매뉴얼 탐색/절 페이징, 완전 폐쇄망 Windows 설치·실행 확인.

## 남은 의사결정
- 실제 Windows Server 버전과 CPU/메모리, 문서 규모, AD 연동 방식, Oracle 버전/드라이버, 파일 반입 정책, 문서 라이선스 목록, 데이터 보존 정책. 각 항목은 ADR로 기록하고 PoC 측정 전 성능 수치를 보장하지 않는다.

## v1.2 재검토 보완
P0-11~14, P1-15~16의 상세 계약은 `BLUE_GREEN_DESIGN_REVIEW.md`를 참조한다. 기존 문서의 blue-green 언급만으로 무중단/무손실을 보장하지 않는다.
