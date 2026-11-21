# 실제 오픈소스 사례와 dok3search 적용 경계

> 설계 참조 자료. 아래 프로젝트의 세부 구현·버전별 동작은 도입 시 소스 코드/라이선스 검토가 필요하다. dok3search가 이들의 운영 성능을 자동으로 상속하는 것은 아니다.

| 프로젝트 | 공개 소스 | 참조할 패턴 | 복사하지 않을 것 |
|---|---|---|---|
| Quickwit | https://github.com/quickwit-oss/quickwit | Rust/Tantivy 기반 색인 파이프라인, 세그먼트·검색/색인 자원 관리 | 분산 클러스터 전체 아키텍처 |
| Meilisearch | https://github.com/meilisearch/meilisearch | 작업 상태/비동기 색인 API, 검색 UX와 오류 모델 | 자체 저장소 구조의 무비판적 이식 |
| Zoekt | https://github.com/sourcegraph/zoekt | 코드 검색, 저장소·파일·라인 단위 결과 | Tantivy 대체 엔진 추가 |
| Elasticsearch | https://www.elastic.co/guide/en/elasticsearch/reference/current/aliases.html | 인덱스 별칭 기반 전환이라는 운영 패턴 | Tantivy에 없는 alias API가 있다고 가정 |
| OpenSearch | https://docs.opensearch.org/latest/im-plugin/index-alias/ | 인덱스 별칭·전환 개념 | 외부 검색 서버 도입 |

## 설계 결정
1. **Tantivy 단일 검색엔진**. Rust Axum 서버 안에 내장하고 검색/색인용 별도 HTTP 서비스는 기본으로 만들지 않는다.
2. **Blue-Green은 dok3search 자체 Generation Manager가 구현**한다. Elasticsearch alias를 직접 사용하는 것이 아니다.
3. **검색 품질과 코드 검색은 별도 검증**한다. Zoekt의 코드 탐색 패턴을 참고하되 Tantivy 필드/토크나이저로 재현 가능한 범위를 먼저 실험한다.
4. **라이선스 검토**: 구현/코드 복사 전 각 저장소의 현재 LICENSE, 의존성 라이선스, 특허·고지 의무를 확인한다. 아키텍처 패턴 참조와 코드 복사는 다르다.
5. **폐쇄망**: 참고 저장소를 운영 필수 의존성으로 만들지 않는다. 소스·빌드 도구·매뉴얼은 승인된 오프라인 공급망으로 반입한다.

## 비교 평가 체크리스트
- [ ] 공식 저장소의 태그/커밋을 고정하여 코드·문서 검토
- [ ] 50만 원본 / 최대 500만 섹션 기준 색인 성능 검증
- [ ] Windows Server 빌드·파일 핸들·백신/EDR 간섭 검증
- [ ] 한국어·영어·코드 심볼 검색 품질 테스트
- [ ] 검색 중 재색인, 장애, 복구, 롤백 재현
- [ ] 라이선스/SBOM/오프라인 설치 패키지 검증
