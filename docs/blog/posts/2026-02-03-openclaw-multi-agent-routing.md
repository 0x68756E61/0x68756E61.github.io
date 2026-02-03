---
title: OpenClaw Multi-Agent Routing (멀티-에이전트 라우팅)
date: 2026-02-03
authors:
  - '0x68756E61'
categories:
  - openclawd
tags:
  - openclawd
  - agent
---

![](../../assets/images/openclaw-multi-agent-routing-hero.png)

> 여러 개의 독립된 에이전트(각각 별도의 워크스페이스, agentDir, 세션)를 한 Gateway에서 운영하고, 채널 계정을 다수(예: 두 개의 WhatsApp)...

<!-- more -->



# 멀티-에이전트 라우팅 (Multi-Agent Routing)

목표: 여러 개의 독립된 에이전트(각각 별도의 워크스페이스, agentDir, 세션)를 한 Gateway에서 운영하고, 채널 계정을 다수(예: 두 개의 WhatsApp) 사용할 수 있게 하는 것입니다. 인바운드는 바인딩에 따라 특정 에이전트로 라우팅됩니다.

"하나의 에이전트"란?
에이전트는 다음을 갖춘 완전한 독립 단위입니다:
- 워크스페이스 (파일: AGENTS.md/SOUL.md/USER.md, 로컬 노트, 페르소나 규칙)
- 상태 디렉터리 (agentDir): 인증 프로필, 모델 레지스트리, 에이전트별 설정
- 세션 저장소: ~/.openclaw/agents/<agentId>/sessions 아래의 채팅 히스토리 및 라우팅 상태

인증 프로필은 에이전트별로 분리되어 있으며 자동으로 공유되지 않습니다. auth-profiles.json을 복사해 공유할 수는 있지만, agentDir을 재사용하면 인증/세션 충돌이 발생하므로 권장하지 않습니다.

스킬은 워크스페이스별로 존재하며(~/<workspace>/skills), 공용 스킬은 ~/.openclaw/skills에 둘 수 있습니다. Gateway는 기본적으로 하나의 에이전트를 호스팅하지만 여러 에이전트를 사이드바이로 운영할 수 있습니다.

워크스페이스 노트: 각 에이전트의 워크스페이스는 기본 작업 디렉터리(cwd)이지만 절대 경로는 호스트의 다른 위치에 접근할 수 있습니다. 샌드박싱을 활성화해 접근을 제한할 수 있습니다.

## 구성 예시 경로
- 설정: ~/.openclaw/openclaw.json
- 상태 디렉터리: ~/.openclaw
- 워크스페이스: ~/.openclaw/workspace 또는 ~/.openclaw/workspace-<agentId>
- agentDir: ~/.openclaw/agents/<agentId>/agent
- 세션: ~/.openclaw/agents/<agentId>/sessions

## 단일-에이전트 모드(기본)
- 아무 설정을 하지 않으면 기본 에이전트(main)가 실행됩니다. 세션 키는 agent:main:<mainKey> 형식으로 지정됩니다.

## 에이전트 헬퍼
- `openclaw agents add work`로 새 격리 에이전트를 추가하고 바인딩을 설정하거나 위저드를 사용해 라우팅을 구성할 수 있습니다.

## 멀티-에이전트의 의미
- 각 agentId는 별도의 페르소나입니다: 서로 다른 전화번호/계정, 서로 다른 AGENTS.md/SOUL.md 등의 워크스페이스 파일, 별도의 인증 및 세션 저장소.
- 이를 통해 하나의 Gateway 호스트에서 여러 사람이 각자의 AI 브레인을 가지도록 할 수 있습니다.

예: 하나의 WhatsApp 계정, 여러 사람(DM split)
- 서로 다른 발신자 번호(E.164, 예 +1555...)로 DM을 매칭해 각 에이전트로 라우팅할 수 있습니다.
- 단, 같은 WhatsApp 번호에서 응답하기 때문에 발신자 아이덴티티는 동일합니다; 완전한 격리를 원하면 에이전트별로 계정을 분리해야 합니다.

라우팅 규칙(메시지가 에이전트를 선택하는 방법)
- 바인딩은 결정론적이며 가장 구체적인 규칙이 우선합니다:
  1. peer match (exact DM/group id)
  2. guildId (Discord)
  3. teamId (Slack)
  4. accountId match for a channel
  5. channel-level match (accountId: "*")
  6. fallback to default agent

## 여러 계정/번호
- 채널이 복수 계정을 지원하면 accountId로 각 로그인(예: 개인, 비즈니스)을 식별할 수 있고, 각 accountId를 다른 에이전트로 라우팅할 수 있습니다.

## 샌드박스와 도구 제어
- 에이전트별로 샌드박스 모드와 도구 허용/차단 목록을 설정할 수 있습니다. 예: family 에이전트는 exec 권한을 비활성화하고 read만 허용.

## 요약
- 에이전트는 데이터·인증·세션을 분리하는 완전한 단위입니다. 라우팅은 바인딩 규칙을 통해 결정되고, 다중 에이전트 환경은 하나의 Gateway 서버에서 여러 계정/페르소나를 안전하게 운영할 수 있게 해 줍니다.
