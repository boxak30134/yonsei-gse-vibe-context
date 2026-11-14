---
title: Gemini 3.x Thinking 토큰 소진(MAX_TOKENS)으로 인한 JSON 중간 끊김 및 후속 적용 실패 해결
major: 교육공학 / 공통
tags: [gemini, thinking-level, max-tokens, json-mode, structured-output, context-cleanup, streaming]
date: 2026-11-14
---

# Gemini 3.x Thinking 토큰 소진(`MAX_TOKENS`)으로 인한 JSON 중간 끊김 및 후속 적용 실패 해결

## [문제 상황]
- AI 슬라이드/레이아웃 디자인 생성 요청 시 응답이 출력되다가 중간에 끊기고, 이후 이어서 보내는 후속 디자인 수정 요청들도 화면에 반영되지 않는 현상이 발생했다.

## [원인 분석]
1. **Thinking 토큰과 `maxOutputTokens` 예산 공유로 인한 `MAX_TOKENS` 조기 절단**:
   - Gemini 3.x 계열 모델(`gemini-3.8-flash` 등)은 기본적으로 추론(`MEDIUM`) 토큰을 소비하며, `maxOutputTokens` 상한(예: `2400`)에 Thinking 토큰이 함께 합산된다. 이로 인해 실제 JSON 출력 중 토큰 예산이 고갈되어 `finishReason: MAX_TOKENS`로 중간에 잘리고 `JSON.parse()`가 실패했다.
2. **직렬화된 JSON 문자열을 단순 `.slice()`로 절삭해 후속 요청 오염**:
   - 현재 캔버스 상태를 AI에 전달할 때 `JSON.stringify(state).slice(0, 3500)`처럼 문자열 길이를 기준으로 자르면, 요소가 많아졌을 때 닫는 괄호가 잘린 불완전한 JSON이 프롬프트에 주입되어 후속 요청이 연쇄 실패했다.
3. **대화 히스토리에 오류/중복 메시지 및 깨진 JSON 잔존**:
   - 직전 턴의 `⚠️` 오류 문구, 플레이스홀더, 중단된 JSON 텍스트가 `history`에 그대로 누적되어 다음 요청의 모델 판단을 방해했다.
4. **이전 스타일 속성(Gradient 등) 잔존**:
   - 후속 디자인 패치에서 배경색(`color`)만 지정하고 `gradient`를 생략했을 때, 기존 상태와 단순 스프레드 병합(`{ ...prev, ...next }`)을 수행해 이전 그라데이션이 새 단색 배경을 덮어쓰는 문제가 있었다.

## [해결 패턴]
1. **Thinking 수준 하향 + 충분한 출력 상한 + JSON 모드(`responseMimeType`)**:
   - `@google/genai/web`의 `ThinkingLevel.LOW` (`minimal`은 Gemini 3.8 Flash 미지원으로 오류 발생)를 지정하고, 구조화 JSON 응답 모드에서는 `maxOutputTokens`를 `8192`로 상향하며 `responseMimeType: 'application/json'`을 적용한다. (미지원 모델 `400` 반환 시 해당 옵션만 제거하고 동일 모델로 1회 재요청)
2. **스트리밍 프리뷰 분리 및 `finishReason` 검증**:
   - JSON 생성 중에는 깨진 JSON 원문 대신 `explanation` 필드를 부분 추출해 사용자 친화적 진행 문구로 표시하고, 완료 시 `MAX_TOKENS` 또는 파싱 실패가 감지되면 무응답 성공 처리하지 않고 명확한 오류 상태와 `[다시 시도]` 버튼을 노출한다.
3. **유효 JSON 단위 컨텍스트 축약 & 히스토리 정제**:
   - 캔버스 JSON을 문자열 `.slice()`로 자르지 않고 요소 개수(`slice(0, 24)`)와 각 텍스트 필드 길이만 제한해 항상 문법적으로 유효한 JSON을 전송한다.
   - `history` 전달 시 현재 턴 메시지·오류 문구(`⚠️`)·원시 JSON을 제외한다.
4. **배경 패치 초기화 병합**:
   - 디자인 패치에 `background`가 포함되면 이전 `gradient`를 유지하지 않고 새 디자인 기준으로 명시적 초기화한다.

## [에이전트 방어 규칙 제안]
- Gemini 3.x 이상 모델로 구조화 JSON을 생성할 때는 `ThinkingLevel.LOW` + `responseMimeType: 'application/json'` + 충분한 `maxOutputTokens(>= 8192)`를 기본 적용한다.
- 프롬프트에 상태 JSON을 주입할 때 `JSON.stringify(...).slice(0, N)` 같은 문자열 절삭을 절대 사용하지 않고, 직렬화 전에 배열/필드 단위로 클리핑한다.
