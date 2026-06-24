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

## 4. 데이터 구조
```js
const RAW = { 1:`베트남어|한국어뜻\n...`, 2:`...`, ... 12:`...` };  // 단원별, 줄마다 "베트남어|뜻"
const SENTENCES = `베트남어 문장|한국어뜻\n...`;                      // 예문(문장)
```
- **⚠️ 중요**: 코드에서 뜻(meaning) 필드 이름은 **`en`** 이지만 **실제로는 한국어**가 들어있음(원래 영어였다가 한국어로 교체한 흔적). `vn`=베트남어, `en`=한국어 뜻.
- 단어 **526개**(Lesson 1~12 = 496개 + **13단원 "오늘 배운 ⭐" 30개**), 문장 **53개**(Lesson 1~2 교재 기반). 단원 추가는 RAW에 같은 형식으로 넣으면 됨. RAW 키는 숫자여야 함(`buildWords`가 `Number(L)`). 13단원 라벨은 `LESSON_NAMES`로 "오늘 배운 ⭐" 표시.
- 사용자 진도는 **Lesson 2까지**. 더 진도 나가면 그 단원 문장을 SENTENCES에 추가.

## 5. 주요 기능 & 코드 위치 (함수명)
- 데이터 빌드: `buildWords()` — `WORDS` 배열(각 항목 `{id,lesson,vn,en,num,gi}`) 생성. 단어 추가(localStorage)도 합침.
- 문제 생성: `makeQ(word,dir,type)`, 방향 `dir`은 `"vn2en"`(베→한) / `"en2vn"`(한→베) / 랜덤.
- 홈/영역: `renderHome()`=**읽기·듣기·쓰기·말하기 4영역 카드**(`SKILLS`,`SKILL_ORDER`)+보조버튼(실전·플래시·단어장…). 카드 클릭→`goSkill(skill)`→`renderSetup()`(범위·문제수 선택)→`startSkill()`. `state.skill`(LS `_skill`)에 마지막 영역 저장. 방향은 영역이 고정(플래시만 사용자 선택).
- 시험 진행: `startSkill()`(영역 시작·표준 경로) / `start()`(구버전, 현재 홈에선 미사용), `startExam()`(실전 단어40+문장10), `startNote()`(오답노트), `setupQ()`, `renderQuiz()`, `next()`.
- 듣기(`mode:"listen"`): `renderQuiz()`의 `isListen` 분기 — 베트남어 글자 숨기고 🔊 자동재생(`.bigspk`/`.listenbox`), 정답 맞히면 글자 공개. 입력·채점은 주관식과 동일(vn2en 고정).
- 말하기(`mode:"speak"`): `startSpeak/renderSpeak/speakAdvance/renderSpeakDone`(상태 `SP`). 🔊 듣고 따라 말한 뒤 자가채점(✅ 잘했어요 / 🔁 더 연습=오답노트 추가). 점수 없음.
- 채점(관대): `norm()`, `acceptable()`, `isCorrectTyped()` — 성조·대소문자·`to`/관사·괄호·슬래시·콤마 복수정답 모두 관대 처리. **답에 든 숫자만 입력해도 정답**(예: "숫자 15, 열다섯"→"15"). **문장은 자가채점**(reveal→맞음/틀림).
- 시험지(인쇄): `genPaper()`, `renderPaper()`, state `P`(format: `wordsent`=단어40+문장10 / `words`=단어만). 중복 단어 자동 제거. `window.print()`. @media print로 컨트롤 숨김.
  - **우선출제**(`P.priority` 토글, 시험지 화면의 "📌 오늘 배운 ⭐ 단어 먼저 출제"): 켜면 '오늘 배운 단어'(`isReviewWord` = 13단원 ∪ `REVIEW_EXTRA` 기존단어 norm매칭)를 선택 범위에 없어도 포함하고 시험지 앞쪽에 배치. 새 시험 단어 묶음을 우선시킬 땐 13단원에 추가하거나 `REVIEW_EXTRA`에 vn 추가.
- 오답 노트: `loadNote/saveNote/noteWrong/noteRight/noteAdd`(키=`lc(vn)||lc(en)`), `renderNote()`. 오답 자동수집, 별표(`starWord`) 직접추가, 연속 `GRAD(=2)`번 정답 시 졸업(제거). `renderNote`에 **시험 방향**(베→한/한→베/랜덤, `noteDir`→`state.dir`) 선택. `startNote`는 듣기/말하기 모드 진입 시 주관식(`type`)으로 폴백.
  - **시험 오답 미리 등록**: `SEED_NOTE`(배열, `{vn,en,lesson,type?}`) + `seedNote()` — 앱 첫 실행 시 1회 오답노트에 머지(`_seedv`=`SEED_NOTE_V` 플래그). 같은 단어가 이미 있으면 `lesson`·`type`만 갱신(streak 유지), 없으면 추가. **새 시험 오답 추가/라벨 정정 시 `SEED_NOTE` 수정 후 `SEED_NOTE_V` 값을 바꾸면** 다음 로드 때 반영됨(기존 등록분도 갱신). 단어는 `type` 생략(word), 작문은 `type:"sentence"`(자가채점). `lesson`엔 시험 라벨(문자열 "6/22 시험" 등 — `lessonLabel`이 문자열은 그대로 표시).
- 백업/복원: `exportData()`, `importBackup()`(합치기/중복제외), `renderBackup()`.
- Telex 도움말: `renderTelex()`.
- 기타: `renderHome()`, `renderList()`(단어장), `renderAdd()`(단어추가), `renderFlash()`(플래시카드), `selectionBlockHTML()`(단원/범위 공용), `speak()`(🔊 Web Speech: 베=vi-VN, 한=ko-KR).
- 클릭 처리: 하단 `actions` 맵 + `view.addEventListener("click", ...)` (data-action 위임).

## 6. 자주 하는 변경
- **단어 추가**: `RAW`의 해당 단원 백틱 문자열에 `베트남어|한국어뜻` 줄 추가.
- **문장 추가**: `SENTENCES`에 `베트남어 문장|한국어뜻` 줄 추가.
- **방향/라벨**: `DIRLABEL`, 홈 방향 버튼(베→한/한→베), `promptLang`/`ansLang`(vi-VN/ko-KR).

## 7. 컨벤션 & 주의사항
- **색상**: 저채도 차분한 팔레트(`:root` CSS변수 `--brand:#5f6fa6` 등). 쨍한 원색 금지(사용자 요청).
- **인쇄 시험지**: 종이에는 **이모지·안내문·정답지 없음**(미니멀). `@page{margin:0}`로 브라우저 머리글/바닥글 제거.
- **표기**: 교재 따라 "**cám ơn**"(O), "cảm ơn"(X).
- **localStorage 키**(접두사 `gybm_wordnote_v1`): `_notebook _custom _mode _dir _count _selMode _wrong`. 브라우저/기기마다 따로 저장됨 → 그래서 **백업/복원** 기능이 있음.
- 홈은 **4영역(읽기·듣기·쓰기·말하기)** 중심: 읽기=베→한 주관식, 쓰기=한→베 주관식, 듣기=소리 듣고 뜻(글자 숨김), 말하기=따라 말하기 자가채점. 실전시험·플래시카드는 하단 보조 버튼. 객관식 모드는 제거됨(죽은 코드 일부 잔존).

## 8. GitHub / 배포
- 원격: **`chan026-normal/word-note`** (Public — GitHub Pages 때문에).
- Pages URL(폰에서 사용): **https://chan026-normal.github.io/word-note/**
- 커밋 이메일은 **noreply**로 설정됨(이 repo git config user.email = `chan026-normal@users.noreply.github.com`, local). 실제 이메일 노출 금지.
- 워크플로: **작업 전 `git pull`**(사용자가 폰 claude.ai/code로 먼저 고쳤을 수 있음), 작업 후 `commit + push`. 커밋 메시지 끝에 `Co-Authored-By: Claude Opus 4.8 <noreply@anthropic.com>`.
- 사용자는 비개발자 → 기술은 쉬운 비유로 설명. 한국어로 소통.
