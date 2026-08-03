# Atlas — 독립 배포 가이드

## V5 Live Intelligence 업데이트 (이번 버전)

기존 디자인·기능은 유지하고, "실시간 투자 플랫폼"에 맞는 구조를 얹었습니다.

- **Provider 아키텍처 확장**: `MockMarketProvider`(풍부한 Mock 시세/히스토리) + `KISProvider`/`YahooProvider`/`FinnhubProvider`(실제 fetch 시도 → 실패 시 자동으로 Mock 전환) + `ETFProvider`(비교/분석) + `AIProvider`(조언/ETF 요약). `state.settings.api.provider` 하나만 바꾸면 시스템 전체가 그 Provider를 사용합니다.
- **ETF 카드 실시간 시세**: 현재가·등락률·전일대비·거래량·시가·고가·저가·업데이트 시간이 카드에 자동 표시되고 30초마다 갱신됩니다. 현재가 수동 입력 기능은 그대로 남아 있습니다.
- **Market Dashboard 확장**: S&P500/Nasdaq100/Dow Jones/Russell2000/VIX/US10Y/Dollar Index/Fear&Greed 8개 지표, 연결 상태 배지 + 마지막 업데이트 시간 + 수동 새로고침 버튼.
- **ETF 검색 자동완성**: ETF 추가 폼에 검색창 추가 — "S&P" 입력하면 TIME/ACE/TIGER/KODEX 등 후보가 뜨고 클릭하면 자동 등록됩니다.
- **Watchlist 실시간화**: 현재가가 5분 단위로 갱신되는 라이브 값으로 표시되고, 목표가 도달 시 배지가 붙습니다.
- **Decision Center 전면 개편**: Mock 문구를 걷어내고 실시간 시장 데이터(오늘 가장 많이 움직인 지수) + 실제 포트폴리오 갭 + 우선 매수 ETF를 조합해 결론을 자동 생성합니다.
- **Alert Center (신규 대시보드 카드)**: 목표비중 초과 / 리밸런싱 필요 / 현금비중 부족 / 관심종목 목표가 도달을 자동 감지해 보여줍니다. 다른 카드처럼 Edit Mode에서 순서 변경·숨기기 가능합니다.
- **ETF Analyzer (신규, ETF 카드의 "분석" 버튼)**: 현재가/등락률/추적오차/총보수/순자산/거래량/1·3·5년 수익률(CAGR)/MDD/변동성/샤프지수/운용사/기초지수/배당여부 + 내 투자정보(보유금액/목표금액/현재비중/목표비중/부족·초과금액) + 상태 배지(🟢적정/🟡부족/🔴초과) + 기간별(1D~1Y) 가격 차트 + Rule 기반 AI Summary.
- **ETF Compare 업그레이드**: 배당 여부·기초지수 필드 추가, Table/Card 보기 전환.
- **History 페이지 (신규, Market Replay)**: 날짜별 S&P500/Nasdaq/VIX/Dollar/US10Y/포트폴리오 총자산/Investment Score/오늘의 행동을 하루 1회 자동 기록. 수동 기록 버튼도 있습니다.
- **API Configuration (Settings)**: Provider 선택(Mock/KIS/Yahoo/Finnhub) + 프록시 Base URL + API Key 입력. 키는 코드가 아니라 이 브라우저의 localStorage에만 저장됩니다.
- **캐시 + 중복요청 방지**: 모든 Provider 호출은 5분 캐시와 진행 중 요청 재사용(dedup) 레이어를 거칩니다.
- **자동 새로고침**: 30초마다 시세 갱신 (Edit Mode 중에는 입력을 방해하지 않도록 일시 중지).

### ⚠️ 반드시 읽어주세요 — "실시간"의 실제 의미
이 앱은 서버가 없는 정적 HTML 파일입니다. 이게 몇 가지 근본적인 한계를 만듭니다.

1. **KIS·Yahoo·Finnhub는 브라우저에서 직접 호출이 불가능합니다.** 대부분의 시세 API는 CORS로 브라우저 직접 호출을 막아두고, KIS는 시크릿 키로 서버 간 OAuth 토큰 교환이 필요해서 브라우저 코드에 키를 넣는 것 자체가 보안상 안 됩니다(요청하신 "API Key는 코드에 저장 금지"와 같은 이유). **그래서 Settings에서 Provider를 KIS로 바꿔도, 본인이 별도로 프록시 서버(그 API를 대신 호출해서 같은 JSON 형태로 돌려주는 작은 백엔드)를 운영하지 않는 한 실제로는 항상 Mock으로 자동 전환됩니다.** 아키텍처는 완전히 준비되어 있어서, 프록시 URL만 넣으면 그 순간부터 실제 데이터로 전환됩니다.
2. **"Mock"이지만 완전히 랜덤은 아닙니다.** ETF/지수 이름과 5분 단위 시간 버킷으로 시드를 만들어서, 새로고침해도 같은 5분 안에는 같은 값이 나오고 5분마다 자연스럽게 값이 바뀝니다. 실제 시세처럼 "그럴듯하게" 움직이지만 진짜 시장 데이터는 아닙니다.
3. **Market Replay(History)의 "매일 시장 종료 후 자동 저장"**도 서버 크론이 없어서, 정확히는 "그 날 앱을 처음 여는 시점"에 기록됩니다. 하루에 한 번도 안 열면 그날 기록은 없습니다 (수동 기록 버튼으로 보완 가능).
4. **AI Summary는 진짜 AI 호출이 아니라 규칙 기반(if/else)**입니다. `AIProvider.getEtfSummary()` 함수 하나만 실제 Claude/OpenAI API 호출로 바꾸면 되도록 분리해뒀습니다.

### 범위를 줄인 부분 (#18, #19)
- Chart lazy-loading이나 컴포넌트 재사용 같은 별도의 성능 레이어는 새로 만들지 않았습니다. 대신 캐시·dedup·차트 destroy 패턴으로 중복 호출과 메모리 누수를 막는 선에서 정리했습니다.
- WebSocket/Cloud Sync/Login/Multi-user는 실제 구현 없이, Provider 인터페이스가 이런 확장에 열려 있다는 설계 의도만 코드 주석으로 남겼습니다 (정적 파일 하나로는 로그인이나 서버 동기화 자체가 불가능합니다).

### ⚠️ 데이터 마이그레이션 안내
저장 키를 `atlas-state-v4` → `atlas-state-v5`로 올렸습니다. V4 데이터는 자동으로 넘어오지 않습니다. V4에서 JSON Export 후 V5에서 Import 해주세요 (구조가 호환되도록 만들었습니다).

## 작업 중 발견한 버그
1. Market Center 상태 바 마크업에서 `<div>` 하나에 `class` 속성을 실수로 두 번 써서 (`class="glass" ... class="row-between"`) 브라우저가 뒤에 쓴 값만 적용하는 버그가 있었습니다 — 검증 중 발견해서 하나로 합쳤습니다.
2. History 페이지를 추가하려고 문자열을 자르고 붙이는 과정에서 `renderJournal` 함수 선언 줄이 통째로 날아가는 사고가 있었습니다 (V4 때 겪었던 것과 같은 유형의 편집 실수). `grep`으로 모든 함수가 정확히 한 번씩만 정의됐는지 확인하는 검증 스크립트를 매 라운드 돌리고 있어서 배포 전에 잡았습니다.

---



## V4 Lite 업데이트 (이번 버전)

이번 버전은 새 기능보다 "쓰기 편하게" 만드는 데 집중했습니다. 디자인과 기존 기능은 전부 유지했습니다.

- **Edit Mode**: 화면 우측 상단 View/Edit 토글. Edit로 전환하면 카드들이 점선 테두리로 바뀌고 드래그·숨기기·인라인 입력이 활성화됩니다.
- **Inline Edit**: 예수금(Account Overview), 현재가·평균단가·목표비중·목표금액(ETF 카드), 은퇴나이·예상수익률(Retirement 페이지)을 Edit Mode에서 클릭 없이 바로 입력 가능한 숫자 필드로 표시. Enter 또는 포커스 아웃 시 자동 저장됩니다.
  - 평균단가는 원래 거래내역에서 자동 계산되는 값이라, 직접 수정하면 "수동 override"로 저장되고 카드에 "(수동)" 표시가 붙습니다. 되돌리기(↺) 버튼으로 다시 자동 계산으로 전환할 수 있습니다.
  - 목표금액을 직접 수정하면 내부적으로 동일한 ETF의 목표비중 값으로 환산되어 저장됩니다 (두 값은 같은 데이터라 하나만 저장합니다).
- **Dashboard 카드 커스터마이징**: Account Overview / Status Strip / 자산 요약 / DC Plan / Score / Mission / Advisor / Recommend / Health / Timeline — 10개 카드를 Edit Mode에서 드래그로 순서 변경, X 버튼으로 숨기기, 상단 "숨긴 카드" 패널에서 다시 표시할 수 있습니다. Decision Center와 Quick Action은 항상 최상단에 고정되어 숨기거나 순서를 바꿀 수 없습니다 (요청하신 "항상 먼저 보이는 카드" / "항상 표시" 요구사항 그대로).
- **Dashboard Layout**: Edit Mode에서 상단에 2열/3열/Auto 버튼으로 그리드 열 수를 바로 전환.
- **Workspace**: 사이드바 최상단에 Home / DC / Market / Retirement / Settings 5개 바로가기 추가. DC를 누르면 대시보드가 DC 계좌 기준으로 즉시 필터링됩니다.
- **Decision Center**: 대시보드 최상단에 "오늘 매수 없음 / 이번달 투자 진행률 42% / S&P 목표보다 3% 부족 / Investment Score 98점" 형태로 오늘 상황을 요약하고 마지막 줄에 결론 문장을 보여주는 카드.
- **Quick Action**: "+ 매수 입력 / + 현재가 수정 / + ETF 추가 / + Journal 작성" — 항상 대시보드 상단에 고정.
- 새로 추가된 설정(Edit Mode 상태, 카드 순서/숨김, 레이아웃 선택 등)도 모두 localStorage에 자동 저장됩니다.

### 알아두셔야 할 점
- **모바일 드래그 제한**: 카드 순서 변경은 브라우저 네이티브 드래그 앤 드롭(HTML5 Drag and Drop API)으로 구현했는데, 이 API는 터치 기기에서 기본적으로 동작하지 않습니다. 모바일에서는 카드 숨기기/다시 표시, 인라인 편집, 레이아웃 전환은 전부 정상 작동하지만 **드래그로 순서 바꾸는 것만 PC에서 하셔야 합니다.** 터치 드래그까지 지원하려면 별도 라이브러리가 필요해서 이번 Lite 버전 범위에서는 제외했습니다.
- **ETF 관리(#3)는 이미 V3에서 구현되어 있던 기능**이라 이번엔 그대로 유지만 했습니다 (추가/삭제/수정/투자이유/메모/색상 모두 Settings → ETF Master, 또는 색상은 여기서, 나머지는 그대로).

### ⚠️ 데이터 마이그레이션 안내
저장 키를 `atlas-state-v3` → `atlas-state-v4`로 올렸습니다. V3에 입력하신 데이터는 자동으로 넘어오지 않으니, V3 페이지에서 JSON 백업(Export)을 먼저 받아두신 뒤 V4에서 불러오기(Import)하시는 걸 권장합니다 — 이번엔 JSON 구조 자체는 V3와 동일해서 Import가 정상적으로 될 겁니다.

## 작업 중 발견한 버그
ETF 카드의 평균단가 "(수동)" 라벨을 만들 때 문자열 이어붙이기 중에 실수로 이스케이프된 백슬래시(`\"`)를 그대로 남겨서, 배포됐다면 화면에 `\"color:...\"` 같은 글자가 그대로 보이는 버그가 있었습니다. 최종 검증 단계에서 템플릿 리터럴 전체를 다시 훑다가 잡아서, 중첩 템플릿 리터럴로 깔끔하게 고쳤습니다.

---



## V3 업데이트 (이번 버전)

기존 디자인·UI·기능은 그대로 유지하고 구조를 계좌 단위로 확장했습니다.

- **Account Dashboard / Account Switch**: 연금저축(NH) · IRP(미래에셋) · DC(미래에셋) 3개 계좌를 개별 관리, 상단 Account Overview는 항상 전체를 보여주고, ALL/개별 계좌 탭을 선택하면 대시보드·포트폴리오·거래내역·통계 전체가 해당 계좌 기준으로 다시 계산됩니다.
- **ETF Master V2**: 운용사(TIME/TIGER/ACE/KODEX/SOL/HANARO), ETF 종류(S&P500/Nasdaq100/Bond/Gold/Dividend/REIT), 총보수 필드 추가
- **Rebalancing Center (신규 메뉴)**: 목표비중 대비 자동 분석 문장 + 추천 매수/매도 금액 테이블 (Portfolio Analysis 요구사항을 이 메뉴에 통합했습니다 — 같은 계산을 두 곳에 중복 구현하지 않기 위한 설계 판단입니다)
- **ETF Compare (신규 메뉴)**: 운용사별 Mock 비교 (수익률/총보수/순자산/추적오차/거래량), ETFProvider로 추상화되어 있어 나중에 실제 API로 교체 가능
- **Market Center (신규 메뉴)**: S&P500/Nasdaq100/VIX/미국10년채/DXY/Fear&Greed Mock 스냅샷, MarketProvider로 추상화
- **Watchlist (신규 메뉴)**: 관심 ETF 추가/삭제/메모/목표가 비교
- **Investment Calendar (신규 메뉴)**: 월별 캘린더에 매수/매도/배당 자동 표시 + 리밸런싱/연말점검 수동 이벤트 등록
- **Statistics (신규 메뉴)**: 월별·누적 투자금 차트, ETF별 평균매수가/손익/배당금 테이블
- **Retirement Upgrade**: 60/65/70/75세 각각의 예상 자산 + 예상 월 연금(4% 인출 기준) 동시 계산
- **Dashboard Upgrade**: Account Overview, Account Switch 탭, Quick Action 버튼(매수/매도 입력, 현재가 수정, Journal 작성, ETF 추가로 바로가기) 추가
- **Provider 추상화 레이어**: `AccountProvider` / `MarketProvider` / `ETFProvider` / `AIProvider` — 지금은 전부 Mock이지만, 나중에 실제 API 키가 생기면 이 객체들 내부만 교체하면 되도록 분리했습니다
- **JSON 구조 개편**: `accounts / etfs / transactions / journal / settings / watchlist / calendarEvents` 최상위 키로 재구성
- **CSV Export 추가** (거래내역), 기존 JSON 전체 백업/복원도 유지

### 범위 관련 솔직한 안내
요청하신 19개 항목 중 상당수를 실제로 동작하는 기능으로 구현했지만, 아래 두 가지는 의도적으로 범위를 좁혔습니다.
- **Performance 섹션(#16)**: 별도의 lazy-loading/캐싱 레이어를 새로 만들지는 않았습니다. 대신 차트 인스턴스를 페이지 전환마다 확실히 destroy하고, 슬라이더·일지 입력처럼 타이핑이 잦은 요소만 전체 리렌더 없이 갱신하는 기존 V2의 최적화를 유지했습니다. 지금 규모(순수 JS, 외부 프레임워크 없음)에서는 이 정도로 충분히 가볍게 동작하고, 데이터가 훨씬 커지기 전까지는 추가 최적화가 체감되지 않을 가능성이 높아서입니다.
- **Investment Calendar**: 이벤트 드래그 앤 드롭이나 일자별 상세 팝오버 같은 고급 UI 대신, 월 그리드 + 이벤트 목록 형태의 단순한 버전으로 구현했습니다. 기능은 다 있습니다 (등록/삭제/월 이동/자동 거래 표시).

### ⚠️ 데이터 마이그레이션 안내
JSON 구조가 계좌 단위로 완전히 바뀌어서 저장 키를 `atlas-state-v2` → `atlas-state-v3`로 분리했습니다. **V2에 입력하신 데이터는 자동으로 넘어오지 않습니다.** V2 페이지의 Settings에서 값을 확인 후 다시 입력해 주세요.

## 작업 중 발견한 버그
Statistics 페이지에 월별 데이터를 넘기려고 처음에는 HTML 문자열 안에 `<script id="stats-months-data">...</script>`를 중첩해서 넣었는데, 이건 제가 만든 문법 검사 도구뿐 아니라 **실제 브라우저에서도 페이지를 깨뜨리는 버그**였습니다. 브라우저의 HTML 파서는 문자열 안이든 밖이든 `</script>`라는 글자만 보이면 그 자리에서 바깥쪽 `<script>` 태그를 끝내버리기 때문입니다. 배포 전에 검증하다가 잡아서, 스크립트 태그를 중첩하는 대신 JS 변수에 데이터를 담아 전달하는 방식으로 고쳤습니다.

---



## V2 업데이트 (이번 버전)

기존 UI/UX와 디자인은 그대로 유지하고 다음을 추가했습니다.

- **실제 초기값 적용**: 예수금 117,812,580원, S&P500 평가금액 약 1,652,364원(반올림 근사치), 목표비중 S&P500 35% / 나스닥100 35% / 채권 30%
- **DC Investment Plan 카드**: 총자산·예수금·주식/채권 목표금액·현재비중·투자율·남은 투자금 등 10개 지표 자동 계산
- **Recommended Purchase (자동 추천 매수)**: 목표 대비 부족한 자산에 이번 달 권장 매수 금액을 자동 배분, 이유 문구 자동 생성
- **Investment Timeline**: 1/3/6개월 마일스톤 진행률 바
- **Investment Score 확장**: 7개 세부지표(Portfolio Balance / Risk / Cash Management / Investment Discipline / Diversification / Rebalancing / Goal Progress) + 원형 게이지
- **Portfolio Health**: Healthy / Good / Warning / Danger 상태 배지
- **CIO Dashboard 스트립**: 대시보드 최상단에서 5초 안에 오늘 상태 파악
- **ETF 카드 개선**: 매수/매도 바로가기, 남은 목표금액, 투자 이유 표시, ETF 수정/삭제
- **ETF Master (Settings)**: ETF 추가/수정/삭제/색상 지정/투자 이유·메모 관리
- **Transactions**: 배당 입력 추가, 실현손익·누적매수금액·누적배당금 자동 집계
- **Journal**: 태그, 검색 기능 추가
- **Retirement (신규 메뉴)**: 현재나이·예상수익률·월투자금 기반 복리 계산 + 차트
- **Settings**: 리밸런싱 기준, 투자계획 개월수/시작일, 예상수익률, 월 투자금 등 추가
- **JSON Export/Import**: 데이터 백업/복원 가능

### ⚠️ 데이터 마이그레이션 안내
V2는 저장 데이터 구조가 많이 바뀌어서 저장 키를 `atlas-state-v1` → `atlas-state-v2`로 분리했습니다.
**이전 V1 배포본에 입력해두신 데이터는 자동으로 넘어오지 않습니다.** 거래내역이나 일지를 이미 입력하셨다면, V1 페이지의 Settings에서 값을 확인 후 V2에 수동으로 다시 입력해 주세요. (다음부터는 Export/Import 기능으로 백업하시면 이런 문제가 없습니다.)


이 폴더에는 `index.html` 하나만 있습니다. 빌드 과정 없이 그대로 정적 호스팅하면 됩니다.
데이터는 브라우저의 실제 `localStorage`에 저장되므로, Claude 로그인과 완전히 무관하게 새로고침 후에도 유지됩니다.

## GitHub Pages로 배포 (무료, 가장 간단)

1. GitHub에서 새 리포지토리 생성 (예: `atlas`)
2. `index.html` 파일을 리포지토리 루트에 업로드 (드래그 앤 드롭 또는 `git push`)
3. 리포지토리 **Settings → Pages**
4. **Source**를 `Deploy from a branch`로 설정, Branch는 `main` / `(root)` 선택 후 저장
5. 1~2분 후 `https://<username>.github.io/atlas/` 로 접속 가능

```bash
git init
git add index.html README.md
git commit -m "Atlas v1"
git branch -M main
git remote add origin https://github.com/<username>/atlas.git
git push -u origin main
```

## 다른 방법

- **Netlify / Vercel**: `index.html`이 든 폴더를 드래그해서 올리면 즉시 URL 발급
- **로컬 파일**: 그냥 `index.html`을 더블클릭해서 브라우저로 열어도 됩니다 (`file://` 로 실행). 다만 폰에서는 파일 앱 → 브라우저로 열기가 번거로울 수 있어 GitHub Pages 쪽이 편합니다.

## 폰 홈 화면에 앱처럼 추가하기

1. 배포된 URL을 사파리(iOS) 또는 크롬(Android)으로 열기
2. 공유 버튼 → **"홈 화면에 추가"**
3. 아이콘을 누르면 주소창 없이 앱처럼 실행됩니다

## 참고

- 이 버전은 Claude 계정, 로그인, `window.storage` 없이 100% 독립적으로 동작합니다.
- 데이터는 그 브라우저/기기의 `localStorage`에만 저장됩니다 — 다른 기기와 자동 동기화되지 않습니다. (여러 기기에서 쓰려면 나중에 백엔드 연동이 필요합니다.)
- 브라우저 데이터를 지우면(캐시 삭제 등) 저장된 내용도 함께 사라지니, 가끔 Transactions/Journal 데이터를 별도로 백업해두시길 권합니다.
