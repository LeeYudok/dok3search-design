# AGENTS.md — dok3search-design

이 저장소는 [`dok3search`](https://github.com/LeeYudok/dok3search) 구현 저장소와 한 세트로 관리하는 **아키텍처·데이터 계약·운영 프로토콜 설계 단일 진실 원천(SSOT)** 이다.
현재 기여 중인 에이전트(`agy`, `codex`, `claude`)뿐 아니라 향후 합류하는 어떤 에이전트(`hermes`, `opencode` 등)도 같은 규칙을 따른다.

## 1. 짝 저장소(`dok3search`) 정합성 원칙

- `dok3search` 에서 API 계약, 분류 체계(`category`·`topic`), 오류 코드, 도메인 포트/DTO, 구현 단계(P0/P1/…)가 바뀌면 이 저장소의 해당 문서(`index.*`, `map.*`, `RUST_WORKSPACE_AND_CONTRACTS.md`, `contracts/*.example.yaml`, `dok3search-implementation-roadmap.md`, `CHANGELOG.md`)도 함께 현행화한다.
- 파일을 추가·이동·삭제할 때는 `index.yaml` 의 `document_tree` 와 `map.yaml` 도 함께 점검한다(PR 간 충돌 방지를 위해 공용 인덱스 변경을 별도 후속 이슈로 분리한 경우에는 이슈에 명시한다).
- 설계 변경이 반영된 뒤에는 구현 저장소의 번들 사본(`frontend/src/samples/design/`) 및 `provenance.json` 을 동기화하고 `python3 scripts/check-design-copy.py --source ../dok3search-design` 로 검증한다.
- 날짜·버전 YAML 스칼라는 항상 작은따옴표로 감싼다(`'2026-11-22'`, `'1.0'`).

## 2. 이슈·브랜치·PR 규칙

- 작업 전에 GitHub 이슈를 열고 `owner:<agent_id>` 라벨과 담당 코멘트(`담당 / 브랜치 / 워크트리 / 상태`)를 남긴다.
- 브랜치는 `batch/<agent_id>-<YYYYMMDD>-<8자리 hex>`(`openssl rand -hex 4`) 를 쓴다. `main` 직접 push 와 squash 머지는 금지하며 PR 본문에 `Closes #N` 을 쓴다.
- 커밋 제목은 `[<모델명>] <type>: <설명> (#N)` 형식, 커밋 작성자는 에이전트 계정(`<agent_id>-ai@doksam.com`)으로 남긴다.
- 여러 에이전트가 병렬로 일할 때는 반드시 자기 `git worktree` 를 쓰며, 다른 에이전트의 워크트리나 수정 중인 PR 파일을 덮어쓰지 않는다.
- **최종 머지 승인은 사용자(`LeeYudok`)가 한다.** 에이전트는 교차 리뷰까지 마친 뒤 머지하지 않고 대기한다.

## 3. 다중 에이전트(N-agent) 협업 및 오케스트레이터 승계(Failover)

전체 협업 프로토콜 원본은 [`LeeYudok/a2a-acp-bridge#8`](https://github.com/LeeYudok/a2a-acp-bridge/issues/8)(1~5절) 이다.

- **N-에이전트 확장**: `agy`, `codex`, `claude` 외에 `hermes`, `opencode` 등 어떤 CLI/MCP 에이전트도 소문자 slug(`^[a-z0-9][a-z0-9-]{0,23}$`, 최대 24자) ID 와 `owner:<agent_id>` 라벨로 동일하게 참여할 수 있다.
- **작업 핸드오프·이어받기**: 담당 에이전트가 중단·토큰 소진 시 변경을 커밋·푸시하고 `handoff` 라벨로 전환한다. 무응답·오류·할당량 소진 시 남은 에이전트가 이슈 예고 후 원브랜치 위에서(자기 워크트리로 받아) 이어받는다.
- **오케스트레이터 무특권 순환 지휘(Failover)**:
  - 특정 에이전트가 영구 지휘권을 독점하지 않으며, 단 한 명만 현재 지휘자 리스(`term` + `lease_until`)를 가진다.
  - 현 지휘자가 중단·오류·할당량 초과·리스 만료 상태가 되면 나머지 활성 에이전트 중 한 명이 `a2a-acp-bridge#8` 에 예고(`지휘권 승계 예고`) 후 `term + 1` 리스를 확정(`지휘권 확정`)하고 모든 활성 에이전트에게 브로드캐스트한다.
  - 복귀한 전임 지휘자는 `a2a-acp-bridge#8` 의 최신 `term` 을 우선 인정하고 일반 작업자·리뷰어로 합류한다.
  - 현재 지휘자는 3개 저장소(`dok3search`, `dok3search-design`, `a2a-acp-bridge`)의 PR/이슈 상태, `dok3search#25` 포트 장부, 그리고 **`dok3search-design` 현행화** 를 함께 챙긴다.
