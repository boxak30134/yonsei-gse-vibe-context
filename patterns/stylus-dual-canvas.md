---
title: S-Pen·태블릿 3계층 저지연 필기 캔버스 및 제스처 격리 패턴
tags: [stylus, s-pen, canvas, desynchronized, palm-rejection, pinch-zoom, lasso, chunking]
updated: 2026-11-13
---

# S-Pen·태블릿 저지연 필기 캔버스 및 제스처 격리 패턴 (`stylus-dual-canvas.md`)

태블릿 S-Pen/스타일러스 필기 앱을 순수 웹뷰(또는 Capacitor 하이브리드) 환경에서 구현할 때 발생하는 **획 증발, 화면 깜빡임, 블러(해상도 저하), 지우개 모양 변형, 스크롤/핀치 줌 충돌**을 해결하는 표준 아키텍처 패턴입니다.

---

## 1. 3계층 캔버스 분리 아키텍처 (3-Layer Canvas)

| 레이어 | 역할 | `desynchronized` | `z-index` | 렌더링 방식 |
| :--- | :--- | :--- | :--- | :--- |
| **Layer 1: Committed Canvas** (`CHUNK_HEIGHT = 1600px` 타일 분할) | 확정된 누적 필기 보관 | `false` (표준 2D) | `30` | `pointerup` 시 1:1 `drawImage` 베이킹. 지우개/이동/Undo 시에만 전체 재렌더 |
| **Layer 2: Active Canvas** | 현재 긋고 있는 S-Pen 1획 전용 | `true` (저지연) | `31` | `pointermove` 중 `clearRect` 없이 증분 세그먼트만 추가 |
| **Layer 3: Interaction Overlay** | 올가미 점선, 객체 드래그 프리뷰, 지우개 커서 | `false` (표준 2D) | `32` | 매 프레임 `clearRect` + 전면 재드로우(깜빡임 0%) |

### 핵심 함정 4가지와 해결책
1. **Chromium `desynchronized: true` + CSS `mix-blend-mode: multiply` 충돌**:
   - `desynchronized: true` 캔버스는 하드웨어 오버레이 플레인에 직접 출력되므로 CSS `mix-blend-mode`를 지원하지 않아 화면 전체가 투명(블랭크) 처리된다.
   - **규칙**: `mix-blend-mode`를 적용할 Dry/Committed 캔버스는 반드시 표준 `canvas.getContext('2d')`로 생성하고, Active 캔버스만 `{ desynchronized: true }`를 사용한다.
2. **`desynchronized: true`에서 `clearRect` 반복 시 깜빡임/소실**:
   - 올가미 점선처럼 매 프레임 `clearRect()` 후 다시 그려야 하는 UI 피드백은 `desynchronized: true`에서 스캔아웃 타이밍 충돌로 보이지 않거나 심하게 깜빡인다.
   - **규칙**: 손글씨 증분 필기는 Layer 2(`desynchronized: true`), 매 프레임 `clearRect`가 필요한 올가미/이동 프리뷰는 Layer 3(`desynchronized: false`)로 물리 분리한다.
3. **25,000px 다중 페이지에서 GPU 텍스처 한계(8192px)로 인한 블러**:
   - 단일 거대 캔버스 높이가 GPU 텍스처 상한(8192px)을 넘겨 `dpr`을 `8192 / 25000 = 0.32`로 다운스케일하면 펜을 떼는 순간 글씨가 모자이크처럼 뭉개진다.
   - **규칙**: 세로 `CHUNK_HEIGHT = 1600px` 단위의 타일 캔버스로 분할해 디바이스 `window.devicePixelRatio`(2.0~3.0)를 100% 무손실 유지한다.
4. **React 리렌더 및 `canvas.width/height` 재할당으로 인한 획 증발**:
   - `canvas.width` 또는 `.height`에 값을 대입하면 GPU 텍스처가 새로 할당되며 픽셀 버퍼가 즉시 초기화된다.
   - **규칙**:
     - `InkCanvas` 최상단에서 `useSyncExternalStore`를 쓰지 말고 `useEffect` 구독으로 분리하며, 필기 루프(`pointerdown`/`move`/`up`) 중 React `setState` 호출을 금지한다.
     - 청크별 ref 콜백을 캐싱하고 `WeakSet`으로 초기 마운트 1회만 크기를 설정한다(실제 픽셀 크기가 바뀔 때만 리사이즈).
     - `pointerup` 시 `chunkCtx.setTransform(1, 0, 0, 1, 0, 0)` 상태에서 `drawImage(wetCanvas, ...)`로 1:1 디바이스 픽셀 전사(Bake)를 완료한 다음 프레임(`rAF` 또는 다음 `pointerdown`)에 Active Canvas를 비운다.

---

## 2. 실시간/재렌더 알고리즘 단일화 & 240Hz 입력

1. **실시간 드로잉 vs 전체 재렌더 알고리즘 일치**:
   - 실시간(`drawIncrementalStrokeSegment`)은 중점 2차 베지어 곡선을 쓰고, 전체 재렌더(`drawStroke`)는 `perfect-freehand` 다각형을 쓰면 지우개가 닿아 재렌더되는 순간 모든 손글씨 모양이 바뀐다.
   - **규칙**: 실시간 증분 드로잉과 `drawStroke` 재렌더링이 100% 동일한 중점 2차 베지어 + Hermite S-커브(`t^2(3 - 2t)`) 알고리즘을 공유하게 한다. 실시간 중에는 끝단 조기 테이퍼링을 끄고(`last: false, end.taper: 0`), `pointerup` 시 끝단 마감을 적용한다.
2. **240Hz 디지타이저 샘플 복원 & `pointercancel` 보존**:
   - `e.getCoalescedEvents()`(React SyntheticEvent 및 nativeEvent 모두 지원)로 숨겨진 하드웨어 샘플을 전수 수집하고, `0.35px` 미만 초미세 지터를 필터링한다.
   - `pointercancel` 발생 시에도 유효한 점(`points.length > 0`)이 있으면 획을 폐기하지 않고 커밋한다.
   - 지우개 `pointerdown` 시 실제 지워진 획이 없으면 Committed Canvas 전체 재렌더(`clearRect`)를 호출하지 않는다.

---

## 3. S-Pen 제스처 격리 & 450ms Palm Rejection

1. **CSS 사전 선언 (`touch-action: none`)**:
   - 브라우저는 터치 시작 시점의 CSS `touch-action`으로 네이티브 스크롤 여부를 즉시 결정한다. `pointerdown` JS 핸들러에서 동적으로 `touchAction = 'none'`을 주면 첫 펜 획이 스크롤로 새어나간다.
   - **규칙**: 뷰포트 CSS에 `touch-action: none; overscroll-behavior: none;`을 기본 선언하고, 1손가락 패닝은 JS 관성 스크롤(감쇠 `0.92/16ms`)로 처리한다.
2. **포인터 타입별 엄격 라우팅 & 용지 외부 바이패스**:
   - `pointerType === 'pen'` 또는 `'eraser'`(또는 배럴 버튼 `buttons === 32`):
     - 터치 좌표가 실제 필기 영역(`.page-canvas-surface`) 내부일 때만 `preventDefault()`, `stopPropagation()`, `setPointerCapture()`를 호출한다.
     - 용지 바깥(회색 여백, 사이드바, 툴바) S-Pen 터치는 가로채지 않고 일반 UI 클릭으로 통과시킨다.
   - `TouchEvent` 핀치 줌 계산 시 `touch.touchType === 'stylus'` 터치를 반드시 제외하여 `펜 + 손바닥`이 2손가락 핀치 줌으로 오인되는 현상을 막는다.
3. **450ms Palm Rejection (`PALM_REJECTION_WINDOW_MS = 450`)**:
   - 펜 드로잉 중 및 펜 떼기 직후 `450ms` 내 손바닥(`touch`) 이벤트를 차단한다(좌우 듀얼 스플릿 패널 환경에서는 전역 `globalPalmRejectionState`로 교차 패널 손바닥 터치까지 차단).
   - **주의**: 펜을 뗀 후 의도적으로 즉시 손가락 스크롤을 재개하려면, 정상 `pointerup`으로 필기가 완전히 끝났을 때 `lastPenTimestamp` 처리 및 `touchAction` 복원을 조율하여 450ms 잠김 불만을 방지한다.
4. **120fps 핀치 줌 (Direct DOM Transform + Focal Point)**:
   - 핀치 줌 중 매 `touchmove`마다 React/Zustand `setZoom`을 호출하면 가상 DOM 재조정으로 심각한 버벅임(Jank)이 발생한다.
   - **규칙**: 제스처 중에는 `sizer.style`과 `page.style.transform`만 DOM에 직접 갱신하고 `FocalZoomAnchor`로 손가락 중심점의 문서 좌표를 유지한 뒤, `touchend`/`pointerup` 시점에만 `setZoom`을 1회 커밋한다.

---

## 4. 올가미(Lasso) 선택·드래그·클립보드 통합 패턴

1. **Layer 3 + 4단계 드래그 파이프라인**:
   - **Stage 1 (Start)**: Committed Canvas에서 선택된 획만 `draw(new Set(ids))`로 1회 마스킹하고 `is-lasso-dragging` 클래스로 액션바를 페이드아웃(`opacity: 0; pointer-events: none`).
   - **Stage 2 (Live Preview)**: 드래그 중 `StrokeStore` 좌표를 절대 수정하지 않는다(수정 시 구독 중인 툴바가 매 프레임 리렌더되며 툴바만 따라다님). Layer 3에만 `ctx.translate(dx, dy)`로 실시간 프리뷰 렌더링.
   - **Stage 3 (Commit)**: `pointerup` 시 `moveStrokes(ids, dx, dy)`를 1회 호출해 원자적 Undo 트랜잭션과 Committed Canvas 베이킹을 수행한다.
   - **Stage 4 (Cancel)**: `pointercancel` 시 Layer 3 클리어 후 Committed Canvas 원위치 복구.
2. **스크롤 동기화 & 열린 궤적 자동 폐합**:
   - `<InkCanvas />`는 `pageEl` 직속 자식(`position: absolute; inset: 0`)으로 배치해 브라우저 컴포지터가 용지와 손글씨를 0ms로 함께 스크롤하게 한다(`scroll` 리스너로 JS 재렌더 금지).
   - `pointerup` 시 시작-끝 거리가 `36px`를 넘어도 유효 궤적이면 자동 폐합(`[...poly, poly[0]]`)해 C/U자 올가미도 선택되게 한다.
   - 액션바는 `FloatingPortal`로 `document.body`(`z-50`)에 마운트하고 `framer-motion` `scale`과 `@floating-ui` `transform` 충돌을 피해 좌표를 동기 계산한다.

---

## 5. 안드로이드 네이티브 스타일러스 오버레이 교훈

- WebView 위에 네이티브 `View` 오버레이를 띄우는 방식은 Window 좌표계와 WebView 내부 DOM(`getBoundingClientRect`, `zoom`, `scrollTop`) 간 비동기 오차, 물리 픽셀 `density` 미보정, `ACTION_UP` 클리어 레이스 컨디션으로 인해 획 소실과 위치 이탈을 유발하기 쉽다.
- 순수 웹 기반 **3계층 고해상도 청크 캔버스**가 좌표 일치성과 유지보수성에서 훨씬 안정적이다. 네이티브 오버레이를 불가피하게 쓸 경우 `DisplayMetrics.density` 변환, `setExclusionRects` UI 바이패스, 용지 경계 외부 `return false` 투과를 필수로 구현한다.

