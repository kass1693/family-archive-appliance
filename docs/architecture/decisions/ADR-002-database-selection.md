# ADR-002: 메타데이터 저장소 DB 선택

## 메타데이터 (Metadata)
- 작성일: 2026-06-16
- 작성자: kass1693
- 상태: Accepted
- 상태 변경 이력:
  - 2026-06-16: Proposed → Accepted

---

## 맥락 (Context)

### 문제 상황
`family-album-api`가 사용할 메타데이터 저장소를 확정해야 한다. 저장 대상은 사진 원본 파일명, 업로드 시각, 촬영 시각, EXIF 데이터, soft delete 상태 등이다. 어떤 DB를 선택하느냐에 따라 EXIF 스키마 설계 방식, Infrastructure 구현 복잡도, 미래 전환 비용이 달라진다.

### 목표
- EXIF 데이터를 유연하게 저장할 수 있어야 한다. EXIF 필드는 카메라 기종, 스마트폰 모델에 따라 키가 달라지며 전체 항목을 사전에 정의하기 어렵다.
- 날짜 범위 쿼리, soft delete 필터 등 기본적인 조회를 지원해야 한다.
- 미래에 DB를 교체하더라도 파급 범위가 Infrastructure 계층에 한정되어야 한다.
- 초기부터 "이 DB는 나중에 바꾼다"는 전제 없이 운영 가능한 선택이어야 한다.

### 제약 조건
- **기술적 제약:** Docker Compose 기반 self-hosted 환경. 별도 컨테이너 추가는 가능하나 운영 복잡도가 올라간다.
- **조직적 제약:** 단일 운영자. DB 관리 부담은 낮을수록 좋다.
- **확장 전제:** 현재는 originals, thumbnails, medium, EXIF, soft delete 수준이나 이후 태그, 앨범, 위치 정보가 추가될 수 있다.

---

## 결정 (Decision)

### 최종 선택된 방안
**MongoDB**

MongoDB를 메타데이터 저장소로 사용한다. `family-album-api` 컨테이너와 별도로 `mongodb` 컨테이너를 Docker Compose에 추가한다.

### 정책 요소 (Policy Breakdown)
- **적용 범위:** `family-album-api`의 메타데이터 저장 전체
- **핵심 규칙:**
  - MongoDB 접근은 Infrastructure 레이어에서만 수행한다.
  - `IPhotoRepository` 인터페이스를 통해 Usecase가 DB에 접근한다. Usecase는 MongoDB의 존재를 알지 않는다.
  - MongoDB document의 `_id`를 UUID로 사용한다. 파일시스템 저장 파일명의 UUID와 동일하다.
  - EXIF는 중첩 문서(embedded document)로 저장한다. 사전 스키마 정의 없이 카메라별 필드를 그대로 담는다.
- **예외 규칙:** 없음

---

## 근거 (Rationale)

### 선택한 방안의 장점
- EXIF 데이터를 별도 직렬화 없이 중첩 문서로 그대로 저장할 수 있다. 카메라 기종마다 필드가 다른 EXIF를 관계형 테이블로 표현하려면 별도 EXIF 테이블이나 JSONB 컬럼이 필요하지만, MongoDB에서는 자연스럽게 담긴다.
- 날짜 범위 쿼리, soft delete 필터는 MongoDB에서도 관계형과 동일하게 동작한다.
- 이 도메인에 JOIN이 없다. 사진 단건 조회, 날짜 범위 목록 조회, hash 존재 여부 확인이 대부분이며, 이 쿼리들은 MongoDB에 더 자연스럽게 맞는다.
- 미래 확장(태그, 앨범, 위치 정보)도 문서 추가 또는 컬렉션 확장으로 자연스럽게 수용된다.
- NoSQL을 더 선호하는 내 기술 성향과 방향이 같다.

### 고려했으나 채택하지 않은 대안

- **PostgreSQL + JSONB**
  - 장점: EXIF를 JSONB 컬럼에 담을 수 있으며, 관계형 모델의 강점인 트랜잭션과 인덱스를 함께 사용할 수 있다.
  - 단점: 관계형 모델과 문서 모델을 동시에 유지하게 된다. JSONB 쿼리는 익숙한 SQL이 아니어서 혼재 감각이 생긴다. 내가 NoSQL을 더 선호하는 상황에서 굳이 PostgreSQL을 선택할 이유가 없다.
  - 채택하지 않은 이유: 두 모델을 섞어 유지하는 인지 부담이 MongoDB 단독 사용보다 크다.

- **SQLite**
  - 장점: 별도 컨테이너가 없어 운영이 단순하다. backup-runner가 `.backup` 명령으로 일관된 snapshot을 만들기 쉽다.
  - 단점: EXIF 유연성 문제가 JSONB 방식과 동일하게 남는다. 이미 "나중에 바꾼다"는 전제가 생긴 상태에서 MVP 단순성을 위해 선택하면 결국 교체 비용만 이연된다.
  - 채택하지 않은 이유: 장기 운영을 전제로 할 때 처음부터 올바른 선택을 하는 것이 낫다.

---

## 결과 (Consequences)

### 긍정적 효과
- EXIF를 그대로 문서에 담으므로 별도 스키마 설계 없이 카메라별 필드를 전부 보존할 수 있다.
- `IPhotoRepository` 추상화로 인해 미래에 DB를 교체하더라도 변경이 Infrastructure 계층에 한정된다.

### 부정적 효과 및 위험
- Docker Compose에 `mongodb` 컨테이너가 추가되어 SQLite 대비 운영 복잡도가 높아진다.
- metadata snapshot 방식이 `mongodump`로 결정된다. backup-policy.md의 snapshot 명령어와 파일 확장자(`.archive`)는 이 결정에 따라 업데이트한다.

### 재검토 조건
- 태그, 앨범 등 확장 기능에서 컬렉션 간 복잡한 JOIN이 요구될 경우.
- 단일 운영자 환경에서 MongoDB 컨테이너 관리 부담이 실질적인 문제가 될 경우.

---

## 참고 (References)

### 관련 ADR
- [ADR-001: 백엔드 API 서버 프레임워크 선택](ADR-001-backend-framework.md)

### 관련 문서
- [시스템 아키텍처 개요](../system/overview.md)
- [백업 정책](../../backup-policy.md)
- [스토리지 정책](../../storage-policy.md)
