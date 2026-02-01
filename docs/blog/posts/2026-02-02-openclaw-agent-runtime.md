---
title: OpenClaw Agent Runtime (에이전트 런타임)
date: 2026-02-02
authors:
  - '0x68756E61'
categories:
  - Agent Runtime
tags:
  - openclawd
  - clawdbot
---

![](../../assets/images/openclaw-agent-runtime-hero.png)


Last updated: 2026-01-22

에이전트 런타임(Agent Runtime)

OpenClaw는 pi-mono에서 파생된 단일 내장 에이전트 런타임을 실행합니다.

워크스페이스(필수)
OpenClaw는 단일 에이전트 워크스페이스 디렉터리(agents.defaults.workspace)를 에이전트의 작업 디렉터리(cwd)로 사용합니다. 도구와 컨텍스트는 이 디렉터리를 기준으로 동작합니다. 누락된 경우 `openclaw setup`을 사용해 `~/.openclaw/openclaw.json`을 생성하고 워크스페이스 파일을 초기화하는 것을 권장합니다.

부트스트랩 파일(주입되는 파일)
agents.defaults.workspace 내부에서 OpenClaw는 다음의 사용자 편집 가능한 파일들을 기대합니다:

- AGENTS.md — 운영 지침 및 “메모리”
- SOUL.md — 페르소나, 경계, 톤
- TOOLS.md — 사용자가 유지보수하는 도구 노트(예: imsg, sag, 규약)
- BOOTSTRAP.md — 처음 실행 시 의식(완료 후 삭제)
- IDENTITY.md — 에이전트 이름/분위기/이모지
- USER.md — 사용자 프로필 및 선호 호칭

새 세션의 첫 턴에서 OpenClaw는 이 파일들의 내용을 에이전트 컨텍스트에 직접 주입합니다. 빈 파일은 건너뛰며, 큰 파일은 프롬프트를 가볍게 유지하기 위해 잘라내기/요약 표식이 붙습니다. 파일이 없으면 OpenClaw는 단일 “missing file” 마커 라인을 주입하며 `openclaw setup`으로 기본 템플릿을 생성할 수 있습니다. BOOTSTRAP.md는 완전히 새로운 워크스페이스에만 생성됩니다.

내장 도구
핵심 도구(read/exec/edit/write 및 관련 시스템 도구)는 항상 사용 가능하며 도구 정책의 적용을 받습니다. `apply_patch`는 선택적이며 `tools.exec.applyPatch`로 제어됩니다. TOOLS.md는 어떤 도구가 존재하는지를 제어하지 않으며, 도구 사용 방식에 대한 가이드입니다.

스킬 로딩
OpenClaw는 세 곳에서 스킬을 로드합니다(워크스페이스가 이름 충돌 시 우선):
- Bundled(설치에 포함)
- Managed/local: ~/.openclaw/skills
- Workspace: <workspace>/skills

스킬은 구성 또는 환경변수로 게이팅될 수 있습니다.

세션 저장
세션 전사(transcript)는 JSONL 형식으로 저장됩니다:
`~/.openclaw/agents/<agentId>/sessions/<SessionId>.jsonl`

스트리밍 중 제어
큐 모드가 `steer`일 때, 들어오는 메시지는 현재 실행 중인 런에 주입됩니다. 큐는 각 도구 호출 후 확인되며, 대기중인 메시지가 있으면 현재 어시스턴트 메시지의 남은 도구 호출을 건너뜁니다(“Skipped due to queued user message.” 에러 도구 결과). `followup` 또는 `collect` 모드에서는 들어오는 메시지를 현재 턴이 끝날 때까지 보관한 뒤 새 에이전트 턴으로 주입합니다.

블록 스트리밍
블록 스트리밍은 완성된 어시스턴트 블록을 즉시 전송합니다(기본값: off). 경계와 청크 크기, 합치기 동작은 agents.defaults.* 설정으로 조정할 수 있습니다. 자세한 설정은 문서를 참조하세요.

모델 참조
에이전트 설정의 모델 참조(예: agents.defaults.model)는 첫 번째 '/'를 기준으로 분리합니다. provider/model 형식을 권장하며, OpenRouter 스타일의 경우 provider 접두사를 포함하세요(e.g., openrouter/moonshotai/kimi-k2). 접두사를 생략하면 기본 프로바이더의 별칭으로 취급됩니다(모델 ID에 '/'가 없을 때만).

기본 구성(최소 설정)
- agents.defaults.workspace 설정
- channels.whatsapp.allowFrom 설정(강력 권장)
