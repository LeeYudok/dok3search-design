# dok3search v1.2 설계 재검토: Generation 기반 Blue-Green 색인

## 결론 및 범위
기존 v1.1은 Blue-Green을 언급하지만 전환 프로토콜이 정의되지 않았다. 이 문서는 구현 전 필수 설계 계약이다. 검색 무중단은 **요청 처리 지속**을 뜻하며 모든 데이터가 실시간으로 반영된다는 뜻이 아니다. 검증 전 성능 보장은 하지 않는다.

## P0-11: 전환 중 변경 유실
- 모든 원본 변경을 `change_log`에 durable sequence로 기록한다. DB별 CDC가 없다면 커넥터가 발견한 변경만 포착되므로 실시간 무손실을 주장하지 않는다.
- BUILDING 세대는 기준 snapshot/watermark를 기록하고 최초 색인 후 `change_log`를 순서대로 replay한다. 신규 세대에 적용한 마지막 sequence를 기록한다.
- 전환 경계에서는 **색인 이벤트의 게시/적용을 짧게 직렬화**하고, 활성 세대와 신규 세대의 target sequence가 같고 해당 변경이 commit/reload된 후 새 세대를 공개한다. 수집 자체는 지속 가능하나 이벤트 적재와 적용 지연을 관측한다.
- 소스별 변경 감지의 지연/누락 위험은 별도 정책과 검증에 남는다. 원본 DB의 일관된 snapshot을 보장할 수 없으면 reconciliation이 완료되기 전 스왑 금지.

## P0-12: Atomic swap과 재시작 정합성
- SQLite `index_generations`에 generation_id, state, schema/analyzer/dictionary version, source boundary, applied_sequence, manifest_checksum, timestamps 기록.
- 상태: `BUILDING -> CATCHING_UP -> VALIDATING -> READY -> ACTIVE -> RETIRED`, 실패 시 `FAILED`; 유일 ACTIVE 제약 및 전환 epoch.
- 새 IndexReader/검색 컨텍스트를 먼저 열어 health-check. SQLite 트랜잭션으로 ACTIVE pointer와 epoch를 영속화한 다음 `ArcSwap`으로 런타임 참조 교체. 두 작업은 분산 원자적이지 않으므로 crash recovery가 영속 포인터를 기준으로 재구성한다.
- 프로세스 시작 시 포인터가 READY 이상이고 manifest/commit/reader 검증이 성공해야 공개. 불일치 시 안전한 이전 세대로 복귀하거나 fail-closed하고 경보. Windows에서는 활성 디렉터리 rename에 의존하지 않는다.
- 이전 요청은 이전 Arc reader를 잡고 종료. 파일 삭제는 참조 해제/보존 기간 이후. 다중 프로세스 배포 시 이 로컬 스왑은 노드 간 원자성을 제공하지 않는다.

## P0-13: 롤백 후 변경 손실
- 이전 세대는 롤백 시점의 최신 변경을 포함하지 않을 수 있다. 이전 세대로 즉시 스왑만 하는 롤백은 기본 금지.
- 이전 세대에 전환 이후 sequence를 replay하고 commit/reload 및 ACL 검증 후 재활성화한다. 실패하면 서비스 정책에 따라 현 세대 유지 또는 제한적 읽기 모드로 전환하고 운영자에게 경보.
- 로그 보존 기간은 롤백 윈도우보다 길어야 한다. 로그가 소실되면 롤백 대신 재구축한다.

## P0-14: 세대 간 메타데이터·ACL 일관성
- 검색 결과 ID/스니펫/섹션은 generation 및 revision으로 결합한다. 검색 시 ACL 필터와 원문 열람 시 실시간 ACL 재확인. 원문 revision을 못 찾으면 임의 최신 문서로 대체하지 않고 명시적 불일치 처리.
- 권한 회수는 일반 배치 반영 지연과 별개로 즉시 서버 측 권한 검사에 적용해야 한다. 캐시 키는 principal/scope/policy_version/generation을 포함하고 변경 시 무효화.
- UI 페이지네이션 커서는 generation/epoch를 포함한다. 전환 후 이전 cursor는 해당 세대 보존 기간 동안 지원하거나 명시적으로 재검색 요청.

## P1-15: 50만 문서 색인 성능과 리소스 경합
- 50만 원본 문서와 200만~500만 section 레코드는 **용량 계획 시나리오**이며 측정된 보장치가 아니다. 1만/10만/50만 대표 샘플에서 원본 GB, PDF 파싱 비용, 초당 docs/MB, segment 크기, merge I/O, 디스크 여유를 기록.
- bounded queue, 파서 CPU 제한, writer 단일 소유, batch/commit 튜닝, 디스크 I/O 제한, 검색 P95 감시. 빌드 중 검색 지연이 한계 초과 시 자동 throttle/pause.
- `initial_build <= 8h`, `search_p95 <= 300ms`, `symbol_p95 <= 150ms`는 **가설 SLO**로 기록하며 하드웨어·코퍼스·동시 요청 조건 없이 보장하지 않는다.
- 여유 공간은 두 세대 + 병합 임시 파일 + 원문 + 로그 + 안전 마진을 실제 측정으로 산정. 디스크 부족 시 스왑 금지 및 현 서비스 유지.

## P1-16: 운영·장애·테스트
- 스왑 전 필수 검증: manifest checksum, parser/analyzer schema 호환, 대상 sequence, 문서/section 수와 삭제 tombstone, 샘플 검색, ACL 부정 테스트, reader reopen, free space, 검색 P95.
- 장애 주입: snapshot 중단, replay 중단, commit 직전/직후, READY 후, SQLite pointer commit 직전/직후, ArcSwap 직전/직후, 롤백 replay 중단, 디스크 부족, Windows 파일 핸들 잔존.
- 모듈: `dok3-index-pipeline`(bounded stages), `dok3-index-manager`(state machine/swap), `dok3-benchmark`(재현 가능한 성능 검증). 먼저 trait/상태 전이 계약과 테스트를 만들고 단일 수직 슬라이스로 구현.

## 정합성 한계
Tantivy와 SQLite 사이에 단일 원자 트랜잭션은 없다. `ArcSwap`은 프로세스 내 포인터 교체만 보장한다. 소스 변경 포착의 완전성, 서비스 가용성, 무손실 롤백은 각 데이터 소스/배포 구성에 대한 테스트 없이는 보장하지 않는다.
