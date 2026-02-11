# OpenClaw (Moltbot) - Dokploy 배포 가이드

> **버전**: 2026.2.11
> **패키지**: `moltbot/moltbot:latest` (Docker Hub)

---

## 개요

OpenClaw는 AI Gateway 플랫폼으로, 다양한 AI 모델(Claude, GPT, DeepSeek 등)을 웹 UI를 통해 관리하고 사용할 수 있게 해주는 셀프호스팅 솔루션입니다.

| 항목 | 값 |
|------|-----|
| **Docker 이미지** | `moltbot/moltbot:latest` (Docker Hub) |
| **기본 포트** | 18789 (Gateway), 18790 (Canvas) |
| **설정 디렉토리** | `/home/node/.moltbot` |
| **워크스페이스** | `/home/node/clawd` |

---

## 설치 및 배포 방법

OpenClaw의 설치, 배포, 업데이트는 모두 **dokploy-skill**을 통해 수행합니다.

### dokploy-skill이란?

이 프로젝트에 포함된 [dokploy-skill](.claude/skills/dokploy-skills/)은 Claude Code의 스킬(skill)로, Dokploy 서버 관리를 위한 전문 도구입니다. SSH 및 API를 통해 다음 작업을 자동으로 처리합니다:

- 애플리케이션 배포 및 재배포
- Docker Compose 설정 및 관리
- 도메인 설정, SSL 인증서(Let's Encrypt), HTTPS 설정
- Traefik 리버스 프록시 설정
- 컨테이너 로그 확인 및 서버 상태 점검
- 볼륨 백업/복원
- 데이터베이스 관리
- 서버 문제 해결 및 디버깅

### 사용 방법

Claude Code에서 dokploy 관련 작업을 요청하면 자동으로 dokploy-skill이 활성화됩니다.

**예시:**

```
# 배포
"Dokploy에 OpenClaw를 배포해줘"

# 업데이트
"OpenClaw를 최신 버전으로 업데이트해줘"

# 상태 확인
"OpenClaw 컨테이너 상태를 확인해줘"

# 도메인 설정
"claw.example.com 도메인을 설정해줘"

# 로그 확인
"OpenClaw 컨테이너 로그를 확인해줘"
```

### 스킬 업데이트

dokploy-skill은 Git 서브모듈로 관리됩니다. 최신 버전으로 업데이트하려면:

```bash
git submodule update --remote .claude/skills/dokploy-skills
git add .claude/skills/dokploy-skills
git commit -m "Update dokploy-skills submodule to latest commit"
```

---

## 프로젝트 구조

```
openclaw/
├── .claude/
│   └── skills/
│       ├── dokploy-skills/    # Dokploy 서버 관리 스킬 (서브모듈)
│       └── skill-creator/     # 스킬 생성 도구 (서브모듈)
├── .gitmodules
└── README.md
```

---

## 참고 사항

- OpenClaw의 모든 서버 관리 작업은 dokploy-skill을 통해 수행하는 것을 권장합니다.
- 수동으로 Docker 명령어를 실행할 필요 없이, Claude Code에 자연어로 요청하면 됩니다.
- dokploy-skill의 상세 문서는 [SKILL.md](.claude/skills/dokploy-skills/SKILL.md)를 참조하세요.
