---
title: 태블릿·모바일 WebView UI 및 포인터 안전 규칙
tags: [tablet, android, capacitor, webview, touch, portal, modal, focus, worker, ui]
updated: 2026-11-13
---

# 태블릿·모바일 WebView UI 및 포인터 안전 규칙 (`tablet-webview-safety.md`)

태블릿 터치 스크린, 가상 키보드, Android WebView(Capacitor) 환경에서 데스크톱 기준의 UI 가정이 깨지는 현상을 방지하기 위한 공통 규칙입니다.

---

## 1. 터치 환경 UI 가시성 및 인터랙션 원칙

1. **호버(`:hover`) 의존 숨김 금지**:
   - 태블릿·모바일 터치 화면에는 마우스 호버가 없다. 삭제·추가 등 주요 액션 버튼에 `opacity-0 group-hover:opacity-100`을 적용하면 터치 사용자가 버튼을 전혀 찾을 수 없다.
   - 상시 노출(`opacity-90 hover:opacity-100`) 또는 명시적인 `[선택]` 모드 토글을 제공한다.
2. **플로팅 메뉴 터치 시 에디터 포커스 보호**:
   - 텍스트 선택 후 나타나는 플로팅 툴바나 보조 바의 버튼을 터치할 때 에디터가 `blur`되어 선택 영역(`Selection`)이 날아가는 것을 막기 위해, 버튼에 `onPointerDown={(e) => e.preventDefault()}`를 적용하고 직전 유효 선택 범위(`SavedSelection`)를 캐싱해 둔다.
3. **모달 포커스 이동으로 인한 부모 플로팅 메뉴 언마운트 방어**:
   - 플로팅 메뉴 내부에서 입력 모달(예: API 키 입력창)을 열 때 모달 `<input>`으로 포커스가 넘어가면 에디터 `blur` 이벤트가 발생해 부모 플로팅 메뉴가 언마운트되면서 모달까지 함께 닫힐 수 있다.
   - 모달은 반드시 `createPortal(..., document.body)`로 최상위 레이어(`z-[120]`)에 마운트하고, `isModalOpenRef` 가드로 모달이 열린 동안 에디터 `blur`에 의한 플로팅 메뉴 숨김을 차단한다.

---

## 2. 스크롤 점프 및 뷰포트 레이아웃 보호

1. **`focus({ preventScroll: true })` 강제**:
   - 동적으로 생성된 인라인 `<textarea>`나 `<input>`에 프로그래밍 방식으로 포커스를 줄 때는 반드시 `el.focus({ preventScroll: true })`를 호출한다. 옵션을 생략하면 브라우저가 요소를 화면 상단으로 끌어오기 위해 컨테이너 `scrollTop`을 강제 점프시킨다.
2. **가상 키보드 및 화면 회전 리플로우 대응**:
   - `AndroidManifest.xml`에 `android:windowSoftInputMode="adjustResize"`와 `android:configChanges="orientation|keyboardHidden|keyboard|screenSize|locale|smallestScreenSize|screenLayout|uiMode"`를 선언해 회전·키보드 팝업 시 WebView 재시작 및 캔버스 초기화를 방지한다.
   - `orientationchange` 및 `visualViewport.resize` 이벤트를 구독하고, 안드로이드 리플로우 완료 직후(`~120ms`) 캔버스 DPR과 툴바 위치를 재계산한다.
3. **바닥 스크롤 Pull-up 자동 페이지 생성 지양**:
   - 터치 패닝과 바닥 당김 제스처가 경합해 의도치 않게 빈 페이지가 계속 생성될 수 있으므로, 페이지 추가는 명시적 버튼(헤더, 썸네일, 시트 하단 버튼)으로 일원화한다.

---

## 3. WebView 런타임 제약 및 폴백

1. **Web Worker 실패 시 메인 스레드 폴백**:
   - 일부 Android WebView 환경이나 번들 설정에서는 `{ type: 'module' }` Web Worker 초기화가 실패하거나 메시지 타임아웃이 발생할 수 있다.
   - Worker 생성에 `try-catch` 및 응답 타임아웃(예: `2000ms`)을 두고, 실패 시 메인 스레드 동기/비동기 엔진으로 즉시 폴백한다.
2. **Blob URL 메모리 누수 방지**:
   - 대용량 페이지 썸네일이나 이미지 미리보기에 `URL.createObjectURL`을 사용할 경우 컴포넌트 언마운트 또는 리스트 갱신 시 반드시 `URL.revokeObjectURL`로 해제한다.
3. **Android 하드웨어 가속 및 오버스크롤 제어**:
   - `<application>`과 `<activity>` 양쪽에 `android:hardwareAccelerated="true"`를 명시하고, `MainActivity`에서 `webView.setLayerType(View.LAYER_TYPE_HARDWARE, null)` 및 `webView.setOverScrollMode(View.OVER_SCROLL_NEVER)`를 설정해 2D 캔버스 필기 지연과 바운스 왜곡을 차단한다.
4. **포인터 캡처 및 오버레이 격리**:
   - 드래그 가능한 플로팅 핀·오버레이 UI에는 `data-overlay` 속성, `e.stopPropagation()`, 필요 시 `setPointerCapture`를 적용해 하위 캔버스나 에디터로 포인터 이벤트가 누수되지 않도록 격리한다.

