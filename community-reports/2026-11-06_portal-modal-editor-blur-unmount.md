---
title: 텍스트 선택 플로팅 툴바 내 입력 모달 포커스 시 에디터 blur로 인한 모달 증발 해결
major: 외국어교육 / 공통
tags: [portal, modal, selection-menu, blur, focus, tiptap, tablet]
date: 2026-11-06
---

# 텍스트 선택 플로팅 툴바 내 입력 모달 포커스 시 에디터 blur로 인한 모달 증발 해결

## [문제 상황]
- 본문 텍스트를 드래그해 띄운 플로팅 선택 툴바(`SelectionMenu`)에서 API 키 등록 모달을 열었을 때, 모달이 나타난 지 약 `0.1~0.5초` 만에 저절로 사라져 입력할 수 없는 현상이 발생했다. 또한 태블릿에서 플로팅 툴바 버튼을 누르면 텍스트 선택 영역이 해제되어 링크 추천·번역 삽입이 실패했다.

## [원인 분석]
1. 모달의 `<input autoFocus>`로 브라우저 포커스가 이동하는 순간 리치 에디터에서 `blur` 이벤트가 발화한다.
2. 에디터 `blur` 리스너(120ms 디바운스)가 `!ed.isFocused`를 감지해 선택 메뉴 상태를 `null`로 초기화하면서, 자식 컴포넌트로 마운트되어 있던 모달까지 통째로 언마운트되었다.
3. 태블릿 터치 시에도 버튼에 `pointerdown` 기본 동작 방지가 없어 터치 즉시 에디터 포커스와 `Selection`이 풀렸다.

## [해결 패턴]
1. 입력 모달은 플로팅 메뉴 내부가 아니라 `createPortal(..., document.body)`를 통해 최상위 레이어(`z-[120]`)에 분리 마운트한다.
2. `isModalOpenRef` 참조 플래그를 두어 모달이 열려 있는 동안에는 에디터 `blur` 이벤트가 발생해도 플로팅 메뉴 및 선택 범위(`selectionRange`)를 초기화하지 않고 보존한다.
3. 플로팅 툴바 액션 버튼에는 `onPointerDown={(e) => e.preventDefault()}`를 적용하고 유효 선택 시 `SavedSelection`을 별도 캐싱해, 버튼 터치로 포커스가 흔들려도 직전 선택 범위를 기반으로 주석·위키링크를 제자리에 삽입한다.

## [에이전트 방어 규칙 제안]
- 에디터 선택 플로팅 메뉴에서 파생되는 모든 모달/팝오버는 반드시 `createPortal(..., document.body)`로 분리하고, 플로팅 메뉴 버튼에는 `onPointerDown={(e) => e.preventDefault()}`와 `isModalOpenRef` 블러 방어 가드를 필수로 적용한다. (참조: `rules/tablet-webview-safety.md`)

