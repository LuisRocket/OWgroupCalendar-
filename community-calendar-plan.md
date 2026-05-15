# 투자밤 커뮤니티 캘린더 구현 계획

## 1. 프로젝트 개요

투자밤 커뮤니티에서 회원가입 없이 누구나 모임을 추가하고 참가 표시·코멘트를 남길 수 있는 공개 캘린더를 구축한다. 노션 캘린더는 편집 시 노션 계정 가입이 필수라 익명 편집이 불가능하여, 정적 호스팅(GitHub Pages) + 익명 API(Google Apps Script) + DB(Google Sheets) 조합으로 대체한다.

## 2. 핵심 요구사항

| # | 요구사항 | 충족 방식 |
|---|---------|----------|
| 1 | 일정을 시각적으로 보여줄 것 | FullCalendar.js로 캘린더 UI 렌더 |
| 2 | 가입 없이 편집 가능 | Apps Script Web App을 익명 실행 허용으로 배포 |
| 3 | 참가자 누구나 모임 추가 | 캘린더 빈 셀 클릭 → 모달 → POST 요청 |
| 4 | 누구나 참가 표시·코멘트 작성 | 모임 상세 모달에서 작성자명+내용 입력 후 POST |

## 3. 대안 검토 결과 (의사결정 기록)

다른 옵션을 검토했으나 아래 이유로 탈락:

- **노션 공유 페이지**: 웹 게시는 읽기 전용. 편집은 워크스페이스 게스트 초대 필요 → 가입 강제.
- **Google Calendar 공유**: 보기 공개는 되지만 편집은 구글 로그인 필수.
- **GitHub Issues를 DB로 사용 (utterances 방식)**: 작성 시 GitHub 로그인 필요.
- **Firebase/Supabase + 익명 인증**: 가능하지만 API 키 노출 위험과 스팸 대응 부담이 큼. 무료 티어 한도 관리 필요.
- **Google Drive API 클라이언트 직접 호출**: OAuth 로그인이 필수라 익명 편집 불가. API 키를 코드에 박으면 노출됨.

**최종 선택: GitHub Pages + Google Apps Script Web App + Google Sheets**
- 익명 사용자가 Apps Script 엔드포인트를 호출하면, Apps Script가 관리자(Luis) 권한으로 Sheets에 쓴다.
- 방문자는 어떠한 계정 로그인도 하지 않는다.
- 모든 구성요소가 무료. 일일 호출 한도 약 20,000회로 소규모 커뮤니티에 충분.

## 4. 시스템 아키텍처

```
┌─────────────────────┐         ┌──────────────────────────┐         ┌─────────────────┐
│   방문자 브라우저    │         │  Apps Script Web App      │         │  Google Sheets  │
│                     │         │  (Luis 계정 권한 실행)      │         │                 │
│  GitHub Pages       │ ──HTTP─▶│  doGet / doPost           │ ──API──▶│  events 시트     │
│  (FullCalendar UI)  │         │                           │         │  attendees 시트  │
│                     │ ◀─JSON──│  CORS 응답                 │ ◀───────│  comments 시트   │
└─────────────────────┘         └──────────────────────────┘         └─────────────────┘
```

데이터 흐름:
1. 페이지 로드 시 `GET /exec?action=list` → 전체 이벤트 조회 → FullCalendar에 표시
2. 모임 추가: `POST /exec` with `action=create_event` → 새 행 삽입
3. 참가 표시: `POST /exec` with `action=join` → attendees 시트에 행 추가
4. 코멘트: `POST /exec` with `action=comment` → comments 시트에 행 추가

## 5. 기술 스택

- **프론트엔드**: 순수 HTML/CSS/JS (빌드 도구 없음, 단순함 우선)
- **캘린더 라이브러리**: [FullCalendar v6](https://fullcalendar.io/) (MIT 라이선스, CDN 사용)
- **호스팅**: GitHub Pages (정적)
- **백엔드**: Google Apps Script Web App
- **데이터베이스**: Google Sheets (3개 시트)
- **인증**: 없음 (익명)

## 6. Google Sheets 데이터 스키마

스프레드시트 하나에 3개 탭을 만든다.

### 6.1 `events` 시트

| 컬럼 | 타입 | 설명 |
|------|------|------|
| id | string | UUID 또는 timestamp 기반 ID |
| title | string | 모임 제목 |
| start | ISO 8601 | 시작 일시 |
| end | ISO 8601 | 종료 일시 |
| location | string | 장소 (선택) |
| description | string | 설명 |
| organizer_name | string | 모임 만든 사람 닉네임 |
| created_at | ISO 8601 | 생성 시각 |

### 6.2 `attendees` 시트

| 컬럼 | 타입 | 설명 |
|------|------|------|
| id | string | UUID |
| event_id | string | events.id 참조 |
| name | string | 참가자 닉네임 |
| status | string | "going" / "maybe" / "not_going" |
| created_at | ISO 8601 | 작성 시각 |

### 6.3 `comments` 시트

| 컬럼 | 타입 | 설명 |
|------|------|------|
| id | string | UUID |
| event_id | string | events.id 참조 |
| author_name | string | 작성자 닉네임 |
| content | string | 코멘트 본문 |
| created_at | ISO 8601 | 작성 시각 |

## 7. Apps Script API 명세

엔드포인트 하나(`https://script.google.com/macros/s/{ID}/exec`)에 action 파라미터로 분기.

### 7.1 GET 요청

- `?action=list` → 전체 이벤트 + 각 이벤트의 참가자수/코멘트수 요약
- `?action=detail&event_id={id}` → 이벤트 상세 + 참가자 목록 + 코멘트 목록

### 7.2 POST 요청 (Content-Type: application/json)

```json
// 이벤트 생성
{
  "action": "create_event",
  "title": "...",
  "start": "2026-06-01T19:00:00+09:00",
  "end": "2026-06-01T21:00:00+09:00",
  "location": "...",
  "description": "...",
  "organizer_name": "..."
}

// 참가 표시
{
  "action": "join",
  "event_id": "...",
  "name": "...",
  "status": "going"
}

// 코멘트
{
  "action": "comment",
  "event_id": "...",
  "author_name": "...",
  "content": "..."
}
```

### 7.3 응답 형식

```json
{ "ok": true, "data": { ... } }
{ "ok": false, "error": "..." }
```

## 8. 구현 단계

### Phase 1: Apps Script 백엔드 구축

1. Google Sheets 새 스프레드시트 생성, 위 스키마대로 3개 탭 + 헤더 행 작성
2. `확장 프로그램 > Apps Script` 진입, 아래 코드 골격 작성
3. `배포 > 새 배포 > 웹 앱` 선택
   - 실행 권한: "나"
   - 액세스: "모든 사용자" (익명 허용)
4. 발급된 `/exec` URL 기록
5. `curl`로 GET/POST 동작 확인

### Phase 2: 프론트엔드 구축

1. GitHub 저장소 생성 (예: `tubam-calendar`), Pages 활성화
2. `index.html`, `style.css`, `app.js` 작성
3. FullCalendar CDN 임포트, `eventSources`에 Apps Script GET URL 연결
4. 이벤트 클릭 시 모달 → 상세 조회 + 참가/코멘트 폼
5. 빈 셀 클릭 → 모임 추가 모달 → POST

### Phase 3: 운영 안정화

1. 캐시 전략: Apps Script `CacheService`로 list 응답 60초 캐싱 (할당량 절감)
2. 입력 검증: 제목/이름 길이 제한, XSS 대비 escape, 욕설 필터(선택)
3. 간단한 봇 방지: 허니팟 필드 + 작성 간격 제한 (Sheets에 IP 해시 기록)
4. 백업: Apps Script 시간 기반 트리거로 매일 시트 사본을 Drive에 저장
5. 관리자 도구: 비공개 URL의 관리 페이지에서 행 삭제 가능하도록 (간단한 토큰 인증)

## 9. Apps Script 코드 골격

```javascript
// ===== code.gs =====
const SHEET_ID = 'YOUR_SHEET_ID_HERE';
const EVENTS_SHEET = 'events';
const ATTENDEES_SHEET = 'attendees';
const COMMENTS_SHEET = 'comments';

function doGet(e) {
  try {
    const action = e.parameter.action;
    let result;
    if (action === 'list') result = listEvents();
    else if (action === 'detail') result = getEventDetail(e.parameter.event_id);
    else throw new Error('Unknown action: ' + action);
    return jsonResponse({ ok: true, data: result });
  } catch (err) {
    return jsonResponse({ ok: false, error: err.message });
  }
}

function doPost(e) {
  try {
    const body = JSON.parse(e.postData.contents);
    let result;
    if (body.action === 'create_event') result = createEvent(body);
    else if (body.action === 'join') result = joinEvent(body);
    else if (body.action === 'comment') result = addComment(body);
    else throw new Error('Unknown action');
    return jsonResponse({ ok: true, data: result });
  } catch (err) {
    return jsonResponse({ ok: false, error: err.message });
  }
}

function jsonResponse(obj) {
  return ContentService
    .createTextOutput(JSON.stringify(obj))
    .setMimeType(ContentService.MimeType.JSON);
}

function getSheet(name) {
  return SpreadsheetApp.openById(SHEET_ID).getSheetByName(name);
}

function uuid() {
  return Utilities.getUuid();
}

function listEvents() {
  const sheet = getSheet(EVENTS_SHEET);
  const rows = sheet.getDataRange().getValues();
  const headers = rows.shift();
  return rows.map(row => Object.fromEntries(headers.map((h, i) => [h, row[i]])));
}

function createEvent(body) {
  // 검증
  if (!body.title || !body.start) throw new Error('title, start required');
  if (body.title.length > 100) throw new Error('title too long');

  const sheet = getSheet(EVENTS_SHEET);
  const id = uuid();
  sheet.appendRow([
    id,
    body.title,
    body.start,
    body.end || '',
    body.location || '',
    body.description || '',
    body.organizer_name || 'anonymous',
    new Date().toISOString()
  ]);
  return { id };
}

function joinEvent(body) {
  if (!body.event_id || !body.name) throw new Error('event_id, name required');
  const sheet = getSheet(ATTENDEES_SHEET);
  const id = uuid();
  sheet.appendRow([
    id, body.event_id, body.name, body.status || 'going', new Date().toISOString()
  ]);
  return { id };
}

function addComment(body) {
  if (!body.event_id || !body.content) throw new Error('event_id, content required');
  const sheet = getSheet(COMMENTS_SHEET);
  const id = uuid();
  sheet.appendRow([
    id, body.event_id, body.author_name || 'anonymous', body.content, new Date().toISOString()
  ]);
  return { id };
}

function getEventDetail(eventId) {
  const events = listEvents();
  const event = events.find(e => e.id === eventId);
  if (!event) throw new Error('event not found');

  const attendees = getSheet(ATTENDEES_SHEET).getDataRange().getValues();
  const aHeaders = attendees.shift();
  const eventAttendees = attendees
    .map(r => Object.fromEntries(aHeaders.map((h, i) => [h, r[i]])))
    .filter(a => a.event_id === eventId);

  const comments = getSheet(COMMENTS_SHEET).getDataRange().getValues();
  const cHeaders = comments.shift();
  const eventComments = comments
    .map(r => Object.fromEntries(cHeaders.map((h, i) => [h, r[i]])))
    .filter(c => c.event_id === eventId);

  return { event, attendees: eventAttendees, comments: eventComments };
}
```

## 10. 프론트엔드 코드 골격

```html
<!-- index.html -->
<!DOCTYPE html>
<html lang="ko">
<head>
  <meta charset="UTF-8">
  <title>투자밤 캘린더</title>
  <link href="https://cdn.jsdelivr.net/npm/fullcalendar@6.1.11/index.global.min.css" rel="stylesheet">
  <link rel="stylesheet" href="style.css">
</head>
<body>
  <header><h1>투자밤 모임 캘린더</h1></header>
  <div id="calendar"></div>
  <div id="modal" class="modal hidden"><!-- 동적으로 채움 --></div>
  <script src="https://cdn.jsdelivr.net/npm/fullcalendar@6.1.11/index.global.min.js"></script>
  <script src="app.js"></script>
</body>
</html>
```

```javascript
// app.js
const API = 'https://script.google.com/macros/s/YOUR_DEPLOY_ID/exec';

document.addEventListener('DOMContentLoaded', () => {
  const calendarEl = document.getElementById('calendar');
  const calendar = new FullCalendar.Calendar(calendarEl, {
    initialView: 'dayGridMonth',
    locale: 'ko',
    selectable: true,
    events: async (info, success, failure) => {
      try {
        const res = await fetch(`${API}?action=list`);
        const json = await res.json();
        if (!json.ok) throw new Error(json.error);
        success(json.data.map(e => ({
          id: e.id, title: e.title, start: e.start, end: e.end
        })));
      } catch (err) { failure(err); }
    },
    select: (info) => openCreateModal(info.startStr, info.endStr),
    eventClick: (info) => openDetailModal(info.event.id),
  });
  calendar.render();
});

async function openDetailModal(eventId) {
  const res = await fetch(`${API}?action=detail&event_id=${eventId}`);
  const json = await res.json();
  // 모달 렌더링 + 참가/코멘트 폼 핸들러 부착
}

async function openCreateModal(start, end) {
  // 폼 모달 표시, 제출 시 POST
}

async function postAction(payload) {
  const res = await fetch(API, {
    method: 'POST',
    body: JSON.stringify(payload),
    // CORS 회피를 위해 Content-Type 헤더 생략 (text/plain로 처리됨)
  });
  return res.json();
}
```

**중요: CORS 처리**
Apps Script 웹 앱은 기본적으로 CORS 헤더를 모두 보내지 않는다. 그래서 `fetch`의 `Content-Type: application/json` 헤더를 명시하면 preflight 요청이 발생하고 실패한다. 헤더를 생략해 단순 요청(simple request)으로 보내면 정상 동작한다. Apps Script 쪽에서는 `e.postData.contents`를 그대로 `JSON.parse`하면 된다.

## 11. 보안·운영 고려사항

- **공개 익명 편집의 본질적 위험**: 누구나 모임을 추가/삭제할 수 있으므로 스팸·장난·악의적 입력 대응 필요.
- **완화책**:
  - 입력 길이/형식 검증을 Apps Script 단에서 수행
  - 시트 변경 이력은 Sheets 자체 버전 관리로 추적 가능
  - 백업 트리거로 매일 사본 저장 (복구용)
  - URL을 공개하지 않고 투자밤 내부 공지로만 공유 (보안이 아닌 노출 제한)
  - 필요 시 간단한 비밀번호를 클라이언트에서 입력받아 Apps Script 검증 (강력하진 않지만 진입장벽)
- **할당량**:
  - Apps Script 무료 일일 한도: URL Fetch 20,000회, 트리거 실행 시간 90분 등
  - 투자밤 규모(수십~수백명)에서는 여유 있음
  - `CacheService`로 list 요청을 캐싱하면 더 안전

## 12. 다른 세션에서 시작할 때 체크리스트

- [ ] Google 계정으로 새 스프레드시트 생성, 3개 탭 + 헤더 행 작성
- [ ] `확장 프로그램 > Apps Script` 진입, `SHEET_ID` 채우고 위 코드 붙여넣기
- [ ] 웹 앱 배포, 익명 액세스 허용, URL 기록
- [ ] `curl "URL?action=list"` 로 빈 배열 응답 확인
- [ ] `curl -X POST URL -d '{"action":"create_event","title":"테스트","start":"2026-06-01T19:00:00+09:00","organizer_name":"luis"}'` 로 생성 확인
- [ ] GitHub 저장소 생성, Pages 활성화
- [ ] `index.html`, `app.js`, `style.css` 작성, `API` 상수에 URL 채우기
- [ ] 로컬에서 `python -m http.server`로 동작 확인
- [ ] push 후 GitHub Pages URL에서 동작 확인
- [ ] 투자밤 공지에 URL 공유

## 13. 향후 확장 아이디어

- 카테고리/태그(스터디, 발표, 친목 등) 색상 구분
- iCal 피드 노출(`?action=ical`)해서 개인 캘린더 앱에서 구독
- 텔레그램/디스코드 봇으로 신규 모임 알림
- 관리자용 비공개 페이지에서 삭제·정리 작업
- 모임 후기 기능(코멘트와 별도의 후기 시트)
