---
title: 태블릿 Android WebView 한글 IME 입력 차단 및 마크다운 InputRules 무력화 해결
major: 교육공학 / 공통
tags: [korean, ime, capacitor, android, webview, tiptap, prosemirror, inputrules]
date: 2026-11-06
---

# 태블릿 Android WebView 한글 IME 입력 차단 및 마크다운 InputRules 무력화 해결

## [문제 상황]
- PC 크롬 브라우저에서는 한글 입력 및 폴더·문서 제목 작성이 정상 작동했으나, Android 태블릿(Capacitor WebView) 환경의 다이얼로그/입력창에서 한/영 전환이 막히고 영문 ASCII로만 고정되는 현상이 발생했다.

## [원인 분석]
1. **Capacitor `captureInput: true` 설정 부작용**: `CapacitorWebView`가 `onCreateInputConnection()` 호출 시 조합형 문자를 처리하지 못하는 단순 키 이벤트용 더미 `BaseInputConnection(this, false)`를 반환한다. 이로 인해 삼성 키보드·Gboard가 해당 뷰를 IME 비지원 입력기로 판정해 한/영 전환을 차단했다.
2. **`autoFocus` 타이밍 충돌 및 IME 메타데이터 누락**: 모달 마운트 시 `autoFocus`가 WebView 키보드 컨텍스트 바인딩 전에 포커스를 강제 전달해 영문 자판이 선행 초기화되었고, `<input>`에 `inputMode="text"`, `lang="ko"` 속성이 빠져 있었다.

## [해결 패턴]
1. `capacitor.config.ts`에서 `android.captureInput: false`로 변경해 WebView가 Chromium 순정 IME 파이프라인을 사용하도록 복원했다.
2. 모달 내 HTML `autoFocus`를 제거하고 `useEffect` + `setTimeout(..., 60)` 지연 포커스로 교체했으며, 입력 요소에 `type="text"`, `inputMode="text"`, `lang="ko"`, `autoCapitalize="off"`, `autoCorrect="off"`, `spellCheck={false}`를 명시했다.

## [에이전트 방어 규칙 제안]
- Capacitor Android 설정 생성/수정 시 `android.captureInput`은 반드시 `false`로 고정하고, 모달 입력창에는 `autoFocus` 대신 지연 포커스와 한국어 IME 속성을 적용한다. (참조: `rules/korean-web-safety.md`)

---

### [추가 사례] 2026-11-10 (교육공학 / 공통)
- **변형 상황**: 태블릿 가상 키보드에서 `# 제목`, `- 목록`, `> 인용`, `**강조**` 등 마크다운 문법을 입력할 때 TipTap/ProseMirror 서식 변환이 전혀 작동하지 않고 플레인 텍스트로 남는 현상이 발생했다.
- **원인**: 안드로이드 가상 키보드는 예측 입력과 조합 버퍼 관리를 위해 기호·공백 입력 중에도 `view.composing === true` 세션을 유지한다. ProseMirror의 `handleTextInput`은 `if (view.composing) return false;`로 `InputRules`를 억제하며, `compositionend` 직후에도 `InputRules`를 재평가하지 않는다.
- **해결**:
  - `isTouchDevice()`로 데스크톱 PC 물리 키보드는 100% 바이패스해 한글 끝 글자 중복(`노트` → `노트트`) 회귀를 원천 차단했다.
  - 터치 디바이스에서만 `compositionend`, `beforeinput`, `keydown`(Space/Enter/229) 직후 `requestAnimationFrame`으로 DOM flush 완료 시점에 마크다운 정규식을 평가·변환하되, 커서 앞 문자가 한글 음절/자모(`/[ㄱ-ㅎㅏ-ㅣ가-힣]$/`)로 끝나면 즉시 바이패스하는 가드를 추가했다.
  - 가상 키패드 전환 피로를 줄이기 위해 `onPointerDown={(e) => e.preventDefault()}`로 에디터 포커스를 보존하는 모바일 마크다운 보조 툴바를 연동하고, `.ProseMirror` CSS 서식을 보강했다.

