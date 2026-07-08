## 메타·개요 (Meta & Overview)
- 문서 ID: UPLOAD-SERVER-SDS
- 버전: 1.0.0
- 작성자: kass1693
- 작성일: 2026-07-08
- 상태: Draft
- 개요: 업로드 서버의 중복 확인, 파일 저장, EXIF 파싱, 변환 파일 생성 흐름을 설계한다.
- 관련 SRS 문서:
  - UPLOAD-SERVER-SRS
- 관련 ADR:
  - ADR-001: 백엔드 API 서버 프레임워크 선택
  - ADR-002: 메타데이터 저장소 DB 선택

---

## 설계 배경 및 목표 (Design Background & Goals)

- 설계 배경:
  - 업로드 서버 SRS에서 파일 수신, EXIF 파싱, 변환 파일 생성이 하나의 흐름으로 묶여 있으며, 중간 실패 시 파일시스템과 DB가 불일치 상태로 남지 않아야 한다는 원자성 요구사항이 있다.
  - 원본 파일 불변성과 임시 경로 기반 저장 방식이 설계의 핵심 제약이다.
- 설계 목표:
  - 업로드 흐름의 각 단계 책임을 레이어별로 명확히 분리한다.
  - 부분 실패 시 일관된 복구 경로를 정의한다.
  - 변환 파일 생성 실패가 업로드 전체를 실패로 만들지 않도록 분리한다.
- 비목표:
  - 변환 파일 생성의 비동기 처리 (MVP에서는 동기 처리)
  - 다중 파일 동시 업로드 최적화

---

## 설계 개요 (Design Overview)

- 설계 범위:
  - 중복 확인 API, 파일 업로드 API, 저장 흐름, EXIF 파싱, 변환 파일 생성
- 전체 구조 요약:
  - Presentation이 HTTP 요청을 수신하고 Usecase에 위임한다. Usecase는 흐름을 조율하며 Infrastructure를 통해 파일 I/O, DB 접근, 파싱, 변환을 수행한다. Domain은 최종 저장 경로의 형식 규칙과 엔티티를 정의한다. 임시 저장 경로(`import/pending/`)는 Infrastructure 내부 상수이며 Domain과 무관하다.
- 주요 구성 요소:
  - Presentation: `PhotoUploadController`, `DuplicateCheckController`
  - Usecase: `UploadPhotoUsecase`, `CheckDuplicatesUsecase`
  - Domain: `Photo` 엔티티, `PhotoStoragePath` 값 객체
  - Infrastructure: `PhotoFileStore`, `PhotoRepository`, `ExifParser`, `ImageConverter`
- 주요 설계 결정:
  - 중복 확인과 업로드는 별도 API로 분리한다. 중복 확인은 클라이언트 UX 목적이며 업로드를 차단하지 않는다.
  - 클라이언트가 계산한 hash는 중복 확인 UX에만 사용한다. 서버는 파일을 수신한 후 직접 SHA-256 hash를 계산하여 DB에 저장한다.
  - 파일은 먼저 Infrastructure 내부 임시 경로에 저장 후 최종 경로로 이동하여, 저장 중단 시 최종 경로에 불완전한 파일이 남지 않도록 한다.
  - DB 기록 실패 시 Usecase가 최종 경로 파일을 삭제하여 파일시스템과 DB 불일치를 방지한다.
  - 변환 파일 생성 실패는 `needsConversion` 플래그로 추적하며 업로드 성공으로 처리한다.
  - 한 번의 요청으로 여러 파일을 수신한다. 파일 중 하나라도 저장에 실패하면 전체 요청을 실패로 처리한다.

---

## 처리 흐름 (Flow)

### 중복 확인 흐름

1. [Presentation] `POST /api/photos/check-duplicates` 수신, hash 목록을 `CheckDuplicatesCommand`로 변환
2. [Usecase] `CheckDuplicatesUsecase`가 `PhotoRepository.findDuplicateHashes(hashes)` 호출
3. [Infrastructure] DB에서 일치하는 hash 목록 조회 후 반환
4. [Presentation] 결과를 응답 DTO로 변환하여 반환

### 업로드 정상 흐름

파일별로 아래 흐름이 독립적으로 실행된다.

1. [Presentation] `POST /api/photos` 수신, 파일 버퍼·원본명·MIME 타입을 `UploadPhotoCommand`로 변환
2. [Usecase] `UploadPhotoUsecase` 실행
3. [Infrastructure] `PhotoFileStore.saveToTemp(buffer)` → 임시 경로에 저장
4. [Infrastructure] `HashCalculator.compute(buffer)` → SHA-256 hash 계산 (서버 계산, 파일 저장과 독립적인 관심사)
5. [Domain] `PhotoStoragePath`로 `originals/YYYY/MM/DD/{UUID}.{ext}` 최종 경로 생성
6. [Infrastructure] `PhotoFileStore.moveToFinal(tempPath, finalPath)` → 최종 경로로 이동
7. [Infrastructure] `ExifParser.parse(finalPath)` → EXIF 파싱 (실패 시 빈 EXIF로 계속)
8. [Infrastructure] `PhotoRepository.save(photo)` → DB 레코드 생성
9. [Infrastructure] `ImageConverter.convert(finalPath)` → thumbnail, medium 생성
10. [Presentation] 전체 결과를 배열로 반환

### 예외 흐름

- **임시 저장 실패**: `PhotoFileStore`가 에러 throw → Usecase가 전파 → Presentation 로깅 후 500 응답
- **최종 경로 이동 실패**: `PhotoFileStore`가 임시 파일 정리 후 에러 throw → Presentation 로깅 후 500 응답
- **DB 기록 실패**: Usecase가 `PhotoFileStore.delete(finalPath)` 호출 후 에러 throw → Presentation 로깅 후 500 응답
- **EXIF 파싱 실패**: `ExifParser`가 내부에서 처리하여 빈 EXIF 반환, 흐름 계속
- **변환 파일 생성 실패**: Usecase가 catch하여 `PhotoRepository.markNeedsConversion(id)` 호출 후 성공 응답
- **다중 파일 중 일부 실패**: 실패 발생 시점에 전체 요청 실패로 처리. 이미 저장 완료된 파일의 롤백 여부는 미결 사항.

### 데이터 흐름

- 입력: multipart/form-data (파일 바이너리 목록, 원본 파일명, MIME 타입)
- 처리: 파일별 독립 실행 — 임시 저장 → hash 계산 → 이동 → EXIF 파싱 → DB 기록 → 변환 파일 생성
- 출력: `[{ id: string }]`
- 저장 대상: 파일시스템(`originals/`, `thumbnails/`, `medium/`), MongoDB(`photos` 컬렉션)

---

## 모듈 설계 (Module Design)

### PhotoUploadController

- 목적: 파일 업로드 HTTP 요청 처리
- 책임: multipart 파싱, 파일별 UploadPhotoCommand 생성, Usecase 위임, 응답 반환, 에러 로깅
- 입력: multipart/form-data (여러 파일)
- 출력: `[{ id: string }]` 또는 HTTP 에러 응답
- 주요 처리: 요청 수신 → 파일별 커맨드 변환 → 파일별 Usecase 호출 → 하나라도 실패 시 전체 실패 처리 → 전체 성공 시 결과 배열 반환
- 의존 대상: `UploadPhotoUsecase`
- 관련 요구사항: REQ-F-002

### DuplicateCheckController

- 목적: 중복 확인 HTTP 요청 처리
- 책임: 요청 파싱, CheckDuplicatesCommand 생성, Usecase 위임, 응답 반환, 에러 로깅
- 입력: `{ hashes: string[] }`
- 출력: `{ duplicates: string[] }`
- 의존 대상: `CheckDuplicatesUsecase`
- 관련 요구사항: REQ-F-001

### UploadPhotoUsecase

- 목적: 단일 파일 업로드 흐름 조율
- 책임: hash 계산 → 임시 저장 → 이동 → EXIF 파싱 → DB 기록 → 변환 파일 생성의 순서 보장, DB 기록 실패 시 파일 삭제, 변환 실패 시 needsConversion 기록
- 입력: `UploadPhotoCommand`
- 출력: `{ id: string }`
- 의존 대상: `PhotoFileStore`, `HashCalculator`, `PhotoRepository`, `ExifParser`, `ImageConverter`
- 관련 요구사항: REQ-F-002, REQ-F-003, REQ-F-004, REQ-NF-001, REQ-NF-002

### CheckDuplicatesUsecase

- 목적: 중복 확인 흐름 처리
- 책임: hash 목록을 DB와 비교하여 중복 항목 반환
- 입력: `CheckDuplicatesCommand`
- 출력: `{ duplicates: string[] }`
- 의존 대상: `PhotoRepository`
- 관련 요구사항: REQ-F-001

### PhotoFileStore

- 목적: 파일시스템 I/O 처리
- 책임: 임시 저장, 최종 경로 이동, 파일 삭제. 최종 경로에 저장된 파일은 이후 쓰기 접근하지 않는다 (REQ-NF-003).
- 입력: 파일 버퍼, 경로
- 출력: 저장된 경로 또는 에러
- 의존 대상: 파일시스템

### PhotoRepository

- 목적: DB CRUD
- 책임: Photo 도큐먼트 저장, hash 기반 조회, needsConversion 갱신
- 의존 대상: MongoDB

### ExifParser

- 목적: 원본 파일에서 EXIF 추출
- 책임: EXIF 파싱, 실패 시 빈 결과 반환
- 입력: 파일 경로
- 출력: `Record<string, unknown>` (파싱 실패 시 빈 객체)

### HashCalculator

- 목적: 파일 바이너리의 SHA-256 hash 계산
- 책임: buffer를 입력받아 SHA-256 hex 문자열 반환. 파일 저장과 무관한 독립적인 연산.
- 입력: 파일 buffer
- 출력: SHA-256 hex 문자열
- 의존 대상: 없음 (표준 crypto 라이브러리)

### ImageConverter

- 목적: 서비스용 변환 파일 생성
- 책임: thumbnail, medium 크기 파일 생성
- 입력: 원본 파일 경로, 대상 저장 경로
- 출력: 생성된 파일 경로 목록 또는 에러

---

## 인터페이스 설계 (Interface Design)

### POST /api/photos/check-duplicates

- 목적: 클라이언트가 제출한 hash 목록 중 DB에 이미 존재하는 항목 반환
- 호출 주체: 클라이언트
- 처리 주체: DuplicateCheckController → CheckDuplicatesUsecase
- 입력: `{ hashes: string[] }` (SHA-256 hex 문자열 목록)
- 출력: `{ duplicates: string[] }` (중복된 hash 목록)
- 반환 규칙:
  - 성공: 200, 중복 없으면 `{ duplicates: [] }`
  - 실패: 500
- 제약 사항: 업로드를 차단하지 않는다. 정보 제공 목적이다.
- 관련 요구사항: REQ-F-001

### POST /api/photos

- 목적: 파일 업로드 수신 및 저장 처리
- 호출 주체: 클라이언트
- 처리 주체: PhotoUploadController → UploadPhotoUsecase (파일별 반복)
- 입력: multipart/form-data (`files[]` 필드, 복수)
- 출력: `[{ id: string }]` (성공한 파일의 UUID 목록)
- 반환 규칙:
  - 전체 성공: 201
  - 실패: 500 (개별 파일 실패는 다른 파일 처리를 중단하지 않음)
- 제약 사항: 동일 hash라도 거부하지 않는다.
- 관련 요구사항: REQ-F-002, REQ-F-003, REQ-F-004

---

## 데이터 설계 (Data Design)

### UploadPhotoCommand (Usecase 입력 DTO)

- 목적: Presentation에서 Usecase로 파일 정보 전달
- 사용 위치: Presentation → Usecase 경계

| 필드 | 타입 | 필수 여부 | 제약 | 설명 |
| -- | -- | ----- | -- | -- |
| buffer | Buffer | 필수 | - | 파일 바이너리 |
| originalName | string | 필수 | - | 클라이언트 원본 파일명 |
| mimeType | string | 필수 | - | MIME 타입 |

### Photo (Domain 엔티티)

- 목적: 파일 식별 및 메타데이터 표현
- 사용 위치: Usecase, Infrastructure

| 필드 | 타입 | 필수 여부 | 제약 | 설명 |
| -- | -- | ----- | -- | -- |
| id | string | 필수 | UUID | 파일 고유 식별자, 파일명에도 사용 |
| hash | string | 필수 | SHA-256 hex | 서버가 파일 수신 후 계산한 hash |
| storagePath | string | 필수 | - | originals/ 기준 상대 경로 |
| capturedAt | Date \| null | 선택 | - | EXIF 기반 촬영 시각 |
| gps | object \| null | 선택 | - | EXIF 기반 GPS 좌표. `{ lat: number, lng: number, altitude?: number }` |
| uploadedAt | Date | 필수 | - | 업로드 시각 |
| exif | object | 필수 | 빈 객체 허용 | EXIF 원본 데이터 전체 |
| needsConversion | boolean | 필수 | - | 변환 파일 재생성 필요 여부 |
| deletedAt | Date \| null | 선택 | - | soft delete 시각 |

### MongoDB photos 컬렉션

- 목적: Photo 영구 저장
- `_id` 필드는 Photo.id (UUID) 사용

---

## 상태 및 정책 설계 (State & Policy Design)

- 주요 상태:
  - `pending`: import/pending/{UUID}에 임시 저장된 파일
  - `stored`: originals/YYYY/MM/DD/{UUID}.{ext}에 최종 저장된 파일
- 정책:
  - 동일 hash라도 업로드 가능. DB에 unique constraint 없음.
  - 변환 파일 생성 실패 시 `needsConversion=true` 기록. 이후 별도 재생성 가능.
  - 원본 파일은 저장 후 수정 불가. 변환 파일 생성 시에도 원본을 읽기 전용으로 접근.
- 정책 적용 위치: 원자성·복구는 Usecase, 최종 저장 경로 형식 규칙은 Domain, 임시 저장 경로는 Infrastructure 내부 상수

| 현재 상태 | 조건/이벤트 | 다음 상태 | 처리 내용 |
| ----- | ------ | ----- | ----- |
| - | 업로드 요청 수신 | pending | import/pending/{UUID}에 임시 저장 |
| pending | 최종 경로 이동 성공 | stored | originals/YYYY/MM/DD/{UUID}.ext로 이동 |
| pending | 최종 경로 이동 실패 | - | 임시 파일 정리, 에러 전파 |
| stored | DB 기록 실패 | - | 원본 파일 삭제, 에러 전파 |
| stored | 변환 파일 생성 실패 | stored (needsConversion=true) | 실패 기록, 업로드 성공 처리 |

---

## 예외 처리 설계 (Exception Handling Design)

- 예외 처리 원칙: Infrastructure는 실패 컨텍스트를 에러 객체에 담아 throw한다. Usecase는 복구 가능한 케이스만 처리하고 나머지는 전파한다. 로깅은 Presentation에서만 수행한다.
- 복구 가능 여부: DB 기록 실패는 Usecase가 파일 삭제 후 복구 불가로 전파. 변환 파일 생성 실패는 Usecase가 플래그 기록 후 성공 처리.

| 예외 상황 | 발생 위치 | 처리 방식 | 반환/응답 | 로그 여부 |
| ----- | ----- | ----- | ----- | ----- |
| 임시 저장 실패 | Infrastructure | 에러 throw | 500 | Presentation |
| 최종 경로 이동 실패 | Infrastructure | 임시 파일 정리 후 에러 throw | 500 | Presentation |
| DB 기록 실패 | Infrastructure | 에러 throw → Usecase가 파일 삭제 후 re-throw | 500 | Presentation |
| 원본 파일 삭제 실패 (롤백 시) | Usecase | 에러 로그 후 원래 에러 전파 | 500 | Presentation |
| EXIF 파싱 실패 | Infrastructure | 내부 처리, 빈 EXIF 반환 | 흐름 계속 | 없음 |
| 변환 파일 생성 실패 | Infrastructure | 에러 throw → Usecase가 needsConversion 기록 후 성공 처리 | 201 | 없음 |

---

## 외부 의존성 (External Dependencies)

| 의존 대상 | 종류 | 사용 목적 | 실패 시 처리 | 비고 |
| ----- | -- | ----- | ------- | -- |
| 파일시스템 | 파일 | 파일 저장·이동·삭제 | 에러 throw | import/pending, originals, thumbnails, medium 경로 |
| MongoDB | DB | Photo 메타데이터 CRUD | 에러 throw | ADR-002 |
| EXIF 파싱 라이브러리 | 라이브러리 | EXIF 추출 | 빈 결과 반환 | exifr 등 |
| 이미지 변환 라이브러리 | 라이브러리 | thumbnail·medium 생성 | needsConversion 기록 | sharp 등 |

---

## 요구사항 매핑 (Requirements Mapping)

| 요구사항 ID | 요구사항 요약 | 설계 반영 위치 | 설계 내용 |
| ---------- | ------- | -------- | ----- |
| REQ-F-001 | 중복 확인 응답 | CheckDuplicatesUsecase, PhotoRepository | hash 목록 비교 후 중복 항목 반환 |
| REQ-F-002 | 파일 수신 및 저장 | UploadPhotoUsecase, PhotoFileStore | pending → stored 상태 전이, 임시 저장 후 이동 |
| REQ-F-003 | EXIF 파싱 및 메타데이터 저장 | ExifParser, PhotoRepository | 파싱 실패 시 빈 EXIF로 저장 |
| REQ-F-004 | 변환 파일 생성 | ImageConverter, UploadPhotoUsecase | 실패 시 needsConversion=true 기록 |
| REQ-NF-001 | 업로드 원자성 | UploadPhotoUsecase | 전 단계 완료 시 성공, 중간 실패 시 복구 |
| REQ-NF-002 | 부분 실패 시 상태 일관성 | UploadPhotoUsecase | DB 실패 시 Usecase가 파일 삭제 |
| REQ-NF-003 | 원본 파일 불변성 | PhotoFileStore | 최종 경로 파일은 저장 이후 쓰기 접근하지 않음. 변환 파일 생성 시에도 원본을 읽기 전용으로 접근. |

---

## 리스크 및 제약 (Risks & Constraints)

### 리스크

| 리스크 | 영향 | 대응 방안 |
| --- | -- | ----- |
| DB 기록 실패 후 파일 삭제도 실패 | 파일시스템에 고아 파일 발생 | 에러 로깅으로 수동 정리 가능하도록 경로 포함 기록 |
| 대용량 파일 업로드 중 서버 재시작 | import/pending에 임시 파일 잔류 | 별도 cleanup 정책으로 처리 (이 문서 범위 외) |

### 제약

- 변환 파일 생성은 업로드 요청 처리 중 동기 수행 (MVP 한계)
- 파일시스템과 MongoDB는 트랜잭션으로 묶을 수 없음. Usecase의 수동 복구에 의존.

---

## 미결 사항 (Open Issues)

| 항목 | 설명 | 담당 | 결정 필요 시점 |
| -- | -- | -- | -------- |
| 변환 파일 비동기 처리 | 업로드 응답 후 백그라운드에서 변환 파일 생성 여부. needsConversion 플래그로 향후 전환 가능. | kass1693 | 성능 문제 발생 시 |
| import/pending 정리 정책 | 임시 파일 잔류 시 정리 주기와 방식 | kass1693 | cleanup SRS 작성 시 |
| 다중 파일 부분 실패 시 롤백 | 일부 파일 저장 성공 후 다른 파일 실패 시 성공한 파일을 롤백할지 여부 | kass1693 | 구현 시 |

---

## 변경 기록 (Change Log)
- 2026-07-08: 초안 작성

---

## 참고 자료 (References)
- SRS 문서:
  - [업로드 서버 요구사항](../requirements/upload-server.md)
- ADR:
  - [ADR-001: 백엔드 API 서버 프레임워크 선택](../decisions/ADR-001-backend-framework.md)
  - [ADR-002: 메타데이터 저장소 DB 선택](../decisions/ADR-002-database-selection.md)
- 관련 문서:
  - [소프트웨어 아키텍처 개요](../architecture/software/overview.md)
  - [에러 전파 정책](../architecture/software/error-policy.md)
