---
title: OpenClaw 게이트웨이 아키텍처 개요
date: 2026-02-01
authors:
  - '0x68756E61'
categories:
  - Architecture
tags:
  - openclawd
  - clawdbot
---

![](/assets/images/openclaw-gateway-architecture-overview-hero.png)


# OpenClaw 게이트웨이 아키텍처 개요

단일 장기 실행 프로세스인 Gateway가 모든 메시징 인터페이스(WhatsApp, Telegram, Slack, Discord, Signal, iMessage, WebChat 등)를 소유하고 관리합니다. 제어 평면 클라이언트(예: macOS 앱, CLI, 웹 UI)는 WebSocket을 통해 Gateway에 연결하며, 각 노드(macOS/iOS/Android/헤드리스)는 role: node로 연결해 명시적 권한과 명령을 제공합니다. 호스트당 하나의 Gateway만 존재하며, WhatsApp 세션은 Gateway에서만 열립니다. 에이전트 편집 가능한 캔버스는 별도 포트에서 제공됩니다.

구성 요소 요약:

- Gateway(데몬): 공급자 연결 유지, 타입화된 WebSocket API 제공, JSON Schema로 수신 프레임 검증, agent/chat/presence/health/heartbeat/cron 등 이벤트 방출.
- 클라이언트(앱/CLI/웹): 각 클라이언트는 하나의 WS 연결을 유지하며 상태·헬스 요청 및 이벤트 구독을 수행.
- 노드(디바이스): role: node로 연결, 디바이스 아이덴티티와 페어링을 통한 인증, 카메라·스크린·로케이션 등 원격 명령 노출.

프로토콜 핵심:
- 전송은 WebSocket(텍스트 프레임, JSON 페이로드)이며 첫 프레임은 반드시 connect여야 합니다.
- 요청/응답과 이벤트 구조가 분리되어 있으며(idempotency 키로 중복 방지), 인증 토큰이 일치하지 않으면 연결이 차단됩니다.

페어링과 보안:
- 신규 디바이스는 페어링 승인이 필요하고, 로컬 연결은 자동 승인될 수 있습니다. 원격 연결은 도전 문자열(challenge) 서명과 명시적 승인을 요구합니다.

운영 팁:
- 권장 원격 접속 방식은 Tailscale 또는 VPN이며, SSH 터널이나 TLS를 통한 보호도 가능합니다.
- 서비스 운영은 launchd/systemd로 감독하고, health 체크는 WS를 통해 수행합니다.

(이 문서는 OpenClaw 문서의 'Gateway Architecture' 항목을 요약·번역한 것입니다.)
