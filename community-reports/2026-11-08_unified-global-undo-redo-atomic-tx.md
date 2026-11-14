---
title: 다중 레이어(필기·텍스트·주석·페이지) 통합 글로벌 Undo/Redo 원자적 트랜잭션 패턴
major: 교육공학 / 공통
tags: [undo-redo, history, atomic-transaction, snapshot, stroke, page-delete]
date: 2026-11-08
---

# 다중 레이어 통합 글로벌 Undo/Redo 원자적 트랜잭션 패턴

## [문제 상황]
- 필기 캔버스, 인라인 텍스트 객체, 리치 에디터 본문, 말풍선 주석, 페이지 추가/삭제가 별도 모듈로 동작하면서 (1) 페이지 삭제 시 소속 필기·텍스트가 한 번의 `Ctrl+Z`로 함께 복구되지 않고, (2) 빈 텍스트 박스 생성이나 문서 초기 마운트만으로도 `canUndo`가 활성화되는 문제가 발생했다.

## [원인 분석]
1. **하위 스토어 개별 히스토리 중복 푸시**: 페이지 삭제 시 페이지 삭제, 스트로크 삭제, 후속 페이지 스트로크 Y좌표 시프트가 각각 별도 히스토리를 쌓아 Undo를 여러 번 눌러야 하는 불일치가 생겼다.
2. **초기 마운트 및 빈 입력 오염**: TipTap 초기화 트랜잭션이 히스토리에 포착되거나, 인라인 텍스트 입력 시작 시점에 미리 히스토리가 푸시되어 빈 입력 취소 후에도 Undo 스택이 남았다.

## [해결 패턴]
1. **단일 `CompositeHistoryManager` + `raw*` 원자적 스냅샷**: 페이지 삭제 등 복합 작업에서는 하위 스토어의 히스토리 기록 없는 `rawRemove`, `rawTranslate`, `rawAdd` 메서드를 호출하고, `{ targetPage, deletedStrokes, deletedTextObjects, shiftStrokeIds, shiftY, templateAttrs }` 전체 스냅샷을 **단 1개의 `HistoryCommand`**로 묶어 1회 Undo로 완벽 복원되게 했다.
2. **완료 시점 기록 & 초기 클린 강제**: 인라인 텍스트는 의미 있는 내용 입력 완료 시점에만 `recordTextAdd`를 1회 푸시하고 빈 입력은 `recordHistory: false`로 무음 폐기하며, 세션 마운트 직후 `this.history.clear()`를 호출해 초기 `canUndo: false`를 보장했다.

## [에이전트 방어 규칙 제안]
- 여러 스토어를 동시에 변경하는 복합 액션은 절대 하위 스토어의 일반 메서드를 연쇄 호출하지 말고 `raw*` API + 단일 `HistoryCommand` 스냅샷으로 원자화한다. (참조: `patterns/pdf-page-layout-and-editor.md`)

