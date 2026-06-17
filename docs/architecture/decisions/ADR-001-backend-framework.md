# ADR-001: 백엔드 API 서버 프레임워크 선택

## 메타데이터 (Metadata)
- 작성일: 2026-06-16
- 작성자: kass1693
- 상태: Accepted
- 상태 변경 이력:
  - 2026-06-16: Proposed → Accepted

---

## 맥락 (Context)

### 문제 상황
`family-album-api`를 Node.js + TypeScript로 구현할 때 어떤 프레임워크를 사용할 것인지 결정해야 한다. 프레임워크 선택은 레이어 분리 구현 방식과 DTO 계층 명시성에 직접 영향을 준다.

### 목표
- Presentation / Usecase / Domain / Infrastructure 레이어를 프레임워크 강제 없이 명시적으로 분리할 수 있어야 한다.
- JSON → DTO → Domain 변환 계층을 구조적으로 명확하게 유지할 수 있어야 한다.
- 단일 운영자가 장기 유지보수하기 용이해야 한다.

### 제약 조건
- **기술적 제약:** Node.js + TypeScript 기반 고정. 런타임은 self-hosted Docker 환경.
- **조직적 제약:** 단일 운영자. 과도한 학습 비용은 피한다.
- **시간적 제약:** MVP 우선. 과잉 설계는 배제한다.

---

## 결정 (Decision)

### 최종 선택된 방안
**Fastify + 직접 레이어 설계**

Fastify를 HTTP 레이어로 사용하고, 레이어 분리(Presentation / Usecase / Domain / Infrastructure)는 프레임워크 구조가 아닌 코드 구조로 직접 구현한다.

### 정책 요소 (Policy Breakdown)
- **적용 범위:** `family-album-api` 컨테이너 전체
- **핵심 규칙:**
  - Fastify route handler는 Presentation 레이어다. 비즈니스 로직을 포함하지 않는다.
  - 요청 스키마(`schema.body`, `schema.querystring`)는 Presentation 레이어 DTO 경계로 사용한다.
  - Usecase, Domain, Infrastructure는 Fastify와 무관한 순수 TypeScript 클래스/함수로 구현한다.
- **예외 규칙:** 없음

---

## 근거 (Rationale)

### 선택한 방안의 장점
- Fastify는 구조를 강제하지 않으므로 내가 정의한 레이어 원칙을 그대로 따를 수 있다.
- 스키마 기반 직렬화/검증이 TypeScript 타입과 연동되어 DTO 경계의 명시성을 구조적으로 지원한다.
- TypeScript-first 설계로 타입 추론이 스키마와 연동된다.
- self-hosted Docker 환경에서 생태계 성숙도와 파일 업로드(`@fastify/multipart`) 지원이 충분하다.

### 고려했으나 채택하지 않은 대안

- **NestJS**
  - 장점: 구조가 이미 잡혀 있어 초기 설정 비용이 낮다. DI 컨테이너 제공.
  - 단점: Module/DI 경계가 내가 원하는 레이어 경계와 충돌한다. 이전 프로젝트에서 "아키텍처 경계가 NestJS에 있었다"는 불편함을 직접 경험했다. 내 레이어 구조를 NestJS 방식 위에 억지로 매핑하게 된다.
  - 채택하지 않은 이유: 아키텍처 경계의 주도권을 프레임워크에 넘기게 된다.

- **Express.js**
  - 장점: 가장 넓은 생태계, 낮은 학습 비용.
  - 단점: DTO 경계를 명시적으로 유지하려면 zod 등을 별도로 통합해야 하며, 그 경계가 느슨해질 수 있다.
  - 채택하지 않은 이유: Fastify와 레이어 설계 자유도는 동일하나 스키마 기반 DTO 명시성이 더 약하다.

- **Hono**
  - 장점: 경량, TypeScript-first.
  - 단점: Edge/Cloudflare Workers 환경 특화. self-hosted Node.js Docker 환경에서 실질적 이점이 없다. 파일 업로드, SQLite 통합 사례가 상대적으로 얕다.
  - 채택하지 않은 이유: 이 프로젝트 환경에서 검증된 이점이 없으며 불필요한 미지 변수를 추가한다.

---

## 결과 (Consequences)

### 긍정적 효과
- 레이어 구조를 내가 원하는 방식으로 직접 제어할 수 있다.
- Fastify 스키마가 Presentation 레이어 DTO 경계를 명시적으로 강제한다.
- 프레임워크와 무관하게 Usecase/Domain 계층을 독립적으로 테스트할 수 있다.

### 부정적 효과 및 위험
- Express에 비해 생태계 레퍼런스가 상대적으로 적다.
- 레이어 구조를 직접 설계해야 하므로 NestJS 대비 초기 구조 설정 비용이 있다.

### 재검토 조건
- Node.js 런타임을 벗어나는 방향으로 기술 스택이 전환될 경우.
- Fastify 생태계에서 해결하지 못하는 요구사항이 등장할 경우.

---

## 참고 (References)

### 관련 ADR
- 없음 (첫 번째 ADR)

### 관련 문서
- [시스템 아키텍처 개요](../system/overview.md)
