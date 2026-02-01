---
title: OpenClaw Agent Loop (에이전트 루프)
date: 2026-02-04
authors:
  - '0x68756E61'
categories:
  - Agent Loop
tags:
  - openclawd
  - clawdbot
---

![](../../assets/images/openclaw-agent-loop-hero.png)

> summary:

<!-- more -->

# 2026 02 04 Openclaw Agent Loop

summary: 


에이전트 루프(Agent Loop)

에이전트 루프는 에이전트의 전체 "실행"입니다: 입력 → 컨텍스트 조합 → 모델 추론 → 도구 실행 → 스트리밍 응답 → 영속화. 메시지를 액션과 최종 응답으로 바꾸는 권위 있는 경로이며 세션 상태의 일관성을 유지합니다.



<!-- more -->





# OpenClaw Agent Loop (에이전트 루프)



에이전트 루프(Agent Loop)

에이전트 루프는 에이전트의 전체 "실행"입니다: 입력 → 컨텍스트 조합 → 모델 추론 → 도구 실행 → 스트리밍 응답 → 영속화. 메시지를 액션과 최종 응답으로 바꾸는 권위 있는 경로이며 세션 상태의 일관성을 유지합니다.

OpenClaw에서 루프는 각 세션마다 단일 직렬화된 실행으로 동작하며, 모델이 사고하고 도구를 호출하며 출력을 스트리밍하는 동안 라이프사이클 및 스트림 이벤트를 방출합니다.

진입점

- Gateway RPC: agent 및 agent.wait
- CLI: agent 명령

작동 방식(개요)

- agent RPC는 파라미터를 검증하고 세션을 해결(sessionKey/sessionId), 세션 메타데이터를 영속화한 후 즉시 { runId, acceptedAt }을 반환합니다.
- agentCommand는 에이전트를 실행합니다:
  - 모델 및 thinking/verbose 기본값 해석
  - 스킬 스냅샷 로드
  - runEmbeddedPiAgent 호출
  - 내장 루프가 라이프사이클 end/error를 방출하지 않으면 라이프사이클 end/error를 방출

runEmbeddedPiAgent

- 세션별 및 전역 큐를 통해 실행을 직렬화
- 모델 및 인증 프로필 해결하고 pi 세션 구성
- pi 이벤트를 구독하고 assistant/tool 델타를 스트리밍
- 타임아웃을 강제 적용하여 초과 시 실행 중단
- 페이로드 및 사용량 메타데이터 반환

구독(subscribeEmbeddedPiSession)

- pi-agent-core 이벤트를 OpenClaw 에이전트 스트림으로 브리지:
  - 도구 이벤트 → tool 스트림
  - 어시스턴트 델타 → assistant 스트림
  - 라이프사이클 이벤트 → lifecycle 스트림(phase: start | end | error)

agent.wait

- runId의 라이프사이클 end/error를 대기
- { status: ok|error|timeout, startedAt, endedAt, error? } 반환

큐잉 및 동시성

- 실행은 세션 키(세션 레인)별로 직렬화되며 선택적으로 전역 레인을 통해 직렬화됩니다.
- 이는 도구/세션 레이스를 방지하고 세션 히스토리의 일관성을 보장합니다.
- 메시징 채널은 collect/steer/followup 같은 큐 모드를 선택할 수 있으며 이들이 레인 시스템에 입력됩니다.

세션 및 워크스페이스 준비

- 워크스페이스를 해결 및 생성; 샌드박스 실행은 샌드박스 워크스페이스 루트로 리디렉션될 수 있음
- 스킬 로드(또는 스냅샷 재사용) 및 환경/프롬프트에 주입
- 부트스트랩/컨텍스트 파일 해결 및 시스템 프롬프트에 주입
- 세션 쓰기 잠금 획득; SessionManager 오픈 및 스트리밍 준비

프롬프트 조립 및 시스템 프롬프트

- 시스템 프롬프트는 OpenClaw의 기본 프롬프트, 스킬 프롬프트, 부트스트랩 컨텍스트, 런별 오버라이드로 구성됩니다.
- 모델별 제한 및 압축을 적용하여 토큰을 확보합니다.

후킹 포인트(Hook points)

- OpenClaw에는 내부 훅(Gateway hooks)과 플러그인 훅 두 가지가 있음
- 내부 훅: agent:bootstrap 등, 명령/라이프사이클 이벤트 중에 실행
- 플러그인 훅: before_agent_start, agent_end, before_tool_call 등 다양한 라이프사이클 지점에 삽입 가능

도구 실행 및 메시징 도구

- 도구 시작/업데이트/종료 이벤트는 tool 스트림에 방출
- 도구 결과는 로깅/방출 전에 크기와 이미지 페이로드를 정리
- 메시징 도구 전송은 중복 어시스턴트 확인을 억제하도록 추적

응답 형성 및 억제

- 최종 페이로드는 어시스턴트 텍스트, 인라인 도구 요약, 오류 텍스트로 구성
- NO_REPLY는 무음 토큰으로 처리되어 송출에서 필터링
- 렌더 가능한 페이로드가 남지 않고 도구 에러가 발생하면 폴백 에러 응답을 생성

컴팩션 및 재시도

- 자동 컴팩션은 컴팩션 스트림 이벤트를 방출하고 재시도를 촉발할 수 있음
- 재시도 시 중복 출력을 피하기 위해 인메모리 버퍼와 도구 요약을 리셋

이벤트 스트림 요약

- lifecycle, assistant, tool 스트림이 주요 스트림임

타임아웃

- agent.wait 기본: 30초
- 에이전트 런타임 기본 타임아웃: agents.defaults.timeoutSeconds 기본 600초 (runEmbeddedPiAgent에서 강제)

그 외 종료 조건: 에이전트 타임아웃, AbortSignal, Gateway 연결 끊김, agent.wait 타임아웃