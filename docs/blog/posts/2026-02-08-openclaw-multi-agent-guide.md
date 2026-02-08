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

> LEAD: OpenClaw(Clawdbot 기반) 환경에서 여러 개의 AI 에이전트를 생성하고, 각기 다른 역할을 부여하여 협업하게 만드는 멀티 에이전트 구성법을 소개합니다.

<!-- more -->

# OpenClaw 멀티 에이전트 완벽 가이드

OpenClaw의 멀티 에이전트 시스템을 사용하면 단일 게이트웨이 내에서 서로 다른 **성격(Persona)**, **권한(Permissions)**, **작업 공간(Workspace)**을 가진 여러 에이전트를 실행할 수 있습니다.

---

## 1. 개요 (Overview)

OpenClaw 멀티 에이전트의 주요 장점은 다음과 같습니다:

*   **개인 비서**: 모든 권한을 가진 메인 에이전트 운영
*   **역할 분담**: 가족/동료용으로 제한된 도구만 사용하는 에이전트 설정
*   **보안 강화**: 공용/테스트용 에이전트를 샌드박스(Docker) 내에서 안전하게 실행

---

## 2. 에이전트 정의 (Defining Agents)

멀티 에이전트는 설정 파일의 `agents.list` 배열에 정의합니다. 각 에이전트는 고유한 `id`를 가집니다.

!!! tip "기본 구조 예시"
    ```json title="config.json"
    {
      "agents": {
        "list": [
          {
            "id": "main",
            "default": true,
            "name": "Main Assistant",
            "workspace": "~/clawd",
            "sandbox": { "mode": "off" }
          },
          {
            "id": "dev_bot",
            "name": "Developer Bot",
            "workspace": "~/clawd-dev"
          }
        ]
      }
    }
    ```

---

## 3. 샌드박스 설정 (Sandbox Configuration)

각 에이전트별로 격리 환경을 적용하여 보안을 조절할 수 있습니다.

*   **mode**:
    *   `"off"`: 호스트 머신에서 직접 실행 (메인 에이전트용)
    *   `"all"`: 항상 Docker 컨테이너 내에서 실행 (안전함)
    *   `"non-main"`: 메인 세션이 아닐 때만 적용
*   **scope**:
    *   `"agent"`: 에이전트 전용 컨테이너 (데이터 격리)
    *   `"session"`: 세션마다 새로운 컨테이너 (휘발성)

---

## 4. 도구 권한 제어 (Tool Restrictions)

에이전트의 역할을 명확히 하기 위해 도구 사용을 제한할 수 있습니다.

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

## 5. 에이전트 간 협업 (Cooperation)

에이전트들은 서로 메시지를 주고받으며 복잡한 작업을 분산 처리할 수 있습니다.

```mermaid
graph LR;
    User((User)) --> Manager[Manager Agent];
    Manager --> Researcher[Researcher Agent];
    Researcher -- 조사 결과 -- > Manager;
    Manager -- 최종 보고 -- > User;
```

### 핵심 도구: `sessions_send` & `sessions_spawn`
*   **`sessions_send`**: 다른 에이전트 세션에 메시지를 전달합니다.
*   **`sessions_spawn`**: 일회성 작업을 위한 임시 서브 에이전트를 생성합니다.

---

## 6. 적용 및 확인

설정 변경 후 다음 명령어로 상태를 확인할 수 있습니다.

1.  **설정 적용**: `clawdbot gateway config.apply`
2.  **에이전트 목록 확인**: `clawdbot agents list`
3.  **바인딩 확인**: `clawdbot agents list --bindings`

!!! note "마치며"
    OpenClaw의 멀티 에이전트 기능을 활용하면 단순한 챗봇을 넘어, 스스로 판단하고 협업하는 AI 팀을 구축할 수 있습니다. 지금 바로 여러분만의 AI 팀을 구성해 보세요!

---
*OpenClaw Documentation v1.0 기반 작성*
