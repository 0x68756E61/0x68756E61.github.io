---
title: OpenClaw 멀티 에이전트 완벽 가이드
date: 2026-02-08
authors:
  - '0x68756E61'
categories:
  - OpenClaw
  - Automation
tags:
  - Multi-Agent
  - Clawdbot
  - Sandbox
  - LLM
---

![](../../assets/images/2026-02-08-openclaw-multi-agents.png)

> LEAD: OpenClaw(Clawdbot 기반) 환경에서 멀티 에이전트를 생성하고 구성하는 방법을 설명합니다. 단일 게이트웨이 내에서 서로 다른 성격, 권한, 작업 공간을 가진 여러 에이전트를 실행하여 효율적인 협업 팀을 구축하는 방법을 다룹니다.

<!-- more -->

# OpenClaw Multi-Agent Creation Guide

OpenClaw (Clawdbot 기반) 환경에서 멀티 에이전트를 생성하고 구성하는 방법을 설명합니다.

---

## 1. 개요 (Overview)

OpenClaw의 멀티 에이전트 시스템을 사용하면 단일 게이트웨이 내에서 서로 다른 **성격(Persona)**, **권한(Permissions)**, **작업 공간(Workspace)**을 가진 여러 에이전트를 실행할 수 있습니다.

!!! info "에이전트 구성 사례"
    - **개인 비서**: 모든 권한을 가진 메인 에이전트
    - **가족/동료용**: 제한된 도구만 사용하는 에이전트
    - **공용/테스트용**: 샌드박스(Docker) 내에서만 실행되는 안전한 에이전트

---

## 2. 에이전트 정의 (Defining Agents)

멀티 에이전트는 설정 파일의 `agents.list` 배열에 정의합니다. 각 에이전트는 고유한 `id`를 가집니다.

### 기본 구조 및 모델 설정

```json title="config.json" linenums="1"
{
  "agents": {
    "list": [
      {
        "id": "main",           // 에이전트 고유 ID
        "default": true,        // 기본 에이전트 여부
        "name": "Main Assistant",
        "model": "anthropic/claude-3-5-sonnet-20240620", // 모델 지정 (Provider/ModelID)
        "workspace": "~/clawd", // 에이전트 전용 작업 공간
        "sandbox": { "mode": "off" }
      },
      {
        "id": "dev_bot",
        "name": "Developer Bot",
        "model": {
            "primary": "openai/gpt-4o",
            "fallbacks": ["openai/gpt-4o-mini"]
        },
        "workspace": "~/clawd-dev"
      }
    ]
  }
}
```

!!! tip
    모델 설정에 대한 더 자세한 내용은 `how-to-set-models.md` 문서를 참고하세요.

---

## 3. 샌드박스 설정 (Sandbox Configuration)

각 에이전트별로 샌드박스(격리 환경) 적용 여부를 설정하여 보안을 강화할 수 있습니다.

- **mode**:
    - `"off"`: 호스트 머신에서 직접 실행 (높은 권한, 메인 에이전트용)
    - `"all"`: 항상 Docker 컨테이너 내에서 실행 (안전함)
    - `"non-main"`: 메인 세션이 아닐 경우에만 샌드박스 적용 (기본값)
- **scope**:
    - `"agent"`: 해당 에이전트 전용 컨테이너 생성 (데이터 격리)
    - `"session"`: 세션마다 새로운 컨테이너 생성 (휘발성)
    - `"shared"`: 여러 에이전트가 동일한 샌드박스 공유

```json title="sandbox configuration"
"sandbox": {
  "mode": "all",
  "scope": "agent"
}
```

---

## 4. 도구 권한 제어 (Tool Restrictions)

에이전트가 사용할 수 있는 도구(Tool)를 제한하여 역할을 명확히 할 수 있습니다.

- **allow**: 허용할 도구 목록
- **deny**: 차단할 도구 목록 (allow보다 우선순위 높음)
- **profile**: 도구 프리셋 (`coding`, `messaging` 등)

### 예시 설정

=== "읽기 전용 에이전트"
    ```json
    "tools": {
      "allow": ["read", "memory_search"],
      "deny": ["write", "edit", "exec", "browser"]
    }
    ```

=== "메시징 전용 에이전트"
    ```json
    "tools": {
      "profile": "messaging",
      "allow": ["slack", "telegram"]
    }
    ```

---

## 5. 전체 설정 예시 (Full Example)

다음은 '메인 에이전트'와 '보안 리서치 에이전트'를 함께 운영하는 설정 예시입니다.

!!! example "Full Configuration"
    ```json title="multi-agent-example.json"
    {
      "agents": {
        "defaults": {
          "sandbox": { "mode": "non-main" }
        },
        "list": [
          {
            "id": "master",
            "default": true,
            "name": "Master Control",
            "workspace": "~/openclaw-master",
            "sandbox": { "mode": "off" }
          },
          {
            "id": "researcher",
            "name": "Safe Researcher",
            "workspace": "~/openclaw-research",
            "sandbox": {
              "mode": "all",
              "scope": "agent"
            },
            "tools": {
              "allow": ["read", "web_search", "browser", "memory_search"],
              "deny": ["exec", "write", "gateway"]
            }
          }
        ]
      }
    }
    ```

---

## 6. 적용 및 확인

설정 변경 후 OpenClaw 게이트웨이를 재시작하거나 설정을 적용해야 합니다.

1.  **설정 적용**: `clawdbot gateway config.apply`
2.  **에이전트 목록 확인**: `clawdbot agents list`
3.  **샌드박스 상태 확인**: `clawdbot sandbox status`

---

# OpenClaw Multi-Agent Channel Bindings

OpenClaw (Clawdbot) 환경에서 각 에이전트와 외부 채널(Telegram, Slack, WhatsApp 등)을 연결하는 방법을 설명합니다.

---

## 1. 바인딩(Bindings) 개요

OpenClaw에서 바인딩은 "누가 어디서 말을 걸었을 때, 어떤 에이전트가 응답할지"를 결정하는 규칙입니다.
설정 파일(`config.json` 또는 `config/` 디렉토리 내 파일)의 `bindings` 배열에 정의합니다.

### 기본 구조
```json title="binding example"
{
  "bindings": [
    {
      "agentId": "support",  // 연결할 에이전트 ID
      "match": {
        "provider": "telegram", // 채널 플러그인 이름
        "peer": {
          "id": "123456789"     // 사용자 또는 그룹 ID
        }
      }
    }
  ]
}
```

---

## 2. 매칭 규칙 (Match Rules)

`match` 객체는 들어오는 메시지의 메타데이터와 대조하여 에이전트를 선택합니다.

- **provider**: (필수) 메시지 제공자 (`telegram`, `slack`, `whatsapp`, `discord` 등)
- **accountId**: (옵션) 봇 계정 ID (여러 봇을 운영할 경우 구분)
- **peer**: (옵션) 상대방(유저/그룹) 정보
    - `kind`: `user` (개인톡) 또는 `group` (그룹챗)
    - `id`: 상대방의 고유 ID (전화번호, 유저 ID 등)
    - `name`: (주의) 이름은 변경될 수 있으므로 ID 사용 권장

!!! note "와일드카드(*) 사용"
    `*`를 사용하여 모든 경우를 매칭할 수 있습니다.
    ```json
    "match": {
      "provider": "slack",
      "peer": {
        "kind": "group",
        "id": "*" // 모든 슬랙 그룹 채널
      }
    }
    ```

---

## 3. 채널별 설정 예시

=== "Telegram (텔레그램)"
    특정 텔레그램 그룹을 '뉴스 요약 에이전트'에 연결합니다.
    ```json
    {
      "agentId": "news_bot",
      "match": {
        "provider": "telegram",
        "peer": {
          "kind": "group",
          "id": "-1001234567890" // 그룹 ID (보통 음수로 시작)
        }
      }
    }
    ```

=== "WhatsApp (왓츠앱)"
    특정 사용자의 1:1 메시지를 '개인 비서 에이전트'에 연결합니다.
    ```json
    {
      "agentId": "secretary",
      "match": {
        "provider": "whatsapp",
        "peer": {
          "kind": "user",
          "id": "821012345678@s.whatsapp.net"
        }
      }
    }
    ```

=== "Slack (슬랙)"
    슬랙의 특정 채널(`#general`) 메시지를 '공지 봇'에 연결합니다.
    ```json
    {
      "agentId": "notice_bot",
      "match": {
        "provider": "slack",
        "peer": {
          "id": "C012345ABCD"
        }
      }
    }
    ```

---

## 4. 우선순위 (Priority)

바인딩은 **위에서 아래로** 순차적으로 검사됩니다. 가장 먼저 매칭된 규칙이 적용됩니다.
따라서 **구체적인 규칙을 위에**, **일반적인 규칙(와일드카드)을 아래에** 배치해야 합니다.

!!! warning "잘못된 예시 (모든 메시지가 'main'으로 감)"
    ```json
    "bindings": [
      { "agentId": "main", "match": { "provider": "*" } },
      { "agentId": "special", "match": { "provider": "telegram" } } // 도달 불가능
    ]
    ```

!!! success "올바른 예시"
    ```json
    "bindings": [
      { 
        "agentId": "vip_support", 
        "match": { "provider": "telegram", "peer": { "id": "VIP_USER_ID" } } 
      },
      { 
        "agentId": "general_support", 
        "match": { "provider": "telegram" } 
      }
    ]
    ```

---

## 5. 설정 확인 및 디버깅

설정 적용 후 다음 명령어로 바인딩이 올바르게 인식되었는지 확인합니다.

1.  **현재 바인딩 목록 확인**:
    ```bash
    clawdbot agents list --bindings
    ```
2.  **로그 모니터링**:
    메시지가 올 때 어떤 에이전트로 라우팅되는지 확인합니다.
    ```bash
    tail -f ~/.clawdbot/logs/gateway.log | grep "routing"
    ```

---

# OpenClaw Multi-Agent LLM Model Configuration

OpenClaw (Clawdbot) 환경에서 각 에이전트별로 사용할 LLM 모델을 설정하는 방법을 설명합니다.

---

## 1. 개요 (Overview)

OpenClaw에서는 전체 기본 모델을 설정할 수도 있고, 각 에이전트마다 서로 다른 모델을 지정하여 비용과 성능을 최적화할 수 있습니다.

- **Main Agent**: 고성능 모델 (e.g. Claude 3.5 Sonnet, GPT-4o)
- **Sub Agent (Simple)**: 빠르고 저렴한 모델 (e.g. GPT-4o-mini, Claude 3 Haiku)
- **Coding Agent**: 코딩 특화 모델 (e.g. Claude 3.5 Sonnet)

---

## 2. 모델 설정 방법 (Model Configuration)

설정 파일(`config.json` 또는 `config/` 내 파일)의 `agents.list` 또는 `agents.defaults` 섹션에서 `model` 속성을 사용합니다.

### 2.1. 문자열 방식 (Simple String)
가장 간단한 방법으로, 모델 ID를 문자열로 지정합니다. 이 경우 해당 모델이 `primary` 모델로 설정됩니다.

```json
"model": "anthropic/claude-3-5-sonnet-20240620"
```

### 2.2. 객체 방식 (Advanced Object)
주 모델(`primary`)과 장애 시 사용할 대체 모델(`fallbacks`)을 상세하게 설정할 수 있습니다.

```json
"model": {
  "primary": "openai/gpt-4o",
  "fallbacks": ["anthropic/claude-3-haiku-20240307"]
}
```

---

## 3. 설정 레벨 및 우선순위 (Priority)

모델 설정은 다음 순서로 적용됩니다 (하위 설정이 상위 설정을 덮어씁니다).

1.  **Global Default** (`agents.defaults.model`)
2.  **Agent Specific** (`agents.list[].model`)
3.  **Session Specific** (런타임 중 `session_status` 도구 등으로 변경 시)

!!! example "계층적 설정 예시"
    ```json
    {
      "agents": {
        "defaults": {
          "model": "openai/gpt-4o-mini" // 기본값: 저렴한 모델
        },
        "list": [
          {
            "id": "boss",
            "model": "anthropic/claude-3-5-sonnet-20240620" // 보스만 좋은 모델
          },
          {
            "id": "intern" // model 설정 없음 -> 기본값(gpt-4o-mini) 사용
          }
        ]
      }
    }
    ```

---

## 4. 서브 에이전트 (Sub-agent) 모델 설정

`sessions_spawn` 도구를 사용하여 동적으로 생성되는 서브 에이전트의 모델은 도구 호출 시 인자(`model`)로 직접 지정합니다.

```json title="tool call example"
{
  "tool": "sessions_spawn",
  "arguments": {
    "task": "데이터 분석해줘",
    "model": "google/gemini-pro-1.5" // 이 작업에만 Gemini 사용
  }
}
```

만약 `model` 인자를 생략하면, 해당 서브 에이전트를 생성한 부모 에이전트의 모델 설정을 상속받지 않고 **시스템 기본값**을 따릅니다.

---

# OpenClaw Multi-Agent Cooperation Guide

OpenClaw (Clawdbot) 환경에서 여러 에이전트가 서로 메시지를 주고받으며 협업하는 방법을 설명합니다.

---

## 1. 개요 (Overview)

OpenClaw의 멀티 에이전트 시스템은 단순히 독립적으로 동작하는 것이 아니라, 서로 메시지를 보내고 작업을 위임할 수 있습니다. 이를 통해 복잡한 작업을 효율적으로 분산 처리할 수 있습니다.

- **작업 분담**: 메인 에이전트가 리서치 에이전트에게 조사를 요청
- **전문성 활용**: 각 에이전트가 특화된 도구(Tool)를 사용
- **비동기 처리**: 오래 걸리는 작업을 다른 에이전트에게 맡기고 즉시 응답

---

## 2. 세션 전송 (Sessions Send)

에이전트 간 소통의 핵심은 `sessions_send` 도구입니다. 이 도구를 사용하면 다른 에이전트의 세션에 메시지를 보낼 수 있습니다.

### 사용 방법
```json title="sessions_send setup"
{
  "tool": "sessions_send",
  "arguments": {
    "sessionKey": "target_agent_session_key",
    "message": "이 작업 좀 처리해줄래?",
    "timeoutSeconds": 60
  }
}
```

- **sessionKey**: 메시지를 받을 대상 세션의 키. (보통 `agentId` 또는 특정 채널 세션 ID)
- **message**: 전달할 내용. (자연어 명령 가능)
- **timeoutSeconds**: 응답을 기다릴 시간 (초 단위).

!!! note "예시: 리서치 요청"
    메인 에이전트가 리서치 에이전트(`researcher`)에게 조사를 요청하는 경우:
    ```json
    {
      "tool": "sessions_send",
      "arguments": {
        "agentId": "researcher", // 대상 에이전트 ID
        "message": "최근 AI 트렌드에 대해 조사해서 요약해줘."
      }
    }
    ```

---

## 3. 서브 에이전트 생성 (Sessions Spawn)

일회성 작업을 위해 새로운 서브 에이전트를 생성하고 싶다면 `sessions_spawn`을 사용합니다. 이 도구는 작업을 완료하면 자동으로 종료되는 임시 에이전트를 생성합니다.

### 사용 방법
```json title="sessions_spawn setup"
{
  "tool": "sessions_spawn",
  "arguments": {
    "task": "이 웹사이트(https://example.com)의 내용을 분석해줘.",
    "model": "openai/gpt-4o", // 사용할 모델 지정 가능
    "thinking": "high",       // 생각(Reasoning) 강도 조절
    "timeoutSeconds": 300
  }
}
```

- **task**: 수행할 작업 내용.
- **model**: (선택) 특정 모델을 사용하여 작업 수행.
- **cleanup**: `"delete"`(기본값) 또는 `"keep"`. 작업 완료 후 세션 유지 여부.

!!! info "활용 시나리오"
    - **복잡한 분석**: 시간이 오래 걸리는 데이터 분석 작업을 별도 에이전트에게 위임.
    - **격리된 실행**: 메인 컨텍스트를 오염시키지 않고 독립적인 환경에서 실행.

---

## 4. 에이전트 간 협업 패턴 (Collaboration Patterns)

### 1) 계층형 구조 (Hierarchical)
- **Manager Agent**: 사용자 입력을 받고 작업을 분배.
- **Worker Agents**: 실제 작업을 수행하고 결과 보고.
    - `Manager` -> `Worker A` (자료 조사)
    - `Manager` -> `Worker B` (보고서 작성)
    - `Manager` -> `User` (최종 결과 전달)

### 2) 파이프라인 구조 (Pipeline)
- 에이전트 A가 작업을 마치고 에이전트 B에게 넘기는 방식.
    - `Agent A` (데이터 수집) -> `Agent B` (데이터 가공) -> `Agent C` (결과 저장)

---

## 5. 주의사항 (Best Practices)

1.  **명확한 지시**: 다른 에이전트에게 작업을 시킬 때는 모호하지 않게 구체적으로 지시해야 합니다.
2.  **무한 루프 방지**: 에이전트 A가 B에게, B가 다시 A에게 작업을 시키는 루프에 빠지지 않도록 설계해야 합니다.
3.  **타임아웃 관리**: 응답이 늦어질 경우를 대비해 적절한 `timeoutSeconds`를 설정하세요.

---

# OpenClaw Multi-Agent Examples

이 문서는 OpenClaw의 멀티 에이전트 기능을 활용한 실전 시나리오와 설정 예시를 담고 있습니다.

---

## Scenario 1: 뉴스 브리핑 팀 (News Briefing Team)

메인 에이전트가 사용자의 요청을 받고, '리서치 에이전트'에게 뉴스를 검색하게 한 뒤 결과를 종합하여 보고하는 시나리오입니다.

### 1. 에이전트 설정 (`config.json`)
```json
{
  "agents": {
    "list": [
      {
        "id": "briefing_bot",
        "default": true,
        "name": "Briefing Master",
        "workspace": "~/briefing-master",
        "sandbox": { "mode": "off" }
      },
      {
        "id": "news_researcher",
        "name": "News Hunter",
        "workspace": "~/news-hunter",
        "sandbox": { "mode": "all", "scope": "agent" },
        "tools": {
          "allow": ["web_search", "web_fetch", "read", "memory_search"],
          "deny": ["exec", "write"]
        }
      }
    ]
  }
}
```

### 2. 워크플로우 (Workflow)
1.  **User -> Briefing Master**: "오늘의 AI 뉴스 요약해줘."
2.  **Briefing Master -> News Hunter** (`sessions_send`):
    ```json
    {
      "agentId": "news_researcher",
      "message": "오늘 날짜 기준 AI 관련 주요 뉴스 5가지를 검색해서 요약해줘."
    }
    ```
3.  **News Hunter**: `web_search` 도구를 사용하여 뉴스 검색 및 요약 생성.
4.  **News Hunter -> Briefing Master**: "1. 구글 제미나이 업데이트... 2. 오픈AI 소식... (요약 내용)"
5.  **Briefing Master -> User**: 최종 정리된 뉴스 브리핑 전달.

---

## Scenario 2: 안전한 코딩 비서 (Secure Coding Assistant)

메인 에이전트는 사용자와 대화하며, 위험한 코드 실행이나 테스트는 격리된 '샌드박스 에이전트'에게 위임합니다.

### 1. 에이전트 설정 (`config.json`)
```json
{
  "agents": {
    "list": [
      {
        "id": "dev_lead",
        "default": true,
        "name": "Dev Lead",
        "tools": { "profile": "coding" }
      },
      {
        "id": "sandbox_runner",
        "name": "Code Runner",
        "sandbox": { "mode": "all" }, // Docker 필수
        "tools": {
          "allow": ["exec", "read", "write"],
          "deny": ["gateway", "nodes"] // 시스템 중요 기능 차단
        }
      }
    ]
  }
}
```

### 2. 워크플로우 (Workflow)
1.  **User -> Dev Lead**: "이 파이썬 스크립트가 안전한지 테스트해줘."
2.  **Dev Lead -> Sandbox Runner** (`sessions_spawn` 또는 `sessions_send`):
    - 일회성 테스트라면 `sessions_spawn` 추천.
    ```json
    {
      "task": "다음 파이썬 코드를 실행하고 결과를 알려줘: print('Hello World')",
      "agentId": "sandbox_runner"
    }
    ```
3.  **Sandbox Runner**: 격리된 Docker 컨테이너 안에서 코드 실행 (`exec`).
4.  **Sandbox Runner -> Dev Lead**: "실행 결과: Hello World (종료 코드 0)"
5.  **Dev Lead -> User**: "테스트 결과 안전하게 실행되었습니다."

---

## Scenario 3: 고객 지원 라우팅 (Customer Support Routing)

들어오는 문의 유형에 따라 기술 지원 팀과 결제 지원 팀으로 자동 분류합니다.

### 1. 채널 바인딩 설정 (`config.json`)
```json
{
  "agents": {
    "list": [
      { "id": "triage_bot", "name": "Receptionist" },
      { "id": "tech_support", "name": "Tech Support" },
      { "id": "billing_support", "name": "Billing Support" }
    ]
  },
  "bindings": [
    {
      "agentId": "triage_bot",
      "match": { "provider": "telegram" } // 모든 텔레그램 메시지를 먼저 받음
    }
  ]
}
```

### 2. 워크플로우 (Workflow)
1.  **User -> Triage Bot**: "결제가 중복으로 됐어요."
2.  **Triage Bot**: 사용자 메시지 분석 -> '결제(Billing)' 관련 이슈로 판단.
    - **Tip**: Triage Bot의 `SOUL.md` 또는 프롬프트에 판단 기준을 명시해야 합니다.
    ```markdown
    # SOUL.md (Triage Bot)
    너는 고객 문의 분류 담당자(Receptionist)야.
    사용자의 메시지를 분석해서 다음 에이전트 ID 중 하나를 선택해 `sessions_send`로 전달해.
    
    - 기술 문제/버그/사용법 -> `tech_support`
    - 결제/환불/요금제 -> `billing_support`
    - 그 외 일반 문의 -> 직접 답변
    ```
3.  **Triage Bot -> Billing Support** (`sessions_send`):
    - 사용자를 직접 넘기거나, 내용을 전달하여 처리 요청.
    ```json
    {
      "agentId": "billing_support",
      "message": "사용자(@user123)가 중복 결제 문의를 했습니다. 확인 부탁드립니다."
    }
    ```
4.  **Billing Support**: 결제 내역 확인 후 처리.

---

!!! note "참고 (References)"
    *OpenClaw Documentation v1.0*
