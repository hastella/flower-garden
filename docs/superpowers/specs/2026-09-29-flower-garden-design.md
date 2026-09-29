# 꽃 정원 (Flower Garden) — 설계 문서

- 작성일: 2026-09-29
- 상태: 설계 확정, 구현 계획 작성 전

## 1. 개요

선물 받은 꽃을 사진으로 찍으면 배경을 지우고 꽃만 오려서, 앱 속 미니 정원에 꽂아 기록하는 모바일 웹 서비스.
꽃마다 누가, 언제, 어떤 마음으로 줬는지 남기고, 정원 링크를 공유해 꽃을 준 사람에게 보여줄 수 있다.

### 확정된 요구사항

| 항목 | 결정 |
|---|---|
| 플랫폼 | 모바일 웹 (PWA) |
| 사용자 | 다중 사용자, 카카오 로그인 |
| 정원 | 1인 여러 개 |
| 테마 | 잔디 정원 1종 + 사용자 배경 사진 업로드(커스텀). 테마 추가가 쉬운 구조 |
| 꽃 이미지 | 촬영 또는 갤러리 업로드 → 배경 제거(누끼) → 정원에 배치 |
| 배치 | 드래그로 자유 배치 |
| 꽃 정보 | 준 사람, 받은 날짜, 메모 (모두 선택 입력) |
| 공유 | 보기 전용 링크 (로그인 불필요) |

### MVP 범위 밖

추가 테마, 꽃 회전, 소셜 기능(구경/좋아요/방명록), 오프라인 지원, 정원 이미지 다운로드.

## 2. 기술 스택

- **프론트/서버**: Next.js (App Router, TypeScript), Vercel 배포
- **백엔드**: Supabase — Auth(카카오 OAuth), Postgres, Storage
- **배경 제거**: `@imgly/background-removal` (브라우저 내 실행, Web Worker)
- **테스트**: Vitest(단위), 로컬 Supabase(RLS), Playwright(E2E, 모바일 뷰포트)

선택 이유: 서버 비용 0원으로 시작 가능, 사진이 외부 AI 서비스로 나가지 않음, Supabase가 카카오 로그인을 기본 지원.
배경 제거는 단일 인터페이스 뒤에 두어 추후 서버 API(remove.bg, Replicate 등)로 교체 가능하게 한다.

## 3. 화면과 흐름

```
[랜딩] ──카카오 로그인──▶ [정원 목록]
                              │  + 새 정원 (이름, 테마 선택 / 배경 업로드)
                              ▼
                        [정원 화면]
             ┌────────────────┼────────────────┐
             ▼                ▼                ▼
      [꽃 추가 시트]     [꽃 상세 카드]     [공유 링크 복사]
                                               ▼
                                    [공유 정원 (보기 전용)]
```

| 경로 | 설명 |
|---|---|
| `/` | 서비스 소개 + "카카오로 시작하기". 로그인 상태면 `/gardens`로 이동 |
| `/gardens` | 내 정원 카드 목록(썸네일, 이름, 꽃 개수) + 새 정원 만들기. 썸네일은 별도 이미지를 만들지 않고 정원 캔버스를 읽기 전용·축소 렌더 |
| `/gardens/[id]` | 메인 정원 화면. 꽃 배치/드래그, 하단 ＋ 버튼, 상단 메뉴(공유 링크 복사, 이름 변경, 삭제) |
| `/g/[shareId]` | 공유 정원. 로그인 없이 열람, 꽃 탭 시 상세 카드, 편집 불가. 카톡 공유용 OG 이미지는 `next/og`로 테마 배경 + 정원 이름 + 꽃 개수를 렌더 (꽃 사진은 포함하지 않음) |

### 꽃 추가 시트

1. 촬영 또는 갤러리 선택 (`<input type="file" accept="image/*">`, 모바일에서 카메라/갤러리 선택 제공)
2. 배경 제거 중 로딩 표시 ("꽃을 다듬는 중… ✂️")
3. 결과 미리보기. 마음에 들지 않으면 "원본으로 꽂기" 선택 가능
4. 준 사람 / 받은 날짜 / 메모 입력 (선택)
5. "꽂기" → 정원 가운데 하단 영역에 배치 → 이후 드래그로 이동

### 꽃 상세 카드

원본 사진, 준 사람, 날짜, 메모 표시. 소유자는 정보 수정, 크기 조절 슬라이더, 삭제 가능.

## 4. 데이터 모델

```sql
profiles (
  id          uuid primary key references auth.users(id),
  nickname    text,
  avatar_url  text
)
-- 카카오 로그인 최초 시 트리거로 자동 생성

gardens (
  id               uuid primary key default gen_random_uuid(),
  owner_id         uuid not null references profiles(id) on delete cascade,
  name             text not null,
  theme            text not null check (theme in ('grass', 'custom')),
  background_path  text,          -- theme = 'custom'일 때 필수
  share_id         text not null unique,  -- 추측 불가능한 랜덤 문자열
  created_at       timestamptz not null default now()
)

flowers (
  id             uuid primary key default gen_random_uuid(),
  garden_id      uuid not null references gardens(id) on delete cascade,
  cutout_path    text not null,   -- 배경 제거 결과 (원본으로 꽂기 선택 시 원본과 동일 경로)
  original_path  text not null,
  x              real not null check (x between 0 and 1),
  y              real not null check (y between 0 and 1),
  scale          real not null default 1.0,
  giver          text,
  received_on    date,
  memo           text,
  created_at     timestamptz not null default now()
)
```

- 좌표 `x, y`는 정원 캔버스 대비 0~1 비율이며 **꽃 이미지 하단 중앙(줄기 끝)** 을 가리킨다.
- 테마는 DB가 아닌 코드(`themes.ts`)에 정의: `{ id, name, background }`. 새 테마 = 항목 1개 + 배경 이미지.

### Storage

- 비공개 버킷 `garden-images`
- 경로: `{user_id}/{garden_id}/{flower_id}-cutout.webp`, `{flower_id}-original.webp`, `background.webp`
- 업로드 전 클라이언트에서 긴 변 1280px로 리사이즈, WebP 변환 (누끼는 알파 채널 유지)
- 화면 표시는 서명 URL 사용

### 권한 (RLS)

- `gardens`, `flowers`: 소유자만 select/insert/update/delete
- Storage: 경로 첫 세그먼트가 본인 `user_id`인 객체만 접근
- 공유 페이지: 서버(Next.js 서버 컴포넌트)에서 service role로 `share_id`에 해당하는 정원 1개와 꽃 목록만 조회하고 서명 URL을 생성해 전달. 익명 사용자의 DB 직접 접근 경로는 열지 않는다.

### 삭제

정원/꽃 삭제 시 DB 행과 해당 Storage 파일을 함께 삭제한다.

## 5. 핵심 모듈

### 배경 제거 `lib/cutout/`

- 인터페이스: `removeBackground(file: Blob): Promise<Blob>`
- `@imgly/background-removal`을 Web Worker에서 실행 (UI 블로킹 방지)
- 결과의 투명 여백을 트림해 꽃 영역만 남김
- 모델은 최초 1회 다운로드 후 브라우저 캐시. 첫 사용 시 "처음 한 번만 준비가 필요해요" 안내

### 이미지 처리 `lib/image/`

- 리사이즈(긴 변 1280px), WebP 인코딩, 알파 트림 — 순수 함수로 분리해 단위 테스트

### 정원 캔버스 `components/garden/`

- 고정 비율(세로 3:4) 캔버스, 꽃은 절대 위치 배치
- Pointer Events 직접 구현 (라이브러리 없음)
  - 300ms 롱프레스 시 드래그 모드 진입 (탭/스크롤과 구분)
  - 드래그 중 살짝 확대 + 그림자
- 손을 떼면 낙관적 업데이트로 좌표 저장, 실패 시 원위치 복구
- 렌더 순서: `y` 오름차순 (아래쪽 꽃이 앞에 보임)
- 편집 모드 / 읽기 전용 모드를 prop으로 전환 (공유 페이지와 공용)

## 6. 에러 처리

| 상황 | 처리 |
|---|---|
| 배경 제거 실패 / 기기 미지원 | "원본으로 꽂기" 제안 |
| 이미지 아닌 파일, 20MB 초과 | 업로드 전 안내 |
| 업로드 실패 (네트워크) | 입력 내용 유지 + "다시 시도" |
| 위치 저장 실패 | 원위치 복구 + 토스트 |
| 없는 공유 링크 | "정원을 찾을 수 없어요" 페이지 |
| 로그인 만료 | 로그인 화면으로 이동, 로그인 후 원래 페이지 복귀 |

## 7. 테스트 전략

- **단위 (Vitest)**: 픽셀 ↔ 비율 좌표 변환, 렌더 순서 정렬, 리사이즈/트림 로직, 테마 설정
- **RLS (로컬 Supabase)**: 타인의 정원/꽃/Storage 객체 읽기·수정 불가 검증
- **E2E (Playwright, 모바일 뷰포트)**: 정원 생성 → 꽃 추가(배경 제거 모듈 목 처리) → 드래그 → 새로고침 후 위치 유지 → 로그아웃 상태로 공유 링크 열람 시 편집 불가. 카카오 로그인은 테스트 세션 주입으로 우회
