# S-Pen 저지연 필기 2-레이어 분리, 표준 컴포지터 가시성 보장 및 ±1 청크 가상화 패턴

- 작성일: 2026-11-21
- 범주: HTML5 2D Canvas · Stylus(S-Pen) Input · Chromium GPU Compositor · Viewport Virtualization

---

## 1. 문제 현상 (Symptoms)

1. **다량 스트로크 및 다중 페이지에서의 필기 지연·버벅임**: 획 수와 페이지 수(20페이지+)가 늘어날수록 펜 이동보다 선 렌더링이 늦어지고 지우기·Undo 시 프레임 드랍이 발생함.
2. **투명 `desynchronized: true` 단독 활성 레이어 전환 시 필기 중 화면 미노출 버그**: `pointermove` 중 정적 캔버스 동시 쓰기를 제거하고 `{ desynchronized: true, alpha: true }` 활성 캔버스에만 그리도록 분리하자, **펜을 모니터에 대고 긋는 동안에는 선이 전혀 보이지 않다가 펜을 떼는(`pointerup`) 순간에야 선이 한꺼번에 나타나는 현상**이 발생함.

---

## 2. 근본 원인 (Root Cause)

1. **`transparent + desynchronized: true` 캔버스의 컴포지터 손상 영역(Damage Rect) 누락**:
   - HTML5 2D Canvas의 `desynchronized: true` 저지연 경로는 다른 DOM 요소와 혼합·중첩되지 않은 단독 하드웨어 오버레이(unoccluded overlay)를 전제한다.
   - 상위 `z-index` 인터랙션 오버레이, `mix-blend-mode: multiply` 정적 레이어, 또는 `transform: scale(zoom)` 컨테이너가 겹친 DOM 구조에서는 Chromium/Edge/WebView GPU 컴포지터가 `{ desynchronized: true, alpha: true }` 캔버스의 프론트버퍼 갱신을 화면 합성으로 올리지 못해 `pointerup` 전까지 화면이 비어 보이게 된다.
2. **단일 획 $O(M^2)$ 재계산 & 다중 페이지 비가상화**:
   - `pointermove`마다 `0..n` 전체 점 스무딩/굵기 재계산, `getBoundingClientRect()` 동기 리플로우, 16~45개 고해상도(`2.0x~3.0x DPR`) 청크 `<canvas>` 상시 마운트(VRAM 폭증), 지우개 드래그 중 전체 청크 재렌더링이 병목을 유발했다.

---

## 3. 해결 패턴 (Solution Pattern)

### 3.1 Active Layer 표준 2D 컴포지터(`desynchronized: false`) + 2-레이어 엄격 분리
- **실시간 가시성 보장**: Active Layer(`wet`)와 Static Layer(`committed`) **모두 표준 2D 컨텍스트(`getContext('2d')`, `desynchronized: false`)** 를 사용해, CSS 블렌딩·줌 트랜스폼·상위 오버레이가 공존하는 환경에서도 `pointermove` 매 프레임 즉시 화면 손상 영역이 합성되도록 보장한다.
- **일정한 $O(1)$ 프레임 비용 유지**: `pointermove` 중에는 Active Layer에만 현재 1개 획의 신규 세그먼트만 증분 렌더링하며(`clearRect` 0회), 수백~수천 개 완료 획이 담긴 Static Layer는 전혀 건드리지 않는다. `pointerup` 시점에만 1:1(`setTransform(1,0,0,1,0,0)` + `drawImage`) 베이크 및 벡터 확정 1회를 수행한 뒤 Active Layer를 초기화한다.

### 3.2 `useRef` 좌표 수집 · `requestAnimationFrame` 배칭 · `WeakMap` 증분 기하 캐시
- **무상태(Zero-Re-render) 입력 수집**: 240Hz `getCoalescedEvents()` 좌표는 React `state`가 아닌 `useRef` 배열에 즉시 적재하고, 실제 캔버스 그리기는 `requestAnimationFrame`으로 프레임당 최대 1회 배칭한다. (`pointerdown` 시작점과 `pointerup` 잔여 구간은 즉시 플러시)
- **제스처 단위 아핀 변환 캐시**: 제스처 시작 시 `scrollLeft`/`scrollTop`/`zoom` 기준 원점을 `useRef`에 1회만 계산해 후속 `pointermove` 샘플에서 `getBoundingClientRect()` 리플로우를 제거한다.
- **정지 필압 감지 `WeakMap` 캐시**: 스트로크별 스무딩·필압 굵기 배열을 `WeakMap`에 증분 캐싱하되, 제자리에서 필압(`last[2]`)만 변한 경우도 감지해 마지막 점 기하만 정확히 재계산한다.

### 3.3 `±1` 뷰포트 청크 가상화 + GPU 컨텍스트 즉시 해제
- 뷰포트의 `scrollTop`, `clientHeight`, `zoom` 기준으로 현재 보이는 청크 인덱스 `[visStart .. visEnd]`의 인접 **`±1` 범위 청크만** 실제 `<canvas>`로 마운트하고, 스크롤 위치에 맞춰 Active Layer의 CSS `top` 계산 시에도 반드시 `scrollTop / zoom`으로 페이지 좌표를 환산한다.
- 스크롤 이벤트는 RAF로 디바운스하며 가시 인덱스 범위가 실제로 변할 때만 React 상태를 갱신한다.
- 화면 밖으로 이탈해 언마운트되는 청크 `<canvas>`는 ref `null` 콜백에서 즉시 `prevEl.width = 0; prevEl.height = 0;`을 호출해 GPU 텍스처와 2D 컨텍스트 메모리를 회수한다.

### 3.4 S-Pen 포인터 격리 및 `destination-out` 무지연 지우개
- **입력 격리**: 스타일러스(`pen`/`eraser` 및 배럴 지우개 버튼 `buttons === 32`) 입력은 `preventDefault()` + `stopPropagation()` + `setPointerCapture()`로 뷰포트 패닝/핀치 줌에서 완전히 차단하고, 펜 사용 종료 후 `450ms` 이내 터치(`touch`)는 팜 리젝션으로 무시한다.
- **지우개 국소 복원**: 드래그 중에는 삭제된 획의 교차 청크에 `globalCompositeOperation = 'destination-out'` 마스크만 즉시 적용해 전체 청크 재렌더를 피하고, `pointerup` 시 오염된 청크만 1회 깨끗이 복원한다.

---

## 4. 주의점 (Pitfalls)

1. **`desynchronized: true` 맹신 금지**: 실기기 GPU/브라우저 환경에서 투명 오버레이 + CSS `transform`/`mix-blend-mode`/형제 오버레이와 함께 `desynchronized: true`를 쓰면 화면이 아예 안 그려지다가 `pointerup`에 나타나는 치명적 증상이 발생할 수 있으므로, 실화면 합성 무결성이 중요할 때는 단일 활성 획 레이어에 표준 2D 컨텍스트 + 2-레이어 분리를 조합하는 것이 훨씬 안정적이다.
2. **JSDOM 가상화 테스트 호환**: 테스트 환경에서 `viewport.clientHeight === 0`인 경우 `getBoundingClientRect().height`로 폴백하면 기존 전체 문서 크기 테스트(`scrollTop = 0`)와 실제 스크롤 뷰포트(`clientHeight = 900`) 기반 `±1` 가상화 테스트를 모두 통과시킬 수 있다.
