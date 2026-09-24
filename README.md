# ECLIPSE Guild Manager v17

기존 ECLIPSE Guild Manager v16을 기준으로 **Google Forms 길드원 DB 연동 + Discord 출석봇 + Supabase**를 연결한 버전입니다.

## 구조

```text
Google Forms → Google Sheets → Apps Script → ECLIPSE Next.js API → Supabase members
Discord → ECLIPSE 출석봇 → ECLIPSE Next.js API → Supabase attendance
                                             ↓
                                      ECLIPSE 웹사이트
```

## 1. Supabase

Supabase SQL Editor에서 `supabase/schema.sql` 전체를 한 번 실행하세요.

`members.name`은 Google Forms 동기화를 위해 UNIQUE 인덱스를 사용합니다. 같은 닉네임을 다시 제출하면 기존 길드원 정보가 업데이트됩니다.

## 2. Vercel 환경변수

기존 환경변수에 아래 2개를 추가하세요.

```env
SUPABASE_SERVICE_ROLE_KEY=Supabase의 service_role key
ECLIPSE_INTEGRATION_TOKEN=길고 랜덤한 비밀 문자열
```

기존 변수도 그대로 필요합니다.

```env
NEXT_PUBLIC_SUPABASE_URL=...
NEXT_PUBLIC_SUPABASE_ANON_KEY=...
NEXT_PUBLIC_ADMIN_PASSWORD=0910
```

**주의:** `SUPABASE_SERVICE_ROLE_KEY`에는 `NEXT_PUBLIC_`을 붙이면 안 됩니다. 브라우저에 노출되면 안 되는 서버 전용 키입니다.

`ECLIPSE_INTEGRATION_TOKEN`은 Google Apps Script와 Discord 봇에서 똑같이 사용합니다.

## 3. Google Forms 연동

`integrations/google-apps-script/Code.gs`를 Google Sheets의 Apps Script에 붙여 넣습니다.

Google Form 질문 제목은 정확히 다음을 권장합니다.

- 닉네임
- 직업
- 투력
- 메모

### 설정

1. Google Form → 응답 → Google Sheets 연결
2. 연결된 Sheet에서 `확장 프로그램 → Apps Script`
3. `Code.gs` 전체 붙여넣기
4. `API_URL`을 실제 Vercel 주소로 변경
5. `TOKEN`을 Vercel의 `ECLIPSE_INTEGRATION_TOKEN`과 동일하게 변경
6. `installTrigger()`를 한 번 실행하고 권한 승인
7. 기존 응답이 이미 있다면 `syncAllRows()`를 한 번 실행

이후 새 Google Form 응답이 들어올 때마다 `members` 테이블에 자동 반영됩니다.

같은 닉네임이 이미 있으면 UPDATE, 없으면 INSERT됩니다.

## 4. Discord 출석봇

`integrations/discord-bot` 폴더를 별도 Python 봇으로 실행합니다.

```bash
pip install -r requirements.txt
```

환경변수:

```env
DISCORD_BOT_TOKEN=디스코드 봇 토큰
ECLIPSE_API_URL=https://실제-Vercel-주소
ECLIPSE_INTEGRATION_TOKEN=Vercel과 동일한 토큰
DISCORD_PREFIX=ㅍ
```

선택적으로 Discord 표시 이름과 게임 닉네임이 다르면:

```env
MEMBER_ALIASES_JSON={"123456789012345678":"게임닉네임"}
```

처럼 Discord 사용자 ID를 게임 닉네임에 연결할 수 있습니다.

### 명령어

```text
ㅍ출석
ㅍ출석취소
ㅍ출석현황
ㅍ도움말
```

`ㅍ출석`은 명령을 입력한 Discord 사용자의 게임 닉네임을 찾아 오늘 날짜로 출석 처리합니다.

등록되지 않은 닉네임이면 출석이 저장되지 않고 오류가 표시됩니다.

`ㅍ출석취소`는 해당 사용자의 오늘 출석 상태를 `absent`로 바꿉니다.

## 5. 웹사이트

- 길드원 목록은 Supabase `members` 데이터를 표시합니다.
- Google Forms 제출 → DB 반영 → 웹사이트 자동 갱신
- 참여율 기록 화면에서 Discord 출석 횟수와 최근 출석을 확인할 수 있습니다.
- Supabase Realtime으로 `members`와 `attendance` 변경도 자동 반영됩니다.

## 보안

Google Apps Script와 Discord 봇은 anon key를 직접 사용하지 않고, Vercel 서버 API에 `ECLIPSE_INTEGRATION_TOKEN`으로 인증합니다.

Vercel API는 서버 전용 `SUPABASE_SERVICE_ROLE_KEY`로 Supabase에 기록합니다.

`SUPABASE_SERVICE_ROLE_KEY`와 `ECLIPSE_INTEGRATION_TOKEN`은 절대 GitHub에 올리지 마세요.

현재 기존 프로젝트의 관리자 기능은 기존 `0910` 클라이언트 비밀번호 방식을 유지합니다.
