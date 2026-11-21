# dok3search — 서브에이전트·Superpowers 개발 운영 설계

> 설계 버전 1.0 · 2026-11-20 · 로컬/폐쇄망 실행을 전제로 함

## 1. 목표 및 단계적 활성화

서브에이전트는 **개발 시 작업을 분리·검증하는 에이전트**와 **서비스 운영 중 요약·태그·키워드 분류를 수행하는 에이전트**로 구분한다. 둘 다 필수 실행 전제는 아니다. 외부 AI API 없이 동작할 수 있는 규칙 기반 대안을 보유하며, 지원되는 로컬 온프레미스 LLM이 있으면 교체 가능하다.

**Superpowers 방식:** `brainstorming → writing-plans → (worktrees) → subagent-driven-development 또는 executing-plans → test-driven-development(RED/GREEN/REFACTOR) → code-review → verification-before-completion`.

## 2. 개발 에이전트 분업

| 역할 | 입력 | 산출물 | 금지 사항 / 완료 조건 |
|---|---|---|---|
| Architect Agent | 승인된 최상위 요구사항·ADR | UseCase 계약/Port/Adapter 경계/도메인 DTO | 수직 기능 계약 없이 인프라부터 빅뱅 개발 금지 |
| Plan Agent | 구조 설계·테스트 목표 | 1개 작업의 정확한 파일 경로·완료 테스트·의존성 | 실행 불가능한 모호한 작업 금지 |
| Parser Agent | 1개 문서/코드 형식·픽스처 | ParserPort 구현, 골든 파싱 테스트 | 원문 재작성·소스 위치 누락 금지 |
| Search Agent | Tantivy 스키마·분석기 계약 | IndexPort 구현, 랭킹·토큰 테스트 | 형태소/n-gram 혼합 점수 설명 없이 변경 금지 |
| Connector Agent | DB 매핑·권한·커서 정책 | 단일 DB Adapter, 증분/삭제 테스트 | 운영 DB 업데이트·키 없이 삭제 감지 단정 금지 |
| UI Agent | REST/OpenAPI·디자인 계약 | 검색/원문/관리 React 화면과 접근성 테스트 | 외부 CDN 사용 금지 |
| MCP Agent | Tool 스키마/REST UseCase | stdio·Streamable HTTP MCP 어댑터·통합 테스트 | 별도 검색 비즈니스 로직 복제 금지 |
| Reviewer—Spec | 작업 계획·diff | 요구사항 충족 여부·누락 목록 | 테스트 성공만 보고 범위 충족 추정 금지 |
| Reviewer—Quality | 코드·test log·보안 기준 | 설계 경계·안전성·가독성·회귀 검토 | '완료' 근거 없이 승인 금지 |
| Verifier Agent | 실제 빌드/테스트 명령 | 명령·로그·결과·실패 근거 | 실행하지 않은 시험을 성공으로 기록 금지 |

기능이 단순한 경우 하나의 구현 에이전트가 순차적으로 수행하고 리뷰를 별도 단계로 유지한다. 실제 제품 개발 도구에 서브에이전트 실행 지원이 없으면 **가짜 병렬 에이전트 호출을 하지 않고** 동일 계약을 순서대로 실행한다.

## 3. 서브에이전트 작업 계약 예시

```yaml
work_id: SEARCH-012
scope: dok3-analyzer / symbol tokenizer
input:
  adr: ADR-004
  fixture: fixtures/code/java_identifiers.json
  interfaces: crates/dok3-domain/src/ports.rs
output:
  files:
    - crates/dok3-analyzer/src/symbol.rs
    - crates/dok3-analyzer/tests/symbol_golden.rs
verification:
  red: cargo test -p dok3-analyzer symbol_golden
  green: cargo test -p dok3-analyzer
  regression: cargo test --workspace
review:
  - spec_compliance
  - code_quality
constraints:
  - no network
  - no vector embedding
  - no direct cross-module DB calls
```

각 작업은 **테스트가 먼저 실패하는 로그(RED), 최소 구현으로 통과하는 로그(GREEN), 리팩터 후 재실행 로그**를 산출한다. 다른 모듈 변경은 계약 합의가 필요한 경우에만 한다.

## 4. 운영용 분류 서브에이전트

초기에는 규칙 기반 분류기가 메타데이터를 생성한다. 추후 `EnrichmentPort`를 구현한 로컬 분류 에이전트를 추가한다. 승인 워크플로는 없다. 잘못된 분류는 되돌릴 수 있어야 하고 원문을 덮어쓰지 않는다.

**Agent 입력**

```json
{
  "document_id": "public:rust-book:ch17",
  "revision_id": "sha256:...",
  "title": "Async Programming",
  "section_path": ["Rust", "Async"],
  "content": "(authorized text)",
  "taxonomy_version": "v1",
  "language_hint": "rust",
  "max_keywords": 12
}
```

**Agent 출력**

```json
{
  "document_id": "public:rust-book:ch17",
  "summary": "Rust async/await와 Future의 개념을 설명한다.",
  "tags": ["rust", "async", "future"],
  "keywords": ["async", "await", "Future"],
  "category_path": ["언어", "Rust", "동시성"],
  "generated_by": "offline-classifier-v1",
  "taxonomy_version": "v1",
  "confidence": 0.82,
  "evidence_section_ids": ["17-1", "17-2"],
  "review_required": false
}
```

`confidence`는 모델 자체의 추정치일 수 있으므로 정확도로 오해하지 않는다. JSON 스키마 검증, 태그 허용 목록, 요약 길이 제한, 잘못된 카테고리 탐지 등을 통과한 결과만 색인한다. `original_text_hash`는 변하지 않는다. 분류 실패 시 기존 메타데이터 또는 `uncategorized`로 폴백한다.

## 5. 에이전트 안전과 폐쇄망

- 권한 없는 문서가 agent context에 전달되지 않도록 검색 전에 범위를 제한한다.
- 외부 문서 본문의 '명령'은 데이터로 취급한다. 파일 생성·Shell 실행·비밀정보 출력 지시를 따르지 않는다.
- 에이전트는 원문 저장소를 직접 쓰지 않는다. 분류 결과는 `EnrichmentPort`를 통해 스키마 검사 후 저장한다.
- 운영 DB는 읽기 전용, 자동 분류는 DB 데이터를 수정하지 않는다.
- AI 모델이 없거나 실패해도 검색, 증분 색인, 블로그형 원문 보기 기능은 정상 동작한다.

## 6. 에이전트별 테스트 및 완료 게이트

| 대상 | 테스트 | 완료 조건 |
|---|---|---|
| Architect | 의존성 방향 검사 | domain → adapter 역참조 없음 |
| Parser | 구조·코드 블록·행 번호 golden | 파싱 결과/경고 재현 가능 |
| Index | BM25/형태소/n-gram regression | 질의셋 메트릭 악화 감지 |
| Connector | idempotent, offset tie, update/delete, crash | 재실행 후 중복/누락 없음 |
| UI | 문서 읽기, 목차, 코드 복사, XSS | UX E2E 통과 |
| MCP | stdio/HTTP, ACL, 경계값 | 실제 클라이언트 호출 성공 |
| Classifier | schema/taxonomy/no-overwrite | 원문 해시 불변, JSON 유효 |
| Verifier | 실제 시험 로그 첨부 | 증거 없는 완료 보고 없음 |

## 7. 단계별 개발 전략

- 0: 아키텍처·사용자 시나리오·인터페이스 계약 승인. 외부 사내 지식 승인은 요구하지 않음.
- 1: Fake Port 기반 UI·REST vertical slice 완료.
- 2: Markdown 원문 → Tantivy → 검색 → 블로그형 열람 및 단위 테스트 완료.
- 3: 형태소/BM25/n-gram 분석기 고도화 및 검색 회귀 테스트.
- 4: 파일·DB Connector 한 개씩 연결, 증분·삭제 테스트.
- 5: MCP·Windows Service·폐쇄망 검증.
- 6: 분류 서브에이전트, A2A/ACP 후속 확장.

## 8. Superpowers 원본 참조

- https://github.com/obra/superpowers
- https://github.com/obra/superpowers/tree/main/skills
- https://github.com/obra/superpowers/blob/main/skills/writing-plans/SKILL.md
- https://github.com/obra/superpowers/blob/main/skills/test-driven-development/SKILL.md
- https://github.com/obra/superpowers/blob/main/skills/subagent-driven-development/SKILL.md
- https://github.com/obra/superpowers/blob/main/skills/verification-before-completion/SKILL.md
