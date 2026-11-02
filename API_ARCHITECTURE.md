# JqueryYa — API Architecture v1.2

> **프로젝트:** JqueryYa (Web File Manager, WFM)  
> **상태:** 구현 전 설계 명세  
> **기술 스택:** Node.js · Express · MySQL · Multer · Sharp · FFmpeg/FFprobe · Jest/Supertest

## 1. 개요 및 변경 사항

JqueryYa는 사용자별 파일 저장·관리, 공유, 휴지통, 파일 형식 변환, 동영상 구간 편집을 제공하는 HTTP 기반 웹 파일 관리자이다. FTP 서버를 구현하는 프로젝트는 아니다.

**v1.2 변경 사항**

- 기존 `Files`와 `Folders`를 **`Storage` 라우터 하나로 통합**한다.
- 파일과 폴더의 공통 동작(조회·이름 변경·이동·복사·삭제)을 동일한 API 체계로 제공한다.
- **`Conversions`**: PNG/JPG/WEBP 이미지 변환 및 MP4 → MP3 오디오 추출을 추가한다.
- **`Media Editor`**: 단일 시간 구간을 추출해 별도 MP4 파일을 생성하는 기능을 추가한다.
- 여러 구간 이어 붙이기, 자막·텍스트 삽입, 화면 전환 효과, 전문 타임라인 편집기는 범위에서 제외한다.

### 평가 대상 라우터

| 라우터 | 담당 영역 | API 수 |
|---|---|---:|
| Storage | 파일·폴더 통합 관리 | 10 |
| Shares | 공유 링크 생성·조회·관리 | 7 |
| Trash | 삭제 항목 조회·복구·영구 삭제 | 6 |
| Conversions | 이미지 변환·영상 오디오 추출 | 7 |
| Media Editor | 동영상 단일 구간 추출 | 7 |
| **합계** | **5개 라우터** | **37** |

별도 인증 라우터(Auth) 5개 및 공개 공유 API 2개는 위 집계에서 제외한다. **위 숫자는 설계된 API 수이며, 구현 또는 테스트 완료를 의미하지 않는다.**

## 2. 공통 규약

- API prefix: `/api`.
- 인증 방식: `express-session`과 MySQL 세션 저장소 기반 로그인 세션.
- 사용자별 파일·폴더·공유·작업 이력 소유권을 모든 요청에서 검사한다.
- 성공 응답: `{ "success": true, "data": { ... } }`.
- 실패 응답: `{ "success": false, "error": { "code": "...", "message": "..." } }`.
- 대표 HTTP 상태 코드: `200`, `201`, `204`, `400`, `401`, `403`, `404`, `409`, `500`.
- `204 No Content` 응답에는 JSON 본문을 넣지 않는다. 다운로드는 바이너리 스트림을 반환한다.
- 파일 내용은 로컬 파일 시스템에 저장하고 메타데이터는 MySQL에 저장한다.
- 이름 충돌, 삭제된 상위 폴더, 파일 시스템 오류 발생 시 정합성 정책을 일관되게 적용한다.

## 3. Storage Router — 10 APIs

파일과 폴더의 공통 API를 하나의 라우터에서 처리한다. **파일 ID와 폴더 ID는 서로 중복될 수 있으므로** 상세·변경·삭제 경로는 `:type`(`files` 또는 `folders`)을 사용한다.

| Method | Endpoint | 기능 | 비고 |
|---|---|---|---|
| POST | `/api/storage/files` | 파일 업로드 | `multipart/form-data` |
| POST | `/api/storage/folders` | 폴더 생성 | 이름·상위 폴더 지정 |
| GET | `/api/storage` | 파일·폴더 통합 목록 | `parentId`, 페이지네이션 |
| GET | `/api/storage/search` | 파일·폴더 검색 | 이름 등 조건 검색 |
| GET | `/api/storage/:type/:id` | 파일·폴더 상세 조회 | 소유권 검사 |
| GET | `/api/storage/:type/:id/download` | 파일 다운로드 | `type=files`만 허용 |
| PATCH | `/api/storage/:type/:id` | 이름 변경 | 공통 처리 |
| PATCH | `/api/storage/:type/:id/move` | 이동 | 파일/폴더 각각 검증 |
| POST | `/api/storage/:type/:id/copy` | 복사 | 폴더는 하위 항목 포함 |
| DELETE | `/api/storage/:type/:id` | 휴지통 이동 | 소프트 삭제 |

**구현 정책**

- `GET /api/storage`는 특정 폴더의 **직접 하위 파일과 폴더**를 한 목록으로 반환한다. `parentId=null`은 루트이다.
- 폴더 이동 시 자기 자신 또는 자신의 하위 폴더로 이동하지 못하게 한다.
- 폴더 복사는 하위 파일·폴더를 재귀적으로 복제한다. 중간 실패 시 생성된 파일과 DB 기록을 정리한다.
- 폴더 다운로드는 이번 범위에서 제외한다. `type=folders` 다운로드 요청은 `400` 또는 `404` 중 한 정책으로 통일한다(구현 시 확정).
- 정적 경로 `/search`는 동적 경로보다 먼저 등록한다.
- API 통합은 **라우터 수준**에서만 수행한다. DB의 `files`, `folders` 테이블과 내부 파일·폴더 서비스는 분리 유지한다.

## 4. Shares Router — 7 APIs

| Method | Endpoint | 기능 |
|---|---|---|
| POST | `/api/shares` | 공유 링크 생성 |
| GET | `/api/shares` | 내 공유 목록 |
| GET | `/api/shares/:id` | 공유 상세 조회 |
| PATCH | `/api/shares/:id` | 만료일·활성 상태 변경 |
| DELETE | `/api/shares/:id` | 공유 해제 |
| GET | `/api/shares/:id/access` | 공유 접근 상태 확인 |
| GET | `/api/shares/:id/download` | 소유자용 공유 파일 다운로드 |

**별도 공개 API(평가 집계 제외)**

- `GET /api/public/shares/:token`: 공유 토큰 정보 확인.
- `GET /api/public/shares/:token/download`: 토큰 기반 공개 다운로드.

공유 토큰은 안전한 난수로 생성하고 만료·해제·원본 삭제 여부를 접근 시 검증한다. 공개 다운로드에는 소유자 로그인 대신 유효한 토큰이 필요하다.

## 5. Trash Router — 6 APIs

| Method | Endpoint | 기능 |
|---|---|---|
| GET | `/api/trash` | 휴지통 목록 |
| GET | `/api/trash/stats` | 휴지통 통계 |
| GET | `/api/trash/:id` | 삭제 기록 상세 |
| POST | `/api/trash/:id/restore` | 삭제 항목 복구 |
| DELETE | `/api/trash/:id` | 항목 영구 삭제 |
| DELETE | `/api/trash` | 휴지통 비우기 |

`/api/trash/:id`의 ID는 `trash_records.id`이다. `/stats`를 `/:id`보다 먼저 등록한다. 폴더의 하위 항목 삭제·복구 및 원래 상위 폴더가 사라진 경우의 복원 정책은 구현 전에 확정한다.

## 6. Conversions Router — 7 APIs

| Method | Endpoint | 기능 |
|---|---|---|
| POST | `/api/conversions` | 파일 변환 작업 생성 |
| GET | `/api/conversions` | 내 변환 이력 조회 |
| GET | `/api/conversions/formats` | 지원 변환 형식 조회 |
| GET | `/api/conversions/:id` | 작업 상태·결과 조회 |
| GET | `/api/conversions/:id/download` | 변환 결과 다운로드 |
| POST | `/api/conversions/:id/cancel` | 작업 취소 |
| DELETE | `/api/conversions/:id` | 변환 작업 기록 삭제 |

### 지원 형식

- 이미지: `PNG → JPG/WEBP`, `JPG → PNG/WEBP`, `WEBP → PNG/JPG`.
- 영상에서 오디오 추출: `MP4 → MP3`.
- 동영상 간 포맷 변환(`MP4 ↔ WEBM` 등)은 MVP에서 제외한다.

### 요청 예시

```http
POST /api/conversions
Content-Type: application/json
```

```json
{
  "fileId": 15,
  "targetFormat": "jpg",
  "options": {
    "quality": 85,
    "background": "#FFFFFF"
  }
}
```

이미지 변환에는 **Sharp**, MP4 → MP3에는 **FFmpeg**를 사용한다. 입력 파일의 실제 형식, 소유권, 허용 변환 조합을 확인한다. PNG → JPG 등 투명도 손실 가능성이 있는 변환은 배경색 정책을 적용한다. 성공한 결과는 새 `files` 레코드로 등록하며 원본은 보존한다.

## 7. Media Editor Router — 7 APIs

| Method | Endpoint | 기능 |
|---|---|---|
| POST | `/api/media/trim` | 동영상 구간 추출 작업 생성 |
| GET | `/api/media/jobs` | 내 편집 작업 목록 |
| GET | `/api/media/jobs/:id` | 작업 상태·결과 조회 |
| GET | `/api/media/jobs/:id/download` | 편집 결과 다운로드 |
| POST | `/api/media/jobs/:id/cancel` | 작업 취소 |
| DELETE | `/api/media/jobs/:id` | 편집 기록 삭제 |
| GET | `/api/media/files/:id/info` | 동영상 길이·해상도·코덱 조회 |

### 요청 예시

4분짜리 `video.mp4`의 **00:13~00:43** 구간을 추출하여 약 30초 길이의 별도 MP4 파일을 생성한다.

```http
POST /api/media/trim
Content-Type: application/json
```

```json
{
  "fileId": 21,
  "startTime": 13,
  "endTime": 43,
  "outputName": "video_trimmed.mp4"
}
```

- 시간 단위는 초이며 `0 <= startTime < endTime <= 실제 영상 길이`를 만족해야 한다.
- **FFprobe**로 재생 시간·스트림 정보를 확인한다.
- **FFmpeg 재인코딩**을 기본으로 사용하여 요청 구간을 가능한 한 정확하게 추출한다. 프레임·타임스탬프에 따라 출력 경계에는 미세한 차이가 있을 수 있다.
- 원본을 보존하고 결과물을 새 `files` 레코드로 등록한다.
- 여러 구간 이어 붙이기, 자막·텍스트 삽입, 화면 전환 효과, 전문 타임라인 편집기는 구현하지 않는다.

## 8. 공통 미디어 작업 처리

변환 및 구간 편집은 처리 시간이 길 수 있으므로 HTTP 요청에서 직접 완료를 기다리지 않고 **작업을 등록한 후 상태를 조회**하는 구조로 구현한다.

```text
클라이언트 요청
  → 로그인·파일 소유권 확인
  → 입력·형식·용량·시간 검증
  → MySQL 작업 레코드 생성 (pending)
  → 제한된 작업 큐에 등록
  → Sharp / FFmpeg 실행 (processing)
  → 결과 파일 저장 및 files 레코드 생성
  → 작업 완료 (completed) / 실패 (failed) / 취소 (cancelled)
```

- 작업 상태: `pending`, `processing`, `completed`, `failed`, `cancelled`.
- 대기 중 취소는 큐에서 제거한다. 실행 중 취소는 자식 프로세스 종료 및 임시 파일 정리를 시도한다.
- 작업 기록 삭제는 이미 생성된 결과 파일을 삭제하지 않는다.
- 사용자 입력을 셸 문자열에 직접 결합하지 않고 허용된 인수로만 실행한다.
- 파일 크기·영상 길이·동시 실행 수·작업 제한 시간을 둔다.
- 서버 재시작 시 중단 작업을 식별하고 실패 처리 또는 재시도하도록 설계한다.
- DB와 파일 시스템은 단일 트랜잭션이 아니므로 오류 발생 시 보상 처리한다.

## 9. Auth Router — 평가 집계 제외

| Method | Endpoint | 기능 |
|---|---|---|
| POST | `/api/auth/register` | 회원가입 |
| POST | `/api/auth/login` | 로그인 |
| POST | `/api/auth/logout` | 로그아웃 |
| GET | `/api/auth/me` | 로그인 사용자 정보 |
| PATCH | `/api/auth/password` | 비밀번호 변경 |

비밀번호 해시, 세션 고정 공격 방지, 세션 쿠키 보안 설정 및 상태 변경 요청의 CSRF 방어를 적용한다.

## 10. 데이터베이스 설계(개념)

| 테이블 | 주요 필드 | 역할 |
|---|---|---|
| `users` | `id`, `username`, `password_hash` | 사용자 |
| `folders` | `id`, `user_id`, `parent_id`, `name`, `deleted_at` | 폴더 트리 |
| `files` | `id`, `user_id`, `folder_id`, `original_name`, `storage_key`, `mime_type`, `size_bytes`, `deleted_at` | 파일 메타데이터 |
| `shares` | `id`, `file_id`, `token_hash`, `expires_at`, `is_active` | 공유 링크 |
| `trash_records` | `id`, `file_id`, `folder_id`, `original_parent_id`, `deleted_at` | 삭제 이력 |
| `conversion_jobs` | `id`, `user_id`, `source_file_id`, `target_format`, `options_json`, `status`, `output_file_id`, `error_message`, `created_at`, `finished_at` | 변환 작업 |
| `media_edit_jobs` | `id`, `user_id`, `source_file_id`, `start_time_ms`, `end_time_ms`, `status`, `output_file_id`, `error_message`, `created_at`, `finished_at` | 편집 작업 |
| `sessions` | 세션 저장소 규격에 따름 | 로그인 세션 |

**참고:** 위 표는 개념 설계이며 실제 컬럼 타입, 외래 키, 인덱스, 체크 제약 및 삭제·복구 정책은 `database/schema.sql` 작성 시 확정한다.

## 11. 권장 프로젝트 구조

```text
src/
  app.js
  server.js
  config/
    database.js
  middleware/
    auth.js
    error-handler.js
  routes/
    storage.routes.js
    shares.routes.js
    trash.routes.js
    conversions.routes.js
    media.routes.js
    auth.routes.js
    public.routes.js
  services/
    storage/
      file.service.js
      folder.service.js
      storage.service.js
    share.service.js
    trash.service.js
    conversion.service.js
    media.service.js
    media-job.service.js
  workers/
    media-worker.js
  repositories/
    ...
database/
  schema.sql
  seed.sql
tests/
storage/
.env.example
.gitignore
README.md
```

`storage/`, `.env`, `node_modules/`, 변환 임시 파일 등은 Git에서 제외한다. 제출 시에는 소스·설정 예시·테스트 코드·필요한 SQL 스크립트를 포함한다.

## 12. 테스트 및 구현 순서

### 필수 테스트

1. **Storage:** 파일 업로드·다운로드, 폴더 계층, 이동 시 순환 방지, 재귀 복사, 타인 데이터 접근 차단.
2. **Shares:** 공유 토큰 유효성, 만료·해제·원본 삭제 상태 확인, 공개 다운로드.
3. **Trash:** 삭제·복구·영구 삭제·비우기, 폴더 하위 항목 처리.
4. **Conversions:** 지원 형식 검증, 실제 이미지 인코딩 결과, MP4 오디오 추출, 작업 실패·취소.
5. **Media Editor:** 영상 길이 검증, 13~43초 구간 추출, 결과 파일 등록, 잘못된 시간 범위 차단.
6. **공통:** 인증, 권한, DB·파일 시스템 정합성, 동시 작업 제한.

### 구현 순서

1. Express·MySQL 연결 및 DB 스키마 확정.
2. Auth 및 공통 미들웨어 구현.
3. Storage 구현 후 기본 파일 관리 검증.
4. Shares·Trash 구현.
5. 공통 작업 큐·미디어 처리 서비스 구축.
6. Conversions 구현(Sharp → FFmpeg).
7. Media Editor 구현(FFprobe → FFmpeg 구간 추출).
8. Jest/Supertest 통합 테스트 및 보고서 작성.

---

## 13. 최종 범위 요약

- **평가 대상 라우터:** 5개.
- **평가 대상 설계 API:** 37개.
- **별도 인증 API:** 5개.
- **별도 공개 공유 API:** 2개.
- **전체 설계 API:** 44개.
- **핵심 신규 기능:** 이미지·오디오 변환, 동영상 단일 구간 추출.
- **제외 기능:** 여러 구간 연결, 자막·텍스트 삽입, 화면 전환 효과, 전문 타임라인 편집기, 동영상 간 포맷 변환.

> **구현 주의:** 교수님의 평가 조건인 '비회원관리 라우터 4개 이상, 각 라우터 API 6개 초과'를 설계상 충족한다. 실제 평가는 구현 코드 및 테스트 결과를 기준으로 확인해야 한다.
