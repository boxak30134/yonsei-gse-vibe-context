---
title: Gemini 스트리밍 SDK Lazy Throw 무응답, 404 종료 모델 및 503 백오프·폴백 해결
major: 교육공학 / 공통
tags: [gemini, streaming, lazy-throw, timeout, backoff, fallback, byok, zustand]
date: 2026-11-09
---

# Gemini 스트리밍 SDK Lazy Throw 무응답, 404 종료 모델 및 503 백오프·폴백 해결

## [문제 상황]
- 사용자가 유효한 Gemini API 키를 등록하고 AI 패널에서 질문해도 로딩 스피너만 무한히 돌거나, 응답도 에러 문구도 없이 요청이 조용히 증발하는 현상이 발생했다.

## [원인 분석]
1. **SDK Lazy Throw로 인한 폴백 루프 탈출 (핵심 구조 결함)**: `@google/genai`의 `generateContentStream()`은 호출 시점에는 HTTP 요청을 보내지 않고 `AsyncIterable`만 즉시 반환하며, 실제 `503 UNAVAILABLE` / `404 NOT_FOUND` 에러는 이후 `for await`의 첫 번째 `next()` 시점에 발생한다. 이로 인해 스트림 생성부의 `try-catch`가 에러를 잡지 못하고 폴백 루프를 빠져나갔다.
2. **Deprecated 폴백 모델(404)**: 폴백 모델로 지정된 `gemini-2.5-flash`가 Google 서버에서 신규 사용자 대상 404 종료되어 폴백 자체가 불능 상태였다.
3. **타임아웃 부재 & `stream.Transform` 노이즈**: 소켓 Hang 시 첫 청크 타임아웃이 없어 무한 대기했고, 클라이언트 번들에서 Node `stream.Transform` 경고가 발생했다.

## [해결 패턴]
1. **첫 청크 조기 프로빙(`probeStreamFirstChunk`) + 3단계 타임아웃**: 모델·재시도 루프 내부에서 첫 `iterator.next()`를 `withTimeout(..., FIRST_CHUNK_TIMEOUT_MS = 12000)`으로 즉시 프로빙해 Lazy Throw를 포착한다. (`thought === true` 토큰도 연결 성공으로 판정, 청크 간 20초·전체 예산 90초 상한 적용)
2. **상태별 에러 분류 & 최신 폴백 체인**:
   - `503/429/500/502/504/timeout`은 지수 백오프+Jitter(`1s→2s→4s`, ±20%; 429 `Retry-After` 우선) 최대 3회 재시도 후 최신 가용 모델 체인으로 폴백.
   - `404`는 재시도 없이 즉시 다음 모델로 전환(종료된 2.x/1.5 모델 체인 편입 금지).
   - `400/401/403`은 즉시 중단(단 `thinkingConfig` 미지원 400은 필드 제거 후 1회 재요청).
3. **`@google/genai/web` & `emptyStreamStub`**: 브라우저 전용 엔트리포인트와 Vite `stream` alias 스텁을 적용했다.

## [에이전트 방어 규칙 제안]
- `generateContentStream` 등 AsyncIterable 기반 LLM SDK 호출 시 반드시 루프 내부에서 첫 청크 `next()`를 타임아웃과 함께 프로빙해야 폴백/재시도가 작동하며, 본문 일부 출력 후 끊김 시에는 중복 방지를 위해 재시도하지 않는다. (참조: `patterns/llm-byok-tutor.md`)

---

### [추가 사례] 2026-11-10 ~ 2026-11-12 (교육공학 / 공통)
- **변형 상황**: (1) 재시도·폴백 중 사용자가 먹통으로 오인했고, (2) AI 패널을 닫거나 새로고침하면 대화 내역이 모두 초기화되고 진행 중인 응답이 중단되었다.
- **해결**:
  - `connecting`/`retrying`/`fallback`/`streaming`/`done`/`error`/`cancelled` 7대 상태 한 줄 배너와 `[취소]`(`AbortController`), `[다시 시도]`, 응답 모델 뱃지, `maskApiKey` 보안 로깅을 연동했다.
  - 대화 상태를 컴포넌트 로컬 `useState`에서 Zustand `persist` 스토어로 격리하고 언마운트 시 자동 `abort()`를 제거해, 패널을 닫거나 새로고침해도 대화가 유지되며 API 전송 시에는 최근 4턴(`slice(-4)`) 슬라이딩 윈도우만 전송하도록 했다.

---

### [추가 사례] 2026-11-20 (교육공학 / 공통)
- **변형 상황**: `probeStreamFirstChunk`를 적용했음에도 간헐적으로 1분 이상 응답이 멈추거나 `[취소]` 버튼과 전체 타임아웃이 작동하지 않고, 타임아웃 종료 시 `"사용자에 의해 요청이 취소되었습니다"`로 오인 표시되었다.
- **원인**:
  1. `@google/genai` SDK의 `HttpRetryOptions.attempts` 기본값이 `5`(최대 60초 지연)여서 `429`/`503` 발생 시 SDK 내부 5회 재시도와 앱 외부 재시도 루프가 곱해져 단일 모델에서 1분 이상 블로킹되었다.
  2. `ai.models.generateContentStream()` 자체가 첫 HTTP 응답 헤더 수신까지 내부 `fetch()`를 `await`하므로, 반환 이후 `iterator.next()`에만 타임아웃을 걸고 `config.abortSignal`을 전달하지 않으면 헤더 대기 구간에서 타임아웃·취소가 무력화되었다.
  3. 전체 예산 `AbortController` 타임아웃이 사용자 취소(`AbortError`)와 구분되지 않았고, 필기/이미지가 없는 텍스트·도형 작업에도 매번 캔버스 이미지를 전송해 지연이 가중되었다.
- **해결**:
  1. `httpOptions: { retryOptions: { attempts: 1 }, timeout: FIRST_CHUNK_TIMEOUT_MS }`로 SDK 내부 중복 재시도를 끄고 앱 폴백 루프가 즉시 제어하게 했다.
  2. `await ai.models.generateContentStream(...)` 호출과 첫 `iterator.next()`를 하나의 `withTimeout`으로 묶고, 시도별 `attemptController.signal`을 `config.abortSignal`에 전달해 타임아웃 시 하부 `fetch()`를 즉시 강제 종료한 뒤 다음 폴백 모델로 전환했다(성공 모델은 `lastWorkingGeminiModel`로 캐싱).
  3. `budgetTimedOut` 플래그로 전체 시간 초과와 사용자 취소를 분리하고, 시각 요소(필기 획·이미지)가 있을 때만 경량 스냅샷(`scale: 0.5, quality: 0.65`)을 전송하도록 최적화했다.

