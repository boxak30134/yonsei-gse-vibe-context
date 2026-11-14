# Google Drive 로컬 우선 동기화 및 PWA OAuth 연동 패턴

> **적용 대상**: 별도 백엔드 서버 없이 클라이언트(React/Vite)에서 Google Drive API를 연동하고, Vercel 및 PWA(태블릿/모바일 포함)로 배포하는 로컬 우선(Local-First) 웹 애플리케이션

---

## 1. 핵심 아키텍처 원칙

1. **사용자 무설정(Zero-Config) 인증 UX**
   - 일반 사용자(원우)에게 Google Cloud Console에서 `Client ID`를 직접 발급받아 입력하게 하지 않는다.
   - 개발자가 발급한 단일 `VITE_GOOGLE_CLIENT_ID`를 빌드 환경 변수로 내장하여 **원클릭 로그인**으로 즉시 연동한다.
2. **최소 권한 원칙 (`drive.file` + Google Picker API)**
   - 사용자의 전체 Google Drive 열람 권한(`drive`) 대신, 앱이 생성했거나 사용자가 Picker로 명시적 선택한 폴더/파일만 접근하는 `https://www.googleapis.com/auth/drive.file` 스코프를 사용한다.
   - Google 보안 심사(OAuth Verification) 장벽을 낮추고 사용자 데이터 프라이버시를 보장한다.
3. **로컬 우선 저장 (`IndexedDB` $\rightarrow$ Drive Sync)**
   - 모든 읽기/쓰기는 기기 내부 `IndexedDB`에 즉시 반영하여 오프라인 및 필기 지연을 방지하고, Drive 동기화는 백그라운드 엔진으로 분리한다.

---

## 2. Google Cloud Console (GCP) 필수 체크리스트

연동 초기 가장 빈번하게 발생하는 `403 PERMISSION_DENIED` 및 `400 redirect_uri_mismatch` 오류를 방지하기 위해 아래 항목을 반드시 확인한다.

- **API 활성화 (2개 모두 필수)**
  - `Google Drive API` 사용 설정
  - `Google Picker API` 사용 설정 *(Drive API만 켜고 Picker API를 누락하면 폴더 선택 창 호출 시 `403` 에러 발생)*
- **OAuth 클라이언트 유형**: `웹 애플리케이션`
- **승인된 JavaScript 원본** (경로 없이 도메인만 입력)
  - 로컬 개발: `http://localhost:5173`
  - 프로덕션: `https://<고정-프로젝트명>.vercel.app`
- **승인된 리디렉션 URI** (콜백 경로 포함, 끝 슬래시 유무 정확히 일치)
  - 로컬 개발: `http://localhost:5173/oauth/callback`
  - 프로덕션: `https://<고정-프로젝트명>.vercel.app/oauth/callback`
- **OAuth 동의 화면**: 외부(External) 유형 선택 후 테스트 기간 중에는 사용할 원우들의 Gmail 주소를 **테스트 사용자** 목록에 등록한다.

---

## 3. Vercel 환경 변수 및 도메인 관리

- **환경 변수 타입 주의 (`Secret`이 아닌 `Config` 사용)**
  - Vite 환경 변수 접두사(`VITE_`)가 붙은 키는 브라우저 번들에 포함되므로 Vercel에서 `Secret` 선택 시 경고가 발생한다.
  - `VITE_GOOGLE_CLIENT_ID`는 공개 클라이언트 식별자이므로 **`Config`** 타입으로 등록한다. (변수 추가/수정 후 반드시 **Redeploy** 실행)
- **고정 프로덕션 URL 유지**
  - Vercel 배포 내역의 스냅샷 해시 URL(`...-git-main-....vercel.app`)은 배포마다 바뀌므로 OAuth 리디렉션에 사용할 수 없다.
  - 작업 브랜치(`feature/*`)는 반드시 **`main` 브랜치로 병합(Merge)**하여 항상 동일한 대표 도메인(`https://<프로젝트명>.vercel.app`)에 최신 빌드가 반영되도록 유지한다.

---

## 4. PWA (`vite-plugin-pwa`) 적용 시 OAuth 충돌 방지

PWA의 Service Worker가 모든 경로 탐색을 캐시된 `index.html`로 가로채면(Navigation Fallback), Google 로그인 직후 돌아오는 `/oauth/callback` 인증 파라미터가 유실된다.

`vite.config.ts`의 `workbox` 설정에 **OAuth 콜백 경로 예외 처리**를 반드시 포함한다.

```ts
VitePWA({
  registerType: 'autoUpdate',
  manifest: {
    display: 'standalone', // 태블릿/모바일에서 브라우저 주소창 제거
    start_url: '/',
    // ...아이콘 및 앱 정보
  },
  workbox: {
    globPatterns: ['**/*.{js,mjs,css,html,ico,png,svg,woff2}'],
    // 핵심: OAuth 리디렉션 경로는 Service Worker가 가로채지 않도록 제외
    navigateFallbackDenylist: [/^\/oauth\/callback/],
  },
})
```

---

## 5. 태블릿 환경 배포 전략: PWA vs 네이티브 래핑(Capacitor APK)

S-Pen 필기 및 Google OAuth가 결합된 웹 앱을 태블릿(갤럭시 탭 등)에 배포할 때는 **네이티브 APK 빌드보다 웹/PWA 배포를 우선 권장**한다.

| 비교 항목 | 웹 / PWA (크롬 홈 화면 설치) | Capacitor 안드로이드 APK (WebView) |
| :--- | :--- | :--- |
| **Google OAuth 동작** | PC와 100% 동일하게 즉시 작동 | `https://localhost` 연결 거부, `window.opener === null`, `disallowed_useragent` 등 네이티브 딥링크 구현 필수 |
| **S-Pen 필기 지연** | 크롬 브라우저의 저지연 캔버스(`desynchronized`) 및 OS 팜 리젝션 기본 최적화 | Android View 합성 레이어로 인한 미세 레이턴시 및 시스템 제스처 간섭 발생 |
| **원우 배포 및 업데이트** | Vercel 고정 URL 공유만으로 즉시 설치, `main` 푸시 시 자동 업데이트 | APK 재빌드·재배포 또는 Play 스토어 14일 비공개 테스트 심사 필요 |

> **에이전트 토큰 최적화 팁**: 웹/PWA 주력 배포 시 루트의 `.claudeignore` 및 `.geminiignore`에 `android/` 디렉토리를 등록해 코딩 에이전트가 불필요한 네이티브 파일을 탐색하지 않도록 차단한다.

---

## 6. 주요 에러 트러블슈팅 요약

| 에러 메시지 / 증상 | 원인 | 해결 방법 |
| :--- | :--- | :--- |
| `400: redirect_uri_mismatch` | 임시 Preview URL로 접속했거나, GCP 콘솔의 리디렉션 URI와 실제 `origin + /oauth/callback`이 불일치 | 에러 팝업의 `[오류 세부정보]`에서 실제 `redirect_uri`를 확인 후 고정 도메인으로 접속하거나 콘솔 URI를 정확히 일치시킴 |
| `403: PERMISSION_DENIED` (Picker 호출 시) | GCP 프로젝트에서 `Google Picker API`가 비활성화됨 | GCP 라이브러리에서 `Google Picker API` 검색 후 사용 설정 |
| PWA 설치 후 로그인 화면 무한 루프 / 백지화 | Service Worker가 `/oauth/callback` 요청을 캐시된 `index.html`로 덮어씀 | `workbox.navigateFallbackDenylist`에 `/^\/oauth\/callback/` 추가 후 재배포 |
| Drive에서 `.md` 동기화 시 원문 기호(`#`, `*`) 노출 | 초기 로드(`setContent`) 시 마크다운 파서를 거치지 않고 플레인 텍스트로 주입됨 | 초기 데이터 로드 및 동기화 반영 파이프라인에 마크다운 파서(`tiptap-markdown` 등) 변환 단계 연결 |