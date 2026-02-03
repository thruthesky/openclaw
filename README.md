# Dokploy에서 OpenClaw (Moltbot) 설치 완벽 가이드

> **작성일**: 2026-02-03
> **패키지**: `moltbot/moltbot:latest` (Docker Hub)

---

## 목차

1. [개요](#1-개요)
2. [사전 요구사항](#2-사전-요구사항)
3. [Dokploy에서 Compose 서비스 생성](#3-dokploy에서-compose-서비스-생성)
4. [Docker Compose YAML 설정](#4-docker-compose-yaml-설정)
5. [환경변수 설정](#5-환경변수-설정)
6. [도메인 및 HTTPS 설정](#6-도메인-및-https-설정)
7. [첫 배포 실행](#7-첫-배포-실행)
8. [Gateway Token 확인](#8-gateway-token-확인)
9. [Device Pairing (기기 승인)](#9-device-pairing-기기-승인)
10. [API 키 설정](#10-api-키-설정)
11. [모델 Provider 설정](#11-모델-provider-설정)
12. [템플릿 파일 생성](#12-템플릿-파일-생성)
13. [최종 확인 및 테스트](#13-최종-확인-및-테스트)
14. [트러블슈팅](#14-트러블슈팅)

---

## 1. 개요

### OpenClaw란?

OpenClaw는 AI Gateway 플랫폼으로, 다양한 AI 모델(Claude, GPT, DeepSeek 등)을 웹 UI를 통해 관리하고 사용할 수 있게 해주는 셀프호스팅 솔루션입니다.

### 중요 정보

| 항목 | 값 |
|------|-----|
| **Docker 이미지** | `moltbot/moltbot:latest` (Docker Hub) |
| **패키지명** | moltbot (OpenClaw의 이전 버전명) |
| **기본 포트** | 18789 (Gateway), 18790 (Canvas) |
| **설정 디렉토리** | `/home/node/.moltbot` |
| **워크스페이스** | `/home/node/clawd` |

### 아키텍처 흐름

```
[사용자 브라우저]
    ↓ HTTPS (443)
[Traefik Reverse Proxy]
    ↓ HTTP (18789)
[Moltbot Gateway Container]
    ↓ API Call
[AI Provider (DeepSeek/OpenAI/Anthropic)]
```

---

## 2. 사전 요구사항

### 필수 요구사항

- [x] Dokploy 서버 설치 완료
- [x] 도메인 (예: `claw.example.com`)
- [x] DNS A 레코드가 서버 IP를 가리킴
- [x] AI Provider API 키 (DeepSeek, OpenAI, Anthropic 중 하나)

### 권장 사양

| 항목 | 최소 | 권장 |
|------|------|------|
| RAM | 1GB | 2GB |
| CPU | 1 Core | 2 Core |
| Storage | 5GB | 10GB |

---

## 3. Dokploy에서 Compose 서비스 생성

### 3.1 프로젝트 생성

1. Dokploy 웹 UI 접속 (`http://서버IP:3000`)
2. **Projects** → **Create Project** 클릭
3. 프로젝트 이름 입력 (예: `openclaw`)
4. **Create** 클릭

### 3.2 Compose 서비스 추가

1. 생성된 프로젝트 클릭
2. **+ Create Service** → **Compose** 선택
3. 서비스 이름 입력 (예: `moltbot`)
4. **Create** 클릭

---

## 4. Docker Compose YAML 설정

### 4.1 General → Raw 탭에서 YAML 입력

Dokploy UI에서 생성된 Compose 서비스의 **General** → **Raw** 탭으로 이동하여 아래 YAML을 입력합니다.

```yaml
services:
  moltbot-gateway:
    image: moltbot/moltbot:latest
    environment:
      HOME: /home/node
      TERM: xterm-256color
      CLAWDBOT_GATEWAY_TOKEN: ${CLAWDBOT_GATEWAY_TOKEN}
      OPENROUTER_API_KEY: ${OPENROUTER_API_KEY}
    volumes:
      - moltbot-config:/home/node/.moltbot
      - moltbot-workspace:/home/node/clawd
    ports:
      - "18789"
      - "18790"
    init: true
    restart: unless-stopped
    command:
      - gateway
      - --bind
      - lan
      - --port
      - "18789"
      - --allow-unconfigured
    networks:
      - dokploy-network

volumes:
  moltbot-config:
  moltbot-workspace:

networks:
  dokploy-network:
    external: true
```

### 4.2 YAML 설정 설명

| 설정 | 설명 |
|------|------|
| `image: moltbot/moltbot:latest` | **필수** - Docker Hub의 공식 이미지 |
| `--bind lan` | 모든 네트워크 인터페이스에서 수신 (0.0.0.0) |
| `--port 18789` | Gateway 포트 |
| `--allow-unconfigured` | 초기 설정 없이 시작 가능 |
| `dokploy-network` | Traefik이 연결된 네트워크 |
| `moltbot-config` | 설정 파일 영구 저장 볼륨 |
| `moltbot-workspace` | 워크스페이스 영구 저장 볼륨 |

### 4.3 주의사항

> ⚠️ **command 설정 주의**
> `node dist/index.js`를 command에 포함하면 안 됩니다.
> 이미지의 ENTRYPOINT가 이미 `node dist/index.js`를 실행하므로 중복됩니다.

**올바른 command:**
```yaml
command:
  - gateway
  - --bind
  - lan
  - --port
  - "18789"
  - --allow-unconfigured
```

**잘못된 command:**
```yaml
command:
  - node
  - dist/index.js  # ❌ 중복!
  - gateway
  - --bind
  - lan
```

---

## 5. 환경변수 설정

### 5.1 Dokploy UI에서 환경변수 추가

1. Compose 서비스 → **Environment** 탭
2. 아래 환경변수 추가:

| 변수명 | 값 | 설명 |
|--------|-----|------|
| `CLAWDBOT_GATEWAY_TOKEN` | 임의의 32자 토큰 | Gateway 접속 토큰 |
| `OPENROUTER_API_KEY` | (선택) OpenRouter API 키 | OpenRouter 사용 시 |

### 5.2 Gateway Token 생성

터미널에서 랜덤 토큰 생성:

```bash
# 32자 랜덤 토큰 생성
openssl rand -hex 16
# 예시 출력: 2atffjjnrmogozfqmv0ds2nyxwgzoq2q
```

이 값을 `CLAWDBOT_GATEWAY_TOKEN` 환경변수에 설정합니다.

---

## 6. 도메인 및 HTTPS 설정

### 6.1 Dokploy UI에서 도메인 추가

1. Compose 서비스 → **Domains** 탭
2. **Add Domain** 클릭
3. 아래 설정 입력:

| 필드 | 값 |
|------|-----|
| **Host** | `claw.example.com` (본인 도메인) |
| **Container Port** | `18789` ⚠️ **중요!** |
| **HTTPS** | ✅ 체크 |
| **Certificate** | `Let's Encrypt` |

### 6.2 포트 설정 주의사항

> ⚠️ **Container Port는 반드시 `18789`로 설정해야 합니다.**
> 기본값 3000이 아닙니다!

```
[Traefik] → Port 18789 → [Moltbot Container]
```

### 6.3 DNS 설정 확인

```bash
# DNS A 레코드 확인
dig +short claw.example.com
# 출력: 서버 IP 주소
```

---

## 7. 첫 배포 실행

### 7.1 배포

1. Dokploy UI → Compose 서비스 → **Deploy** 버튼 클릭
2. 배포 로그 확인

### 7.2 정상 시작 로그 확인

```
[gateway] agent model: anthropic/claude-opus-4-5
[gateway] listening on ws://0.0.0.0:18789 (PID 7)
[browser/service] Browser control service ready (profiles=2)
```

### 7.3 접속 테스트

브라우저에서 접속:
```
https://claw.example.com/?token=YOUR_GATEWAY_TOKEN
```

예시:
```
https://claw.example.com/?token=2atffjjnrmogozfqmv0ds2nyxwgzoq2q
```

---

## 8. Gateway Token 확인

### 8.1 Token이 없거나 모르는 경우

SSH로 서버에 접속하여 확인:

```bash
# 컨테이너 이름 확인
docker ps --format '{{.Names}}' | grep moltbot

# 예시 출력: myproject-moltbot-abc123-moltbot-gateway-1
```

```bash
# Gateway Token 확인 (환경변수에서)
docker exec <컨테이너이름> printenv CLAWDBOT_GATEWAY_TOKEN
```

### 8.2 접속 URL 형식

```
https://도메인/?token=GATEWAY_TOKEN
```

---

## 9. Device Pairing (기기 승인)

### 9.1 Device Pairing이란?

Moltbot은 보안을 위해 각 브라우저/기기마다 일회성 승인이 필요합니다.

### 9.2 Pairing Required 에러

처음 접속 시 아래 에러가 표시됩니다:
```
disconnected (1008): unauthorized: pairing required
```

### 9.3 기기 승인 방법

**SSH로 서버 접속 후:**

```bash
# 컨테이너 이름 확인
CONTAINER=$(docker ps --format '{{.Names}}' | grep moltbot | head -1)

# 승인 대기 중인 기기 목록 확인
docker exec $CONTAINER node dist/index.js devices list
```

**출력 예시:**
```
Pending approval:
  Request ID: abc123def456
  Platform: MacIntel
  Client: moltbot-control-ui webchat
```

```bash
# 기기 승인
docker exec $CONTAINER node dist/index.js devices approve abc123def456
```

### 9.4 승인 후

브라우저를 새로고침하면 Control UI에 접속됩니다.

> 💡 **팁**: 다른 컴퓨터/브라우저에서 접속할 때마다 같은 과정을 반복해야 합니다.

---

## 10. API 키 설정

### 10.1 지원 AI Provider

| Provider | API 키 형식 |
|----------|------------|
| DeepSeek | `sk-xxxxxxxx` |
| OpenAI | `sk-xxxxxxxx` |
| Anthropic | `sk-ant-xxxxxxxx` |
| OpenRouter | `sk-or-xxxxxxxx` |

### 10.2 방법 1: Web UI에서 설정 (권장) ⭐

Moltbot Control UI에서 직접 API 키를 설정할 수 있습니다.

#### 10.2.1 Config 페이지 접속

1. `https://도메인/?token=YOUR_TOKEN` 으로 접속
2. 좌측 메뉴에서 **Settings** → **Config** 클릭

#### 10.2.2 DeepSeek Provider 추가

Config 페이지에서 아래 설정을 찾아 입력합니다:

**models.providers.deepseek 설정:**

| 필드 | 값 |
|------|-----|
| `models.providers.deepseek.baseUrl` | `https://api.deepseek.com` |
| `models.providers.deepseek.api` | `openai-completions` |

> 💡 Config UI에서 JSON 형식으로 직접 편집할 수도 있습니다.

#### 10.2.3 모델 설정

| 필드 | 값 |
|------|-----|
| `agents.defaults.model.primary` | `deepseek/deepseek-chat` |

#### 10.2.4 API 키 설정

1. 좌측 메뉴에서 **Agent** → **Skills** 또는 **Nodes** 탭 확인
2. 또는 CLI로 설정 (아래 방법 2 참조)

> ⚠️ **참고**: API 키는 보안상 Web UI에서 직접 설정이 제한될 수 있습니다.
> 이 경우 아래 CLI 방법을 사용하세요.

---

### 10.3 방법 2: CLI에서 설정

**SSH로 서버 접속 후:**

```bash
# 컨테이너 이름 확인
CONTAINER=$(docker ps --format '{{.Names}}' | grep moltbot | head -1)

# 볼륨 이름 확인 (docker inspect로 확인)
docker inspect $CONTAINER | grep -A5 "Mounts"
```

**auth-profiles.json 파일 생성:**

```bash
# DeepSeek API 키 설정 예시
docker exec $CONTAINER bash -c 'mkdir -p /home/node/.moltbot/agents/main/agent && cat > /home/node/.moltbot/agents/main/agent/auth-profiles.json << EOF
{
  "version": 1,
  "profiles": {
    "deepseek:default": {
      "type": "api_key",
      "provider": "deepseek",
      "key": "sk-여기에-API-키-입력"
    }
  }
}
EOF'
```

### 10.4 다른 Provider 예시

```json
// Anthropic
{
  "version": 1,
  "profiles": {
    "anthropic:default": {
      "type": "api_key",
      "provider": "anthropic",
      "key": "sk-ant-여기에-API-키-입력"
    }
  }
}

// OpenAI
{
  "version": 1,
  "profiles": {
    "openai:default": {
      "type": "api_key",
      "provider": "openai",
      "key": "sk-여기에-API-키-입력"
    }
  }
}
```

---

## 11. 모델 Provider 설정

### 11.1 기본 모델 문제

기본 모델은 `anthropic/claude-opus-4-5`입니다. DeepSeek 등 다른 모델을 사용하려면 provider 설정이 필요합니다.

### 11.2 방법 1: Web UI에서 모델 설정 (권장) ⭐

Moltbot Control UI의 Config 페이지에서 모델을 설정할 수 있습니다.

#### 11.2.1 Config 페이지 접속

1. `https://도메인/?token=YOUR_TOKEN` 으로 접속
2. 좌측 메뉴에서 **Settings** → **Config** 클릭

#### 11.2.2 DeepSeek Provider JSON 설정

Config 페이지에서 JSON 편집 모드로 전환하여 아래 내용을 추가합니다:

```json
{
  "models": {
    "providers": {
      "deepseek": {
        "baseUrl": "https://api.deepseek.com",
        "api": "openai-completions",
        "models": [
          {
            "id": "deepseek-chat",
            "name": "DeepSeek Chat",
            "contextWindow": 64000
          }
        ]
      }
    }
  },
  "agents": {
    "defaults": {
      "model": {
        "primary": "deepseek/deepseek-chat"
      }
    }
  }
}
```

#### 11.2.3 개별 필드 설정 (UI 폼 사용 시)

| 설정 경로 | 값 |
|----------|-----|
| `models.providers.deepseek.baseUrl` | `https://api.deepseek.com` |
| `models.providers.deepseek.api` | `openai-completions` |
| `models.providers.deepseek.models[0].id` | `deepseek-chat` |
| `models.providers.deepseek.models[0].name` | `DeepSeek Chat` |
| `models.providers.deepseek.models[0].contextWindow` | `64000` |
| `agents.defaults.model.primary` | `deepseek/deepseek-chat` |

#### 11.2.4 설정 저장 및 적용

1. **Save** 버튼 클릭
2. 자동으로 Gateway가 재시작됨
3. 로그에서 `agent model: deepseek/deepseek-chat` 확인

---

### 11.3 방법 2: CLI에서 moltbot.json 설정

**DeepSeek 모델 설정 예시:**

```bash
# 볼륨에서 직접 설정 파일 생성/수정
docker run --rm -v <볼륨이름>:/data alpine sh -c 'cat > /data/moltbot.json << EOF
{
  "gateway": {
    "trustedProxies": [
      "10.0.0.0/8",
      "172.16.0.0/12"
    ]
  },
  "models": {
    "providers": {
      "deepseek": {
        "baseUrl": "https://api.deepseek.com",
        "api": "openai-completions",
        "models": [
          {
            "id": "deepseek-chat",
            "name": "DeepSeek Chat",
            "contextWindow": 64000
          }
        ]
      }
    }
  },
  "agents": {
    "defaults": {
      "model": {
        "primary": "deepseek/deepseek-chat"
      }
    }
  }
}
EOF'
```

### 11.4 볼륨 이름 확인 방법

```bash
# Dokploy compose 서비스의 볼륨 확인
docker volume ls | grep moltbot

# 예시 출력:
# local     myproject-moltbot-abc123_moltbot-config
# local     myproject-moltbot-abc123_moltbot-workspace
```

### 11.5 설정 후 재시작

```bash
docker restart $CONTAINER
```

### 11.6 지원 모델 ID 형식

| Provider | 모델 ID 형식 |
|----------|-------------|
| DeepSeek | `deepseek/deepseek-chat` |
| Anthropic | `anthropic/claude-opus-4-5`, `anthropic/claude-sonnet-4-20250514` |
| OpenAI | `openai/gpt-4o`, `openai/gpt-4-turbo` |

---

## 12. 템플릿 파일 생성

### 12.1 필요한 템플릿 파일

Moltbot은 워크스페이스에 특정 템플릿 파일이 필요합니다. 없으면 Chat에서 에러가 발생합니다.

| 파일 | 설명 |
|------|------|
| `AGENTS.md` | 에이전트 정의 |
| `SOUL.md` | AI 성격/페르소나 |
| `TOOLS.md` | 도구 정의 |
| `USER.md` | 사용자 정보 |
| `IDENTITY.md` | 신원 정보 |
| `BOOT.md` | 부팅 스크립트 |
| `BOOTSTRAP.md` | 초기화 스크립트 |
| `HEARTBEAT.md` | 하트비트 설정 |
| `HOOK.md` | 훅 설정 |
| `MEMORY.md` | 메모리 설정 |
| `SKILL.md` | 스킬 정의 |
| `DD.md` | 추가 설정 |
| `SOUL_EVIL.md` | 대체 페르소나 |

### 12.2 템플릿 파일 일괄 생성

```bash
# 워크스페이스 볼륨에 템플릿 생성
WORKSPACE_VOL="myproject-moltbot-abc123_moltbot-workspace"

docker run --rm -v $WORKSPACE_VOL:/data alpine sh -c '
for file in AGENTS.md SOUL.md TOOLS.md USER.md IDENTITY.md BOOT.md BOOTSTRAP.md HEARTBEAT.md HOOK.md MEMORY.md SKILL.md DD.md SOUL_EVIL.md; do
  touch /data/$file
done
ls -la /data/*.md
'
```

### 12.3 AGENTS.md 기본 내용 (선택)

```bash
docker run --rm -v $WORKSPACE_VOL:/data alpine sh -c 'cat > /data/AGENTS.md << EOF
# Agents Configuration

## Main Agent
- Name: Assistant
- Role: General purpose AI assistant

EOF'
```

### 12.4 컨테이너 재시작

```bash
docker restart $CONTAINER
```

---

## 13. 최종 확인 및 테스트

### 13.1 로그 확인

```bash
docker logs $CONTAINER --tail 20
```

**정상 로그:**
```
[gateway] agent model: deepseek/deepseek-chat
[gateway] listening on ws://0.0.0.0:18789 (PID 7)
[browser/service] Browser control service ready (profiles=2)
[ws] webchat connected conn=xxx remote=10.0.1.9
```

### 13.2 접속 테스트

1. 브라우저에서 `https://claw.example.com/?token=YOUR_TOKEN` 접속
2. Device Pairing 승인 (필요시)
3. Chat 탭에서 "Hello" 메시지 전송
4. AI 응답 확인

### 13.3 Health Check

Control UI 우상단의 **Health OK** 표시 확인

---

## 14. 트러블슈팅

### 14.1 404 Not Found

**원인**: Traefik이 컨테이너에 연결되지 않음

**해결**:
1. Compose YAML에서 `dokploy-network` 확인
2. 포트가 `18789`로 설정되었는지 확인
3. 재배포

### 14.2 502 Bad Gateway

**원인**: 컨테이너가 크래시되거나 포트 불일치

**해결**:
```bash
# 컨테이너 상태 확인
docker ps -a | grep moltbot

# 로그 확인
docker logs $CONTAINER --tail 50
```

### 14.3 unauthorized: gateway token missing

**원인**: URL에 token 파라미터 없음

**해결**: `?token=YOUR_TOKEN`을 URL에 추가

### 14.4 unauthorized: pairing required

**원인**: 새 브라우저/기기에서 첫 접속

**해결**: [섹션 9. Device Pairing](#9-device-pairing-기기-승인) 참조

### 14.5 Unknown model: xxx

**원인**: 모델 provider가 설정되지 않음

**해결**: [섹션 11. 모델 Provider 설정](#11-모델-provider-설정) 참조

### 14.6 No API key found for provider

**원인**: auth-profiles.json에 API 키 없음

**해결**: [섹션 10. API 키 설정](#10-api-키-설정) 참조

### 14.7 Missing workspace template: XXX.md

**원인**: 워크스페이스에 필요한 템플릿 파일 없음

**해결**: [섹션 12. 템플릿 파일 생성](#12-템플릿-파일-생성) 참조

### 14.8 memory-core plugin not found

**원인**: moltbot 이미지의 알려진 버그

**해결**:
- `--allow-unconfigured` 플래그가 command에 포함되어 있는지 확인
- 이 플래그가 있으면 이 에러는 무시됨

### 14.9 컨테이너 재시작 루프

**원인**: 설정 파일 유효성 검사 실패

**해결**:
```bash
# 설정 파일 확인
docker run --rm -v <config-volume>:/data alpine cat /data/moltbot.json

# 잘못된 설정 수정 또는 삭제
docker run --rm -v <config-volume>:/data alpine rm /data/moltbot.json
```

---

## 부록 A: 전체 설정 스크립트

서버에서 한 번에 실행할 수 있는 스크립트:

```bash
#!/bin/bash

# 변수 설정
CONTAINER="myproject-moltbot-xxx-moltbot-gateway-1"  # 실제 컨테이너 이름으로 변경
CONFIG_VOL="myproject-moltbot-xxx_moltbot-config"    # 실제 볼륨 이름으로 변경
WORKSPACE_VOL="myproject-moltbot-xxx_moltbot-workspace"  # 실제 볼륨 이름으로 변경
DEEPSEEK_API_KEY="sk-your-api-key-here"              # 실제 API 키로 변경

# 1. 템플릿 파일 생성
docker run --rm -v $WORKSPACE_VOL:/data alpine sh -c '
for file in AGENTS.md SOUL.md TOOLS.md USER.md IDENTITY.md BOOT.md BOOTSTRAP.md HEARTBEAT.md HOOK.md MEMORY.md SKILL.md DD.md SOUL_EVIL.md; do
  touch /data/$file
done
'

# 2. moltbot.json 설정
docker run --rm -v $CONFIG_VOL:/data alpine sh -c "cat > /data/moltbot.json << 'EOF'
{
  \"gateway\": {
    \"trustedProxies\": [\"10.0.0.0/8\", \"172.16.0.0/12\"]
  },
  \"models\": {
    \"providers\": {
      \"deepseek\": {
        \"baseUrl\": \"https://api.deepseek.com\",
        \"api\": \"openai-completions\",
        \"models\": [{\"id\": \"deepseek-chat\", \"name\": \"DeepSeek Chat\", \"contextWindow\": 64000}]
      }
    }
  },
  \"agents\": {
    \"defaults\": {
      \"model\": {\"primary\": \"deepseek/deepseek-chat\"}
    }
  }
}
EOF"

# 3. auth-profiles.json 설정
docker run --rm -v $CONFIG_VOL:/data alpine sh -c "mkdir -p /data/agents/main/agent && cat > /data/agents/main/agent/auth-profiles.json << 'EOF'
{
  \"version\": 1,
  \"profiles\": {
    \"deepseek:default\": {
      \"type\": \"api_key\",
      \"provider\": \"deepseek\",
      \"key\": \"$DEEPSEEK_API_KEY\"
    }
  }
}
EOF"

# 4. 권한 설정
docker run --rm -v $CONFIG_VOL:/data alpine chown -R 1000:1000 /data
docker run --rm -v $WORKSPACE_VOL:/data alpine chown -R 1000:1000 /data

# 5. 컨테이너 재시작
docker restart $CONTAINER

# 6. 로그 확인
sleep 5
docker logs $CONTAINER --tail 20
```

---

## 부록 B: 빠른 참조

### 중요 경로

| 경로 | 설명 |
|------|------|
| `/home/node/.moltbot/moltbot.json` | 메인 설정 파일 |
| `/home/node/.moltbot/agents/main/agent/auth-profiles.json` | API 키 설정 |
| `/home/node/clawd/` | 워크스페이스 (템플릿 파일) |

### 중요 포트

| 포트 | 용도 |
|------|------|
| 18789 | Gateway (WebSocket + HTTP) |
| 18790 | Canvas |

### 유용한 명령어

```bash
# 컨테이너 목록
docker ps | grep moltbot

# 로그 실시간 확인
docker logs -f $CONTAINER

# 컨테이너 내부 접속
docker exec -it $CONTAINER bash

# 설정 파일 확인
docker exec $CONTAINER cat /home/node/.moltbot/moltbot.json

# 기기 목록
docker exec $CONTAINER node dist/index.js devices list

# 기기 승인
docker exec $CONTAINER node dist/index.js devices approve <request-id>
```

---


> 📝 **문서 작성**: Claude Code
> 🔧 **테스트 환경**: Dokploy on Ubuntu 22.04, Moltbot 2026.1.29
