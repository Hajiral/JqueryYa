# JqueryYa — Web File Manager API Architecture v1.0

## 1. 목적 및 범위
Node.js(Express), MySQL, 로컬 파일 시스템을 활용한 웹 기반 파일 관리 시스템. FTP 프로토콜은 사용하지 않는다. 회원 인증을 제외한 평가 대상 라우터는 Files(8), Folders(7), Shares(7), Trash(6)로 총 28개 API이다. 이 문서는 구현 전 설계안이며 API 구현 완료를 의미하지 않는다.

기본 경로: `/api`; 요청·응답: JSON(파일 업로드는 multipart/form-data, 다운로드는 바이너리); 인증: 서버 세션과 HttpOnly 쿠키. 소유자 검증은 모든 비공개 API에 적용한다.

## 2. 라우터 및 엔드포인트

### 2.1 Files — `/api/files` (8개)
| Method | Path | 설명 | 주요 입력 | 성공 |
|---|---|---|---|---|
| POST | `/api/files` | 파일 업로드 | multipart `file`, `folderId`(선택) | 201 |
| GET | `/api/files` | 파일 목록 | `folderId`, `page`, `limit`(선택) | 200 |
| GET | `/api/files/:id` | 파일 메타데이터 | 경로 `id` | 200 |
| GET | `/api/files/:id/download` | 파일 다운로드 | 경로 `id` | 200 |
| PATCH | `/api/files/:id` | 파일 이름 변경 | JSON `name` | 200 |
| DELETE | `/api/files/:id` | 휴지통 이동 | 경로 `id` | 204 |
| POST | `/api/files/:id/copy` | 파일 복사 | JSON `targetFolderId` | 201 |
| PATCH | `/api/files/:id/move` | 파일 이동 | JSON `targetFolderId` | 200 |

### 2.2 Folders — `/api/folders` (7개)
| Method | Path | 설명 | 주요 입력 | 성공 |
|---|---|---|---|---|
| POST | `/api/folders` | 폴더 생성 | JSON `name`, `parentId`(선택) | 201 |
| GET | `/api/folders` | 폴더 목록 | `parentId`, `page`, `limit`(선택) | 200 |
| GET | `/api/folders/:id` | 폴더 상세 | 경로 `id` | 200 |
| GET | `/api/folders/:id/contents` | 하위 폴더·파일 조회 | 경로 `id` | 200 |
| PATCH | `/api/folders/:id` | 이름 변경 | JSON `name` | 200 |
| PATCH | `/api/folders/:id/move` | 폴더 이동 | JSON `targetParentId` | 200 |
| DELETE | `/api/folders/:id` | 휴지통 이동 | 경로 `id` | 204 |

### 2.3 Shares — `/api/shares` (7개)
| Method | Path | 설명 | 주요 입력 | 성공 |
|---|---|---|---|---|
| POST | `/api/shares` | 공유 링크 생성 | JSON `fileId`, `expiresAt`(선택) | 201 |
| GET | `/api/shares` | 내 공유 목록 | `page`, `limit`(선택) | 200 |
| GET | `/api/shares/:id` | 공유 정보 | 경로 `id` | 200 |
| PATCH | `/api/shares/:id` | 공유 설정 변경 | JSON `expiresAt`, `isActive` | 200 |
| DELETE | `/api/shares/:id` | 공유 해제 | 경로 `id` | 204 |
| GET | `/api/shares/:id/access` | 공유 상태 확인(소유자) | 경로 `id` | 200 |
| GET | `/api/shares/:id/download` | 소유자용 공유 파일 다운로드 | 경로 `id` | 200 |

공개 링크용 보조 API(28개에 미포함): `GET /api/public/shares/:token`(공유 파일 정보), `GET /api/public/shares/:token/download`(다운로드). 무작위 토큰을 발급하고 DB에는 해시만 저장하며 만료·비활성·파일 삭제 여부를 검사한다.

### 2.4 Trash — `/api/trash` (6개)
| Method | Path | 설명 | 주요 입력 | 성공 |
|---|---|---|---|---|
| GET | `/api/trash` | 삭제 항목 목록 | `page`, `limit`(선택) | 200 |
| GET | `/api/trash/stats` | 삭제 항목 통계 | 없음 | 200 |
| GET | `/api/trash/:id` | 삭제 항목 상세 | 경로 `id` | 200 |
| POST | `/api/trash/:id/restore` | 복원 | 경로 `id` | 200 |
| DELETE | `/api/trash/:id` | 영구 삭제 | 경로 `id` | 204 |
| DELETE | `/api/trash` | 휴지통 비우기 | 없음 | 204 |

**라우팅 주의:** `/api/trash/stats`는 `/api/trash/:id`보다 먼저 선언한다.

### 2.5 Auth — `/api/auth` (평가 대상 제외)
| Method | Path | 설명 |
|---|---|---|
| POST | `/api/auth/register` | 회원가입 |
| POST | `/api/auth/login` | 로그인·세션 생성 |
| POST | `/api/auth/logout` | 로그아웃·세션 무효화 |
| GET | `/api/auth/me` | 현재 사용자 조회 |
| PATCH | `/api/auth/password` | 비밀번호 변경 |

## 3. 공통 응답 형식
일반 성공(200/201):
```json
{"success":true,"data":{"id":101,"name":"example.pdf"}}
```
실패:
```json
{"success":false,"error":{"code":"FILE_NOT_FOUND","message":"파일을 찾을 수 없습니다."}}
```
204 응답은 본문이 없다. 바이너리 다운로드는 JSON 래퍼 없이 Content-Type, Content-Disposition 등 적절한 헤더를 사용한다.

상태 코드: 200 조회/수정, 201 생성, 204 본문 없는 삭제 성공, 400 입력 오류, 401 미인증, 403 권한 부족, 404 없음, 409 이름·상태 충돌, 500 서버 오류.

## 4. 데이터 구조
- `users`: 사용자 ID, 로그인 식별자, 비밀번호 해시.
- `sessions`: 세션 저장소(개발/배포 환경에 맞는 MySQL 기반 저장소 사용).
- `folders`: ID, 소유자 ID, 부모 폴더 ID(nullable), 이름, 삭제 시각(nullable).
- `files`: ID, 소유자 ID, 폴더 ID(nullable), 원본 이름, 내부 저장 키, MIME, 크기, 삭제 시각(nullable).
- `shares`: ID, 파일 ID, 토큰 해시, 만료 시각, 활성 여부, 다운로드 수.
- `trash_records`: ID, 소유자 ID, 파일 또는 폴더 참조(둘 중 하나), 원래 부모 ID, 삭제 시각.

실제 파일 바이트는 서버의 로컬 저장소에 두고 MySQL에는 경로 대신 안전한 내부 저장 키 및 메타데이터를 보관한다.

## 5. 구현 규칙
1. 모든 비공개 조회·변경은 세션의 사용자 ID와 리소스 소유자를 함께 확인한다.
2. 폴더를 자신 또는 자손 폴더로 이동하는 작업을 금지한다.
3. 삭제·복원 시 하위 폴더와 파일의 처리 정책을 일관되게 적용한다(구현 전 상세 확정 필요).
4. 휴지통에 있는 파일은 일반 다운로드와 공개 공유 다운로드에서 접근을 차단한다.
5. 파일 이동·복사·삭제 과정에서 DB 트랜잭션과 파일 시스템 작업 간 불일치 복구를 설계한다.
6. 이름 중복 정책을 동일 사용자·부모 폴더 범위에서 정의하고 충돌 시 409를 반환한다.
7. 업로드 크기 제한, 허용 파일명 검증, 경로 탐색 방지, 안전한 다운로드 파일명 처리를 적용한다.
8. `node_modules/`, `.env`, 실제 업로드 파일은 Git에 포함하지 않는다.

## 6. 코드 디렉터리 제안
```text
src/
  routes/{auth,files,folders,shares,trash}.routes.js
  controllers/{files,folders,shares,trash}.controller.js
  services/{files,folders,shares,trash}.service.js
  repositories/{files,folders,shares,trash}.repository.js
  middleware/{requireAuth,errorHandler}.js
  config/database.js
  app.js
  server.js
database/schema.sql
tests/
storage/
.env.example
.gitignore
README.md
```

## 7. 테스트 기준
- Files: 업로드 후 DB 레코드 및 실제 파일 존재 검증, 다운로드 내용 일치.
- Folders: 부모-자식 관계, 순환 이동 차단.
- Shares: 생성·만료·비활성·토큰 접근 검증.
- Trash: 삭제 후 목록 노출, 복원 후 원래 위치 확인, 영구 삭제.
- 공통: 미로그인 401, 타 사용자 접근 차단, 잘못된 입력 400, 없는 ID 404, 중복 409.

## 8. 과제 체크리스트
회원 관리 제외 4개 라우터, 각 라우터 최소 6개 API(Files 8 / Folders 7 / Shares 7 / Trash 6), 실제 DB 연동, 핵심 API 구현 및 테스트 코드, 지정 보고서 형식, 대면 회의 증빙 사진, 제출 시 `node_modules` 제외.
