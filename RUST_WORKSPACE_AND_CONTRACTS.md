# Rust Workspace 골격과 인터페이스 우선 개발

## 원칙
전체 crate 구조·핵심 도메인 타입·포트(Trait)·계약 테스트를 먼저 고정한다. **수백 개의 빈 함수나 실행 경로의 `todo!()`는 만들지 않는다.** 기능별 Vertical Slice로 RED → GREEN → REFACTOR를 반복한다.

아래 경로는 [구현 저장소](https://github.com/LeeYudok/dok3search)의 구성이다. P0 구현은 `dok3-core`·`dok3-domain`의 계약 타입과 `dok3-search`·`dok3-api`·`dok3-server`의 인메모리 HTTP 검색을 제공한다.

```text
dok3search/
  backend/rust/
    Cargo.toml               # workspace members, resolver
    Cargo.lock
    rust-toolchain.toml
    crates/
      dok3-core/             # 오류, 설정, 공통 타입
      dok3-domain/           # DocumentId, Revision, SectionId, GenerationId
      dok3-storage/          # SQLite 작업저널/메타데이터/활성 세대 상태
      dok3-connectors/       # 파일, Git, SVN, DB 소스 포트
      dok3-parser/           # MD, HTML, PDF, DOCX, Tree-sitter
      dok3-analyzer/         # 한국어/영어/코드 분석기
      dok3-index/            # Tantivy writer, schema, reader
      dok3-index-manager/    # 세대 빌드/검증/활성화/복구
      dok3-search/           # 검색, ACL 필터, 원문 접근
      dok3-agent/            # MCP 우선, A2A/ACP 후속
      dok3-api/              # Axum 라우터와 검색·문서 조회
      dok3-benchmark/        # 재현 가능한 부하 생성/측정
      dok3-server/           # HTTP 서버 진입점
  frontend/                  # React + TypeScript
  config/                    # 예시 YAML
  contracts/                 # YAML 계약 및 버전
  tests/                     # 통합/장애 주입/성능
  docs/                      # 설계/운영/자체 매뉴얼
  scripts/                   # 오프라인 빌드/패키징
```

## 핵심 Port 계약(개념 예시)
```rust
pub trait ChangeJournal: Send + Sync {
    fn append(&self, change: ChangeEvent) -> Result<Sequence, JournalError>;
    fn read_after(&self, seq: Sequence, limit: usize) -> Result<Vec<ChangeEvent>, JournalError>;
}

pub trait IndexBuilder: Send + Sync {
    fn build(&self, generation: GenerationId, snapshot: SnapshotToken) -> Result<BuildReport, IndexError>;
    fn replay_until(&self, generation: GenerationId, seq: Sequence) -> Result<Sequence, IndexError>;
    fn validate(&self, generation: GenerationId) -> Result<ValidationReport, IndexError>;
}

pub trait SearchSnapshotProvider: Send + Sync {
    fn snapshot(&self) -> SearchSnapshot; // generation + reader + epoch를 한 묶음으로 보유
}
```

> 위 코드는 컴파일 가능한 전체 구현이 아니라 인터페이스 설계 스케치이다. 실제 비동기/취소/동시성 모델은 Vertical Slice 테스트로 확정한다.

## 단계별 산출물
1. Workspace, CI 오프라인 빌드, 계약 타입, 테스트 러너
2. MD 한 파일 → 파싱 → 섹션 → Tantivy → REST 검색/읽기
3. Generation Manager: 신규 세대 빌드 → 검증 → 교체 → 복구
4. 지속 변경 이벤트, ACL, 변경/삭제 동기화
5. React 원문 뷰, MCP 도구, 코드/한국어 분석기
6. DB 어댑터별 계약 테스트, 50만 건 벤치마크, Windows 서비스

## 의존성 방향
`domain`은 외부 크레이트 구현에 의존하지 않는다. `index-manager`는 `storage`/`index`의 포트에 의존하며, `api`/`agent`는 `search` 응용 서비스만 호출한다. 파서가 Tantivy를 직접 수정하지 않도록 한다.
