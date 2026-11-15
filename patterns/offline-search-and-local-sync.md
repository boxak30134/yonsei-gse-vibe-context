---
title: 순수 JS 오프라인 하이브리드 검색·문서 연결 및 Local-First 격리 동기화 패턴
tags: [offline-search, bm25, tfidf, wikilink, local-first, dexie, opfs, google-drive, sync]
updated: 2026-11-13
---

# 오프라인 하이브리드 검색·문서 연결 및 Local-First 격리 동기화 패턴 (`offline-search-and-local-sync.md`)

네트워크나 외부 LLM API 토큰을 소모하지 않고 노트 간 지식 연결을 실시간 추천하며, 로컬(IndexedDB/OPFS) 우선 아키텍처를 유지한 채 클라우드 지정 폴더와 안전하게 양방향 동기화하는 패턴입니다.

---

## 1. 순수 JS 오프라인 하이브리드 검색·문서 연결

### 1.1 3단계 하이브리드 토크나이저
1. **$\LaTeX$ 수식 명령어 보존**: `\\[a-zA-Z]{2,}` 패턴을 `math:\cmd` 독립 토큰으로 추출해 수식 구조가 유사한 문서를 연결한다.
2. **불용어 제거 + 한국어 2단계 조사 제거**: 어절 끝의 복합 조사·단일 조사를 제거해 어간을 정규화한다.
3. **한글 2-gram 보조 토큰**: 3음절 이상 한글 단어에 Bi-gram 토큰을 추가 생성해 부분 일치 검색을 보강한다.

### 1.2 이중 랭킹 및 위키링크 즉시 치환
- **선택 구절 기반 추천**: BM25($k_1=1.5, b=0.75$) + 제목 직접 포함 보너스(`+5.0`)로 500개 문서 기준 `< 10ms` 내 상위 5개 연관 문서와 문맥 스니펫을 반환한다. 클릭 시 `[[문서제목|선택 구절]]`로 즉시 치환한다.
- **연관 문서 추천**: TF-IDF 코사인 유사도(`0.7`) + `#태그` Jaccard 유사도(`0.2`) + `[[위키링크]]` 보너스(`0.25`)를 결합한다.
- **성능 및 WebView 폴백**: 변경 플래그(`isDirty`)로 지연 리빌드를 수행하고, Web Worker 초기화 실패/타임아웃(Android WebView 등) 시 메인 스레드 인메모리 엔진으로 자동 폴백한다.

---

## 2. Local-First (Dexie + OPFS) & 지정 폴더 격리 동기화

### 2.1 최소 권한 및 폴더 경계 격리 (`Folder Boundary Guard`)
- OAuth 스코프는 `https://www.googleapis.com/auth/drive.file`만 허용하며 `drive` / `drive.readonly`는 차단한다.
- 모든 원격 조회 쿼리에 `'<targetFolderId>' in parents and trashed = false`를 강제하고, 쓰기/삭제 전 `parents` 검증으로 경계 이탈 시 보안 경계 오류를 발생시킨다.
- **API Key 없는 UX 단순화**: Google Identity Services(GIS) SDK 자동 로드 + 로그인 직후 앱 전용 폴더 자동 생성/연결 + 인앱 폴더 선택 모달을 결합해 Google Picker API Key 없이도 원클릭 연동을 지원한다. (Google Cloud Console에서 `Drive API` 활성화 필수)

### 2.2 증분 동기화 및 충돌·Undo 보호
- 로컬 편집은 즉시 Dexie(IndexedDB)와 OPFS(`images/` 바이너리)에 기록하고 동기화 대기 플래그(`syncDirty = true`)를 마킹하며, 오프라인 편집분은 `online` 이벤트 복귀 시 자동 재시도한다.
- **양방향 충돌 분기**: 로컬 수정 시각과 원격 수정 시각이 모두 마지막 동기화 시각(`lastSyncedAt`)보다 최신이면 덮어쓰지 않고 `<문서제목> (충돌 사본 YYYYMMDD-HHmm).md`로 분기 저장한다.
- **Undo 스택 보호**: 원격 변경분을 현재 열린 문서에 반영할 때는 반드시 히스토리 제외 옵션(`addToHistory: false`)을 사용해 사용자의 로컬 `Ctrl+Z` 히스토리를 오염시키지 않는다.

