# dok3search — 폐쇄망 완전 내재화·공급망·운영 설계

> 2026-11-20 · Windows 온프레미스 · 설계 버전 1.0
>
> 목적: 외부 인터넷이 전혀 없는 개발·빌드·실행 환경에서 소스, 라이브러리, 사전, UI 정적 리소스, 매뉴얼을 모두 확보하고 재현 가능하게 운영한다. **본 문서는 반입 목록과 검증 기준**이다. 설치 바이너리/서적 본문 자체를 포함하지 않는다.

## 1. 폐쇄망 경계와 운영 모델

- 인터넷 연결 수집/빌드 스테이징 영역과 사내 폐쇄망을 분리한다. 반입은 기관의 승인된 망간 자료 이동 절차로 수행한다.
- 운영 장비는 외부 네트워크 접근에 의존하지 않는다. 검색·색인·문서 열람·MCP·A2A 선택 확장은 내부 주소로만 제공한다.
- 기본 권장: `dok3search.exe` + React 정적 빌드 자산 + `ProgramData\\dok3search`의 데이터/설정 + 로컬 또는 내부 DB. 애플리케이션은 외부 CDN, 웹 폰트, 외부 자바스크립트, 외부 이미지, 외부 라이선스 확인 서버를 호출하지 않는다.
- **개발 폐쇄망**까지 인터넷이 없다면 빌드 도구와 의존성 소스를 반입한다. **운영 폐쇄망**만 인터넷이 없다면 검증 완료된 실행 파일과 런타임 의존성만 배포한다. 두 경로를 혼동하지 않는다.

## 2. 완전 내재화 자산 목록 (BOM)

| 범주 | 반드시 포함할 항목 | 검증 산출물 |
|---|---|---|
| Rust 도구 | 고정 버전 rustc, cargo, rustfmt, clippy, 해당 Windows 타깃 표준 라이브러리 | 버전 출력·해시·설치 매뉴얼 |
| Windows 빌드 | MSVC Build Tools/SDK, 링크 라이브러리·헤더 또는 승인된 GNU 타깃 툴체인 | 오프라인 Hello World 링크 성공 |
| Rust 코드 | Workspace 전체 소스·Cargo.lock·`.cargo/config.toml`·vendored crates·Git 종속성 | `cargo build --frozen` 성공 |
| 네이티브 라이브러리 | Oracle Client/Instant Client, 필요 C 라이브러리/DLL, 오픈소스 라이선스 | dependency/DLL 검사·실행 스모크 |
| React | Node.js/npm 고정 버전(빌드 환경), `package-lock.json`, npm 캐시/사설 레지스트리 미러, 전체 아이콘/스타일/폰트 | `npm ci --offline` + `npm run build` 성공 |
| 문서 파서 | Tree-sitter grammar 바이너리·원본·라이선스, PDF/DOCX 파서 의존성, 테스트 픽스처 | 실제 형식 샘플 파싱 테스트 |
| 한국어 검색 | 형태소 분석기, 한국어 사전, 사용자 사전, bi-gram/n-gram 설정, 사전 라이선스·버전 | 한국어 골든 질의·토큰 검증 |
| 검색엔진 | Tantivy 버전 고정, 인덱스 스키마·분석기 버전·릴리스 체크섬 | 신규 색인/재구축·복구 검증 |
| 검색 매뉴얼 | dok3search 사용자/관리자/API/MCP/색인/복구/검색 DSL 매뉴얼 | 원문 열람·검색 가능 |
| 기술 지식 | Rust Book·std API·Cargo Book·Tokio 등 실제 사용 버전 문서·코드 예제·출처/라이선스 | Knowledge Pack manifest·원문 검증 |
| 보안/공급망 | SBOM, 바이너리/팩 SHA-256, 라이선스 목록, 취약점 점검 결과, 내부 서명/승인 이력 | 반입 목록과 해시 대조 |

## 3. 오프라인 패키징 구조

```text
dok3search-airgap-bundle/
├── manifest.json                 # 번들 ID·버전·대상 OS/아키텍처·해시·라이선스
├── checksums.sha256
├── sbom/                         # CycloneDX/SPDX 등
├── src/                          # Rust workspace + React 소스 + 문서 원본
├── rust/
│   ├── toolchain/                # rustc/cargo/target components
│   ├── vendor/                   # cargo vendor 산출물
│   └── build-tools/              # Windows SDK/MSVC 승인 배포물 참조 또는 설치본
├── web/
│   ├── node-distribution/
│   ├── npm-cache/                # 모든 tarball 포함 확인
│   └── dist/                     # 사전 생성 정적 자산
├── native/
│   ├── oracle-client/            # 필요 시 별도 배포권 확인
│   └── dependencies/
├── analyzers/
│   ├── korean-dictionaries/
│   ├── user-dictionary/
│   └── tokenizer-profiles/
├── grammars/                     # Tree-sitter grammar 소스·컴파일 아티팩트
├── knowledge-packs/
│   ├── rust-official/
│   ├── dok3search-manual/
│   └── internal-samples/
├── tests/                         # 오프라인 실행 골든 데이터와 검증 스크립트
└── runbooks/                      # 설치/백업/복구/장애/재색인 매뉴얼
```

**주의**: 번들에 `build-tools`, Oracle Client 및 유료 자료를 실제로 담을 수 있는지는 라이선스·기업 재배포 조건을 확인한다. 목록은 설계된 구성이지 자료 재배포 허가를 의미하지 않는다.

## 4. Rust 의존성 오프라인 빌드

### 4.1 인터넷 연결 스테이징 영역

1. 빌드할 Windows 타깃(`x86_64-pc-windows-msvc` 권장)을 고정하고 Rust 및 MSVC/SDK 버전을 기록한다.
2. `Cargo.lock`을 커밋하고 Git 의존성도 고정된 revision으로 지정한다.
3. `cargo vendor vendor`를 실행하고 명령에서 출력한 Cargo Source Replacement 설정을 `.cargo/config.toml`에 저장한다.
4. 서드파티 C/C++/네이티브 빌드 스크립트가 빌드 중 별도 다운로드하지 않는지 검사한다.
5. Windows를 포함한 동등한 환경에서 클린 빌드를 시험하고 `vendor`·toolchain·SDK·DLL·런타임을 반입 패키지로 고정한다.

예시 `.cargo/config.toml`:

```toml
[source.crates-io]
replace-with = "vendored-sources"

[source.vendored-sources]
directory = "vendor"

[net]
offline = true
```

폐쇄망 검증 명령:

```powershell
cargo build --workspace --release --frozen
cargo test --workspace --frozen
cargo clippy --workspace --frozen -- -D warnings
```

`--frozen`은 lockfile 고정 + 네트워크 금지다. **vendor가 완전하지 않으면 오류를 내는 것이 정상**이며, 보이지 않게 외부 접근을 시도하면 안 된다. 또한 `cargo vendor`는 컴파일러·Windows SDK·Oracle DLL까지 제공하지 않는다.

## 5. React/npm 오프라인 빌드

1. `package.json`, `package-lock.json`, Node/npm 버전을 고정한다.
2. 온라인 스테이징에서 종속 패키지의 tarball과 선택·플랫폼별 패키지를 모두 캐시에 확보하거나 조직 내 npm 레지스트리 미러를 구성한다.
3. Windows 폐쇄망 클린 환경에서 `npm ci --offline`을 실제로 실행한다. `--prefer-offline`은 부족한 리소스를 외부에서 받으므로 **사용 금지**.
4. `npm run typecheck && npm run test && npm run build`를 실행한다. 프런트 산출물은 Rust 웹 서버가 제공하므로 **운영에 Node 서버가 필요 없다**.
5. 아이콘/폰트/수식 렌더러/코드 하이라이트·Worker 파일이 CDN을 요청하지 않는지 브라우저 네트워크 차단 테스트를 수행한다.

예시:

```powershell
npm ci --offline --cache .\offline-npm-cache
npm run build
```

캐시가 불완전하면 실패한다. lockfile만으로 오프라인 설치를 보장하지 않는다.

## 6. 검색엔진 매뉴얼도 검색 가능한 데이터로 탑재

`source_id = dok3search-product-manual`, `category = 운영/검색/API/개발`, `doc_type = manual`, `version = product-version`으로 별도 논리 출처를 등록한다. 최초 설치 시 수동 `docs/manual`의 Markdown 원문을 파싱해 검색한다.

매뉴얼 필수 장:

- 사용자: 블로그형 탐색, 검색 필터, 정확 검색, 형태소·2/3/4-gram 차이, 문서 원문·목차 이동, 태그·키워드.
- 관리자: 소스 등록, DB 연결 권한, 워터마크, 증분·삭제 처리, 백업·복구, 재색인, 오류 코드와 로그.
- 개발자: REST·MCP 도구 스키마, 결과/오류 계약, SDK 예제, A2A/ACP 연동, 테스트.
- 아키텍처: Parser/Analyzer/Tantivy/SourceAdapter, 저장 구조, 보안 모델, 검색 점수 해석.
- 트러블슈팅: 라이브러리 누락, DB 접속 실패, 잘못된 문자셋, 인덱스 손상·복구, 원문 누락.

**자기문서화(Self-indexing)**: 첫 기동에서 매뉴얼 원본을 기본 지식팩으로 등록한다. 내부 사용자도 검색·원문 보기 가능하며 MCP `kb_search`로 같은 매뉴얼을 찾는다. 매뉴얼 버전과 실행 버전을 매칭하고 버전 불일치 경고를 표시한다.

## 7. 지식팩의 ‘문서 교육’ 기능

기술서적은 **법적 이용 권한이 있는 원문**을 수집한다. Rust Book은 공개 소스와 명시된 라이선스를 확인할 수 있지만, 상업 출판물·유료 PDF 등은 별도 권한 확인이 필요하다. 기술 지식을 모델 파라미터에 학습시키는 것이 아니라, **검색 → 장/절 원문 → 코드 예제 → 문맥 이해 → 구현·검증**의 과정을 제공한다.

문서에 목차, 분류·태그·키워드·요약, 소스 버전, 코드 블록, 앞뒤 문서 링크를 붙이고 `get_document/get_section/get_adjacent_sections`로 AI가 읽을 수 있게 한다. 긴 문서의 일괄 응답은 길이 제한·페이지네이션을 제공한다. 코드 예제는 가능하면 `rustc`/`cargo test`로 별도 실행 검증하고, 검색 결과에 검증 여부를 표기한다.

## 8. 네트워크 차단 및 오프라인 인수 테스트

| 테스트 | 성공 조건 |
|---|---|
| 운영 프로세스 외부 연결 차단 | 정상 검색·원문·MCP·증분 색인에 외부 DNS/HTTP 요청 0건 |
| 개발 PC 외부 연결 차단 | 빈 캐시/클린 Windows 환경에서도 패키지 의존성 해결 후 빌드 성공 |
| UI CDN 차단 | 폰트, JS, CSS, 아이콘, 하이라이트가 모두 로컬에서 로드 |
| 기술 교재 자체 탑재 | Rust Book 예시 장의 검색·전체 원문·앞뒤 문맥 조회 성공 |
| 제품 매뉴얼 자체 탑재 | MCP와 UI 모두 `dok3search` 자체 문서를 검색·열람 가능 |
| DB Connector 재시작 | 작업 중단·재시작 후 중복/누락 없이 재수집 |
| Windows 재시작 | 서비스 자동 시작 및 이전 인덱스 조회·원문 열람 정상 |
| 보안 | 권한 없는 인덱스/문서와 원문 접근 차단 |
| 재구축 | 원본 Knowledge Pack + 동일 파서/분석기 버전으로 색인 재생성 성공 |

## 9. 운영 절차와 리스크

- 릴리스마다 **애플리케이션 버전, 색인 스키마 버전, 분석기·사전 버전, 제품 매뉴얼 버전, 빌드 도구 버전**을 함께 고정한다.
- 새 형식 파서·새 Rust crate를 추가하면 BOM/반입 패키지/골든 테스트도 같이 갱신한다.
- Windows 바이너리 서명·DLL 검색 경로·파일 권한을 검증한다. Oracle Client는 설치·비트수·버전을 반드시 확인한다.
- 개인정보·인증정보는 색인 전 규칙으로 필터링하고 외부 문서는 prompt injection을 포함할 수 있으므로 지침으로 실행하지 않는다.
- 원본과 색인을 구분해 원본·SQLite·인덱스 복구 순서를 문서화한다.

## 10. 공식 참고문서

1. Cargo Vendor: https://doc.rust-lang.org/cargo/commands/cargo-vendor.html
2. Cargo Source Replacement: https://doc.rust-lang.org/cargo/reference/source-replacement.html
3. Cargo build / frozen: https://doc.rust-lang.org/cargo/commands/cargo-build.html
4. rustup Windows toolchain: https://github.com/rust-lang/rustup/blob/main/doc/user-guide/src/installation/windows.md
5. npm offline config: https://docs.npmjs.com/cli/v11/using-npm/config/
6. npm ci: https://docs.npmjs.com/cli/npm-ci/
7. Rust Book: https://github.com/rust-lang/book
8. Oracle Rust driver: https://docs.rs/oracle/latest/oracle/
9. Tantivy Rust: https://docs.rs/tantivy/latest/tantivy/
10. MCP 2026-07-28 transport: https://github.com/modelcontextprotocol/modelcontextprotocol/blob/main/docs/specification/2026-07-28/basic/transports/index.mdx
