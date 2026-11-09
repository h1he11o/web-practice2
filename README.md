# 9주차 실습 · 검색·정렬·페이지네이션 게시판 개선

첨부 예제를 실행해 요구사항별로 검증하고, **요구사항을 충족하지 못한 부분만** 최소한으로 수정했습니다.
항목별 테스트 입력과 기대·실제 결과, 테스트 캡처는 LMS에 제출한 실습 보고서에 정리했습니다.

## 실행 방법

```bash
npm install express
cd 실습코드
node server.js
# 브라우저: http://localhost:3000
```

## 폴더 구성

| 경로 | 설명 |
|---|---|
| `실습코드/server.js`, `실습코드/index.html` | 수정한 코드 (수정 위치에 `[수정]` 주석, 자세한 내용은 아래 "원본 대비 수정한 부분") |
| `원본코드/` | 첨부된 원본 (비교용) |

## 원본 대비 수정한 부분

원본을 실행해 검증한 결과, 아래 14가지를 수정했습니다. 그 밖의 요구사항(검색, 조회순 동점 id DESC, 서버의 page·limit 보정, 1페이지 초기화, limit+1, 디바운스, AbortController, AbortError 처리)은 **원본이 이미 충족해서 수정하지 않았습니다.**

### ① 정렬 키 화이트리스트: 상속 속성 차단 (`server.js`)
```diff
- const sortComparator = SORT_WHITELIST[sortKey] || SORT_WHITELIST.latest;
+ const sortComparator = Object.hasOwn(SORT_WHITELIST, sortKey) ? SORT_WHITELIST[sortKey] : SORT_WHITELIST.latest;
```
- 원본에서 `sort=__proto__`를 보내면 `Object.prototype`이 반환되어 정렬 중 오류가 났습니다(**HTTP 500**).
- `sort=constructor`, `toString`은 상속받은 함수라서 화이트리스트를 통과해 버렸습니다.
- `Object.hasOwn`은 객체에 직접 정의한 키(latest, oldest, views)만 허용합니다. 나머지는 모두 latest로 처리합니다.

### ② 커서 API에 제목·본문 검색 추가 (`server.js`)
```diff
- let candidate = POSTS;
- if (!isNaN(rawCursor) && rawCursor > 0) {
-   candidate = POSTS.filter(p => p.id < rawCursor);
- }
+ const search = (req.query.search || '').trim().toLowerCase();
+ let candidate = POSTS.filter(p => {
+   if (!isNaN(rawCursor) && rawCursor > 0 && p.id >= rawCursor) return false;
+   if (!search) return true;
+   return p.title.toLowerCase().includes(search) || p.content.toLowerCase().includes(search);
+ });
```
- 원본은 `search`를 보내도 무시하고 전체 글을 반환했습니다.
- 검색 조건과 커서 조건을 모두 적용한 뒤, 기존 코드 그대로 `limit + 1`개를 잘라 `hasNext`를 판단합니다.
- `POSTS`가 id 내림차순으로 들어 있으므로 결과도 id 내림차순입니다.

### ③ 커서 모드에서 검색어 전달 + 정렬 선택 비활성화 (`index.html`)
```diff
  if (state.cursor) q.set('cursor', state.cursor);
+ if (state.search) q.set('search', state.search);
```
```diff
+ document.getElementById('sortSelect').disabled = (mode === 'cursor');
```
- 원본 화면은 커서 요청에 검색어를 붙이지 않았습니다.
- 커서 모드는 id 내림차순으로 고정인데도 정렬 선택이 활성화돼 있었습니다. 바꿔도 결과가 그대로였습니다.

### ④ URL의 정렬 값 검증 (`index.html`)
```diff
- state.sort = params.get('sort') || 'latest';
+ state.sort = validSort(params.get('sort'));   // ['latest','views','oldest'] 외에는 latest
+ // 새로고침 시 잘못된 값이면 주소창도 sort=latest 로 교체 (replaceState)
```
- 원본에서 `?sort=__proto__`로 새로고침하면 정렬 선택창이 빈칸이 됐습니다.
- 결과는 latest인데 URL은 `__proto__`로 남아 URL·정렬·결과가 일치하지 않았습니다.

### ⑤ 검색어 파라미터가 중복될 때 서버 오류 방지 (`server.js`, 오프셋·커서 API)
```diff
- const search = (req.query.search || '').trim().toLowerCase();
+ const search = String(req.query.search || '').trim().toLowerCase();
```
- `?search=a&search=b`처럼 같은 이름이 두 번 오면 Express가 배열로 넘깁니다. 배열에는 `.trim()`이 없어 **HTTP 500**이 났습니다.
- `String()`으로 감싸 문자열로 바꾼 뒤 처리하므로 이제 200으로 응답합니다.

### ⑥ 커서 탭에서 뒤로 가기 시 화면·URL 불일치 (`index.html`, popstate)
- 원본은 커서 탭에서 뒤로 가면 주소는 오프셋 검색인데 화면은 커서 탭에 남았습니다. 이전 누적 목록에 새 결과가 붙어 20건이 됐습니다.
- popstate에서 모드를 오프셋으로 되돌리고(탭, 페이지 바, 정렬 선택 활성화) 커서와 누적 목록을 초기화합니다.

### ⑦ 범위를 벗어난 page 값의 URL 보정 (`index.html`, loadData)
```js
state.page = json.meta.page;
if (urlParams.has('page') && urlParams.get('page') !== String(state.page)) {
  urlParams.set('page', state.page);
  window.history.replaceState({}, '', `?${urlParams.toString()}`);
}
```
- 원본은 `?page=9999`로 들어오면 화면은 100페이지인데 주소는 9999로 남았습니다. `-1`, `abc`도 마찬가지였습니다.
- 서버가 보정한 `meta.page`로 상태와 주소를 맞춥니다.

### ⑧ 최신순인데 작성일이 올라가던 문제 (`server.js`, 데이터 생성)
```diff
- createdAt: `2026-${(10 - Math.floor(i / 150))...}-${((i % 28) + 1)...}`
+ createdAt: new Date(Date.UTC(2026, 9, 31) - Math.floor(i / 3) * 86400000).toISOString().slice(0, 10)
```
- 원본은 정렬(id 기준)은 맞지만 날짜가 `10-01, 10-02, 10-03…`으로 올라가 캡처상 정렬이 틀린 것처럼 보였습니다.
- id가 클수록 날짜가 늦도록(하루 3건씩) 바꿨습니다. id 1000 = 2026-10-31, id 1 = 2025-12-02.

### ⑨ 입력 직후 조회 버튼을 누르면 요청이 2번 나가던 문제 (`index.html`, applySearch)
```diff
  function applySearch() {
+   clearTimeout(debounceTimer);
```

### ⑩ 검색 결과가 없을 때 페이지 버튼 숨김 (`index.html`, renderPagination)
- 원본은 결과가 0건이어도 페이지 버튼 `1`이 남았습니다. `totalCount === 0`이면 페이지 바를 비웁니다.

### ⑪ 마지막 묶음의 nextCursor (`server.js`, 커서 API)
```diff
- const nextCursor = items.length > 0 ? items[items.length - 1].id : null;
+ const nextCursor = hasNext ? items[items.length - 1].id : null;
```
- 다음 묶음이 없는데도 마지막 id를 돌려주던 것을 `null`로 바꿨습니다.

### ⑫ SQL 검사기를 파라미터 바인딩 형태로 표시 (`index.html`, updateSqlInspector)
- 칸 제목은 "보안 파라미터 바인딩"인데, 원본은 검색어를 SQL 문자열에 그대로 넣어 보여 줬습니다. 커서 탭 SQL에는 검색 조건이 빠져 있었습니다.
- `?` 자리표시자로 표시하고 바인딩 값을 따로 보여 줍니다. 커서 SQL에도 검색 조건을 추가했습니다.
```sql
SELECT ... FROM posts WHERE (title LIKE ? OR content LIKE ?) AND id < ? ORDER BY id DESC LIMIT ?;
-- 바인딩: ['%Redis%', '%Redis%', 883, 11]
```

### ⑬ 요청 실패 표시 (`index.html`, loadData)
```diff
  const res = await fetch(endpoint, { signal });
+ if (!res.ok) throw new Error(`HTTP ${res.status}`);
```
- 원본은 서버가 에러를 보내도 화면에 표시가 없고 콘솔에만 찍혔습니다.
- 실패하면 표에 "목록을 불러오지 못했습니다."를 표시해 빈 결과("검색 조건에 일치하는 글이 없습니다.")와 구분합니다. AbortError는 기존대로 조용히 넘어갑니다.

### ⑭ 같은 검색어 재요청 방지 (`index.html`, 디바운스)
```diff
  debounceTimer = setTimeout(() => {
+   if (e.target.value.trim() === state.search) return;
```
- 원본은 검색어 뒤에 공백만 추가해도 같은 검색어로 요청이 다시 나갔습니다.

### 수정하지 않은 항목
- **커서 탭에서 새로고침하면 오프셋 탭으로 돌아감**: 과제의 URL 상태 복원은 오프셋 모드만 요구하므로 그대로 두었습니다.
- **server.js의 `path`, `dateNum`**: 원본에 있던 쓰이지 않는 코드입니다. 동작에 영향이 없어 그대로 두었습니다.

---

## 개념 설명

### ① 오프셋·커서에 적합한 화면
- **오프셋**(`LIMIT ? OFFSET ?`): 게시판, 관리자 목록, 검색 결과처럼 **페이지 번호로 바로 이동**하거나 전체 건수·페이지 수를 보여줘야 하는 화면. 정렬 기준도 자유롭게 바꿀 수 있습니다. 대신 뒤 페이지로 갈수록 OFFSET만큼 읽고 버리고, 조회 중에 글이 추가·삭제되면 항목이 밀립니다.
- **커서**(`WHERE id < ? ORDER BY id DESC LIMIT ?`): SNS 피드, 채팅, 알림, 무한 스크롤처럼 **"더 보기"만 있는** 화면. 마지막으로 본 id 다음부터 읽으므로 깊이 내려가도 속도가 일정하고, 새 글이 추가돼도 누락·중복이 없습니다. 대신 특정 페이지로 점프할 수 없고 정렬 기준이 고정됩니다. 그래서 이 예제의 커서 모드는 정렬 선택을 비활성화했습니다.

### ② 디바운스·요청 취소의 역할
- **디바운스(300ms)**: 입력이 멈추고 300ms 동안 추가 입력이 없을 때만 요청합니다. "Redis"를 100ms 간격으로 치면 요청이 5번이 아니라 1번 나갑니다. **요청 횟수**를 줄입니다.
- **요청 취소(AbortController)**: 요청이 여러 개 나가면 먼저 보낸 요청의 응답이 나중에 도착할 수 있습니다. 그러면 입력창은 "Redis"인데 표는 이전 검색어 결과로 덮어써집니다. 새 요청 전에 이전 요청을 `abort()`하면 이전 fetch는 `AbortError`로 끝나 화면을 바꾸지 못합니다. `AbortError`는 정상적인 취소이므로 에러로 처리하지 않습니다. **응답 순서** 문제를 막습니다.

### ③ limit + 1 원리
다음 페이지가 있는지 알려고 `COUNT(*)`로 전체를 세는 대신 **limit보다 1개 더** 조회합니다.
- limit + 1개가 오면 뒤에 더 있다는 뜻입니다. `hasNext = true`로 두고 마지막 1개는 버린 뒤 limit개만 응답합니다.
- limit개 이하가 오면 마지막 페이지입니다. `hasNext = false`.
- 예: limit 10, 남은 글이 정확히 10개면 11개를 요청해도 10개만 옵니다 → `hasNext = false`.

### ④ offset · totalPages 계산식
```
offset     = (page − 1) × limit
totalPages = max(1, ceil(totalCount ÷ limit))
```
예: "Redis" 검색 83건, limit 10 → totalPages = ceil(83 ÷ 10) = 9, 3페이지 offset = (3 − 1) × 10 = 20.
page가 0·−1·abc이면 1로, totalPages보다 크면 마지막 페이지로, limit 1000은 100으로 보정합니다.

---

## AI 활용 내역
- **AI(Claude Code)를 사용한 부분**: 원본 코드 분석과 미충족 항목 확인, 수정 코드 작성, README·실습 보고서 초안 작성
- **AI가 실행해 확인한 내용**: 원본에서 `sort=__proto__`가 HTTP 500이 나는 것을 재현했습니다. 수정 후 API 요청과 브라우저에서 디바운스 요청 1회·요청 취소·커서 끝까지 누락/중복 0건·뒤로 가기 복원을 확인했습니다.
- **직접 검증한 내용**: 브라우저에서 직접 검증한 내용들을 캡처해서 보고서에 사진을 올렸다. 모든 결과가 기대와 일치함을 확인했다. 네트워크 속도를 3G로 낮춰서 이전 요청이 취소되고 마지막 검색어 결과만 나오는 것, 뒤로 가기 후 URL과 검색창과 정렬 결과가 일치한 것, 브라우저 주소창에 page=-1 등을 입력해 나온 meta 등을 확인했다.
