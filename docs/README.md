# Mindcraft 코드베이스 문서

Mindcraft는 LLM(대규모 언어 모델)으로 구동되는 Minecraft AI 봇 프레임워크다. 봇이 자연어로 대화하고, 명령을 실행하고, 코드를 생성하여 Minecraft 월드에서 자율적으로 행동할 수 있게 한다.

## 📚 문서 목록

| 문서 | 설명 |
|------|------|
| [ARCHITECTURE.md](./ARCHITECTURE.md) | 전체 시스템 아키텍처, 데이터 흐름, Mermaid 다이어그램 |
| [MODULES.md](./MODULES.md) | 모든 소스 파일의 상세 모듈별 분석 |
| [MODELS.md](./MODELS.md) | LLM 통합 상세 — 각 모델 프로바이더 동작 방식 |
| [AGENT_SYSTEM.md](./AGENT_SYSTEM.md) | 에이전트 시스템 심층 분석 — 라이프사이클, 명령, 메모리, NPC 등 |

## 🔑 핵심 개념

- **MindServer**: 모든 에이전트 프로세스를 관리하는 중앙 허브 (Socket.IO 기반)
- **Agent Process**: 각 봇은 별도의 Node.js 자식 프로세스로 실행
- **Prompter**: LLM과의 모든 상호작용을 관리하는 프롬프트 엔진
- **Skills/World**: mineflayer 기반의 Minecraft 동작 라이브러리
- **Modes**: 틱마다 환경에 자동 반응하는 행동 모드 시스템

## 🛠 기술 스택

- **런타임**: Node.js (ES Modules)
- **Minecraft 클라이언트**: mineflayer + 플러그인들 (pathfinder, pvp, collectblock, auto-eat, armor-manager)
- **통신**: Socket.IO (MindServer ↔ Agent Process)
- **LLM**: OpenAI, Anthropic, Google Gemini, Ollama, 그 외 다수
- **렌더링**: prismarine-viewer (브라우저 뷰어 + 오프스크린 캡처)
- **보안**: SES (Secure EcmaScript) Compartment로 생성 코드 샌드박싱
