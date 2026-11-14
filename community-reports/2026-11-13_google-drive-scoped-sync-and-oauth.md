---
title: Google Drive 지정 폴더 격리 동기화(drive.file) 및 Picker API Key 없는 원클릭 연동
major: 교육공학 / 공통
tags: [google-drive, oauth, drive-file, security-boundary, sync, local-first, gis]
date: 2026-11-13
---

# Google Drive 지정 폴더 격리 동기화 및 Picker API Key 없는 원클릭 연동

## [문제 상황]
- 노트 앱에 Google Drive 동기화를 도입할 때 (1) 전체 드라이브 열람 권한 요구로 인한 보안 불안, (2) 사용자가 OAuth Client ID를 직접 입력해야 하는 높은 진입 장벽, (3) Google Picker API가 별도 API Key 및 콘솔 활성화를 요구해 폴더 선택 창이 열리지 않는 문제가 발생했다.

## [원인 분석]
1. `drive` / `drive.readonly` 상위 스코프나 부모 폴더 경계 검증 부재 시 앱 버그로 타 폴더 파일이 손상될 위험이 있다.
2. Google Picker SDK는 OAuth Client ID 외에 추가 Picker API Key가 필요하며, 미설정 시 창이 열리지 않는다. 또한 Google Cloud 프로젝트에서 `Drive API`가 비활성화되어 있으면 403 `accessNotConfigured` 오류가 발생한다.

## [해결 패턴]
1. **최소 권한(`drive.file`) + `Folder Boundary Guard`**: `drive.file` 스코프만 요청하고 모든 조회 쿼리에 `'<targetFolderId>' in parents and trashed = false`를 강제하며, 쓰기/삭제 전 `parents` 불일치 시 `SecurityBoundaryError`를 발생시켰다.
2. **GIS 자동 로드 + 전용 폴더 자동 생성 + 인앱 모달**: `index.html` 및 동적 로더(`ensureGoogleGisLoaded`)로 GIS 팝업을 즉시 실행하고, Picker API Key 없이도 Drive API만으로 앱 전용 폴더를 자동 생성/연결(`ensureOrCreateDefaultSyncFolder`)하며, 폴더 변경은 인앱 모달(`listAppAccessibleFolders`)로 대체했다.
3. **충돌 분기 & Undo 보호**: 양방향 동시 수정 시 `(충돌 사본 ...)`으로 분기하고, 지정 폴더 삭제 시 루트 재생성 대신 `SYNC_FOLDER_MISSING`으로 안전 중단하며, 원격 반영 시 `addToHistory: false`로 로컬 Undo 스택을 보호했다.

## [에이전트 방어 규칙 제안]
- 클라우드 드라이브 연동 시 전체 권한(`drive`) 대신 `drive.file` 최소 스코프와 부모 폴더 ID 경계 가드를 강제하고, 배포 안내 시 Google Cloud Console의 `Drive API` 활성화 및 OAuth 테스트 사용자 등록 체크리스트를 포함한다. (참조: `rules/byok-security-and-oauth.md`, `patterns/offline-search-and-local-sync.md`)

