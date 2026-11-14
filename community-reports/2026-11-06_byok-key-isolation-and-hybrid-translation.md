---
title: 동일 공급사(Google) AI 키와 번역 API 키 충돌 해결 및 카드 없는 3-Tier 하이브리드 번역 설계
major: 외국어교육 / 공통
tags: [byok, api-key, translation, gemini, adaptive-glossary, ruby, zero-config]
date: 2026-11-06
---

# 동일 공급사(Google) AI 키와 번역 API 키 충돌 해결 및 카드 없는 3-Tier 하이브리드 번역 설계

## [문제 상황]
- Google Gemini AI 키가 이미 등록되어 있는데도 번역 기능에서 키 충돌 및 오작동이 발생하거나, 반대로 결제 신용카드 등록이 필수인 Google Cloud Translation API 키를 요구해 일반 학습자가 앱 설치 직후 번역 기능을 전혀 쓰지 못하는 장벽이 존재했다.

## [원인 분석]
1. **스토리지 식별자 공유**: Gemini 학술 튜터 키와 Google Translate 키가 동일한 `'google'` 공급자 ID를 공유해 서로의 키를 오인하거나 덮어쓸 위험이 있었다.
2. **유료 결제 카드 장벽**: GCP Cloud Translation v2는 결제 계정 연결이 필수라 교육 현장의 학생·교사가 즉시 발급해 쓰기 어려웠다.

## [해결 패턴]
1. **서비스별 키 식별자 완전 격리**: AI 튜터(`ai:google`)와 번역 전용(`translate:google`)의 저장·조회·삭제 API 및 환경변수 폴백을 완전히 분리했다.
2. **3-Tier Zero-Config + Card-Free BYOK 아키텍처**:
   - **Tier 1 (0ms 오프라인)**: 로컬 적응형 사전(공기어 스코어링, 복수형 stem 정규화, IndexedDB 사용자 학습 승격)으로 신뢰도 `>= 70%` 단일 단어를 즉시 처리(다의어 도메인 `baseWeight`는 동등하게 중립화).
   - **Tier 2 (무료 BYOK)**: 신용카드 없는 Google AI Studio 무료 Gemini 키로 문맥 번역 수행, 키 오류 시 중단 없이 Tier 3으로 자동 폴백.
   - **Tier 3 (무키 폴백)**: 키 미등록 시에도 공개 경량 번역 엔드포인트로 즉시 동작하며, 삽입 방식(`parentheses` 괄호 / `ruby` 루비 / `replace` 대체)을 선택 가능하게 했다.

## [에이전트 방어 규칙 제안]
- 동일 벤더라도 엔드포인트/서비스가 다르면 API 키 스토리지 ID를 반드시 분리하고, 교육용 웹앱의 기본 번역·보조 기능은 결제 카드나 키 등록 없이도 즉시 동작하는 Zero-Config 폴백 경로를 기본 제공한다. (참조: `rules/byok-security-and-oauth.md`, `patterns/llm-byok-tutor.md`, `majors/lang-edu.md`)

