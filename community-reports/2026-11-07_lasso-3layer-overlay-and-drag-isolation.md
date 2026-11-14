---
title: 올가미 점선 소실, 드래그 시 툴바만 따라다니는 충돌 및 3계층 Interaction Overlay 해결
major: 교육공학 / 수학교육
tags: [lasso, canvas, interaction-overlay, desynchronized, floating-portal, drag-preview]
date: 2026-11-07
---

# 올가미 점선 소실, 드래그 시 툴바만 따라다니는 충돌 및 3계층 Interaction Overlay 해결

## [문제 상황]
- 올가미(Lasso) 도구로 손글씨를 감쌀 때 (1) 파란 점선 궤적이 전혀 보이지 않거나 깜빡이고, (2) 열린 C/U자 궤적에서 선택이 취소되며, (3) 선택된 획을 드래그하면 정작 손글씨는 멈춰 있고 상단 HTML 액션 바(`LassoActionBar`)만 펜을 따라다니며 화면 맨 위 `(0, 0)`으로 튕기는 문제가 연쇄 발생했다.

## [원인 분석]
1. **`desynchronized: true` 캔버스의 Full Clear-and-Redraw 아티팩트**: 올가미 점선을 손글씨용 Active Canvas(`desynchronized: true`)에 그리면서 매 프레임 `clearRect()`를 호출하자, 하드웨어 프론트 버퍼 스캔아웃 타이밍과 겹쳐 점선이 통째로 안 보이거나 깜빡였다. 형광펜 사용 직후 잔류 `opacity: 0.35`와 `mixBlendMode: multiply`가 남는 문제도 겹쳤다.
2. **레이어 계층 역전 & `transform` 충돌**: 액션 바가 `z-20`으로 캔버스(`z-30`) 뒤에 깔렸고, `framer-motion` `scale`이 `@floating-ui`의 `transform: translate(...)`를 덮어써 메뉴가 `(0, 0)`으로 튀었다.
3. **드래그 중 `StrokeStore` 변조**: `pointermove`마다 `previewTranslate`로 스토어 좌표를 수정해 `OverlayLayer`가 매 프레임 리렌더되며 툴바만 움직였고, 정작 캔버스는 스트로크 수 가드에 의해 정지해 있었다.
4. **열린 올가미 폐합 제한 & 스크롤 분리**: 시작-끝점 거리가 `36px` 초과 시 선택을 취소했고, `<InkCanvas />`가 `pageEl` 밖에 있어 스크롤 시 0.1~0.3초 고무줄 뒤처짐이 발생했다.

## [해결 패턴]
1. **Layer 3 Interaction Overlay (`desynchronized: false`, `z-32`) 신설**: 매 프레임 `clearRect`가 필요한 올가미 점선·객체 이동 프리뷰를 표준 더블 버퍼링 캔버스로 물리 분리했다.
2. **4단계 드래그 파이프라인**: 드래그 시작 시 Committed Canvas에서 선택 획만 1회 마스킹하고 `is-lasso-dragging`으로 툴바를 페이드아웃한 뒤, 드래그 중에는 `StrokeStore`를 일절 수정하지 않고 Layer 3에만 `(dx, dy)` 변위를 렌더링하다가 `pointerup` 시 단 1회 `moveStrokes`를 커밋한다.
3. **좌표 동기화 & 액션 바 고정**: `<InkCanvas />`를 `pageEl` 직속 자식으로 이동해 0ms 컴포지터 스크롤을 확보하고, 열린 올가미 궤적을 자동 폐합했으며, 액션 바는 `FloatingPortal(z-50)`으로 승격해 선택 영역 직상단 좌표를 동기 계산했다.

## [에이전트 방어 규칙 제안]
- `desynchronized: true` 캔버스에서는 매 프레임 `clearRect`를 수반하는 인터랙션 피드백(올가미·커서)을 절대 그리지 말고 `desynchronized: false` 오버레이 캔버스로 분리하며, 드래그 프리뷰 중에는 React가 구독하는 스토어를 직접 변조하지 않는다. (참조: `patterns/stylus-dual-canvas.md`)

