---
title: 카드 없는 BYOK 무료 AI 튜터·3-Tier 하이브리드 번역 및 스트리밍 복구 패턴
tags: [llm, gemini, byok, streaming, fallback, retry, translation, token-optimization, zustand]
updated: 2026-11-13
---

# 카드 없는 BYOK AI 튜터·3-Tier 폴백 및 스트리밍 안정화 패턴 (`llm-byok-tutor.md`)

교육용 웹앱에서 사용자가 신용카드 없이 발급한 무료 AI API 키(Google AI Studio 등)로 튜터·번역 기능을 사용할 때 발생하는 **스트림 무응답(Hang), Lazy Throw 폴백 실패, 404/503 에러, 토큰 낭비, 채팅 내역 유실**을 해결하는 표준 패턴입니다.

---

## 1. 3-Tier 하이브리드 엔진 (Zero-Config + Card-Free BYOK)

결제 카드 등록이 필요한 유료 API(예: GCP Cloud Translation) 의존을 없애고 설치 즉시 동작하도록 3단계 파이프라인을 구성한다.

1. **Tier 1 (Local First — 0ms, 0 Token)**:
   - 로컬 용어 사전/적응형 사전(`seedGlossary` + 사용자 학습 `IndexedDB`)에서 문맥 공기어(Co-occurrence) 스코어링으로 신뢰도 `>= 70%`인 단일 단어는 네트워크 호출 없이 즉시 반환.
2. **Tier 2 (Card-Free BYOK — 무료 AI Studio / 멀티 프로바이더)**:
   - 사용자가 입력한 Gemini/OpenAI/Anthropic 무료·개인 키로 문맥 번역 및 학술 튜터 응답 수행.
   - 키 오류(만료/할당량/과부하) 시 번역 작업은 중단하지 않고 Tier 3으로 자동 폴백.
3. **Tier 3 (Zero-Config 공개 무키 폴백)**:
   - 키가 없거나 Tier 2가 실패한 번역 요청은 공개 경량 번역 엔드포인트로 자동 폴백해 앱 진입 장벽을 제거.

---

## 2. Gemini SDK Lazy Throw 방어 & 타임아웃·폴백 파이프라인

### 2.1 핵심 원인: `generateContentStream()`의 지연 평가 (Lazy Throw)
- `@google/genai` SDK(브라우저에서는 반드시 `@google/genai/web` 사용)의 `generateContentStream()`은 호출 시점에 HTTP 요청을 보내지 않고 `AsyncIterable` 객체만 즉시 반환한다.
- 실제 `503 Server Unavailable` 또는 `404 Not Found`는 이후 `for await (const chunk of stream)`의 **첫 번째 `next()` 호출 시점**에 발생한다. 따라서 `generateContentStream()`만 `try-catch`로 감싸면 모델 폴백 루프가 완전히 무력화된다.

### 2.2 해결 패턴: 첫 청크 조기 프로빙 (`probeStreamFirstChunk`) + 3단계 타임아웃
```ts
const FIRST_CHUNK_TIMEOUT_MS = 12_000;   // 첫 청크(thought 포함) 도착 제한
const CHUNK_GAP_TIMEOUT_MS = 20_000;     // 스트리밍 중 청크 간 무응답 제한
const TOTAL_REQUEST_BUDGET_MS = 90_000;  // 전체 재시도·폴백 합산 상한

async function probeStreamFirstChunk(stream, signal) {
  const it = stream[Symbol.asyncIterator]();
  const first = await withTimeout(it.next(), FIRST_CHUNK_TIMEOUT_MS, signal, 'first-chunk');
  if (first.done) throw new EmptyStreamError();
  return { first: first.value, rest: it };
}
```
- **규칙 1**: 첫 청크의 모든 part가 `thought === true`(thinking 토큰)여도 연결 성공으로 판정한다. (단, 노트 앱 짧은 질의는 `thinkingConfig`를 최소 레벨로 낮춰 첫 본문 지연을 줄이고, `thinkingConfig` 미지원 400 시 필드 제거 후 같은 모델로 1회만 재요청한다.)
- **규칙 2**: 본문 일부가 이미 UI에 스트리밍된 뒤 `CHUNK_GAP_TIMEOUT_MS`로 끊긴 경우 **절대 재시도하지 않는다**(내용 중복 출력 방지). 기수신 본문을 보존하고 안내 문구만 덧붙인다.

### 2.3 에러 분류 및 지수 백오프 + Jitter
SDK 에러 형태(`{ status }`, `{ error: { code } }`, 메시지 내 상태 코드)를 통합 파싱(`extractHttpStatus`)해 다음과 같이 분기한다.

| 상태 코드 / 유형 | 분류 | 처리 동작 |
| :--- | :--- | :--- |
| `503`, `429`, `500`, `502`, `504`, `timeout` | `retryable` | 동일 모델로 최대 3회 지수 백오프(`1s → 2s → 4s`, ±20% Jitter; 429는 `Retry-After` 우선, 상한 10s) 후 다음 폴백 모델로 전환 |
| `404` | `fatal-model` | 재시도 없이 **즉시 다음 모델**로 전환 (`gemini-2.5-*` 등 종료 모델 체인 편입 금지) |
| `400` | `fatal-request` | 재시도·폴백 없이 즉시 종료 |
| `401`, `403` | `fatal-auth` / `fatal-permission` | 즉시 종료 + 원인별 안내(무료 티어/결제/키 오류) |
| `AbortError` | `cancelled` | 백오프 타이머 즉시 `clearTimeout`, 에러 UI 미표시 |

- **7단계 UI 상태 배너**: `connecting` → `retrying` → `fallback` → `streaming` → `done`(사용 모델 뱃지) / `error` / `cancelled`를 상단 한 줄 배너 + `[취소]`/`[다시 시도]` 버튼으로 노출해 무한 로딩 착시를 없앤다.

---

## 3. 런타임 토큰 최적화 & 대화 내역 영구 보존

1. **런타임 API 페이로드 절삭**:
   - **슬라이딩 윈도우**: UI에는 전체 대화(`최대 200개`)를 스크롤 가능하게 보존하되, 실제 API 전송은 최근 4턴(`history.slice(-4)`)만 전송한다.
   - **컨텍스트 클리핑**: 문서 전체 대신 선택 문장과 앞뒤 문맥(최대 `1,000자`)만 주입한다.
   - **출력 상한 및 조건부 프롬프트**: `maxOutputTokens`를 작업별로 제한하고, 시스템 프롬프트는 범용 학술·리서치 튜터로 중립화하며 수식 관련 질의일 때만 KaTeX LaTeX 규칙을 조건부 적용한다.
2. **Zustand `persist` 기반 채팅 영속화**:
   - 채팅 상태를 컴포넌트 로컬 `useState`에 두면 패널 닫기/새로고침 시 대화가 증발한다. 전역 `persist` 스토어(`localStorage`)로 분리하고, **패널 언마운트 시 자동 `abort()`를 제거**해 사용자가 다른 화면을 보는 중에도 백그라운드 응답이 완료되도록 한다.
3. **멀티 프로바이더 모델 선택기(`AiModelSelector`) & 사전 검증**:
   - 모델 ID 직접 입력 대신 `[Google / OpenAI / Anthropic]` 그룹별 표시명·한 줄 설명·예상 소진 토큰 뱃지 + `"기타 모델 ID 직접 입력"` 옵션을 제공한다.
   - 제공자별 키 형식 검증(`isValidKeyFormat`)과 미입력 시 크래시 없는 안내 배너·발급 URL을 연결하며, 채팅창 하단에 예상 입력 토큰 + 최대 출력 상한을 미리보기로 표시한다.
4. **Vite `stream.Transform` 브라우저 경고 해소**:
   - 클라이언트 번들링 시 Node `stream` 참조 경고를 없애기 위해 `vite.config.ts`의 `resolve.alias`에 경량 `emptyStreamStub`을 매핑한다.

