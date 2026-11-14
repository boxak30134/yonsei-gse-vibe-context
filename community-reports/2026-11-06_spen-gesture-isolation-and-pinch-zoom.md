---
title: S-Pen 필기 직후 터치 스크롤 잠김, 핀치 줌 (0,0) 점프 및 제스처 격리 해결
major: 교육공학 / 수학교육
tags: [s-pen, stylus, touch-action, pinch-zoom, focal-point, palm-rejection, gesture-isolation]
date: 2026-11-06
---

# S-Pen 필기 직후 터치 스크롤 잠김, 핀치 줌 (0,0) 점프 및 제스처 격리 해결

## [문제 상황]
- 태블릿에서 S-Pen으로 필기한 직후 손가락으로 화면을 밀어 올릴 때 스크롤이 잠겨 움직이지 않았고, 두 손가락 핀치 줌 시 손가락 중심이 아닌 화면 최상단 `(0, 0)`으로 튕기며 확대되는 문제가 발생했다.

## [원인 분석]
1. **필기 후 `touchAction: 'none'` 미해제 및 과도한 잠금**: 호버링 중에도 `touchAction = 'none'`을 강제하고 필기 종료 시 초기화가 조건부로 누락되어 브라우저 터치 스크롤 판정이 차단되었다.
2. **핀치 줌 시 스크롤 오프셋 미역산 & 컨테이너 크기 클램핑**: 배율만 키우고 손가락 중심점의 문서 좌표를 역산해 `scrollTop`/`scrollLeft`를 보정하지 않았으며, 줌 컨테이너 CSS 크기가 먼저 커지지 않아 브라우저가 이전 `scrollHeight` 한계로 스크롤을 `0`에 클램핑했다.

## [해결 패턴]
1. 핀치 시작 시 손가락 중심의 unscaled 문서 좌표를 고정하는 `FocalZoomAnchor`를 도입하고, 줌 변경 시 DOM 크기를 선제 갱신한 뒤 `computeFocalZoomScrollOffsets`로 `scrollTop`/`scrollLeft`를 실시간 보정했다.
2. 필기 종료 시 `touchAction` 및 팜 리젝션 상태를 즉시 정리해 스크롤 전환 지연을 제거했다.

## [에이전트 방어 규칙 제안]
- 줌/스크롤 컨테이너 구현 시 배율 변경과 동시에 크기 래퍼 DOM 치수를 먼저 갱신하고 `FocalZoomAnchor` 역산으로 시점을 고정한다. (참조: `patterns/stylus-dual-canvas.md`)

---

### [추가 사례] 2026-11-07 (교육공학 / 공통)
- **변형 상황**: `pointerdown` 시 JS에서 `touchAction = 'none'`을 동적으로 지정해도 첫 펜 획이 네이티브 스크롤로 새어 화면이 끌리고, 필기 중 손바닥이 닿으면 `펜 + 손바닥`이 2손가락 핀치 줌으로 오인되었다.
- **원인**: 브라우저는 터치/펜이 닿는 순간의 CSS `touch-action` 값으로 네이티브 팬 여부를 즉시 결정하므로 `pointerdown` JS 변경은 이미 한 발 늦다. 또한 `TouchEvent`에 S-Pen의 `touchType: 'stylus'`가 함께 포함되어 있었다.
- **해결**:
  - 뷰포트 CSS에 `touch-action: none; overscroll-behavior: none;`을 사전 선언하고, 1손가락 터치 패닝은 JS 관성 스크롤(감쇠 `0.92/16ms`)로 대체했다.
  - `TouchEvent` 핀치 판정 시 `touch.touchType === 'stylus'`를 필터링해 제외했다.

### [추가 사례] 2026-11-08 (수학교육 / 교육공학)
- **변형 상황**: (1) 핀치 줌 도중 화면이 심하게 버벅였고, (2) S-Pen으로 용지 바깥 회색 여백·사이드바·툴바를 누를 때 클릭이 차단되었으며, (3) 좌우 분할 화면에서 한쪽 패널에 필기할 때 반대편 패널이 손바닥 터치로 끌렸다.
- **원인**: 핀치 `touchmove`마다 Zustand `setZoom`을 호출해 초당 60~120회 React 리렌더링이 발생했고, 뷰포트 전역 포인터 캡처로 용지 바깥 펜 터치까지 가로챘으며, 450ms 팜 리젝션이 단일 패널 로컬 상태로만 격리되어 있었다.
- **해결**:
  - 핀치 줌 중에는 `setZoom`을 호출하지 않고 `sizer.style`과 `page.style.transform`만 Direct DOM Transform으로 갱신한 뒤 `touchend`/`pointerup` 시 1회 커밋해 120fps를 달성했다.
  - `.page-canvas-surface` 바운딩 박스 히트 테스트를 추가해 용지 바깥 S-Pen 터치는 100% 바이패스했다.
  - 전역 `globalPalmRejectionState`를 도입해 좌우 듀얼 스플릿 패널 간에도 `PALM_REJECTION_WINDOW_MS = 450` 교차 팜 리젝션을 적용했다.

