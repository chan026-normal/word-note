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

## 4. 데이터 구조
```js
const RAW = { 1:`베트남어|한국어뜻\n...`, 2:`...`, ... 12:`...` };  // 단원별, 줄마다 "베트남어|뜻"
const SENTENCES = `베트남어 문장|한국어뜻\n...`;                      // 예문(문장)
```
- **⚠️ 중요**: 코드에서 뜻(meaning) 필드 이름은 **`en`** 이지만 **실제로는 한국어**가 들어있음(원래 영어였다가 한국어로 교체한 흔적). `vn`=베트남어, `en`=한국어 뜻.
- 단어 **532개**(Lesson 1~12 = 496개 + **13단원 "오늘 배운 ⭐" 36개**, 사용자가 그날 배운 단어 모음). 단원 추가는 RAW에 같은 형식. RAW 키는 숫자여야 함(`buildWords`가 `Number(L)`). 13단원 라벨은 `LESSON_NAMES`로 "오늘 배운 ⭐" 표시.
- **문장 데이터 2종(별개!)**: ① `SENTENCES`/`SENTS` — 단원 구분 없는 53개, **실전시험·시험지 2부**용. ② `SENT_RAW`={단원번호:문장} + `SENT_BY_LESSON`/`sentLessons()` — **✍️ 문장 시험·시험지 문장범위**용, 현재 **1단원 35 + 2단원 46 + 3단원 41개**. 교재 사진→문장 추출, 한국어는 직접 번역. (섹션5 참고)
- 사용자 진도: 단어 L1~12 + 그날 학습 단어(13단원), 문장은 **1·2·3단원** 넣음. 새 단원 문장은 교재 사진 받아 `SENT_RAW`에 번호 키로 추가.

## 5. 주요 기능 & 코드 위치 (함수명)
- 데이터 빌드: `buildWords()` — `WORDS` 배열(각 항목 `{id,lesson,vn,en,num,gi}`) 생성. 단어 추가(localStorage)도 합침.
- 문제 생성: `makeQ(word,dir,type)`, 방향 `dir`은 `"vn2en"`(베→한) / `"en2vn"`(한→베) / 랜덤.
- 홈/영역: `renderHome()`=**읽기·듣기·쓰기·말하기 4영역 카드**(`SKILLS`,`SKILL_ORDER`)+보조버튼(실전·플래시·단어장…). 카드 클릭→`goSkill(skill)`→`renderSetup()`(범위·문제수 선택)→`startSkill()`. `state.skill`(LS `_skill`)에 마지막 영역 저장. **방향은 모든 영역(실전 제외)에서 베→한/한→베/랜덤 선택**(2026-06-25, `showDir=skill!=="exam"`; `goSkill` 진입 시 영역 기본 `dir` 세팅, `startSkill`은 `exam`만 random 고정). 말하기는 카드별 `SP.cards[].dir`(`renderSpeak`가 방향 따라 베트남어/한국어 먼저 보여줌).
- 시험 진행: `startSkill()`(영역 시작·표준 경로) / `start()`(구버전, 현재 홈에선 미사용), `startExam()`(실전 단어40+문장10), `startNote()`(오답노트), `setupQ()`, `renderQuiz()`, `next()`.
- ✍️ 문장 시험(단원별, 2026-06-24): `SENT_RAW`={단원번호:`베트남어 문장\|한국어`}(현재 1단원 35 + 2단원 46 + 3단원 41개) + `SENT_BY_LESSON`/`sentLessons()`. 홈 "✍️ 문장 시험"→`goSent`→`renderSentSetup`(단원·방향 선택)→`startSent`(`sentence` 타입 자가채점). **기존 `SENTENCES`/`SENTS`(실전시험·시험지용, 단원 구분 없는 53개)와는 별개**. 단원 추가는 `SENT_RAW`에 번호 키로 문장 넣으면 됨(교재 사진→문장 추출, 한국어 뜻은 직접 번역).
- 듣기(`mode:"listen"`): `renderQuiz()`의 `isListen` 분기 — 베트남어 글자 숨기고 🔊 자동재생(`.bigspk`/`.listenbox`), 정답 맞히면 글자 공개. 입력·채점은 주관식과 동일(vn2en 고정).
- 말하기(`mode:"speak"`): `startSpeak/renderSpeak/speakAdvance/renderSpeakDone`(상태 `SP`). 🔊 듣고 따라 말한 뒤 자가채점(✅ 잘했어요 / 🔁 더 연습=오답노트 추가). 점수 없음.
- 채점: `norm()`, `acceptable()`, `isCorrectTyped()` — 대소문자·`to`/관사·괄호·슬래시·콤마 복수정답 관대 처리. **답에 든 숫자만 입력해도 정답**(예: "숫자 15, 열다섯"→"15"). **⚠️ 베트남어 답(`en2vn` 단어)은 성조·đ까지 채점**(`checkVnTyped`/`normTone`/`acceptableTone`, 2026-06-25): 성조까지 맞아야 정답, 철자만 맞고 성조 틀리면 `Q.toneOnly`→"⚠️ 성조가 틀려요(철자는 맞아요)" 안내(성조 학습 위해). **문장은 자가채점**(reveal→맞음/틀림).
- 시험지(인쇄): `genPaper()`, `renderPaper()`, state `P`(format: `wordsent`=단어40+문장10 / `words`=단어만). 중복 단어 자동 제거. `window.print()`. @media print로 컨트롤 숨김.
  - **우선출제**(`P.priority` 토글, 시험지 화면의 "📌 오늘 배운 ⭐ 단어 먼저 출제"): 켜면 '오늘 배운 단어'(`isReviewWord` = 13단원 ∪ `REVIEW_EXTRA` 기존단어 norm매칭)를 선택 범위에 없어도 포함하고 시험지 앞쪽에 배치. 새 시험 단어 묶음을 우선시킬 땐 13단원에 추가하거나 `REVIEW_EXTRA`에 vn 추가.
  - **문장 범위**(`P.sentLesson`, wordsent 2부 문장): "전체"(`SENTS` 53개) 또는 단원별(`SENT_BY_LESSON[n]`) 선택. `paperSentLesson` 액션, 시험지 화면 "문장 범위" 칩. `genPaper`에서 `sentPool` 분기.
  - **정답지**(`P.withKey` 토글 "📝 정답지 같이 인쇄", 기본 ON, 2026-06-27): `renderPaper`가 시험지 `.paper` 뒤에 `.paper.pkey` 정답지 블록 추가(문제 순서 그대로 `프롬프트 → 정답`, 정답은 `.pkq i` 초록 강조). `@media print`에서 `.pkey{break-before:page}`로 **새 페이지(맨 마지막 장)**. `paperKey` 액션은 `renderPaper()`만 호출(재섞기 X). 학생용은 끄기. CSS는 `.pkeygrid/.pkq`.
  - **A4 한 장에 맞춤**(`P.onePage` 토글 "📄 A4 한 장에 맞춤", **기본 ON**, 2026-06-27): 켜면 두 `.paper`에 `onepage` 클래스 → `.paper.onepage *` 압축 CSS(폰트·줄간격·여백 축소)로 단어40+문장10이 **A4 1장**에 들어감(정답지까지 합쳐 총 2장). `paperOne` 액션(`renderPaper()`만). 압축값은 실제 인쇄 PDF로 맞춤(문장 10번째까지 1장 보장). 끄면 글씨 큰 3장.
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
- **색상**: 저채도 차분한 팔레트(`:root` CSS변수 `--brand:#5f6fa6` 등). 쨍한 원색 금지(사용자 요청).
- **인쇄 시험지**: 종이에는 **이모지·안내문 없음**(미니멀). `@page{margin:0}`로 브라우저 머리글/바닥글 제거. (정답지는 2026-06-27부터 `P.withKey`로 **별도 마지막 장**에 출력 — 토글로 끌 수 있음.)
- **표기**: 교재 따라 "**cám ơn**"(O), "cảm ơn"(X).
- **localStorage 키**(접두사 `gybm_wordnote_v1`): `_notebook _custom _mode _dir _count _selMode _skill _seedv _wrong`. 브라우저/기기마다 따로 저장 → **백업/복원** 필요(홈 맨 아래 작은 회색 링크로 축소함, 사용자 요청).
- **단원 선택(`state.lessons`)은 LS 저장 안 함**: 앱 시작 시 1회만 전체선택(시작부 `lessonsList().forEach(add)`), 이후 '전체 해제'가 유지됨(`renderSetup`/`goHome`은 자동 전체선택 안 함 — 예전엔 여기서 자동선택해 '전체 해제'가 무효화되던 버그였음). 시험지(`genPaper`)는 범위 0개여도 alert로 튕기지 않고 `renderPaper`에서 "범위 골라주세요" 안내(`1f36d1f`, ⚠️ preview 권한오류로 미검증).
- 홈은 **4영역(읽기·듣기·쓰기·말하기)** 중심. **4영역 모두 양방향**(베↔한, `renderSetup`에서 방향 선택). 홈 카드 방향 태그는 읽기·쓰기 **둘 다 "베 ↔ 한"으로 통일**(사용자 요청). 읽기/쓰기는 둘 다 주관식(type)이라 사실상 방향만 다른 비슷한 기능. 실전시험·플래시카드는 하단 보조. 객관식 모드 제거됨(죽은 코드 잔존).

## 8. GitHub / 배포
- 원격: **`chan026-normal/word-note`** (Public — GitHub Pages 때문에).
- Pages URL(폰에서 사용): **https://chan026-normal.github.io/word-note/**
- 커밋 이메일은 **noreply**로 설정됨(이 repo git config user.email = `chan026-normal@users.noreply.github.com`, local). 실제 이메일 노출 금지.
- 워크플로: **작업 전 `git pull`**(사용자가 폰 claude.ai/code로 먼저 고쳤을 수 있음), 작업 후 `commit + push`. 커밋 메시지 끝에 `Co-Authored-By: Claude Opus 4.8 <noreply@anthropic.com>`.
- 사용자는 비개발자 → 기술은 쉬운 비유로 설명. 한국어로 소통.

## 9. 현재 상태 & 미해결 (2026-06-27 세션)
- **2026-06-27 추가 세션**: 시험지 **정답지**(`P.withKey`, 별도 마지막 장, 기본 ON) + **A4 한 장에 맞춤**(`P.onePage` 압축, **기본 ON** → 시험지1+정답지1=2장) 추가. Chrome 헤드리스 print-to-pdf로 페이지수 실측 검증(섹션3 참고). 사용자 확인: 토글 OFF면 시험지가 단어1장+문장1장으로 쪼개져 3장이 나오던 것이 원인이었음.
- **이전 세션 큰 작업(커밋순)**: 4영역 학습(`ccf0e37`) → 오답노트 시드(6/22~25 시험 오답) → 숫자채점·오답방향 → 시험지 우선출제+13단원 단어 → 단원별 문장시험(`SENT_RAW`)+시험지 문장범위 → 오답 직접입력 → 성조채점+양방향+**SEED 비활성**(`32610fa`) → 6/25 학습단어 → 전체해제 버그수정+양방향표시 → 1·3단원 문장 → 백업버튼 축소 → 시험지 빈범위 처리.
- **⚠️ 동기 공유 워크플로(중요)**: 사용자가 앱을 GYBM 동기들과 Pages URL로 공유 중. 그래서 `seedNote()`를 껐음(새 사용자=동기는 빈 오답노트). **그 결과, 사용자 본인의 시험 오답을 코드(`SEED_NOTE`)에 넣으면 동기에게도 떠서 안 됨.** → 사용자 오답은 **백업 JSON으로 만들어 주고** 사용자가 폰 백업/복원으로 가져오거나(예: 6/27 오답은 JSON으로 제공함), **'➕ 오답 직접 추가'**로 입력. **코드에 넣는 건 '모두에게 공유되는 학습자료'(13단원 단어·`SENT_RAW` 문장)만.** ← 이 구분 꼭 지킬 것.
- **⚠️ 미검증**: 마지막 커밋(시험지 빈범위 처리)은 **preview 서버가 권한 오류(`PermissionError`)로 안 떠 화면 검증 못 함**. 사용자 폰 확인 대기. (평소엔 preview 잘 됐는데 세션 끝에 막힘 — 새 세션에서 재시도해볼 것.)
- **문장/단어 정확도**: 교재 사진(카톡, 90° 회전 → `sips -r 90` 보정 후 읽음)에서 베트남어 본문 추출, 한국어는 직접 번역. **성조 일부 오류 가능** — 사용자가 발견 시 수정. 각 단원 사진 010~ 일부(받아쓰기·반복연습)는 미추출(핵심 회화·예문·읽기 위주).
- **다음 예상**: 4단원~ 교재 사진 주면 `SENT_RAW`에 추가. 디자인/효과음/통계 등은 사용자가 '나중에'로 미뤄둠. 사용자가 매 시험 오답을 줄 때마다 위 '동기 공유 워크플로'대로 처리.
