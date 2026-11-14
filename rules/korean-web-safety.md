---
title: 한국어 웹·모바일 안전 규칙 (한글 IME, 폰트, 인코딩, 조사 토크나이징)
tags: [korean, ime, composition, capacitor, webview, tiptap, prosemirror, encoding, cp949, font]
updated: 2026-11-13
---

# 한국어 웹·모바일 안전 규칙 (`korean-web-safety.md`)

교육용 웹앱 및 태블릿 하이브리드 앱(Capacitor/Android WebView)에서 한글 입력기(IME) 조합 깨짐, 마크다운 입력 무력화, 한글 인코딩(`CP949`) 깨짐, 한국어 검색 누락을 방지하기 위한 공통 규칙입니다.

---

## 1. 한글 IME 조합 보호 (`isComposing` 및 이벤트 방어)

### 1.1 검색창·채팅창·인라인 입력창 기본 방어
- 모든 텍스트 입력 핸들러(`keydown`)에서 `e.nativeEvent.isComposing === true` 또는 `e.keyCode === 229` 상태에서는 `Enter` 제출·단축키 실행을 차단하여 끝 글자가 중복 제출되거나 마지막 음절이 잘리는 현상을 방지한다.
- `<input>` 및 `<textarea>`에는 모바일/태블릿 가상 키보드가 한국어 입력 컨텍스트를 인식하도록 다음 속성을 명시한다.
  ```tsx
  <input
    type="text"
    inputMode="text"
    lang="ko"
    autoCapitalize="off"
    autoCorrect="off"
    spellCheck={false}
  />
  ```
- 모달 다이얼로그의 `<input>`에는 HTML `autoFocus` 속성을 직접 사용하지 않는다. Android WebView에서는 키보드 컨텍스트 바인딩 전에 포커스가 전달되어 영문 자판으로 고정될 수 있으므로, 마운트 후 `setTimeout(..., 60)` 지연 포커스를 사용한다.

### 1.2 Capacitor Android WebView `captureInput` 함정 금지
- `capacitor.config.ts`에서 `android.captureInput`은 반드시 `false`로 유지한다.
- **원인**: `captureInput: true` 설정 시 `CapacitorWebView`가 `onCreateInputConnection()`에서 단순 키 이벤트용 더미 `BaseInputConnection(this, false)`를 반환한다. 이로 인해 삼성 키보드·Gboard가 조합형 한글(`setComposingText`, `commitText`)을 지원하지 않는 뷰로 오판하여 한/영 전환을 차단하고 영문 ASCII로 고정한다.

---

## 2. 리치 텍스트 에디터(TipTap/ProseMirror) 가상 키보드 마크다운 IME 우회

### 2.1 문제 구조
- 안드로이드 가상 키보드는 예측 입력과 조합 버퍼 관리를 위해 `# `, `- `, `**` 같은 기호 입력 중에도 `view.composing === true` 세션을 유지한다.
- ProseMirror 코어의 `handleTextInput`은 `if (view.composing) return false;`로 `InputRules`를 억제하고, `compositionend` 이후에도 `InputRules`를 재평가하지 않아 마크다운이 플레인 텍스트로 남는다.

### 2.2 플랫폼 격리 및 안전한 변환 규칙
1. **데스크톱 PC 물리 키보드 100% 바이패스 (`isTouchDevice()` 격리)**:
   - 데스크톱(Windows/macOS/Linux) 환경에서는 브라우저 순정 IME 파이프라인이 정상 작동하므로 커스텀 IME 보정 훅을 절대 실행하지 않는다. 개입 시 한글 끝 글자 중복(`노트` → `노트트`) 회귀가 발생한다.
2. **한글 음절 종결 무변조 가드**:
   - 태블릿 환경에서도 커서 직전 텍스트가 한글 음절/자모(`/[ㄱ-ㅎㅏ-ㅣ가-힣]$/`)로 끝나면 어떠한 트랜잭션도 발생시키지 않고 즉시 종료한다.
3. **`requestAnimationFrame` 지연 평가**:
   - `compositionend`, `beforeinput`, `keydown`(Space `32`, Enter `13`, IME `229`) 직후 DOM 동기화가 완료된 다음 프레임(`requestAnimationFrame`)에서 `state.tr` 트랜잭션으로 헤딩(`# `~`###### `), 리스트(`- `, `1. `), 인용구(`> `), 인라인 마크(`**bold**`, `*italic*`, `~~strike~~`, `` `code` ``)를 원자적으로 변환한다.
4. **모바일 마크다운 보조 툴바 포커스 보호**:
   - 가상 키보드에서 `#`, `*`, `~`, `$` 입력 피로를 줄이는 보조 툴바 버튼에는 반드시 `onPointerDown={(e) => e.preventDefault()}`를 적용해 터치 시 에디터 포커스와 선택 영역이 풀리지 않도록 한다.
5. **CSS 서식 보장**:
   - 모바일 웹뷰 CSS 리셋 환경에서도 마크다운 서식이 플레인 텍스트로 보이지 않도록 `.ProseMirror` 셀렉터에 `h1~h6`, `strong`, `em`, `s`, `blockquote`, `pre`, `code`, `a`, `hr` 스타일을 명시한다.

---

## 3. 한국어 인코딩(`UTF-8` / `CP949`) 및 커스텀 폰트 처리

### 3.1 파일 인코딩 무결성
- 앱 내부 저장 및 마크다운 직렬화는 `UTF-8`을 기본으로 하되, 학교·공공기관에서 배포한 `.csv`·`.txt`·`.md` 외부 파일 로드 시 `UTF-8`, `UTF-8 BOM`, `CP949(EUC-KR)`를 판별하는 디코더(`TextDecoder('utf-8', { fatal: true })` 실패 시 `TextDecoder('euc-kr')` 폴백)를 사용한다.
- 브라우저/클라이언트 번들에서 `iconv-lite` 등이 Node.js `stream.Transform`을 참조해 경고가 발생하지 않도록 경량 브라우저 스텁(`emptyStreamStub`) alias를 구성하거나 Web Standard `TextDecoder` API를 우선 사용한다.

### 3.2 로컬 커스텀 폰트(`.ttf`, `.otf`, `.woff2`) 영속화
- 사용자가 로컬에서 추가한 한글 폰트 파일은 용량 상한(예: `30MB`) 및 확장자 검증 후 `IndexedDB`에 저장하고 `@font-face` / `FontFace` API로 동적 등록하여 오프라인·재시작 시에도 유실 없이 복원한다.
- 에디터 기본 폰트 크기는 CSS 변수(`--note-font-size`)로 관리하고 8~72px 범위로 클램핑한다.

---

## 4. 한국어 조사 제거 및 하이브리드 토크나이징 (오프라인 검색)

- 한국어는 체언 뒤에 격조사·보조사가 결합하므로(`"직육면체를"`, `"직육면체에서"`, `"직육면체에서는"`), 단순 공백 분리로는 검색이 실패한다.
- **2단계 복합 조사 제거(`stripKoreanJosa`) + 2-gram 보조 토큰**:
  1. 긴 복합 조사(`에서는`, `에서도`, `으로는`, `에게서`, `로부터` 등)를 먼저 제거한 뒤 단일 조사(`은`, `는`, `이`, `가`, `을`, `를`, `의`, `에`, `에서`, `로`, `으로`, `와`, `과` 등)를 제거해 어간을 추출한다. (어간 최소 길이 2음절 이상 보호)
  2. 3음절 이상의 한글 단어는 2음절 Bi-gram 보조 토큰을 함께 색인하여 부분 어절·복합어 매칭 정확도를 확보한다.

