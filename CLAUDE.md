# Dreampea 프로젝트 claude.md

## 프로젝트 개요
드림피아(Dreampea)는 학교, 진로센터 등 교육기관에 진로 수업을 제공하는 업체의 내부 관리 시스템이다.
강사를 모집하여 기관에 파견하는 비즈니스 모델을 운영하며, 반복 업무를 시스템화하는 것이 목적이다.

---

## 기술 스택

| 항목 | 기술 |
|---|---|
| Framework | Next.js (App Router) |
| Language | TypeScript |
| Database | Supabase (PostgreSQL) |
| DB 클라이언트 | Supabase Client (`@supabase/supabase-js`) 주력 |
| 스타일링 | Tailwind CSS |
| 폼 관리 | React Hook Form |
| 유효성 검사 | Zod |
| 상태 관리 | Zustand |
| 테이블 | TanStack Table |
| 배포 | Vercel |

---

## 프로젝트 구조

```
/app                  # Next.js App Router 페이지
/components
  /ui                 # 버튼, 인풋 등 공통 컴포넌트
  /features           # 도메인별 컴포넌트 (행사관리, 강사관리 등)
/lib                  # 유틸 함수, Supabase 클라이언트 초기화 등
/types                # TypeScript 타입 정의
/hooks                # 커스텀 훅
/constants            # enum 값, 상수 등
```

---

## Supabase 설정

- Supabase Client를 주력으로 사용한다.
- `/lib/supabase.ts`에서 클라이언트를 초기화하고 전역에서 import하여 사용한다.
- 관리자 웹과 강사 웹이 동일한 Supabase 프로젝트를 공유한다.
- RLS(Row Level Security) 정책은 두 웹의 접근 권한을 모두 고려하여 설계한다.
- 환경변수는 아래 형식을 따른다.

```env
NEXT_PUBLIC_SUPABASE_URL=
NEXT_PUBLIC_SUPABASE_ANON_KEY=
SUPABASE_SERVICE_ROLE_KEY=
```

---

## 인증 및 권한

### 로그인 주체
- **관리자(admins)** — 현재 웹(관리자 웹)에서 로그인
- **멘토(mentors)** — 별도 강사 웹에서 회원가입 및 로그인 (현재 웹에서는 관리자가 관리만 함)

### 관리자 역할 (admins 테이블)
| 컬럼 | 설명 |
|---|---|
| `is_super` | 슈퍼관리자. 다른 관리자 승인 권한 보유 |
| `is_authenticated` | 승인된 관리자 여부. `is_super`인 관리자만 변경 가능 |
| `is_sales` | 영업담당자 여부 |
| `is_comm` | 소통담당자 여부 |

- 역할은 동시에 보유 가능하다.
- `is_authenticated`가 `false`인 관리자는 로그인 후 접근이 제한된다.
- `is_authenticated` 승인은 `is_super = true`인 관리자만 가능하다.

### 멘토 정보 수정
- 멘토는 직접 정보를 수정할 수 없다.
- 멘토가 수정 요청(`mentor_requests`)을 제출하면 관리자가 처리한다.

---

## 비즈니스 로직 핵심 규칙

### 재고 차감
- `event_rows`에 row가 생성되고 `session_headcount` 값이 입력되면 재고가 차감된다.
- 재고는 `supply_logs` 테이블에 로그를 남기는 방식으로 관리한다.
- 재고 종류는 `total`(총재고)과 `kit`(키트재고) 두 가지이며, `free`(여유재고)는 `total - kit`으로 계산한다.
- `delta` 값은 양수=입고, 음수=출고를 의미한다.
- 소모성(`is_consumable = true`) 재료만 차감 처리한다.

### 행사 체크 상태 (event_check_status)
- `smallint` 타입으로 1~4 값만 허용한다.
- 각 단계의 의미는 추후 확정 예정이다.

### 강사료/재료비 입금자
- 기본값은 `mentor_occupation_programs` 테이블의 `lecture_fee_payer_id`, `material_fee_payer_id`에서 가져온다.
- `event_rows`에 값이 있으면 오버라이드한다. 없으면(`null`) 기본값을 사용한다.

### 여유재고 계산
```
여유재고(free) = 총재고(total) - 키트재고(kit)
```

---

## 코딩 컨벤션

### 네이밍
- 컴포넌트 — PascalCase (`EventTable.tsx`)
- 함수/변수 — camelCase (`fetchEvents`)
- 타입/인터페이스 — PascalCase (`EventRow`)
- 상수 — UPPER_SNAKE_CASE (`MAX_STOCK`)
- DB 컬럼명 — snake_case (Supabase 기본)

### 컴포넌트
- 함수형 컴포넌트만 사용한다.
- `export default`는 페이지 컴포넌트에만 사용하고, 나머지는 named export를 사용한다.

### 폼
- 모든 폼은 React Hook Form + Zod 조합으로 작성한다.
- Zod 스키마는 `/lib/validations/` 디렉토리에 도메인별로 분리하여 관리한다.

### 타입
- Supabase 테이블 타입은 Supabase CLI로 자동 생성하여 `/types/supabase.ts`에서 관리한다.
- 추가 커스텀 타입은 `/types/` 디렉토리에 도메인별로 분리한다.

### 상태관리
- 서버 데이터(DB 조회)는 Supabase Client로 직접 패치한다.
- 전역 클라이언트 상태(로그인 정보, UI 상태 등)는 Zustand로 관리한다.
- Zustand 스토어는 `/lib/store/` 디렉토리에 도메인별로 분리한다.

### 주석
- 주석은 한글로 작성한다.

---

## 주요 테이블 관계 요약

```
fields → occupations → occupation_programs
                              ↓
institutions → events → event_rows ← mentors
                  ↓
           event_schedules (교시별 시간표)
                              ↓
                        supply_logs ← supplies
```

---

## 미결 사항 (확정 시 업데이트 필요)

- `event_check_status` 1~4 단계별 의미 정의
- 도메인 확정 후 Vercel 환경변수 업데이트
