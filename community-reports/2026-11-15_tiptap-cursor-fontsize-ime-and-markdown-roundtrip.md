# TipTap/ProseMirror 커서 위치 인라인 글자 크기(`storedMarks`), 한글 IME 지연 적용 및 `@tiptap/markdown` `<span>` 왕복 패턴

- 작성일: 2026-11-15
- 범주: Rich Text Editor (TipTap v3 / ProseMirror) · `@tiptap/markdown` · Korean IME

---

## 1. 문제 현상 (Symptoms)

1. **드래그 선택 없이 커서 위치에서 글자 크기 변경 시 미적용**: 텍스트를 드래그하지 않고 커서만 둔 상태에서 툴바 글자 크기 프리셋을 누르면 직후 타이핑하는 글자에 크기가 적용되지 않거나, 초기화(`보통`)를 눌러도 기존 `<span>` 스타일이 계속 따라옴.
2. **마크다운 저장·재열기 시 인라인 크기 유실**: 에디터에서는 크게 보이지만 Markdown으로 직렬화(`getMarkdown()` / `jsonToMarkdown()`)하면 `<span style="font-size: ...">` 태그가 사라져 일반 텍스트로 저장됨.
3. **한글 입력기(IME) 조합 중 툴바 조작 시 음절 깨짐/포커스 이탈**: 한글 조합 중 툴바 버튼을 누르면 포커스가 툴바로 넘어가거나 ProseMirror 트랜잭션이 즉시 실행되어 조합 중인 음절이 끊기거나 중복 입력됨.
4. **전역 기본 폰트 크기와 커서 인라인 크기의 충돌**: 툴바에서 커서 크기를 바꿀 때 문서 전체 기본 폰트 크기 상태(`fontSize`)까지 함께 덮어써서, 기존에 스타일 없이 작성된 다른 문단까지 한꺼번에 크기가 변함.

---

## 2. 근본 원인 (Root Cause)

1. **`Mark.renderMarkdown` 기본 구현의 스타일 무시**: 커스텀 `Mark`에서 `renderMarkdown: (node, h) => h.renderChildren(node)`만 두면 태그 래퍼 없이 자식 텍스트만 출력되어 `font-size` 속성이 직렬화되지 않는다. 또한 `@tiptap/markdown` 역직렬화 시 `<span style="font-size: ...">` 인라인 HTML을 `Mark`로 복원할 토크나이저(`markdownTokenizer` / `parseMarkdown`)가 없으면 왕복 파싱이 끊긴다.
2. **빈 선택 영역(`selection.empty`)에 대한 `storedMarks` 미사용**: `setMark`만 호출하거나 특정 도구 모드에서만 분기하면 커서 상태의 `storedMarks`가 일관되게 갱신되지 않는다. 특히 초기화 시에는 `storedMarks`에서 해당 마크를 명시적으로 제거(`markType.removeFromSet(baseMarks)`)해야 이후 입력부터 기본 크기로 분리된다.
3. **IME 조합 중 즉시 트랜잭션 및 포커스 강탈**: `view.composing` 또는 `compositionstart`~`compositionend` 구간에서 `.focus()`와 `dispatch(tr)`를 바로 실행하면 브라우저 IME 조합 버퍼가 강제 확정/손상된다.
4. **HardBreak(`\n`) 포함 시 블록 경계 손상**: `<span style="font-size: ...">줄1\n줄2</span>`처럼 줄바꿈을 감싼 채 직렬화하면 마크다운 파서에 따라 블록 경계가 깨지거나 닫히지 않은 인라인 HTML로 취급될 수 있다.

---

## 3. 해결 패턴 (Solution Pattern)

### 3.1 `storedMarks` 기반 커서 즉시 적용 및 초기화
- 선택 영역이 있을 때(`!empty`)는 `tr.addMark` / `tr.removeMark`를 사용하고, 커서만 있을 때(`empty`)는 `baseMarks = state.storedMarks ?? $from.marks()`를 기준으로 `tr.setStoredMarks(...)`를 호출한다.
- **크기 적용**: `fontSizeType.create({ size: sizeCss }).addToSet(baseMarks)`
- **기본 크기 복귀(초기화)**: `fontSizeType.removeFromSet(baseMarks)` (이후 입력 문자부터 즉시 해당 마크가 제외됨)

### 3.2 `@tiptap/markdown` 왕복 직렬화/파싱 및 줄바꿈 안전 처리
- `renderMarkdown`에서 내용이 비어 있으면(`!content`) 빈 `<span ...></span>` 유령 태그를 출력하지 않고 즉시 `''`를 반환한다.
- `content`에 `\n`(`HardBreak` 등)이 포함된 경우 줄 단위로 분리해 각 비어 있지 않은 세그먼트마다 `<span style="font-size: ...;">...</span>`를 닫고 열어 마크다운 블록 구조를 보호한다.
- `markdownTokenizer`(inline level)와 `parseMarkdown`을 함께 정의해 `<span style="font-size: ...">...</span>` 내부의 중첩 마크다운(`**굵게**` 등)까지 `lexer.inlineTokens` + `h.applyMark`로 손실 없이 복원한다.

### 3.3 한글 IME 조합 중 지연 큐잉(`compositionend` 플러시)
- ProseMirror `Plugin`으로 `compositionstart`/`compositionend` 상태와 대기 중인 크기 변경 값(`pendingSize: string | null | undefined`)을 관리한다.
- `view.composing` 또는 플러그인 `composing` 상태일 때는 즉시 `storedMarks`를 수정하거나 `.focus()`를 호출하지 않고 메타데이터 트랜잭션으로 `pendingSize`만 적재한 뒤, `compositionend` 직후 `setStoredMarks` 트랜잭션을 한 번에 반영한다.
- 툴바 버튼(`-`, `+`, 프리셋, 초기화)에는 `onMouseDown={(e) => e.preventDefault()}`를 적용해 클릭 시 에디터 커서와 IME 상태가 이탈하지 않게 한다(직접 숫자 입력 `<input>`은 제외).

### 3.4 툴바 실시간 커서 상태 동기화
- 툴바가 에디터 인스턴스의 `selectionUpdate`, `transaction`, `focus` 이벤트를 구독해 `state.storedMarks ?? $from.marks()`에서 현재 커서의 인라인 크기를 읽어 하이라이트한다.
- 에디터 인스턴스가 활성화된 상태에서는 전역 기본 폰트 설정을 덮어쓰지 않고 커서/선택 범위의 인라인 마크만 변경하여 문서 내 다른 일반 문단 크기를 보존한다.

---

## 4. 주의점 (Caveats)

- **목(Mock) 기반 UI 테스트 호환**: 단위 테스트에서 에디터 인스턴스 없이 세션 목만 주입하는 경우가 있으므로 `session?.editor?.on(...)`처럼 옵셔널 체이닝 가드를 반드시 둔다.
- **중첩 마크 직렬화 순서**: `**<span style="font-size: 13px;">텍스트</span>**`처럼 마크 우선순위에 따라 `**`가 `<span>` 바깥에 감싸질 수 있으므로, 테스트 단언 시 태그와 마크의 중첩 순서를 과도하게 고정하지 않는 것이 안전하다.
- **`tr.setStoredMarks([])`의 의미**: ProseMirror에서 `storedMarks`를 `[]`로 설정하는 것은 `null`과 다르며, 직전 문자의 마크 상속을 끊고 "지금 커서부터는 마크 없음"을 명시하는 데 필수적이다.
