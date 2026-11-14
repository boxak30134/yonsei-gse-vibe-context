---
title: 캔버스 필기 획 즉각 소실, 깜빡임, 지우개 모양 변형 및 다중 페이지 블러 해결
major: 교육공학 / 수학교육
tags: [canvas, s-pen, desynchronized, mix-blend-mode, flickering, blur, chunking, webview]
date: 2026-11-06
---

# 캔버스 필기 획 즉각 소실, 깜빡임, 지우개 모양 변형 및 다중 페이지 블러 해결

## [문제 상황]
- 태블릿 S-Pen 또는 마우스로 손글씨를 쓸 때 필기 도중에는 선이 보이다가 펜을 떼는 순간(`pointerup`/`pointercancel`) 방금 쓴 글씨가 화면에서 통째로 사라지는 치명적 결함이 발생했다.

## [원인 분석]
1. **Chromium `desynchronized: true` + CSS `mix-blend-mode: multiply` 충돌**: 영구 획을 보관하는 Dry 캔버스에 `{ desynchronized: true }`와 `mix-blend-mode: multiply`가 동시에 적용되었다. Chromium 사양상 `desynchronized: true` 캔버스는 하드웨어 오버레이 플레인으로 우회 출력되므로 CSS 블렌드 모드를 지원하지 않아 표면 전체가 투명(블랭크) 처리되었다.
2. **`pointercancel` 시 획 폐기 및 비동기 전환 공백**: 태블릿 OS 제스처 경계에서 `pointercancel`이 오면 유효한 점 데이터가 있어도 획을 버렸고, `pointerup` 시 rAF 비동기 드로우로 1프레임 공백이 발생했다.

## [해결 패턴]
1. Dry(Committed) 캔버스는 `desynchronized` 없는 표준 `getContext('2d')`로 분리하고, 실시간 획을 그리는 Wet(Active) 캔버스만 `{ desynchronized: true }`를 유지했다.
2. `pointercancel`에서도 `points.length > 0`이면 획을 영구 커밋하고, 뷰포트 밖 빠른 이동까지 수신하도록 `window.pointerup`/`pointercancel` 리스너를 보강했다.

## [에이전트 방어 규칙 제안]
- `mix-blend-mode`가 적용되는 캔버스에는 절대 `{ desynchronized: true }`를 설정하지 않으며, 실시간 Wet 캔버스와 영구 Dry 캔버스의 컨텍스트 생성 옵션을 엄격히 분리한다. (참조: `patterns/stylus-dual-canvas.md`)

---

### [추가 사례] 2026-11-07 (수학교육 / 교육공학)
- **변형 상황**: 연속 필기 시 글자가 깜빡이고, 지우개가 페이지에 닿기만 해도 페이지 내 모든 손글씨 굵기와 모양이 갑자기 변하는 현상이 발생했다.
- **원인**: `pointermove` 중 `desynchronized: true` 캔버스에 매 프레임 `clearRect`를 호출해 스캔아웃 티어링이 발생했고, 실시간 드로잉(중점 2차 베지어)과 지우개 진입 시 전체 재렌더(`perfect-freehand` 다각형 외곽선)의 렌더링 알고리즘이 서로 달랐다. 또한 매 획마다 동일한 `canUndo` 값으로 스토어가 패치되어 워크스페이스가 리렌더되었다.
- **해결**:
  - `pointermove` 중 `clearRect`를 0회로 제거하고 새로 유입된 `getCoalescedEvents()` 구간만 증분 베지어 곡선으로 덧그렸다.
  - 실시간 증분 드로잉과 전체 재렌더링(`drawStroke`)을 동일한 중점 2차 베지어 + Hermite S-커브 알고리즘으로 단일화했고, 지우개 접촉 시 실제 삭제된 획이 없으면 전체 재렌더를 호출하지 않게 했다.
  - 세션 스토어 `patch`에 얕은 동등성 가드를 추가해 연속 필기 중 React 리렌더링을 0회로 차단했다.

### [추가 사례] 2026-11-08 (수학교육 / 교육공학)
- **변형 상황**: (1) 21페이지 이상(높이 25,000px) 문서에서 펜을 떼자마자 획이 모자이크처럼 심하게 흐려졌고, (2) 1페이지에서도 `pointerup` 직후 획이 증발했으며, (3) 안드로이드 네이티브 오버레이를 도입하자 상단바/여백 엉뚱한 위치에 그려지고 획이 소실되었다.
- **원인**:
  1. 단일 캔버스가 GPU 텍스처 한계(8192px)를 넘어 `dpr = 8192 / 25000 = 0.327`로 다운스케일되었다.
  2. `finish()`에서 `dryCanvas.height = wetCanvas.height` 대입으로 캔버스 픽셀 버퍼가 초기화되었고, `setTransform(1,0,0,1,0,0)` 누락으로 `drawImage` 시 DPR이 중복 곱해졌으며, 최상단 `useSyncExternalStore`와 인라인 ref 콜백(`ref={(el) => ...}`) 때문에 `addStroke` 리렌더마다 캔버스가 언마운트/재할당되며 버퍼가 지워졌다.
  3. 네이티브 오버레이의 Window 좌표계와 WebView DOM 좌표계 간 비동기 오차 및 `ACTION_UP` 클리어 레이스 컨디션이 발생했다.
- **해결**:
  - 세로 `CHUNK_HEIGHT = 1600px` 타일 캔버스로 분할해 DPR 2.0~3.0 고해상도를 무손실 보존했다.
  - `InkCanvas`의 `useSyncExternalStore`를 `useEffect` 구독으로 대체하고, `WeakSet` 초기화 가드 + 캐시된 ref 콜백으로 리렌더 시 `canvas.width/height` 재할당을 원천 차단했다. `pointerup` 시 단위 변환 행렬(`setTransform(1,0,0,1,0,0)`)로 1:1 `drawImage` 베이킹 후 다음 프레임에 Active Canvas를 비우도록 순서 확정.
  - 불안정한 네이티브 오버레이(`overlayEnabled = false`)를 비활성화하고 순수 웹 고해상도 3계층 청크 캔버스로 단일화했다.

