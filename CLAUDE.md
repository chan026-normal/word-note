# 단어 시험 (베트남어 ↔ 한국어) — 인수인계서

> 이 문서는 새 작업 세션(Claude Code / claude.ai/code)이 이 프로젝트를 바로 이어서 작업할 수 있도록 정리한 핸드오버 문서입니다.

## 1. 프로젝트 개요
- GYBM 베트남 과정 참가자(한국인, 베트남어 학습자)를 위한 **베트남어 ↔ 한국어 단어/문장 시험·암기 웹앱**.
- **`단어시험.html` 파일 하나로 동작**하는 자기완결 앱: 프레임워크·빌드·서버 불필요, 더블클릭하면 브라우저에서 열림, **오프라인 작동**, 데이터는 브라우저 **localStorage**에 저장.
- 순수 **바닐라 JS**(외부 라이브러리 없음). UI는 한국어.

## 2. 파일 구조
- **`단어시험.html`** — 앱 본체(HTML+CSS+JS 한 파일). 거의 모든 작업은 여기서.
- `index.html` — 루트 URL용 리디렉트(→ `단어시험.html`). GitHub Pages 주소를 깔끔하게 하려고 둠. 수정 거의 안 함.
- `README.md`, `.gitignore`, 이 `CLAUDE.md`.
- `contents.zip` / 이미지 — 교재 사진 원본. **.gitignore로 제외**(저장소에 안 올라감).

## 3. 실행 / 미리보기 (중요)
- 이 환경엔 **node가 없음**. JS 검증은 브라우저 미리보기로 함.
- 미리보기 서버: `prj_code/.claude/launch.json`의 **`wordnote`** 설정(python http.server, 포트 8137, `word_note` 폴더 serve). `preview_start name=wordnote`.
- 파일명이 한글이라 URL은 인코딩 필요: `preview_eval`로 `location.href = location.origin + '/' + encodeURIComponent('단어시험.html')`.
- 검증은 `preview_eval`로 함수 직접 호출(예: `WORDS.length`, `start()`, `loadNote()`) + `preview_screenshot`.
- **⚠️ preview 서버가 샌드박스 권한오류로 못 뜰 때(2026-06-27 지속)**: 미리보기 프로세스가 `word_note`(심지어 `prj_code`)를 못 읽어 404. 그땐 **인쇄/PDF 레이아웃은 Chrome 헤드리스로 검증** 가능(Bash는 샌드박스 아님): ①`단어시험.html`을 scratchpad에 복사하고 끝에 `<script>setTimeout(()=>{P.onePage=true;P.withKey=true;genPaper(true);},400)</script>` 주입 ②`"/Applications/Google Chrome.app/Contents/MacOS/Google Chrome" --headless=new --no-pdf-header-footer --print-to-pdf=out.pdf --virtual-time-budget=3000 file://.../test.html` ③PDF 페이지수 확인+Swift(PDFKit)로 페이지를 PNG 렌더해 Read로 눈으로 확인(`mdls -name kMDItemNumberOfPages`도 가능). 시험지 페이지수·압축 맞춤은 이 방식으로 실측했음.
  - **앱 화면 스샷**(인쇄 아닌 일반 화면 검증): 같은 복사본 끝에 `<script>setTimeout(()=>{actions.goList()/*등 액션 호출*/},400)</script>` 주입 후 `chrome --headless=new --screenshot=out.png --window-size=430,1500 --force-device-scale-factor=2 file://...` → PNG를 Read. 아이콘 교체는 이 방식으로 7화면 검증함.

## 4. 데이터 구조
```js
const RAW = { 1:`베트남어|한국어뜻\n...`, 2:`...`, ... 12:`...` };  // 단원별, 줄마다 "베트남어|뜻"
const SENTENCES = `베트남어 문장|한국어뜻\n...`;                      // 예문(문장)
```
- **⚠️ 중요**: 코드에서 뜻(meaning) 필드 이름은 **`en`** 이지만 **실제로는 한국어**가 들어있음(원래 영어였다가 한국어로 교체한 흔적). `vn`=베트남어, `en`=한국어 뜻.
- 단어 **532개**(Lesson 1~12 = 496개 + **13단원 "오늘 배운 ⭐" 36개**, 사용자가 그날 배운 단어 모음). 단원 추가는 RAW에 같은 형식. RAW 키는 숫자여야 함(`buildWords`가 `Number(L)`). 13단원 라벨은 `LESSON_NAMES`로 "오늘 배운 ⭐" 표시.
- **문장 데이터 2종(별개!)**: ① `SENTENCES`/`SENTS` — 단원 구분 없는 53개, **실전시험(`startExam`)**용. ② `SENT_RAW`={단원번호:문장} + `SENT_BY_LESSON`/`sentLessons()` — **✍️ 문장 시험·시험지 문장범위**용, 현재 **1과 53·2과 62·3과 56·4과 81·5과 63·6과 88 = 403개(L1~6)**. 교재 사진→문장 추출(+쓰기연습 정답 생성분), 한국어는 직접 번역. (섹션5 참고)
  - **⚠️ 시험지 "전체"=`allSentences()`(=②의 단원 합집합 122개), `SENTS`(①53개) 아님**(2026-06-27 수정). 예전엔 "전체"가 옛 SENTS 53개라 단원 문장이 하나도 안 나오는 버그였음(사용자 발견). `allSentences = () => sentLessons().flatMap(l=>SENT_BY_LESSON[l])` 헬퍼로 단원 추가 시 자동 반영. 실전시험은 여전히 SENTS 사용.
- 사용자 진도: 단어 L1~12 + 그날 학습 단어(13단원)+7/2복습(14단원), 문장은 **1~6단원** 넣음. 새 단원 문장은 교재 사진 받아 `SENT_RAW`에 번호 키로 추가.

## 5. 주요 기능 & 코드 위치 (함수명)
- 데이터 빌드: `buildWords()` — `WORDS` 배열(각 항목 `{id,lesson,vn,en,num,gi}`) 생성. 단어 추가(localStorage)도 합침.
- 문제 생성: `makeQ(word,dir,type)`, 방향 `dir`은 `"vn2en"`(베→한) / `"en2vn"`(한→베) / 랜덤.
- 홈/영역: `renderHome()`=**읽기·듣기·쓰기·말하기 4영역 카드**(`SKILLS`,`SKILL_ORDER`)+보조버튼(실전·플래시·단어장…). 카드 클릭→`goSkill(skill)`→`renderSetup()`(범위·문제수 선택)→`startSkill()`. `state.skill`(LS `_skill`)에 마지막 영역 저장. **방향은 모든 영역(실전 제외)에서 베→한/한→베/랜덤 선택**(2026-06-25, `showDir=skill!=="exam"`; `goSkill` 진입 시 영역 기본 `dir` 세팅, `startSkill`은 `exam`만 random 고정). 말하기는 카드별 `SP.cards[].dir`(`renderSpeak`가 방향 따라 베트남어/한국어 먼저 보여줌).
- 시험 진행: `startSkill()`(영역 시작·표준 경로) / `start()`(구버전, 현재 홈에선 미사용), `startExam()`(실전 단어40+문장10), `startNote()`(오답노트), `setupQ()`, `renderQuiz()`, `next()`.
- ✍️ 문장 시험(단원별, 2026-06-24): `SENT_RAW`={단원번호:`베트남어 문장\|한국어`}(현재 L1~6, 총 403개 — 단원별 수는 섹션4/9 참고) + `SENT_BY_LESSON`/`sentLessons()`. 홈 "✍️ 문장 시험"→`goSent`→`renderSentSetup`(단원·방향 선택)→`startSent`(`sentence` 타입 자가채점). **기존 `SENTENCES`/`SENTS`(실전시험·시험지용, 단원 구분 없는 53개)와는 별개**. 단원 추가는 `SENT_RAW`에 번호 키로 문장 넣으면 됨(교재 사진→문장 추출, 한국어 뜻은 직접 번역).
- 듣기(`mode:"listen"`): `renderQuiz()`의 `isListen` 분기 — 베트남어 글자 숨기고 🔊 자동재생(`.bigspk`/`.listenbox`), 정답 맞히면 글자 공개. 입력·채점은 주관식과 동일(vn2en 고정).
- 말하기(`mode:"speak"`): `startSpeak/renderSpeak/speakAdvance/renderSpeakDone`(상태 `SP`). 🔊 듣고 따라 말한 뒤 자가채점(✅ 잘했어요 / 🔁 더 연습=오답노트 추가). 점수 없음.
- 채점: `norm()`, `acceptable()`, `isCorrectTyped()` — 대소문자·`to`/관사·괄호·슬래시·콤마 복수정답 관대 처리. **답에 든 숫자만 입력해도 정답**(예: "숫자 15, 열다섯"→"15"). **⚠️ 베트남어 답(`en2vn` 단어)은 성조·đ까지 채점**(`checkVnTyped`/`normTone`/`acceptableTone`, 2026-06-25): 성조까지 맞아야 정답, 철자만 맞고 성조 틀리면 `Q.toneOnly`→"⚠️ 성조가 틀려요(철자는 맞아요)" 안내(성조 학습 위해). **문장은 자가채점**(reveal→맞음/틀림).
- 시험지(인쇄): `genPaper()`, `renderPaper()`, state `P`(format: `wordsent`=단어40+문장10 / `words`=단어만). 중복 단어 자동 제거. `window.print()`. @media print로 컨트롤 숨김.
  - **우선출제**(`P.priority` 토글, 시험지 화면의 "📌 오늘 배운 ⭐ 단어 먼저 출제"): 켜면 '오늘 배운 단어'(`isReviewWord` = 13단원 ∪ `REVIEW_EXTRA` 기존단어 norm매칭)를 선택 범위에 없어도 포함하고 시험지 앞쪽에 배치. 새 시험 단어 묶음을 우선시킬 땐 13단원에 추가하거나 `REVIEW_EXTRA`에 vn 추가.
  - **문장 범위**(`P.sentLesson`, wordsent 2부 문장): "전체"(`allSentences()` = 모든 단원 문장 합 403개(L1~6), 2026-06-27 수정 — 옛 `SENTS`53 아님) 또는 단원별(`SENT_BY_LESSON[n]`) 선택. `paperSentLesson` 액션, 시험지 화면 "문장 범위" 칩. `genPaper`에서 `sentPool` 분기.
  - **정답지(시험지와 동일 레이아웃) + 시험지/정답지 따로 저장**: `renderPaper`가 시험지 `.paper` 뒤에 `.paper.pkey` 정답지 블록을 **항상** 추가. **정답지는 시험지와 똑같은 `.pq`(단어)·`.psq`(문장) 구조**를 쓰되 빈 밑줄(`.pqline`/`.psline`) 자리에 **정답을 초록으로 채움**(`.pqline.answ`/`.psline.answ`, 2026-06-27 개편 — 옛 `.pkeygrid/.pkq`의 `프롬프트 → 정답` 한 줄 형식 폐기). 그래서 번호·열정렬·압축이 시험지와 100% 동일. **인쇄 버튼 2개**(`printPaperOnly`/`printKeyOnly` → `printOnly("paper"|"key")`): `body`에 `ponly-paper`/`ponly-key` 클래스를 잠깐 붙여 `@media print`에서 한쪽만 보이게 한 뒤 `window.print()`, `afterprint`+setTimeout 백업으로 제거 → **시험지 PDF / 정답지 PDF 따로 저장**. `ponly-key`일 땐 `.pkey` `break-before` 무효화로 빈 첫 장 방지. (옛 `P.withKey` 토글은 대체·제거됨.) 정답지 압축 미세조정: `.paper.onepage .pqline.answ`/`.psline.answ`로 1장 유지. ⚠️ 문장 정답은 `.psq` 기본 폰트를 물려받아 너무 컸던 걸 `.psline.answ{font-size:13px}`(onepage 11px)로 본문보다 작게 지정 → 밑줄 안에 들어옴(단어 정답은 `.pq` 폰트 상속이라 OK).
  - **A4 한 장에 맞춤**(`P.onePage` 토글 "📄 A4 한 장에 맞춤", **기본 ON**, 2026-06-27): 켜면 두 `.paper`에 `onepage` 클래스 → `.paper.onepage *` 압축 CSS(폰트·줄간격·여백 축소)로 단어40+문장10이 **A4 1장**에 들어감(시험지·정답지 각각 1장). `paperOne` 액션(`renderPaper()`만). 압축값은 실제 인쇄 PDF로 맞춤(문장 10번째까지 1장 보장).
  - **단어 번호 세로(열 방향) 정렬**(2026-06-27): 2열 단어 그리드를 행→지그재그 대신 **왼쪽 열 1~N 내려간 뒤 오른쪽 열**로. `.pgrid`/`.pkeygrid.colflow`에 `grid-auto-flow:column; grid-template-rows:repeat(var(--prows),auto)`, 인라인 `style="--prows:${Math.ceil(wQs.length/2)}"`로 행 수 지정(2열 미디어=인쇄·≥560px에만 적용, 폰 1열은 그대로 순차). 시험지·정답지 번호 순서 일치.
- 오답 노트: `loadNote/saveNote/noteWrong/noteRight/noteAdd`(키=`lc(vn)||lc(en)`), `renderNote()`. 오답 자동수집, 별표(`starWord`) 직접추가, 연속 `GRAD(=2)`번 정답 시 졸업(제거). `renderNote`에 **시험 방향**(베→한/한→베/랜덤, `noteDir`→`state.dir`) 선택. `startNote`는 듣기/말하기 모드 진입 시 주관식(`type`)으로 폴백. **오답 직접 추가**(2026-06-25): `renderNote` 상단 폼(베트남어+뜻 입력) → `addNoteWord`/`addNoteSent` 액션 → `addNoteManual("word"|"sentence")` → `noteAdd`(lesson="내 오답", `noteHas` 중복체크). 누구나 종이 시험 오답을 앱에서 직접 입력(동기 공유 대비).
  - **시험 오답 미리 등록** (⚠️ **2026-06-25 비활성화**: 동기 공유 위해 `seedNote()`를 빈 함수로 만듦 → 새 사용자는 빈 오답노트, 기존 등록분은 각 기기 localStorage 유지, `SEED_NOTE` 배열은 코드에 잔존하나 미사용. 되살리려면 `seedNote` 본문 복원): `SEED_NOTE`(배열, `{vn,en,lesson,type?}`) + `seedNote()` — 앱 첫 실행 시 1회 오답노트에 머지(`_seedv`=`SEED_NOTE_V` 플래그). 같은 단어가 이미 있으면 `lesson`·`type`만 갱신(streak 유지), 없으면 추가. **새 시험 오답 추가/라벨 정정 시 `SEED_NOTE` 수정 후 `SEED_NOTE_V` 값을 바꾸면** 다음 로드 때 반영됨(기존 등록분도 갱신). 단어는 `type` 생략(word), 작문은 `type:"sentence"`(자가채점). `lesson`엔 시험 라벨(문자열 "6/22 시험" 등 — `lessonLabel`이 문자열은 그대로 표시).
- 백업/복원: `exportData()`, `importBackup()`(합치기/중복제외), `renderBackup()`.
- Telex 도움말: `renderTelex()`.
- 기타: `renderHome()`, `renderList()`(단어장), `renderAdd()`(단어추가), `renderFlash()`(플래시카드), `selectionBlockHTML()`(단원/범위 공용), `speak()`(🔊 Web Speech: 베=vi-VN, 한=ko-KR).
- 클릭 처리: 하단 `actions` 맵 + `view.addEventListener("click", ...)` (data-action 위임).

## 6. 자주 하는 변경
- **단어 추가**: `RAW`의 해당 단원 백틱 문자열에 `베트남어|한국어뜻` 줄 추가.
- **문장 추가(단원별)**: `SENT_RAW`에 단원 번호 키로 `베트남어 문장|한국어뜻` 줄. (`SENTENCES`는 실전·시험지용 기존 53개라 거의 안 건드림)
- **방향**: 홈엔 방향 버튼 없음(영역 카드). 방향은 `renderSetup`의 방향 세그(`setupDir`)·오답노트(`noteDir`)·문장시험(`sentDir`)에서. 영역별 기본은 `SKILLS[].dir`. `DIRLABEL`, `promptLang`/`ansLang`(vi-VN/ko-KR).

## 7. 컨벤션 & 주의사항
- **아이콘(이모지 금지, 2026-06-27)**: UI 그림 이모지는 **인라인 SVG 라인 아이콘**으로 교체함. `<body>` 첫머리 **`<svg><defs><symbol id="i-*">` 스프라이트**(home/book/note/pencil/cards/volume/plus/check/x/printer/star/eye/eye-off/headphones/mic/save/keyboard/alert/repeat/bulb/shuffle/pin/target/inbox/clipboard/folder/search/trophy/thumb) + CSS `.ic{width:1em;height:1em;stroke:currentColor;fill:none…}`. 사용: `<svg class='ic' viewBox='0 0 24 24'><use href='#i-NAME'/></svg>`(크기는 부모 font-size=1em). **새 UI에도 이모지 대신 이 아이콘 쓸 것**(없으면 `<symbol>` 추가). ⚠️ **속성값(placeholder/title/value)·`<option>`·`alert()`엔 SVG 못 넣음 → 거기선 문자**. **유지(문자)**: 방향 화살표 `→ ← ↔`, 언어 국기 `🇰🇷 🇻🇳`, 별표 토글 `☆ ★`(빈/채움 구분 위해). 이모지 컬러별 `⭐→★`.
- **색상**: 저채도 차분한 팔레트(`:root` CSS변수 `--brand:#5f6fa6` 등). 쨍한 원색 금지(사용자 요청).
- **인쇄 시험지**: 종이에는 **이모지·안내문 없음**(미니멀). `@page{margin:0}`로 브라우저 머리글/바닥글 제거. (시험지/정답지는 2026-06-27부터 **"시험지 저장"·"정답지 저장" 버튼으로 따로 PDF 출력** — `printOnly`. 미리보기엔 둘 다 표시.)
- **표기**: 교재 따라 "**cám ơn**"(O), "cảm ơn"(X).
- **localStorage 키**(접두사 `gybm_wordnote_v1`): `_notebook _custom _mode _dir _count _selMode _skill _seedv _wrong _zoom`. 브라우저/기기마다 따로 저장 → **백업/복원** 필요(홈 맨 아래 작은 회색 링크로 축소함, 사용자 요청). (`_zoom`=화면 확대 배율 0.8~2.0, 헤더 −/100%/+ 버튼.)
- **단원 선택(`state.lessons`)은 LS 저장 안 함**: 앱 시작 시 1회만 전체선택(시작부 `lessonsList().forEach(add)`), 이후 '전체 해제'가 유지됨(`renderSetup`/`goHome`은 자동 전체선택 안 함 — 예전엔 여기서 자동선택해 '전체 해제'가 무효화되던 버그였음). 시험지(`genPaper`)는 범위 0개여도 alert로 튕기지 않고 `renderPaper`에서 "범위 골라주세요" 안내(`1f36d1f`, ⚠️ preview 권한오류로 미검증).
- 홈은 **4영역(읽기·듣기·쓰기·말하기)** 중심. **4영역 모두 양방향**(베↔한, `renderSetup`에서 방향 선택). 홈 카드 방향 태그는 읽기·쓰기 **둘 다 "베 ↔ 한"으로 통일**(사용자 요청). 읽기/쓰기는 둘 다 주관식(type)이라 사실상 방향만 다른 비슷한 기능. 실전시험·플래시카드는 하단 보조. 객관식 모드 제거됨(죽은 코드 잔존).

## 8. GitHub / 배포
- 원격: **`chan026-normal/word-note`** (Public — GitHub Pages 때문에).
- Pages URL(폰에서 사용): **https://chan026-normal.github.io/word-note/**
- 커밋 이메일은 **noreply**로 설정됨(이 repo git config user.email = `chan026-normal@users.noreply.github.com`, local). 실제 이메일 노출 금지.
- 워크플로: **작업 전 `git pull`**(사용자가 폰 claude.ai/code로 먼저 고쳤을 수 있음), 작업 후 `commit + push`. 커밋 메시지 끝에 `Co-Authored-By: Claude Opus 4.8 <noreply@anthropic.com>`.
- 사용자는 비개발자 → 기술은 쉬운 비유로 설명. 한국어로 소통.

## 9. 현재 상태 & 미해결 (2026-07-09 세션)
- **2026-07-09 (시험지 문장범위 다중선택 + 스크롤 유지)**: 사용자 버그제보 2건 수정. ①**문장 범위 다중선택**: `P.sentLesson`(단일값 "all"|번호)→**`P.sentPick`(Set, 빈 Set=전체)**. 칩 토글(추가/해제), '전체'=비움. `genPaper`의 `sentPool`=선택 단원들 `paperSents` 합(빈 Set이면 `allPaperSentences`). 칩 on 상태 `P.sentPick.has(l)`/전체 `size===0`. 액션 `paperSentLesson` 토글로 변경. ②**시험지 컨트롤 클릭 시 맨위로 튀는 것 수정**: `set()`이 매 렌더 `scrollTo(0,0)` 하던 걸, 헬퍼 **`paperKeep(fn)`**(스크롤 y 저장→fn→복원)로 감싸 시험지 컨트롤(`paperDir`/`paperCount`/`paperFormat`/`paperPriority`/`paperOne`/`paperSentLesson`) 재생성 때 위치 유지(첫 진입 `goPaper`는 그대로 맨위). 헤드리스 검증: 3과+5과 동시선택 pool=119, 토글/전체 정상, 다중선택 칩 하이라이트 확인.

- **2026-07-06 (Cowork — 쓰기 연습 '정답 문장' SENT_RAW 추가, 데이터만)**: 각 단원 쓰기(THỰC HÀNH VIẾT) 변환문제(문장→의문문/부정문 등)의 빈칸 정답을 **문법 규칙대로 생성**해 `SENT_RAW`에 추가(코드/함수 변경 없음). 1과 +18(có…không 의문 9+부정 9)·2과 +16(ai 의문 10+부정 7)·3과 +15(mấy 의문 8+đã 과거 7)·4과 +10(bao nhiêu 의문 10)·5과 +14(sẽ 미래 10+quá 강조 4)·6과 +22(có·phải không/ai·gì·đâu·nào/mấy·bao nhiêu 의문). **문장 총 308→403.** 제외: 철자 고르기(ten/tên)·cũng 완성·빈칸 대화(회화 중복)·받아쓰기 지문. **⚠️ 교재 인쇄 정답이 아니라 규칙대로 생성한 변환문이라 성조·표현 오류 가능 — 나중에 검수 필요.** 헤드리스로 단원별 수(L1 53·L2 62·L3 56·L4 81·L5 63·L6 88=403)·형식오류0·부팅 검증.
- **2026-07-06 (Cowork 작업 — 읽기/쓰기 개편·동의어 확대·6단원)**: ①**쓰기에 방향 선택 추가**: `renderSetup`의 `showDir`에서 write 제외를 풀어 쓰기도 한→베·베→한·랜덤 선택 가능(읽기는 vn2en 고정 유지). `startSkill`을 `makeQ(w, dirOf(state.dir))`로 바꿔 '랜덤'이 문항마다 실제로 섞이게. `SKILLS.write` tag "한 ↔ 베"·desc 갱신. ②**읽기를 '입력 없이 읽고 뜻 확인'으로 개편**: `SKILLS.read.mode` "type"→"read". `renderQuiz`에 `kind==="read"` 분기(입력칸 대신 '뜻 보기'=`revealRead`로 한국어뜻+발음 공개, 채점·점수 없음, ☆'다시 볼 단어'=`readStar`로 오답노트 추가). 완료화면 `renderReadDone`, `renderResult`가 read면 그리로. `startNote` 모드 폴백에 read 추가. ③**쓰기 동의어 정답 확대**: `checkVnAny`를 넓혀, 뜻 표기가 완전 일치 안 해도 **뜻 조각(콤마/슬래시)이 겹치는 실제 교재 단어의 베트남어**면 정답(`meaningTokens`/`meaningsOverlap`). 예: 뜻 "얼마, 몇"에 `mấy`(몇) 입력→정답(성조채점·과인정 방지 유지). ④**6단원 문장 66개 추가**: `SENT_RAW` 6단원(교재 복습 87~106쪽 회화2·읽기·작문·문법예문, 한국어 직접번역). **문장 총 242→308(L1~6).** (사진 추출이라 성조/번역 오류 가능 — 회전 페이지 재확인 권장.) 헤드리스 검증: 부팅·읽기모드 화면·쓰기 방향선택·동의어확대·SENT6=66 확인.
- **2026-07-04 (Cowork 미커밋분 — 이번에 함께 올림)**: ⑤**단어장 '여러 단원 모아 저장'**: 단어장에서 여러 단원을 골라 한 Word(.doc)로 저장(단원 열 포함). `L.multi`/`L.pick`(Set), `pickedWordsAll`/`pickedLabel`/`lessonTag`/`downloadDoc`/`docFileName`/`exportPicked`, `wordDocHtml`에 `withLesson` 옵션, `renderList` 다중선택 분기, actions `toggleMulti`/`pickLesson`/`pickAll`/`pickClear`/`exportPicked`.

## (이전) 현재 상태 & 미해결 (2026-07-04 세션)
- **2026-07-04 (5단원 문장 49개 + 발음예문 시험지 재포함)**: ①`5단원.zip`(제68~86과 "BÂY GIỜ CHÚNG TA ĐI ĐÂU?/어디 가요", 주소·장소·큰숫자 nghìn/triệu/tỷ·chúng ta·sẽ·quá·à·đấy) 사진 19장에서 회화3(Vân/David, 택시/Dorothy, Lee/Toàn)+읽기(보행자거리 Nguyễn Huệ/Bùi Viện)+받아쓰기+문법예문(chúng ta·đấy·quá·à·sẽ) 추출 → `SENT_RAW` 5단원 **49문장** 추가. 문장 총 193→**242**. (5단원은 발음섹션에 '각 câu' 없어 lr 대상 없음.) ②**'발음 예문(lr) 시험지 제외' 되돌림**(사용자 요청): `paperSents = l => (SENT_BY_LESSON[l]||[])`로 필터 제거 → 시험지도 모든 문장 포함(allPaperSentences=allSentences=242). `@`/lr 표시는 데이터에 남겨둠(다시 제외하려면 `.filter(s=>!s.lr)` 추가). 사진 정방향(회전 없음).
- **2026-07-04 (수업 노트 단어 42개 추가)**: 사용자가 준 `베트남어_수업_정리노트.docx`(7/2 수업, 직업·장소·숫자·시간·형용사·일상활동·언어실력)에서 어휘 85개 추출 → 기존 RAW와 중복검사(43개 기존, 42개 신규) → **신규 42개를 `RAW` 14단원으로 추가**. 여러 주제라 특정 교재단원에 못 넣어(사용자 의견) **번호 대신 라벨 `LESSON_NAMES[14]="7/2 복습"`**(칩에 '7/2 복습'으로 뜸, '오늘 배운 ★'과 별개). WORDS 532→**574**. 영어 뜻(ENG_MAP) 없어 시험지 en2vn은 한국어 폴백. docx 파싱은 zip+document.xml. (중복 43개는 기존 뜻 유지—일부는 노트 뜻이 더 정확: gần=가깝다 vs 기존 거의/가까이, lái xe=운전기사 vs 운전하다 — 필요시 갱신.)
- **2026-07-04 (읽기/쓰기 방향 고정 + 문장 시험 타자화)**: ①**읽기·쓰기 방향 고정**(사용자: "읽기·쓰기가 같아 보임" → 구분). `read`=`vn2en`(베→한, 읽고 뜻 맞히기)·`write`=`en2vn`(한→베, 뜻 보고 베트남어 쓰기)로 **방향 선택 버튼 제거**(`renderSetup`의 `showDir`에서 read/write 제외 — 듣기·말하기는 방향 선택 유지). SKILLS 태그 `베↔한`→`베→한`/`한→베`, 설명·cardDesc 갱신. **⚠️ 예전 '읽기·쓰기 둘 다 양방향' 방침을 뒤집음**(이번 사용자 요청). ②**문장 시험 타자 입력**(사용자: "직접 타자로", "딱 안 맞아도 맞는 말이면 정답"). 문장 화면에 `<textarea id="sentInput">` + '채점하기'/'모르겠어요'. `checkSent`→`sentMatch`로 **관대한 자동채점**: `normSent`(소문자+**성조/diacritic 제거**(NFD+combining 제거+đ→d)+문장부호·띄어쓰기 무시) 후 완전일치 또는 편집거리 유사도≥0.8이면 자동 정답. 못 맞추면 모범답안+'내 답' 보여주고 **자가채점**(맞음/틀림 — 표현 달라도 맞으면 '맞았어요'). Enter=채점, Shift+Enter=줄바꿈. `Q.autoOk` 플래그. (오프라인이라 완전 의미판정 불가 → 관대매칭+자가채점 하이브리드.) 실전시험 문장 10개에도 동일 적용. 헤드리스 검증: 정답/성조없이=자동정답, 엉뚱=자가채점, 방향선택 read/write 없음·listen 유지.

## (이전) 현재 상태 & 미해결 (2026-07-03 세션)
- **2026-07-03 (시험지 '뜻→베' 영어 + 단원 전체해제 시작)**: ①**시험지 '뜻→베트남어'(en2vn) 문제의 뜻을 교재 영어로 출제**(사용자 요청, '베→뜻'(vn2en)·화면학습·플래시카드는 한국어 유지). 원본 영어판(커밋 `a3f3653`, 한국어화 전)에서 영어 뜻 복원 → **`ENG_RAW`(단원별 vn|english, RAW 뒤)** + **`ENG_MAP`**(키 `단원||norm(vn)` — 동형이의 정확 매칭: 예 anh=you vs Anh=England, thứ hai=monday/second, mất=to take/to die). `buildWords`가 `w.eng` 부여, `genPaper`에서 en2vn 프롬프트=`w.eng||w.en`(영어 없으면 한국어 폴백). **교재 1~12과 496개 영어 있음, 13과 30개·custom은 영어 없어 한국어 폴백.** 정답지도 같은 qs라 영어 프롬프트로 나옴. ②**앱 시작 시 단원 '전체 해제' 상태**(시험지·홈 모두, 예전 시작부 전체선택 1줄 제거) — 필요한 단원만 골라 씀. 헤드리스 검증: en2vn 290개 프롬프트 100% 영어(29번 'England'=Anh 확인)·vn2en 정답 100% 한국어·startup state.lessons=0.
- **2026-07-03 (시험지에서 '발음 예문' 제외)**: 교재 'Nghe và lặp lại các câu'(Listen and repeat, 발음 연습용 예문) 문장을 **시험지(genPaper)에서만 제외**(사용자 요청 — 아직 안 배운 단어가 섞이고 발음 위주라 시험 가치 낮음). **문장 시험(goSent 자습)에는 그대로 유지.** 구현: `SENT_RAW` 해당 줄 앞에 **`@` 접두사** → 파서가 `lr:true` 플래그 부여. 헬퍼 `paperSents(l)`=lr 제외, `allPaperSentences()`. genPaper `sentPool`·시험지 '문장 범위' 칩 개수는 이 헬퍼로 교체(`allSentences`/`SENT_BY_LESSON`은 goSent용 그대로). **대상: 1과 0 · 2과 0 · 3과 3(34쪽 초록박스: Các ông ấy muốn.../Có mười người đến muộn./Xin lỗi buổi tối...) · 4과 14(51쪽 ②섹션 6 + 52쪽 ④섹션 8).** 1·2과 발음예문은 애초에 SENT_RAW에 없음(2과 소개예문은 문법섹션 출처라 유지). 문장 총계: goSent 193 / 시험지 176(전체), 3과 41→38·4과 71→57. Chrome 헤드리스로 40회 생성 시 lr 0회·goSent L4 71문장 유지 검증. (교재 사진: `~/Downloads/N단원.zip`, 3·4단원은 회전 없음, 1·2단원도 upright.)
- **2026-07-03 (4단원 문장 추가)**: 교재 4단원(제50~67과 "ANH LÀM NGHỀ GÌ?/직업") 사진 18장에서 핵심 회화·예문·읽기 추출 → `SENT_RAW`에 `4:` 71문장 추가(회화 3개, 예문 섹션2·4, 읽기 ĐỌC HIỂU, 받아쓰기 Chính tả, 문법 예문 đang·rất·lắm·nào·thế nào·bao nhiêu). 발음연습(c/k/qu/g/gh/ng/ngh)·학생 손글씨 빈칸문제는 제외. 단어(WORDS)는 4과 44개가 이미 있어 문장만 추가. Chrome 헤드리스로 문장시험 화면(Lesson 1~4)·1번문항·부팅 검증. **문장 총 122→193개.** (⚠️ 검증 팁: 헤드리스 주입 시 `</body>`로 replace하면 `wordDocHtml` 템플릿 안의 `</body>`에 걸려 스크립트가 깨져 보임 — `rfind`로 마지막 `</body>` 앞이나 `</script>` 기준 주입할 것.)
- **2026-07-03 (Cowork 작업)**: ⑤**단어장 Word(.doc) 저장** — 지금 보이는 목록(단원/검색/내 단어)을 Word 문서로 내려받기(`wordDocHtml`/`exportWordDoc`, '단어장 보기'에 'Word로 저장' 버튼, `exportWord` 액션). 화면·저장 공용 필터로 `curListWords`/`curListLabel` 추출해 `renderList` 정리. ⑥**쓰기(한→베) 동의어 정답 인정** — 같은 한국어 뜻을 가진 다른 베트남어도 정답 처리(성조 채점 유지). `checkVnAny` 추가, `checkTyped`의 `en2vn` 분기를 `checkVnTyped`→`checkVnAny`로 교체(예: 뜻 "매우, 아주" → `rất`·`lắm` 모두 정답, 헤드리스로 확인). ⑦**화면 확대** — 헤더에 −/100%/+ 버튼, `#view`에 `zoom` 적용(0.8~2.0, LS `_zoom`), 직접 리스너 `bindZoom`, 인쇄 시 리셋(`@media print .wrap{zoom:1!important}`), 좁은 화면(≤430px)에선 `.sub` 숨김. (JS 문법·부팅·기능 헤드리스 검증 후 로컬 커밋/push.)
- **2026-07-01 (Cowork 작업)**: ①단어장 '내 단어' 칩을 '전체' 뒤로+개수, 전체보기에서 내 단어 맨 위 고정·accent 구분(`.chip.mine`/`.lz.mine`, `--accentsoft`/`--warn`), '내 단어'만 볼 때 추가·관리 힌트. ②**단어 추가 자동완성**: 한쪽만 입력→나머지 채움. 교재(`WORDS` 532)에 있으면 오프라인 즉시, 없으면 MyMemory 무료 API(vi↔ko) 번역+'확인 필요' 강조(`lookupLocal`/`translateOnline`/`doAutoFill`, `autoFill` 액션, blur/Enter/버튼 트리거). ③추가 후 '추가됐어요' 메시지 자동 페이드(`setAddStatus` autoClear+`.addstatus`). ④쓰기(주관식) Enter 버그: 입력창 Enter `preventDefault`/`stopPropagation` → 첫 Enter는 채점만, 다음은 두번째 Enter/'다음'. (Cowork에서 편집, push는 로컬에서)
- **2026-06-27 추가 세션 ④**: 시험지 문구 다듬기 — ①"문장 번역"→"문장" ②범위 표기 `L1,L2`→`1과,2과`(`rangeTitle` `l+"과"`, 13단원은 라벨) ③제목 "베트남어 시험 (단어+문장)"→"베트남어 시험" ④**정답지를 시험지와 동일 밑줄 레이아웃**으로 개편(정답을 줄에 초록 채움). 2장 유지(Chrome PDF 검증).
- **2026-06-27 추가 세션 ③**: 시험지 단어 번호 **세로(열 방향) 정렬**(왼1~N→오른N+1~, `grid-auto-flow:column`+`--prows`). 시험지·정답지 모두. Chrome 헤드리스 PDF로 1~20/21~40 확인.
- **2026-06-27 추가 세션 ②**: **UI 이모지 → 인라인 SVG 라인 아이콘 전면 교체**(87+곳, `<symbol>` 스프라이트 + `.ic`). 화살표·국기·☆★는 문자 유지(섹션7). Chrome 헤드리스 `--screenshot`으로 홈·설정·시험지·단어장·오답노트·말하기·퀴즈 7화면 검증. 검색창 placeholder는 속성이라 아이콘 빼고 텍스트로.
- **2026-06-27 추가 세션**: 시험지 **정답지**(문제 순서대로, 정답 초록) + **A4 한 장에 맞춤**(`P.onePage` 압축, **기본 ON** → 시험지1장·정답지1장) + **시험지/정답지 따로 저장**(`printOnly`, 인쇄 버튼 2개 — `P.withKey` 토글은 이걸로 대체·제거) + **시험지 문장 "전체"=단원 합 122개 버그수정**(`allSentences`). 모두 Chrome 헤드리스 print-to-pdf로 페이지수·내용 실측 검증(섹션3 참고).
- **이전 세션 큰 작업(커밋순)**: 4영역 학습(`ccf0e37`) → 오답노트 시드(6/22~25 시험 오답) → 숫자채점·오답방향 → 시험지 우선출제+13단원 단어 → 단원별 문장시험(`SENT_RAW`)+시험지 문장범위 → 오답 직접입력 → 성조채점+양방향+**SEED 비활성**(`32610fa`) → 6/25 학습단어 → 전체해제 버그수정+양방향표시 → 1·3단원 문장 → 백업버튼 축소 → 시험지 빈범위 처리.
- **⚠️ 동기 공유 워크플로(중요)**: 사용자가 앱을 GYBM 동기들과 Pages URL로 공유 중. 그래서 `seedNote()`를 껐음(새 사용자=동기는 빈 오답노트). **그 결과, 사용자 본인의 시험 오답을 코드(`SEED_NOTE`)에 넣으면 동기에게도 떠서 안 됨.** → 사용자 오답은 **백업 JSON으로 만들어 주고** 사용자가 폰 백업/복원으로 가져오거나(예: 6/27 오답은 JSON으로 제공함), **'➕ 오답 직접 추가'**로 입력. **코드에 넣는 건 '모두에게 공유되는 학습자료'(13단원 단어·`SENT_RAW` 문장)만.** ← 이 구분 꼭 지킬 것.
- **⚠️ 미검증**: 마지막 커밋(시험지 빈범위 처리)은 **preview 서버가 권한 오류(`PermissionError`)로 안 떠 화면 검증 못 함**. 사용자 폰 확인 대기. (평소엔 preview 잘 됐는데 세션 끝에 막힘 — 새 세션에서 재시도해볼 것.)
- **문장/단어 정확도**: 교재 사진(카톡, 90° 회전 → `sips -r 90` 보정 후 읽음)에서 베트남어 본문 추출, 한국어는 직접 번역. **성조 일부 오류 가능** — 사용자가 발견 시 수정. 각 단원 사진 010~ 일부(받아쓰기·반복연습)는 미추출(핵심 회화·예문·읽기 위주).
- **다음 예상**: 7단원~ 교재 사진 주면 `SENT_RAW`에 추가(6단원까지 완료). 6단원·쓰기연습 생성문장은 **검수 대기**(성조·번역 확인). 디자인/효과음/통계 등은 사용자가 '나중에'로 미뤄둠. 사용자가 매 시험 오답을 줄 때마다 위 '동기 공유 워크플로'대로 처리.
