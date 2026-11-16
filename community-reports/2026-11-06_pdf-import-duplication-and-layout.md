---
title: PDF 가져오기 시 상단 빈 공간·손글씨 중복 및 다중 페이지 오프셋 누적 오차 해결
major: 교육공학 / 공통
tags: [pdf, pdfjs, layout, offset, scroll-spy, page-source-of-truth, thumbnail]
date: 2026-11-06
---

# PDF 가져오기 시 상단 빈 공간·손글씨 중복 및 다중 페이지 오프셋 누적 오차 해결

## [문제 상황]
- PDF 파일을 노트로 가져왔을 때 (1) 각 페이지 상단 80%가 흰 빈 공간으로 비어 나오거나, (2) 페이지 하단에 손글씨 이미지가 한 번 더 중복 렌더링되고, (3) 페이지 추가 시 새 페이지가 보이지 않으며, (4) 썸네일 20쪽을 클릭하면 18~19쪽으로 빗나가고 하이라이트가 중간 페이지로 깜빡이며 되돌아가는 현상이 복합적으로 발생했다.

## [원인 분석]
1. **텍스트/이미지 누락 및 XObject 중복**: `pdf.js` 렌더링 파이프라인에 `operationsFilter`가 걸려 인쇄 텍스트(`TEXT_OPS`)와 이미지(`IMAGE_OPS`)가 지워졌고, 반대로 `page.render()`가 이미 그린 손글씨 XObject 비트맵을 별도 `{ type: 'image' }` 노드로 한 번 더 추출해 중복 마운트했다.
2. **페이지 진실 소스 이원화**: 페이지 추가 시 UI 페이지 메타데이터만 늘리고 에디터 Live 문서(`editor.state.doc`)에 페이지 템플릿 노드를 삽입하지 않아 화면에 빈 페이지가 나타나지 않았다.
3. **추정 오프셋 누적 오차 & 스파이 경합**: 메타데이터 치수로만 좌표를 추정해 DOM의 border/배지/패딩 오차(페이지당 6~10px)가 20페이지에서 120~200px로 누적되었다. 또한 프로그램 smooth scroll 도중 스크롤 스파이가 지나가는 중간 페이지로 활성 페이지 인덱스를 덮어써 하이라이트가 요동쳤다.

## [해결 패턴]
1. **`pdf.js` 필터 제거 & 단일 마운트**: `operationsFilter`와 중복 XObject 이미지 노드 생성을 제거해 페이지당 단일 PDF 페이지 템플릿 노드만 마운트하고, 추출 텍스트는 검색용 `plainText` 필드에만 저장한다.
2. **페이지 진실 소스 단일화**: 모든 페이지 추가·삭제·재정렬을 단일 세션 메서드에서 TipTap 트랜잭션(`tr.insert`/`tr.delete` + `1..N` 및 `pageId` 불변식 동기화)과 UI 페이지 불변 배열 교체로 일원화하고, 가로/세로 혼합 PDF는 대표 폭을 컨테이너 기준으로 고정한다.
3. **DOM 실측 & 내비게이션 상태 기계 분리**: 스크롤 목표와 스파이 판정에는 DOM 실측(`offsetTop`/`offsetHeight` + `ResizeObserver`)을 사용하고, 목표 페이지(`target`)와 관찰 페이지(`visible`) 상태 기계를 분리해 프로그램 스크롤 중 스파이가 활성 페이지 하이라이트를 덮어쓰지 못하게 격리했다.

## [에이전트 방어 규칙 제안]
- 다중 페이지 스크롤 좌표 계산에는 메타데이터 추정치나 임의 보정 상수(`+ 8 * index`)를 쓰지 말고 반드시 DOM 실측(`offsetTop`/`offsetHeight`)과 `ResizeObserver`를 사용하며, 클릭 내비게이션 목표 상태(`target`)와 스크롤 스파이 관찰 상태(`visible`)를 분리한다. (참조: `patterns/pdf-page-layout-and-editor.md`, `patterns/pptx-client-export.md`)

