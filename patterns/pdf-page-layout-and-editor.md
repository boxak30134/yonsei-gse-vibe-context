---
title: 리치 에디터(TipTap) + PDF 오버레이 레이아웃 및 글로벌 Undo/Redo 패턴
tags: [tiptap, prosemirror, pdf, layout, scroll-spy, text-layer, undo-redo, atomic-transaction]
updated: 2026-11-13
---

# 리치 에디터(TipTap) + PDF 오버레이 레이아웃 및 글로벌 Undo/Redo 패턴 (`pdf-page-layout-and-editor.md`)

ProseMirror/TipTap 기반 마크다운 에디터 위에 다중 PDF 페이지 템플릿, 페이지 앵커 기반 독립 텍스트 레이어, 말풍선 주석, 통합 Undo/Redo를 결합할 때 발생하는 **커서 상단 점프, 누적 오프셋 빗나감, 스크롤 스파이 경합, 부분 Undo 불일치**를 해결하는 아키텍처 패턴입니다.

---

## 1. PDF/멀티페이지 위 인라인 텍스트 입력 (커서 점프 3중 방어)

PDF `inkTemplate` 노드는 ProseMirror의 원자 노드(`atom: true`, `contentDOM` 없음)이므로, 페이지 위를 탭할 때 텍스트 커서가 1페이지 바깥 최상단으로 튀는 현상이 발생하기 쉽다. 아래 4가지가 모두 갖춰져야 해결된다.

1. **`InkTemplate` `selectable: false` 및 `stopEvent` 분기**:
   - `selectable: false`로 클릭 시 페이지 전체 `NodeSelection` 파란 테두리가 잡히는 것을 막는다.
   - `NodeView`의 `stopEvent(event)`에서 텍스트 입력 도구(`activeTool === 'none'`)일 때만 `pointerdown`/`mousedown`/`touchstart`/`click`에 `true`를 반환해 ProseMirror가 노드 직전에 자체 `TextSelection`을 생성하는 것을 차단한다(필기 도구일 때는 `false`를 반환해 240Hz S-Pen 경로 보존).
2. **Capture 단계 격리 & Markdown 본문 바이패스**:
   - 독립 텍스트 레이어(`TextObjectLayer`)는 `pointerdown`/`mousedown`을 `{ capture: true }` 단계에서 가로채되, **타깃이 일반 Markdown 본문(`.ProseMirror, p, h1~h6, blockquote, li, pre, code, table`)인 경우에는 즉시 바이패스**해야 태블릿에서 본문 텍스트 드래그/선택/링크 추천이 죽지 않는다.
   - 오직 PDF `.ink-template` 여백을 탭했을 때만 `preventDefault()`, `stopPropagation()` 후 투명 인라인 `<textarea>`(Direct-to-Note, 상자 테두리·그림자 없음)를 띄운다.
3. **1-based / 0-based 셀렉터 분리 & `{0, 0}` 폴백 금지**:
   - `querySelector('[data-page="..."], [data-page-index="..."]')`처럼 0-based와 1-based를 한 셀렉터에 쉼표로 섞으면 2페이지 검색 시 DOM 앞쪽의 1페이지가 먼저 매칭된다. 1-based(`data-page`, `#pdf-page-N`)와 0-based(`data-page-index`)를 분리 조회하고, 클릭 시 확정된 DOM 요소를 `WeakMap`에 캐싱한다.
   - 오프셋 계산 실패 시 `{ x: 0, y: 0 }`을 반환하면 객체가 1페이지 상단으로 날아가므로, `null`을 반환해 해당 틱 렌더링을 보류한다.
4. **`focus({ preventScroll: true })` & 주석 스크롤 튐 차단**:
   - 인라인 `<textarea>` 포커스 시 반드시 `{ preventScroll: true }`를 전달하고 `editor.commands.blur()`를 호출한다.
   - 말풍선 주석 생성 시에도 `chain().focus().setMark(...)`처럼 불필요한 `.focus()`를 호출하면 문서 끝으로 스크롤이 튀므로, 순수 드래그 텍스트(`quote.length > 0`)일 때만 마크를 적용하고 미선택 시 현재 뷰포트 중앙 좌표에 핀을 꽂는다.

---

## 2. 페이지 진실 소스 단일화 & DOM 실측 레이아웃 (`getMeasuredPageLayout`)

### 2.1 페이지 진실 소스(Single Source of Truth) + `pageId` 불변식
- 페이지 추가·삭제·재정렬은 `NoteSession` 단일 메서드에서 TipTap 단일 트랜잭션(`tr.insert`/`tr.delete` + `tr.setNodeMarkup`으로 `1..N` 재정렬)과 `ui.pages` 불변 새 배열 교체, `record.content`(`view.state.doc.toJSON()`) 동기화를 한 번에 처리한다.
- 모든 `InkTemplate` 노드에는 `pageId` 속성과 DOM `data-page-id`를 부여해 `NotePage.id === InkTemplate.attrs.pageId` 불변식을 유지하며, `extractInkTemplateDimensions`에서 PDF 노드가 1개 이상일 때 부족한 페이지를 가상 생성하지 않도록 차단한다.
- 가로/세로 혼합 규격 PDF에서는 첫 페이지 대표 폭(`pdfContainerWidth`)을 컨테이너 기준으로 고정해 페이지 전환 시 레이아웃 폭 출렁임을 방지한다.

### 2.2 DOM 실측 기반 오프셋 (`getMeasuredPageLayout`)
- 메타데이터(`width/height`) 기반 추정 함수(`computePdfPageLayout`)는 초기 `minHeight` 선확보용으로만 쓰고, 실제 스크롤 이동과 스크롤 스파이 판정에는 DOM 요소의 `offsetTop`과 `offsetHeight`를 실측하는 `getMeasuredPageLayout` + `ResizeObserver`(16ms 디바운스)를 사용한다.
- **이유**: 캔버스 border, 배지, 패딩 등 페이지당 6~10px의 오차가 20페이지에 누적되면 120~200px(1~2페이지) 빗나가 20쪽 클릭 시 18~19쪽으로 이동한다. (임의 보정 상수 `+ 8 * index` 땜질 금지)

### 2.3 내비게이션 의도와 스크롤 스파이 비경합 상태 기계 (`NavState`)
- `targetPageIndex: number | null`(사용자가 클릭으로 의도한 목표)와 `visiblePageIndex: number`(스파이 관찰값)를 분리하고, 외부 노출은 `targetPageIndex ?? visiblePageIndex`를 사용한다.
- `scrollTo({ top, behavior: 'smooth' })` 1회 실행 후 `programmatic === true` 동안 스파이는 `visible`만 갱신하고 하이라이트(`currentPageIndex`)를 덮어쓰지 않는다.
- `scrollend`, 150ms 정지, `|scrollTop - targetOffset| <= 2px`, 1500ms 타임아웃, 또는 사용자 직접 입력(`wheel`/`touchmove`/`pointerdown`) 시 즉시 settle하며, 뷰포트 중앙선(`scrollTop + viewportH / 2`) 기준으로 현재 페이지를 판정한다.

---

## 3. 통합 글로벌 Undo/Redo 원자적 트랜잭션 (`CompositeHistoryManager`)

- 필기 스트로크, 텍스트 객체, 에디터 본문, 말풍선 주석, 페이지 추가/삭제/복제/재정렬은 **단일 타임라인(`CompositeHistoryManager`)**에서 관리한다.
- **원자적 복합 트랜잭션**: 페이지 삭제처럼 여러 스토어(페이지 메타 + 에디터 노드 + 해당 페이지 스트로크 + 텍스트 객체 + 후속 페이지 스트로크 Y좌표 시프트)가 동시에 변하는 작업은, 하위 스토어의 히스토리 기록 없는 `raw*` 메서드(`rawRemove`, `rawTranslate`, `rawAdd`)를 호출하고 전체 스냅샷을 **단 1개의 `HistoryCommand`**로 푸시해 `Ctrl+Z` 1회로 완벽 복구되게 한다.
- **빈 입력 및 마운트 오염 방지**:
  - Direct-to-Note 인라인 텍스트 입력은 작성 완료 시점에만 `recordTextAdd`로 1회 기록하고, 빈 값으로 종료 시 `recordHistory: false`로 무음 삭제한다.
  - 문서 초기 마운트 완료 직후 `history.clear()`를 호출해 에디터 초기화 트랜잭션이 `canUndo: true`로 잘못 잡히는 것을 방지한다.

