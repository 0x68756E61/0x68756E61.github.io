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


Last updated: 2026-01-22

개요

단일 장기 실행 Gateway는 모든 메시징 인터페이스(WhatsApp via Baileys, Telegram via grammY, Slack, Discord, Signal, iMessage, WebChat)를 소유합니다. 제어 평면 클라이언트(macOS 앱, CLI, 웹 UI, 자동화)는 구성된 바인드 호스트(기본 127.0.0.1:18789)의 WebSocket을 통해 Gateway에 연결합니다. 노드(macOS/iOS/Android/헤드리스)도 WebSocket으로 연결하되 role: node를 선언하고 명시적 권한/명령을 제공합니다. 호스트당 하나의 Gateway가 존재하며, WhatsApp 세션은 Gateway에서만 엽니다. 캔버스 호스트(기본 포트 18793)는 에이전트가 편집 가능한 HTML 및 A2UI를 제공합니다.

구성 요소 및 흐름

Gateway(데몬)
- 공급자 연결 유지
- 타입화된 WS API(요청, 응답, 서버 푸시 이벤트) 제공
- 수신 프레임을 JSON Schema로 검증
- agent, chat, presence, health, heartbeat, cron 등의 이벤트를 발생

클라이언트(mac 앱 / CLI / 웹 관리자)
- 클라이언트당 하나의 WS 연결
- health, status, send, agent, system-presence 등의 요청 전송
- tick, agent, presence, shutdown 등의 이벤트 구독

노드(macOS / iOS / Android / 헤드리스)
- role: node로 동일 WS 서버에 연결
- connect 시 디바이스 아이덴티티 제공; 페어링은 디바이스 기반이며 승인 정보는 페어링 저장소에 보관
- canvas.*, camera.*, screen.record, location.get 같은 명령 노출

프로토콜 세부사항
- 전송: WebSocket(텍스트 프레임, JSON 페이로드)
- 첫 프레임은 반드시 connect여야 함
- 핸드셰이크 이후 요청/응답 및 이벤트 교환
- OPENCLAW_GATEWAY_TOKEN(또는 --token)이 설정된 경우 connect.params.auth.token이 일치해야 하며 그렇지 않으면 소켓이 닫힘
- send, agent 같은 부작용을 일으키는 메서드는 idempotency 키 필요; 서버는 짧은 중복 제거 캐시를 유지
- 노드는 connect에 role:"node"와 caps/commands/permissions를 포함해야 함

페어링 및 로컬 신뢰
- 모든 WS 클라이언트(운영자 + 노드)는 connect 시 디바이스 아이덴티티 포함
- 신규 디바이스 ID는 페어링 승인 필요; Gateway는 이후 연결을 위한 디바이스 토큰 발행
- 로컬 연결(loopback 또는 호스트의 tailnet 주소)은 자동 승인 가능
- 비로컬 연결은 connect.challenge nonce 서명 및 명시적 승인 필요
- Gateway 인증(gateway.auth.*)은 모든 연결에 적용

원격 액세스
- 권장: Tailscale 또는 VPN
- 대안: SSH 터널(예: ssh -N -L 18789:127.0.0.1:18789 user@host)
- 터널 상에서도 동일한 핸드셰이크+토큰 규칙 적용; TLS 및 선택적 핀닝 사용 가능

운영 스냅샷
- 시작: openclaw gateway(포그라운드, stdout에 로그)
- 헬스: WS를 통한 health(hello-ok에 포함)
- 감독: launchd/systemd로 자동 재시작 설정

불변 조건
- 정확히 하나의 Gateway가 호스트당 Baileys 세션을 제어
- 핸드셰이크 필수; 비-JSON 또는 첫 프레임이 connect가 아니면 강제 종료
- 이벤트는 재생되지 않으므로 클라이언트는 갭 발생 시 갱신 필요
