---
title: OpenClaw Subagents 가이드 - sessions_spawn과 멀티 에이전트 설정
date: 2026-02-04
authors:
  - '0x68756E61'
categories:
  - openclaw
tags:
  - openclaw
  - ai-agent
  - subagents
  - automation
---

![](../../assets/images/openclaw-subagents-guide.png)

> OpenClaw의 강력한 기능 중 하나는 서브 에이전트(Sub-agents)를 활용한 태스크 위임과 정교한 멀티 에이전트 라우팅입니다. `sessions_spawn` 툴을 사용하여 복잡한 작업을 분산 처리하는 방법을 알아봅니다.

<!-- more -->

---

# OpenClaw Subagents & Multi-Agent Guide

OpenClaw는 단순한 챗봇 그 이상입니다. 여러 독립적인 에이전트를 구성하고, 각 에이전트가 서브 에이전트를 호출하여 협업하게 함으로써 고도의 자동화 워크플로우를 구축할 수 있습니다.

## 1. sessions_spawn: 서브 에이전트 실행

`sessions_spawn`은 현재 세션에서 새로운 독립 세션을 생성하고 서브 에이전트에게 특정 작업을 할당하는 핵심 도구입니다.

### 주요 파라미터
- **task**: 서브 에이전트가 수행할 작업 내용 (필수)
- **agentId**: 실행할 에이전트의 ID (옵션)
- **model**: 사용할 모델 오버라이드 (옵션)
- **runTimeoutSeconds**: 실행 제한 시간 설정
- **cleanup**: 완료 후 세션 삭제(`delete`) 또는 유지(`keep`) 여부

### 활용 예시
```json
{
  "action": "sessions_spawn",
  "task": "최근 5개 메시지의 내용을 분석하여 기술적 이슈만 요약해줘.",
  "agentId": "analyst-agent"
}
```

---

## 2. 멀티 에이전트 구성 (agents.list)

OpenClaw Gateway 설정 파일에서 `agents.list`를 통해 여러 에이전트를 정의할 수 있습니다. 각 에이전트는 자신만의 워크스페이스와 모델 설정을 가집니다.

```json5
{
  agents: {
    list: [
      {
        id: "main",
        workspace: "~/.openclaw/workspace-main"
      },
      {
        id: "analyst",
        workspace: "~/.openclaw/workspace-analyst",
        model: "anthropic/claude-3-5-sonnet"
      }
    ]
  }
}
```

---

## 3. 서브 에이전트 허용 목록 (Allowlist)

보안과 리소스 관리를 위해, 특정 에이전트가 호출할 수 있는 서브 에이전트를 제한할 수 있습니다. 이는 `subagents.allowAgents` 설정을 통해 제어됩니다.

```json5
{
  id: "main",
  subagents: {
    allowAgents: ["analyst", "summarizer"] // 특정 에이전트만 호출 가능
    // "allowAgents": ["*"] // 모든 에이전트 호출 허용
  }
}
```

---

## 4. 메시지 라우팅 (Bindings)

인바운드 메시지를 특정 채널(Telegram, WhatsApp 등)이나 특정 사용자로부터 특정 에이전트에게 전달되도록 규칙을 정할 수 있습니다.

- **match.channel**: 메시지가 들어온 채널
- **match.peer**: 특정 사용자 또는 그룹 ID
- **agent**: 연결할 에이전트 ID

---

## 결론

OpenClaw의 서브 에이전트와 라우팅 기능을 활용하면, 한 명의 비서가 아닌 '팀 단위'의 AI 시스템을 구축할 수 있습니다. 각 분야에 특화된 에이전트들을 배치하고 필요할 때마다 `sessions_spawn`으로 호출하여 효율을 극대화해 보세요!
