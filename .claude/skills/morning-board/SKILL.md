---
name: morning-board
description: 연합뉴스 RSS에서 경제 뉴스 위주 3건과 영주시 상망동 오늘 날씨를 가져와 news.md·weather.md·weather_alert.md·dashboard.html을 morningbox 폴더에 생성/갱신하는 아침 브리핑 워크플로우. 사용자가 /morning-board 라고 입력하면 실행한다.
---

# morning-board — 아침 브리핑 자동 생성

`/morning-board` 명령이 들어오면 아래 순서를 그대로, 매번 처음부터 실행한다.

## 지켜야 할 공통 규칙

- 산출물 파일(`news.md`, `weather.md`, `weather_alert.md`, `dashboard.html`)은 전부 이 프로젝트 폴더(morningbox) 안에만 만들거나 덮어쓴다. 다른 폴더는 건드리지 않는다.
- 모든 파일은 한글이 깨지지 않도록 **UTF-8**로 저장한다.
- 이 워크플로우는 현재 공개 API(연합뉴스 RSS, open-meteo)만 사용하며 별도 인증 키가 필요 없다. 앞으로 어떤 단계에서든 API 키·토큰 같은 값이 필요해지면 **그 값을 코드나 파일에 절대 직접 적지 말고, `.env` 파일을 읽어서 쓴다고만 기술**한다 (실제 키 값은 스킬 파일이나 대화, 산출물 어디에도 남기지 않는다).
- 매 실행은 "오늘" 기준으로 새로 조회한다. 이전 실행 결과(과거 날짜·수치)를 재사용하지 않는다.

## 1단계 — 뉴스 가져오기 → news.md

1. PowerShell로 `https://www.yna.co.kr/rss/news.xml` 을 가져와 XML로 파싱한다.
   - `title`은 CDATA이므로 `$item.title.'#cdata-section'` 형태로 읽는다.
   - `link`, `pubDate`도 함께 읽는다.
2. 전체 기사 중 제목에 "경제"가 포함된 기사를 찾는다 (발행시각 최신순).
3. 경제 기사가 3건 미만이면, 부족한 개수만큼 전체 기사 중 가장 최신 기사로 채운다(이미 고른 기사와 중복되지 않게).
4. 최종 3건의 제목·링크·발행시각을 정리해 `news.md`에 한 줄씩 저장한다.
   - 경제 기사는 표시를 남긴다 (예: `[경제]` 접두어).
   - 파일 하단에 출처 URL과, 경제 기사가 부족해 최신 기사로 채웠는지 여부를 각주로 남긴다.

## 2단계 — 날씨 가져오기 → weather.md, weather_alert.md

1. open-meteo 지오코딩 API로 "영주시" 좌표를 조회한다.
   `https://geocoding-api.open-meteo.com/v1/search?name=영주시&count=10&language=ko&format=json`
   - "상망동" 단위 좌표는 open-meteo에 없으므로, 영주시 중심 좌표로 대체하고 그 사실을 메모로 남긴다.
2. open-meteo 예보 API로 오늘의 최고기온·최저기온·강수확률(최대)을 가져온다.
   `https://api.open-meteo.com/v1/forecast?latitude={lat}&longitude={lon}&daily=temperature_2m_max,temperature_2m_min,precipitation_probability_max&timezone=Asia%2FSeoul&forecast_days=1`
3. `weather.md`에 오늘 날짜, 최고기온, 최저기온, 강수확률, 출처, 좌표 대체 메모를 저장한다.
4. 판정 규칙(고정 기준값): **강수확률 ≥ 60% 또는 최저기온 ≤ 18℃** 이면 「⚠️ 기준 넘음」, 아니면 「기준 미달」.
5. `weather_alert.md`에 판정 결과와 근거 수치를 한 줄로 저장한다.

## 3단계 — 대시보드 만들기 → dashboard.html

1. 같은 폴더의 `Design.md`를 읽어 색·레이아웃·글자 기준을 확인하고 그대로 따른다 (임의로 스타일을 바꾸지 않는다). Design.md가 바뀌면 다음 실행부터는 바뀐 기준을 따른다.
2. 화면 위쪽에 날씨 상자를 배치한다.
   - 최고·최저기온·강수확률을 큼직한 숫자로, 어울리는 이모지(☀️/🌙/💧 등)와 함께 가로로 배치한다.
   - `weather_alert.md`의 판정을 배지로 표시한다. 「기준 넘음」이면 Design.md의 경고 톤(다크 옐로우/오렌지)으로, 「기준 미달」이면 차분한 톤으로 표시한다.
3. 화면 아래쪽에 뉴스 3건을 배치한다.
   - 제목은 실제 기사 링크로 연결한다(새 탭).
   - 발행 시각·출처는 보조 글자색으로 표시한다.
   - [경제] 기사에는 Design.md의 강조색 배지를 붙인다.
   - 데스크톱/모바일 반응형은 Design.md의 레이아웃 규칙(예: 데스크톱 다열 그리드, 좁은 화면은 1열)을 따른다.
4. `dashboard.html`을 UTF-8로 저장한다.
5. Claude Browser 도구로 `dashboard.html`을 열어 데스크톱 폭과 모바일 폭(375px 안팎) 양쪽에서 레이아웃이 깨지지 않는지 확인한다.

## 4단계 — 배포하기 (홈페이지 갱신)

1. `dashboard.html`을 `index.html`로 복사한다(덮어쓰기).
2. 변경 사항을 확인한다: `git status --porcelain`
   - 변경 사항이 없으면 이 단계는 건너뛴다.
3. 변경 사항이 있으면 커밋하고 푸시한다.
   ```
   git add -A
   git commit -m "갱신"
   git push
   ```
   - git/GitHub 로그인이 안 되어 있거나 push가 실패하면, 사용자에게 실패 원인 1줄과 다음 행동(예: 홈페이지.md 1~2번 다시 실행)을 안내한다.

## 완료 후 보고

작업이 끝나면 사용자에게 다음을 간단히 요약해서 알려준다:
- 갱신/생성한 파일 목록
- 오늘의 최고·최저기온·강수확률과 기준 판정 결과
- 선택된 뉴스 3건의 제목 (경제 기사 표시 포함)
- 홈페이지에 반영됐는지 여부 (커밋/푸시 성공·건너뜀·실패)
