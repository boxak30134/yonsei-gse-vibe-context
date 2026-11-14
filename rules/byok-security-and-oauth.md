---
title: BYOK API 키 격리·보안 및 OAuth 최소 권한 규칙
tags: [byok, security, api-key, aes-gcm, oauth, google-drive, release, sanitization]
updated: 2026-11-13
---

# BYOK API 키 격리·보안 및 OAuth 최소 권한 규칙 (`byok-security-and-oauth.md`)

교육용 웹·모바일 앱에서 사용자의 API 키와 클라우드 계정 권한을 안전하게 보호하기 위한 보안 규칙입니다.

---

## 1. API 키 저장 격리 및 번들 유출 원천 차단

1. **서비스별 저장소 식별자 엄격 분리**:
   - 동일한 공급사(Google 등)라도 서비스가 다르면 API 키를 절대 같은 스토리지 키에 저장하거나 폴백으로 공유하지 않는다.
   - 예: 생성형 AI 키(`ai:google`)와 번역 API 키(`translate:google`)는 완전히 독립된 저장·조회·삭제 함수로 격리한다.
2. **Release 빌드 시 개발용 환경변수 키 번들링 차단**:
   - `.env.local`에 개발자 개인 테스트 키가 있더라도, 마켓 배포 빌드(`--mode release` 또는 `VITE_BYOK_ONLY=true`)에서는 `vite.config.ts`에서 환경변수 키를 빈 문자열(`""`)로 치환하고 런타임 env 폴백도 차단해 APK/AAB 디컴파일 유출을 막는다.
3. **클라이언트 암호화 저장 + 폴백**:
   - 사용자 입력 키는 Web Crypto API(`AES-GCM`)로 암호화해 IndexedDB에 저장한다. Web Crypto 또는 IndexedDB가 제한된 환경/테스트 환경을 대비해 안전한 폴백 레이어를 둔다.
4. **로그 및 에러 메시지 마스킹**:
   - 콘솔 로그, 에러 객체, UI 안내 문구, 테스트 스냅샷에 원문 API 키가 절대 노출되지 않도록 `maskApiKey()`(앞 5자 + `***` + 뒤 4자만 노출) 및 `sanitizeLogMessage()`를 경유한다.

---

## 2. Google Drive 등 클라우드 연동 최소 권한 및 경로 경계 가드

1. **최소 권한 스코프 강제 (`drive.file`)**:
   - 사용자의 전체 드라이브를 열람할 수 있는 `drive`, `drive.readonly` 스코프 요청을 **절대 금지**한다.
   - 반드시 사용자가 선택하거나 앱이 생성한 단일 지정 폴더 및 파일에만 접근 가능한 `https://www.googleapis.com/auth/drive.file`만 사용한다.
2. **경로 경계 가드 (`Folder Boundary Guard`)**:
   - 모든 파일 목록 조회 쿼리에 `'<targetFolderId>' in parents and trashed = false` 조건을 강제한다.
   - 읽기·업로드·수정·삭제 요청 직전 대상 파일의 `parents`에 `targetFolderId`(또는 직계 `images/` 서브폴더)가 포함되어 있는지 검증하고, 이탈 시 API 호출 전에 `SecurityBoundaryError`를 던져 차단한다.
3. **지정 폴더 삭제 시 루트 임의 생성 금지**:
   - 사용자가 클라우드에서 지정 폴더를 삭제(`404` 또는 `trashed: true`)한 경우, 드라이브 루트에 새 폴더를 임의로 재생성하지 말고 동기화를 일시 중지(`SYNC_FOLDER_MISSING`)한 뒤 사용자에게 폴더 재지정을 안내한다.
4. **Android WebView OAuth 차단(`disallowed_useragent`) 우회**:
   - Google OAuth는 인앱 WebView에서의 로그인 요청을 차단하므로, 네이티브(Capacitor) 환경 감지 시 외부 시스템 브라우저로 인증 URL을 위임하고 리다이렉트 콜백으로 토큰을 수신하는 경로를 제공한다.
5. **연동 해제 시 로컬 데이터 보존**:
   - 계정 연동 해제 시 암호화 저장된 OAuth 토큰만 파기하며, 사용자의 로컬 노트 데이터는 절대 삭제하지 않는다.

