# Express + TypeScript 프로젝트 구조

#express #typescript #typedi #layered-architecture

> Express 5 + TypeScript + typedi 기반 REST API 프로젝트의 레이어드 아키텍처 구성

## 기술 스택

| 항목 | 기술 | 비고 |
|-|-|-|
| 런타임 | Node.js + TypeScript | ESM (`"type": "module"`) |
| 프레임워크 | Express 5 | |
| DI | typedi | `@Service()` 데코레이터 + `Container.get()` |
| ORM | TypeORM | PostgreSQL |
| 인증 | JWT | bcryptjs + jsonwebtoken |
| 빌드 | tsc → `dist/` | 개발 시 `tsx watch` |

## 디렉토리 구조

```
src/
├── index.ts              # 앱 진입점 (서버 시작)
├── config/               # 환경 설정 (dotenv → config 객체)
│   ├── index.ts
│   └── dtos/             # 공통 DTO (페이징, 응답 래퍼 등)
├── loaders/              # 앱 초기화 (Express, DB 연결)
│   ├── index.ts          # load() 함수 — 초기화 순서 관리
│   ├── express.ts        # 미들웨어 등록, 라우트 마운트
│   └── database.ts       # TypeORM DataSource 초기화
├── routes/               # 라우터 (URL → 핸들러 매핑)
│   └── index.ts          # 전체 라우트 등록
├── controllers/          # 컨트롤러 (요청/응답 처리)
├── services/             # 서비스 (비즈니스 로직, @Service)
├── entities/             # TypeORM 엔티티 (DB 테이블 매핑)
├── dtos/                 # 요청/응답 DTO 인터페이스
└── middleware/           # 공통 미들웨어 (인증, 에러 핸들러)
```

## 레이어별 역할

### config/
- `.env` 값을 `dotenv`로 로드
- `config` 객체로 export (`port`, `jwtSecret`, `cors.origin` 등)
- 새 환경변수 필요 시 `config/index.ts`에 추가

### loaders/
- Express 미들웨어 등록 (cors, json 파싱, 라우트, 에러 핸들러)
- DB 연결 등 비동기 초기화
- `loaders/index.ts`의 `load()` 함수에서 순서 제어

### routes/
- HTTP 엔드포인트 정의 → service 호출
- 새 도메인 추가 시 `routes/{domain}.ts` 생성 → `routes/index.ts`에 등록

### services/
- `@Service()` 데코레이터로 typedi 컨테이너에 등록
- DB CRUD, 데이터 가공, 외부 API 호출
- 새 도메인 추가 시 `services/{Domain}Service.ts` 생성

### entities/
- TypeORM `@Entity()` 데코레이터로 DB 테이블 매핑
- 컬럼 정의, 관계 설정

### dtos/
- 요청 body, 응답의 TypeScript interface 정의
- `Create{Domain}DTO`, `Update{Domain}DTO` 패턴

### middleware/
- `isAuth` — JWT 토큰 검증
- `errorHandler` — 전역 에러 처리
- 유효성 검증, 로깅, 권한 체크 등 공통 로직

## 요청 흐름

```
클라이언트 요청
  → Express 미들웨어 (cors, json 파싱)
    → routes (URL 매핑 + middleware 적용)
      → service (비즈니스 로직)
        → entity (DB 조회/저장)
          → 응답 반환
  → errorHandler (에러 발생 시)
```
