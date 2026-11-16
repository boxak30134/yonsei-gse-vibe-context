---
title: PDF 원자 노드 위 텍스트 입력 시 커서 최상단 점프 및 본문 텍스트 선택 차단 해결
major: 교육공학 / 공통
tags: [pdf, prosemirror, tiptap, cursor-jump, stopEvent, capture, focus, preventScroll]
date: 2026-11-08
---

# PDF 원자 노드 위 텍스트 입력 시 커서 최상단 점프 및 본문 텍스트 선택 차단 해결

## [문제 상황]
- PDF 문서 페이지를 클릭해 텍스트를 입력하려 할 때 커서가 클릭한 위치가 아닌 1페이지 바깥 최상단으로 튀고 뷰포트가 맨 위로 스크롤되는 현상이 발생했다.

## [원인 분석]
1. **ProseMirror `NodeSelection` / `TextSelection` 가로채기 (주원인)**: PDF 페이지 템플릿 노드가 `atom: true`, `contentDOM: null` 구조인데 `NodeView`에 `stopEvent`가 없고 텍스트 오버레이 레이어가 버블 단계에서만 이벤트를 처리해, 클릭 즉시 ProseMirror가 PDF 노드 직전(문서 최상단)에 자체 캐럿을 생성했다.
2. **1-based / 0-based 셀렉터 혼용 & `{0, 0}` 폴백**: `querySelector`에 `[data-page="..."]`와 `[data-page-index="..."]`를 쉼표로 결합해 2페이지 탐색 시 1페이지가 우선 매칭되었고, 탐색 실패 시 `{ x: 0, y: 0 }`으로 폴백했다.
3. **`focus({ preventScroll: true })` 누락**: 인라인 `<textarea>` 포커스 시 브라우저가 화면 스크롤을 점프시켰다.

## [해결 패턴]
1. PDF 페이지 노드에 `selectable: false`를 지정하고, `NodeView.stopEvent`에서 텍스트 입력 도구일 때만 포인터/마우스 이벤트에 `true`를 반환해 ProseMirror 개입을 차단했다(필기 도구 시 `false`로 S-Pen 경로 보존).
2. 텍스트 오버레이 레이어는 `capture` 단계에서 이벤트를 처리하되 1-based/0-based 셀렉터를 분리 조회하고, `WeakMap`에 클릭된 페이지 DOM을 캐싱하며, 오프셋 조회 실패 시 `{0,0}` 대신 `null`을 반환해 오배치를 보류한다.
3. `textarea.focus({ preventScroll: true })`와 `editor.commands.blur()`를 적용하고, 상자 테두리·배경 없는 투명 인라인 입력기로 구현했다.

## [에이전트 방어 규칙 제안]
- TipTap `atom: true` 노드 위에 커스텀 오버레이 입력을 구현할 때는 좌표 수식 단위 테스트만으로 안심하지 말고, `NodeView.stopEvent` + `capture` 단계 차단 + `document.activeElement`·`editor.state.selection` 불변성 통합 테스트를 반드시 작성한다. (참조: `patterns/pdf-page-layout-and-editor.md`)

---

### [추가 사례] 2026-11-10 (교육공학 / 공통)
- **변형 상황**: 텍스트 오버레이 레이어에 `capture` 단계 `preventDefault()`/`stopPropagation()`을 적용하자, 일반 Markdown 본문에서 태블릿 터치 드래그 및 롱프레스 텍스트 선택이 무력화되어 어휘 유사도 링크 추천 툴바가 열리지 않았다.
- **원인**: `capture` 단계 핸들러가 타깃 요소 구분 없이 페이지 내 모든 클릭을 가로챘기 때문이다.
- **해결**: 타깃이 `.ProseMirror, p, h1~h6, blockquote, ul, ol, li, pre, code, table` 등 본문 Markdown 영역이면 `capture` 핸들러를 즉시 반환(바이패스)하고, 오직 PDF 페이지 여백 탭 시에만 인라인 텍스트 입력이 작동하도록 격리했다.

### [추가 사례] 2026-11-12 (교육공학 / 공통)
- **변형 상황**: PDF 문서에서 우클릭 또는 메뉴로 말풍선 주석을 추가할 때 뷰포트가 문서 맨 끝으로 스크롤 점프했다.
- **원인**: PDF 템플릿 선택 잔재로 인해 주석 생성 내부에서 `chain().focus().setMark(...)`가 호출되어 에디터 끝 캐럿으로 스크롤되었다.
- **해결**: 순수 드래그 텍스트(`quote.length > 0`)일 때만 `.focus()` 없이 마크를 적용하고, 미선택 시에는 클릭 좌표 또는 현재 뷰포트 중앙 좌표에 핀을 생성하도록 수정했다.

