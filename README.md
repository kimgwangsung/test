# 차세대 AI 운영체제 프롬프트 (Next-Generation AI Operating System Prompt)

이 저장소는 프로덕션 환경의 Claude Fable 5 (claude.ai 컨슈머용) 시스템 프롬프트를
완전히 역설계(리버스 엔지니어링)하고, 그 후속 버전을 설계한 결과물을 담고 있습니다.
후속 버전은 계층화된(layered) 모델 독립적(model-agnostic) "AI 운영체제" 프롬프트로,
장기 실행(long-horizon) 에이전트 작업, 대규모 코드베이스 소프트웨어 엔지니어링,
리서치에 최적화되어 있으며, 동시에 Claude Fable 5에 맞게 튜닝되어 있습니다.

## 구성

| 파일 | 산출물 |
|------|--------|
| `01-architecture-analysis.md` | Part 1 — 원본 프롬프트의 아키텍처 분석; Part 2 — 약점 보고서; Part 3 — 개선 전략 |
| `02-universal-master-os-prompt.md` | Part 4 — 범용 마스터 AI 운영체제 프롬프트 (모델 독립적; Claude, GPT-5, Codex, Gemini CLI, Antigravity, RooCode, OpenHands, Aider, Cline에 배포 가능) |
| `03-claude-fable-5-os-prompt.md` | Part 5 — Claude Fable 5 최적화 AI 운영체제 프롬프트 |

## 설계 요약

원본 프롬프트는 약 42k 토큰 규모의 모놀리식(monolithic) 프롬프트로, 다음 다섯 가지
기법 덕분에 효과를 발휘합니다: 추상적 가치 대신 의사결정 절차(decision procedure)로
행동을 명세하는 방식, 핵심 하드 리밋(hard limit)의 의도적 반복, 스킬(skill) 파일을
통한 점진적 공개(progressive disclosure), 긍정/부정 예시의 쌍(pair) 제공, 그리고
메타인지 트립와이어(meta-cognitive tripwire). 반면 다음과 같은 실패 요인도 안고
있습니다: 통제되지 않은 중복(토큰의 약 15–20% 낭비), 복사-편집 드리프트(copy drift)로
인한 규칙 간 충돌, 하드코딩된 가변(volatile) 정보, 에이전트 서브시스템의 부재
(검증/복구 루프 없음, 장기 실행 상태 관리 없음, 엔지니어링 표준 없음), 그리고
하나의 제품 표면(surface) 밖에서는 재사용할 수 없는 구조.

후속 버전은 다섯 가지 기법은 유지하고, 실패 요인은 제거하며, 누락된 서브시스템을
작은 커널 위의 조합 가능한(composable) 모듈로 추가합니다. 우선순위(precedence)는
명시적으로 선언됩니다:

```
L0  커널(Kernel)      정체성, 우선순위, 안전 불변식(invariants)        (항상 로드)
L1  추론(Reasoning)   분해, 계획, 검증, 복구                           (항상 로드)
L2  모듈(Modules)     swe / research / files / connectors / memory     (배포 단위별)
L3  도구 계층(Tool)   라우팅 규칙; 스키마는 하네스(harness)에 위치      (배포 단위별)
L4  세션(Session)     날짜, 사용자 컨텍스트, 환경 정보                  (세션별)
TAIL 요약(Recap)      최근성(recency) 윈도우에 하드 리밋 앵커 재배치    (항상 로드)
```

안전(safety) 관련 내용은 전부 원래 강도 그대로 보존됩니다 — 중복만 제거했을 뿐,
약화시키지 않았습니다.
