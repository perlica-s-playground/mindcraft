# MODULES.md — Mindcraft 코드베이스 모듈 분석

> 이 문서는 `src/` 디렉토리 내 모든 `.js` 파일을 모듈별로 분석한 레퍼런스입니다.

---

## 1. src/mindcraft/ — 핵심 진입점 및 서버

### `src/mindcraft/mindcraft.js`
**목적:** Mindcraft 애플리케이션의 메인 진입점. 에이전트 프로세스 생성/관리/종료를 담당한다.

| 내보내기 | 설명 |
|---|---|
| `init(host_public, port, auto_open_ui)` | MindServer를 생성하고 초기화한다. UI 자동 열기 옵션 포함. |
| `createAgent(settings)` | 설정 기반으로 에이전트 프로세스를 생성한다. MC 서버 연결 및 AgentProcess 시작. |
| `getAgentProcess(agentName)` | 이름으로 에이전트 프로세스 객체를 반환한다. |
| `startAgent(agentName)` | 중단된 에이전트를 강제 재시작한다. |
| `stopAgent(agentName)` | 에이전트 프로세스를 중지한다. |
| `destroyAgent(agentName)` | 에이전트를 중지하고 프로세스 맵에서 삭제한다. |
| `shutdown()` | 모든 에이전트를 중지하고 프로세스를 종료한다. |

---

### `src/mindcraft/mindserver.js`
**목적:** Socket.io 기반 중앙 허브 서버. 에이전트 프로세스 간 통신, 외부 API, 웹앱 호스팅을 담당한다.

| 내보내기 | 설명 |
|---|---|
| `registerAgent(settings, viewer_port)` | 에이전트를 연결 목록에 등록한다. |
| `logoutAgent(agentName)` | 에이전트를 로그아웃 상태로 변경하고 상태를 갱신한다. |
| `createMindServer(host_public, port)` | Express+Socket.io HTTP 서버를 생성하고 모든 이벤트 핸들러를 등록한다. |
| `getIO()` | Socket.io 인스턴스를 반환한다. |
| `getServer()` | HTTP 서버 인스턴스를 반환한다. |
| `numStateListeners()` | 현재 상태 리스너(브라우저 뷰어) 수를 반환한다. |

**주요 소켓 이벤트:** `create-agent`, `get-settings`, `login-agent`, `chat-message`, `restart-agent`, `stop-agent`, `destroy-agent`, `shutdown`, `send-message`, `bot-output`, `listen-to-agents`

**내부 클래스:**
- `AgentConnection` — 에이전트별 소켓, 설정, 게임 상태를 보관한다.

---

### `src/mindcraft/mcserver.js`
**목적:** 마인크래프트 서버 탐색 및 연결 검증을 수행한다.

| 내보내기 | 설명 |
|---|---|
| `serverInfo(ip, port, timeout, verbose)` | 지정된 IP:포트의 MC 서버 정보(버전, 핑, 이름)를 조회한다. |
| `findServers(ip, earlyExit, timeout)` | LAN 서버를 포트 스캔으로 검색한다 (49000–65000). |
| `getServer(host, port, version)` | MC 서버를 찾고 버전 호환성을 검증한다. 자동 버전 감지 지원. |

---

## 2. src/process/ — 에이전트 프로세스 관리

### `src/process/agent_process.js`
**목적:** 에이전트를 별도의 Node.js 자식 프로세스로 생성하고 관리한다.

| 내보내기 | 설명 |
|---|---|
| `AgentProcess` (class) | 에이전트 프로세스의 라이프사이클을 관리하는 클래스. |
| `.start(load_memory, init_message, count_id)` | `init_agent.js`를 자식 프로세스로 실행한다. 비정상 종료 시 자동 재시작. |
| `.stop()` | SIGINT로 프로세스를 종료한다. |
| `.forceRestart()` | 실행 중인 프로세스를 강제 종료 후 재시작한다. |

---

### `src/process/init_agent.js`
**목적:** 에이전트 프로세스의 실제 진입점. CLI 인자를 파싱하고 Agent 인스턴스를 생성/시작한다.

| 내보내기 | 설명 |
|---|---|
| (없음 — 스크립트 실행) | yargs로 인자 파싱 후 `serverProxy.connect()` → `Agent.start()` 순서로 실행. |

**CLI 옵션:** `-n` (이름), `-l` (메모리 로드), `-m` (초기 메시지), `-c` (카운트 ID), `-p` (포트)

---

## 3. src/agent/ — 에이전트 코어

### `src/agent/agent.js`
**목적:** 에이전트의 핵심 클래스. 봇 초기화, 메시지 처리, 이벤트 루프, 태스크 관리를 통합한다.

| 내보내기 | 설명 |
|---|---|
| `Agent` (class) | 마인크래프트 AI 에이전트의 메인 클래스. |
| `.start(load_mem, init_message, count_id)` | 모든 컴포넌트(History, Coder, NPC 등)를 초기화하고 봇을 로그인시킨다. |
| `.handleMessage(source, message, max_responses)` | 수신 메시지를 처리한다. LLM 응답 생성, 명령어 감지/실행, 대화 라우팅. |
| `.routeResponse(to_player, message)` | 응답을 적절한 대상(다른 봇 또는 오픈 채팅)으로 라우팅한다. |
| `.openChat(message)` | 번역 처리 후 인게임 채팅 또는 위스퍼로 메시지를 전송한다. |
| `.startEvents()` | 커스텀 이벤트(시간, 체력, 사망 등)를 등록하고 업데이트 루프를 시작한다. |
| `.update(delta)` | 모드 업데이트, 셀프 프롬프터 업데이트, 태스크 완료 확인을 수행한다. |
| `.requestInterrupt()` | 현재 실행 중인 코드를 인터럽트한다. |
| `.clearBotLogs()` | 봇 출력 로그를 초기화한다. |
| `.shutUp()` | 모든 채팅과 셀프 프롬프팅을 중지한다. |
| `.isIdle()` | 에이전트가 유휴 상태인지 확인한다. |
| `.cleanKill(msg, code)` | 히스토리 저장 후 프로세스를 종료한다. |
| `.killAll()` | MindServer 전체를 종료한다. |

---

### `src/agent/action_manager.js`
**목적:** 에이전트 액션의 실행, 타임아웃, 인터럽트, 재개를 관리한다.

| 내보내기 | 설명 |
|---|---|
| `ActionManager` (class) | 액션 실행 관리자. |
| `.runAction(actionLabel, actionFn, options)` | 액션을 실행한다. 타임아웃, 재개(resume) 옵션 지원. |
| `.resumeAction(actionFn, timeout)` | 이전 액션을 재개하거나 새 재개 액션을 등록한다. |
| `.stop()` | 현재 실행 중인 액션을 중지한다 (10초 타임아웃). |
| `.cancelResume()` | 재개 함수를 취소한다. |
| `.getBotOutputSummary()` | 봇 출력을 요약한다 (500자 제한). |

---

### `src/agent/coder.js`
**목적:** LLM을 통해 자바스크립트 코드를 생성하고, 린트/검증/실행한다.

| 내보내기 | 설명 |
|---|---|
| `Coder` (class) | 코드 생성 및 실행 관리자. |
| `.generateCode(agent_history)` | LLM에 코드 생성을 요청하고, 린트 후 샌드박스 환경에서 실행한다. 최대 5회 시도. |
| `._lintCode(code)` | ESLint로 생성된 코드를 검증하고 존재하지 않는 함수를 확인한다. |
| `._stageCode(code)` | 코드를 정리하고, SES Compartment에서 안전하게 평가한다. |
| `._sanitizeCode(code)` | 코드블록에서 언어 접두사를 제거한다. |

---

### `src/agent/connection_handler.js`
**목적:** 봇 연결 끊김/킥 이유를 분석하고 사람이 읽을 수 있는 메시지로 변환한다.

| 내보내기 | 설명 |
|---|---|
| `log(agentName, msg)` | 콘솔과 MindServer 양쪽에 로그를 출력한다. |
| `parseKickReason(reason)` | 킥 사유를 분석하여 타입, 메시지, 치명도를 반환한다. |
| `handleDisconnection(agentName, reason)` | 연결 끊김을 중앙에서 처리한다. |
| `validateNameFormat(name)` | 에이전트 이름이 유효한 형식(3-16자, 영숫자/_)인지 검증한다. |

**에러 유형:** `name_conflict`, `access_denied`, `server_full`, `version_mismatch`, `maintenance`, `network_error`, `behavior`

---

### `src/agent/conversation.js`
**목적:** 봇 간 대화를 관리한다. 대화 시작/종료, 메시지 큐, 응답 타이밍 제어.

| 내보내기 | 설명 |
|---|---|
| `default` (ConversationManager 인스턴스) | 싱글턴 대화 관리자. |
| `.initAgent(agent)` | 에이전트 참조를 설정한다. |
| `.startConversation(send_to, message)` | 다른 봇과의 대화를 시작한다. |
| `.sendToBot(send_to, message, start, open_chat)` | 다른 봇에게 메시지를 전송한다. |
| `.receiveFromBot(sender, received)` | 다른 봇으로부터 메시지를 수신하고 큐에 추가한다. |
| `.isOtherAgent(name)` | 주어진 이름이 다른 에이전트인지 확인한다. |
| `.inConversation(other_agent)` | 현재 대화 중인지 확인한다. |
| `.endConversation(sender)` | 특정 봇과의 대화를 종료한다. |
| `.endAllConversations()` | 모든 대화를 종료한다. |
| `.updateAgents(agents)` | 에이전트 목록을 갱신한다. |
| `.responseScheduledFor(sender)` | 해당 발신자에 대한 응답이 예약되어 있는지 확인한다. |

---

### `src/agent/history.js`
**목적:** 대화 히스토리와 메모리 요약을 관리한다. 턴 저장, 메모리 압축, 파일 저장/로드.

| 내보내기 | 설명 |
|---|---|
| `History` (class) | 대화 히스토리 관리자. |
| `.getHistory()` | 현재 턴 목록을 딥카피로 반환한다. |
| `.add(name, content)` | 새 턴을 추가한다. 최대 메시지 수 초과 시 자동으로 메모리를 요약한다. |
| `.summarizeMemories(turns)` | LLM으로 턴을 요약하여 메모리를 업데이트한다 (500자 제한). |
| `.appendFullHistory(to_store)` | 전체 히스토리를 JSON 파일에 추가한다. |
| `.save()` | 메모리, 턴, 셀프프롬프트 상태를 `memory.json`에 저장한다. |
| `.load()` | `memory.json`에서 히스토리를 로드한다. |
| `.clear()` | 턴과 메모리를 초기화한다. |

---

### `src/agent/memory_bank.js`
**목적:** 이름-좌표 쌍으로 장소를 기억/회상하는 간단한 키-값 저장소.

| 내보내기 | 설명 |
|---|---|
| `MemoryBank` (class) | 장소 기억 관리자. |
| `.rememberPlace(name, x, y, z)` | 이름과 좌표를 저장한다. |
| `.recallPlace(name)` | 이름으로 좌표를 조회한다. |
| `.getJson()` | 전체 메모리를 JSON으로 반환한다. |
| `.loadJson(json)` | JSON에서 메모리를 로드한다. |
| `.getKeys()` | 저장된 장소 이름 목록을 반환한다. |

---

### `src/agent/mindserver_proxy.js`
**목적:** 에이전트 프로세스에서 MindServer로의 Socket.io 클라이언트 연결을 관리한다.

| 내보내기 | 설명 |
|---|---|
| `serverProxy` (MindServerProxy 싱글턴) | MindServer와의 연결 프록시. |
| `.connect(name, port)` | MindServer에 연결하고 설정을 수신한다. |
| `.setAgent(agent)` | 에이전트 참조를 설정한다. |
| `.login()` | MindServer에 로그인 이벤트를 전송한다. |
| `.shutdown()` | MindServer 종료를 요청한다. |
| `.getNumOtherAgents()` | 다른 에이전트 수를 반환한다. |
| `sendBotChatToServer(agentName, json)` | 다른 봇에게 채팅 메시지를 전송한다. |
| `sendOutputToServer(agentName, message)` | 일반 출력을 서버에 전송한다. |

---

### `src/agent/modes.js`
**목적:** 자동 행동 모드(자기 보호, 전투, 횃불 설치 등)를 정의하고 업데이트 루프에서 실행한다.

| 내보내기 | 설명 |
|---|---|
| `initModes(agent)` | 모드 컨트롤러를 초기화하고 봇에 연결한다. |

**정의된 모드 (modes_list):**
- `self_preservation` — 익사, 화재, 저체력 시 회피/대응
- `unstuck` — 같은 위치에 오래 머물 때 이동 시도
- `cowardice` — 적 몹으로부터 도망
- `self_defense` — 근처 적 몹 공격
- `hunting` — 유휴 시 사냥 가능한 동물 공격
- `item_collecting` — 유휴 시 아이템 수집
- `torch_placing` — 유휴 시 횃불 설치
- `elbow_room` — 다른 플레이어와 거리 유지
- `idle_staring` — 유휴 시 주변 엔티티 바라보기 애니메이션
- `cheat` — 치트 모드 (텔레포트, /setblock 사용)

**ModeController 클래스 (bot.modes에 할당):**
- `.exists(name)`, `.setOn(name, on)`, `.isOn(name)` — 모드 존재/활성화 확인/설정
- `.pause(name)`, `.unpause(name)`, `.unPauseAll()` — 모드 일시정지/해제
- `.getMiniDocs()`, `.getDocs()` — 모드 문서 반환
- `.update()` — 모든 활성 모드를 우선순위 순으로 업데이트
- `.flushBehaviorLog()` — 행동 로그를 반환하고 초기화
- `.getJson()`, `.loadJson(json)` — 모드 상태 직렬화/역직렬화

---

### `src/agent/self_prompter.js`
**목적:** 에이전트가 스스로 목표를 설정하고 반복적으로 자기 자신에게 프롬프트하는 기능.

| 내보내기 | 설명 |
|---|---|
| `SelfPrompter` (class) | 셀프 프롬프팅 관리자. |
| `.start(prompt)` | 주어진 목표로 셀프 프롬프팅을 시작한다. |
| `.startLoop()` | 셀프 프롬프트 루프를 실행한다. 명령어 미사용 시 3회 후 자동 중지. |
| `.stop(stop_action)` | 셀프 프롬프팅을 완전히 중지한다. |
| `.pause()` | 셀프 프롬프팅을 일시 정지한다 (대화 중 등). |
| `.update(delta)` | 유휴 시간 기반으로 루프를 자동 재시작한다. |
| `.isActive()` / `.isStopped()` / `.isPaused()` | 상태 확인 메서드. |
| `.shouldInterrupt(is_self_prompt)` | 핸들 메시지에서 인터럽트 여부를 결정한다. |
| `.handleUserPromptedCmd(is_self_prompt, is_action)` | 사용자 메시지로 인한 액션 시 루프를 중지한다. |
| `.handleLoad(prompt, state)` | 저장된 상태에서 복원한다. |

---

### `src/agent/settings.js`
**목적:** 모든 파일에서 import 가능한 경량 전역 설정 객체.

| 내보내기 | 설명 |
|---|---|
| `default` (settings 객체) | 현재 에이전트 설정을 담는 빈 객체. |
| `setSettings(new_settings)` | 기존 설정을 새 설정으로 교체한다. |

---

### `src/agent/speak.js`
**목적:** TTS(Text-to-Speech) 기능. 시스템 TTS 또는 OpenAI/Gemini API를 통한 음성 합성.

| 내보내기 | 설명 |
|---|---|
| `speak(text, speak_model)` | 텍스트를 음성으로 변환한다. 큐 기반으로 순차 재생. |

**지원 모델:** `system` (OS 내장 TTS), `openai/*` (OpenAI TTS), `google/*` (Gemini TTS)

---

## 4. src/agent/commands/ — 명령어 시스템

### `src/agent/commands/index.js`
**목적:** 명령어 파싱, 검증, 실행의 중앙 허브. 명령어 문법 파싱 및 타입 체크.

| 내보내기 | 설명 |
|---|---|
| `getCommand(name)` | 이름으로 명령어 객체를 반환한다. |
| `blacklistCommands(commands)` | 지정된 명령어를 비활성화한다 (`!stop`, `!stats` 등은 블록 불가). |
| `containsCommand(message)` | 메시지에서 명령어를 감지한다. |
| `commandExists(commandName)` | 명령어 존재 여부를 확인한다. |
| `parseCommandMessage(message)` | 명령어를 파싱하고 인자 타입을 검증한다. |
| `truncCommandMessage(message)` | 명령어 이후의 텍스트를 잘라낸다. |
| `isAction(name)` | 명령어가 액션인지 확인한다. |
| `executeCommand(agent, message)` | 명령어를 파싱하고 실행한다. |
| `getCommandDocs(agent)` | 모든 명령어의 도움말을 생성한다. |

---

### `src/agent/commands/actions.js`
**목적:** 에이전트가 실행할 수 있는 모든 액션 명령어를 정의한다.

| 내보내기 | 설명 |
|---|---|
| `actionsList` (배열) | 모든 액션 명령어 정의 배열. |

**정의된 명령어:**
- `!newAction(prompt)` — LLM 기반 커스텀 코드 생성/실행
- `!stop` — 모든 액션 강제 중지
- `!stfu` — 채팅/셀프프롬프팅 중지 (액션은 계속)
- `!restart` — 에이전트 프로세스 재시작
- `!clearChat` — 대화 히스토리 초기화
- `!goToPlayer(name, closeness)` — 플레이어에게 이동
- `!followPlayer(name, dist)` — 플레이어를 무한 따라가기
- `!goToCoordinates(x, y, z, closeness)` — 좌표로 이동
- `!searchForBlock(type, range)` — 블록 검색 및 이동
- `!searchForEntity(type, range)` — 엔티티 검색 및 이동
- `!moveAway(distance)` — 현재 위치에서 이탈
- `!rememberHere(name)` — 현재 위치 저장
- `!goToRememberedPlace(name)` — 저장된 위치로 이동
- `!givePlayer(name, item, num)` — 플레이어에게 아이템 전달
- `!consume(item)` — 아이템 섭취
- `!equip(item)` — 아이템 장착
- `!putInChest(item, num)` — 상자에 아이템 넣기
- `!takeFromChest(item, num)` — 상자에서 아이템 꺼내기
- `!viewChest` — 상자 내용 확인
- `!discard(item, num)` — 아이템 버리기
- `!collectBlocks(type, num)` — 블록 수집
- `!craftRecipe(name, num)` — 아이템 제작
- `!smeltItem(item, num)` — 아이템 제련
- `!clearFurnace` — 화로 비우기
- `!placeHere(type)` — 현재 위치에 블록 설치
- `!attack(type)` — 가까운 몹 공격
- `!attackPlayer(name)` — 플레이어 공격
- `!goToBed` — 침대에서 수면
- `!stay(seconds)` — 현재 위치에서 대기
- `!setMode(name, on)` — 모드 활성화/비활성화
- `!goal(prompt)` — 셀프 프롬프팅 목표 설정
- `!endGoal` — 셀프 프롬프팅 종료
- `!showVillagerTrades(id)` — 주민 거래 목록 표시
- `!tradeWithVillager(id, index, count)` — 주민과 거래
- `!startConversation(name, msg)` — 다른 봇과 대화 시작
- `!endConversation(name)` — 봇 대화 종료
- `!lookAtPlayer(name, direction)` — 플레이어를 바라보기
- `!lookAtPosition(x, y, z)` — 좌표를 바라보기
- `!digDown(distance)` — 아래로 채굴
- `!goToSurface` — 지표면으로 이동
- `!useOn(tool, target)` — 도구를 대상에 사용
- `!panoramicScan(prompt)` — 360도 촬영 후 비전 분석

---

### `src/agent/commands/queries.js`
**목적:** 정보 조회 명령어를 정의한다 (세계 상태에 영향 없음).

| 내보내기 | 설명 |
|---|---|
| `queryList` (배열) | 모든 쿼리 명령어 정의 배열. |

**정의된 명령어:**
- `!stats` — 위치, 체력, 배고픔, 시간, 날씨 등 봇 상태 조회
- `!inventory` — 인벤토리 및 착용 장비 조회
- `!nearbyBlocks` — 주변 블록 목록
- `!craftable` — 현재 인벤토리로 제작 가능한 아이템
- `!entities` — 주변 플레이어/엔티티/주민 목록
- `!modes` — 모드 목록 및 상태
- `!savedPlaces` — 저장된 장소 목록
- `!checkBlueprintLevel(levelNum)` — 블루프린트 레벨 완성도 확인
- `!checkBlueprint` — 블루프린트 전체 확인
- `!getBlueprint` — 블루프린트 설명
- `!getBlueprintLevel(levelNum)` — 레벨별 블루프린트 설명
- `!getCraftingPlan(item, qty)` — 상세 제작 계획 생성
- `!searchWiki(query)` — 마인크래프트 위키 검색
- `!help` — 명령어 도움말

---

## 5. src/agent/library/ — 스킬 및 월드 라이브러리

### `src/agent/library/index.js`
**목적:** 스킬과 월드 모듈의 JSDoc 문서를 추출하여 프롬프트에 사용할 수 있도록 한다.

| 내보내기 | 설명 |
|---|---|
| `docHelper(functions, module_name)` | 함수 배열에서 `/** ... **/` 형식의 문서를 추출한다. |
| `getSkillDocs()` | skills + world 모듈의 전체 문서 배열을 반환한다. |

---

### `src/agent/library/skills.js`
**목적:** 마인크래프트 봇이 수행할 수 있는 모든 게임 내 행동(이동, 채굴, 제작, 전투 등)의 구현체.

| 내보내기 | 설명 |
|---|---|
| `log(bot, message)` | 봇 출력에 메시지를 추가한다. |
| `craftRecipe(bot, itemName, num)` | 레시피로 아이템을 제작한다. 제작대 자동 배치 지원. |
| `wait(bot, milliseconds)` | 인터럽트 가능한 대기. |
| `smeltItem(bot, itemName, num)` | 화로에서 아이템을 제련한다. 연료 자동 투입. |
| `clearNearestFurnace(bot)` | 가장 가까운 화로의 모든 아이템을 회수한다. |
| `attackNearest(bot, mobType, kill)` | 가장 가까운 특정 몹을 공격한다. |
| `attackEntity(bot, entity, kill)` | 특정 엔티티를 공격한다. |
| `defendSelf(bot, range)` | 주변 적을 모두 처치할 때까지 방어한다. |
| `collectBlock(bot, blockType, num, exclude)` | 특정 블록을 수집한다. |
| `pickupNearbyItems(bot)` | 주변 아이템을 줍는다. |
| `breakBlockAt(bot, x, y, z)` | 지정 좌표의 블록을 파괴한다. |
| `placeBlock(bot, blockType, x, y, z, placeOn, dontCheat)` | 블록을 배치한다. 치트 모드 지원. |
| `equip(bot, itemName)` | 아이템을 장착한다 (무기/방어구/도구). |
| `discard(bot, itemName, num)` | 아이템을 버린다. |
| `putInChest(bot, itemName, num)` — 상자에 보관 |
| `takeFromChest(bot, itemName, num)` — 상자에서 꺼내기 |
| `viewChest(bot)` — 상자 내용 표시 |
| `consume(bot, itemName)` — 아이템 소비(먹기) |
| `giveToPlayer(bot, itemType, username, num)` — 플레이어에게 아이템 전달 |
| `goToGoal(bot, goal)` — 경로 탐색으로 목표까지 이동 (문 열기 지원) |
| `goToPosition(bot, x, y, z, min_distance)` — 좌표로 이동 |
| `goToNearestBlock(bot, blockType, min_distance, range)` — 가장 가까운 블록으로 이동 |
| `goToNearestEntity(bot, entityType, min_distance, range)` — 가장 가까운 엔티티로 이동 |
| `goToPlayer(bot, username, distance)` — 플레이어에게 이동 |
| `followPlayer(bot, username, distance)` — 플레이어를 끝없이 따라가기 |
| `moveAway(bot, distance)` — 현재 위치에서 이탈 |
| `moveAwayFromEntity(bot, entity, distance)` — 엔티티로부터 이탈 |
| `avoidEnemies(bot, distance)` — 적 몹 회피 |
| `stay(bot, seconds)` — 현재 위치에서 대기 (모든 모드 일시정지) |
| `useDoor(bot, door_pos)` — 문 사용 (열고 통과) |
| `goToBed(bot)` — 가장 가까운 침대에서 수면 |
| `tillAndSow(bot, x, y, z, seedType)` — 땅 경작 및 파종 |
| `activateNearestBlock(bot, type)` — 가장 가까운 블록 활성화 (레버 등) |
| `showVillagerTrades(bot, id)` — 주민 거래 목록 표시 |
| `tradeWithVillager(bot, id, index, count)` — 주민과 거래 실행 |
| `digDown(bot, distance)` — 아래로 채굴 (용암/물/낙사 안전 확인) |
| `goToSurface(bot)` — 지표면으로 이동 |
| `useToolOn(bot, toolName, targetName)` — 도구를 대상에 사용 |
| `useToolOnBlock(bot, toolName, block)` — 도구를 블록에 사용 |

---

### `src/agent/library/world.js`
**목적:** 마인크래프트 월드 상태 조회 함수 모음. 블록, 엔티티, 인벤토리, 경로 등.

| 내보내기 | 설명 |
|---|---|
| `getNearestFreeSpace(bot, size, distance)` | 빈 공간을 찾는다 (크기×크기, 아래에 단단한 블록). |
| `getBlockAtPosition(bot, x, y, z)` | 봇의 상대 좌표에서 블록을 가져온다. |
| `getSurroundingBlocks(bot)` | 봇 주변(아래/발/머리) 블록을 반환한다. |
| `getFirstBlockAboveHead(bot, ignore_types, distance)` | 머리 위 첫 번째 고체 블록을 찾는다. |
| `getNearestBlocks(bot, block_types, distance, count)` | 특정 타입의 가까운 블록 목록을 반환한다. |
| `getNearestBlocksWhere(bot, predicate, distance, count)` | 조건을 만족하는 블록을 검색한다. |
| `getNearestBlock(bot, block_type, distance)` | 가장 가까운 특정 블록 하나를 반환한다. |
| `getNearbyEntities(bot, maxDistance)` | 거리순 주변 엔티티 목록을 반환한다. |
| `getNearestEntityWhere(bot, predicate, maxDistance)` | 조건을 만족하는 가장 가까운 엔티티를 반환한다. |
| `getNearbyPlayers(bot, maxDistance)` | 주변 플레이어 목록을 반환한다. |
| `getVillagerProfession(entity)` | 주민의 직업과 레벨을 반환한다. |
| `getInventoryStacks(bot)` | 인벤토리 아이템 스택 목록을 반환한다. |
| `getInventoryCounts(bot)` | 인벤토리 아이템별 수량 객체를 반환한다. |
| `getCraftableItems(bot)` | 현재 인벤토리로 제작 가능한 아이템 목록을 반환한다. |
| `getPosition(bot)` | 봇의 현재 위치를 반환한다. |
| `getNearbyEntityTypes(bot)` | 주변 엔티티 종류 목록을 반환한다. |
| `isEntityType(name)` | 유효한 엔티티 타입인지 확인한다. |
| `getNearbyPlayerNames(bot)` | 주변 플레이어 이름 목록을 반환한다. |
| `getNearbyBlockTypes(bot, distance)` | 주변 블록 종류 목록을 반환한다. |
| `isClearPath(bot, target)` | 파괴 없이 이동 가능한 경로가 있는지 확인한다. |
| `shouldPlaceTorch(bot)` | 횃불을 설치해야 하는지 판단한다. |
| `getBiomeName(bot)` | 현재 바이옴 이름을 반환한다. |

---

### `src/agent/library/skill_library.js`
**목적:** 임베딩 기반 스킬 문서 검색 시스템. 현재 대화 맥락에 관련된 스킬 문서를 선택한다.

| 내보내기 | 설명 |
|---|---|
| `SkillLibrary` (class) | 스킬 문서 라이브러리. |
| `.initSkillLibrary()` | 모든 스킬 문서를 임베딩한다. |
| `.getAllSkillDocs()` | 전체 스킬 문서를 반환한다. |
| `.getRelevantSkillDocs(message, select_num)` | 코사인 유사도 또는 단어 중복으로 관련 스킬 문서를 선택한다. |

---

### `src/agent/library/full_state.js`
**목적:** 에이전트의 전체 상태를 구조화된 객체로 반환한다 (MindServer 상태 리스너용).

| 내보내기 | 설명 |
|---|---|
| `getFullState(agent)` | 위치, 게임플레이, 인벤토리, 주변 환경, 모드 등 전체 상태를 반환한다. |

---

### `src/agent/library/lockdown.js`
**목적:** SES(Secure EcmaScript) 기반 코드 실행 격리 환경을 설정한다.

| 내보내기 | 설명 |
|---|---|
| `lockdown()` | SES lockdown을 실행한다 (한 번만). localeTaming, consoleTaming 등 unsafe 설정. |
| `makeCompartment(endowments)` | 제한된 전역 환경의 Compartment를 생성한다. |

---

## 6. src/agent/npc/ — NPC 자율 행동

### `src/agent/npc/controller.js`
**목적:** NPC 자율 행동 컨트롤러. 목표 설정, 아이템 수집, 건축, 일상 루틴을 관리한다.

| 내보내기 | 설명 |
|---|---|
| `NPCContoller` (class) | NPC 행동 관리자. |
| `.init()` | 건축 파일을 로드하고, idle 이벤트에 자율 행동을 연결한다. |
| `.setGoal(name, quantity)` | 현재 목표를 설정한다. LLM 기반 목표 설정 지원. |
| `.executeNext()` | 다음 행동을 실행한다 (건물 출입, 목표 수행, 귀가, 취침). |
| `.executeGoal()` | 아이템/건축 목표를 수행한다. |
| `.currentBuilding()` | 봇이 현재 어느 건물 안에 있는지 반환한다. |
| `.getBuildingDoor(name)` | 건물의 문 좌표를 반환한다. |
| `.getBuiltPositions()` | 건축된 블록 좌표 목록을 반환한다. |

---

### `src/agent/npc/data.js`
**목적:** NPC 데이터 모델. 목표, 건축물, 집, 루틴 설정 등.

| 내보내기 | 설명 |
|---|---|
| `NPCData` (class) | NPC 상태 데이터 클래스. |
| `.toObject()` | 직렬화 가능한 객체로 변환한다. |
| `NPCData.fromObject(obj)` | 객체에서 NPCData를 복원한다. |

**필드:** `goals`, `curr_goal`, `built`, `home`, `do_routine`, `do_set_goal`

---

### `src/agent/npc/build_goal.js`
**목적:** 건축 목표를 수행한다. JSON 블루프린트를 기반으로 블록을 배치/파괴한다.

| 내보내기 | 설명 |
|---|---|
| `BuildGoal` (class) | 건축 실행자. |
| `.executeNext(goal, position, orientation)` | 블루프린트를 한 단계 진행한다. 부족한 자재는 missing 객체로 반환. |

---

### `src/agent/npc/item_goal.js`
**목적:** 아이템 획득 목표를 위한 의존성 트리 기반 계획/실행 시스템.

| 내보내기 | 설명 |
|---|---|
| `ItemGoal` (class) | 아이템 목표 실행자. |
| `.executeNext(item_name, item_quantity)` | 다음 단계를 실행한다 (수집/제작/제련/사냥). |

**내부 클래스:**
- `ItemNode` — 개별 아이템의 획득 방법(채집/제작/제련/사냥)과 의존성을 표현
- `ItemWrapper` — 아이템의 여러 획득 경로를 관리하고 최적 경로를 선택

---

### `src/agent/npc/utils.js`
**목적:** NPC 관련 유틸리티 함수.

| 내보내기 | 설명 |
|---|---|
| `getTypeOfGeneric(bot, block_name)` | 나무 종류나 침대 색상 등 일반 블록을 구체적 타입으로 변환한다. |
| `blockSatisfied(target_name, block)` | 블록이 목표 조건을 만족하는지 확인한다. |
| `itemSatisfied(bot, item, quantity)` | 인벤토리에 해당 아이템(또는 상위 등급)이 충분한지 확인한다. |
| `rotateXZ(x, z, orientation, sizex, sizez)` | XZ 좌표를 회전 방향에 따라 변환한다. |

---

## 7. src/agent/vision/ — 비전 시스템

### `src/agent/vision/camera.js`
**목적:** prismarine-viewer 기반 1인칭 스크린샷 캡처 시스템.

| 내보내기 | 설명 |
|---|---|
| `Camera` (class, extends EventEmitter) | 1인칭 카메라. |
| `.capture()` | 현재 시점의 JPEG 스크린샷을 저장하고 파일명을 반환한다. |

---

### `src/agent/vision/browser_viewer.js`
**목적:** prismarine-viewer의 브라우저 기반 실시간 뷰어를 추가한다.

| 내보내기 | 설명 |
|---|---|
| `addBrowserViewer(bot, count_id)` | 설정에서 활성화된 경우 웹 뷰어를 시작한다 (포트 3000+count_id). |

---

### `src/agent/vision/vision_interpreter.js`
**목적:** 스크린샷을 LLM 비전 모델로 분석하여 환경을 해석한다.

| 내보내기 | 설명 |
|---|---|
| `VisionInterpreter` (class) | 비전 해석기. |
| `.lookAtPlayer(player_name, direction)` | 플레이어를 바라보거나(at), 같은 방향을 보고(with) 스크린샷을 분석한다. |
| `.lookAtPosition(x, y, z)` | 지정 좌표를 바라보고 스크린샷을 분석한다. |
| `.getCenterBlockInfo()` | 시야 중앙의 블록 정보를 반환한다. |
| `.analyzeImage(filename)` | 스크린샷을 비전 모델로 분석한다. |

---

## 8. src/agent/tasks/ — 태스크 시스템

### `src/agent/tasks/tasks.js`
**목적:** 태스크의 정의, 초기화, 검증, 타임아웃, 봇 텔레포트 등을 관리한다.

| 내보내기 | 설명 |
|---|---|
| `Task` (class) | 태스크 관리자. |
| `.getAgentGoal()` | 에이전트별 목표 문자열을 반환한다. |
| `.isDone()` | 태스크 완료 여부를 확인한다 (검증 + 타임아웃). |
| `.setAgentGoal()` | 에이전트에 목표를 설정한다 (`!goal` 실행). |
| `.initBotTask()` | 인벤토리 초기화, 텔레포트, 대화 시작 등 태스크를 세팅한다. |
| `.teleportBots()` | 봇을 적절한 위치로 텔레포트한다. |
| `.updateAvailableAgents(agents)` | 사용 가능한 에이전트 목록을 갱신한다. |

**태스크 타입:** `construction`, `cooking`, `techtree`

---

### `src/agent/tasks/construction_tasks.js`
**목적:** 건축 태스크 검증, 블루프린트 관리, 절차적 건물 생성.

| 내보내기 | 설명 |
|---|---|
| `ConstructionTaskValidator` (class) | 블루프린트 대비 건축 완성도를 검증한다. |
| `Blueprint` (class) | 블루프린트 데이터를 관리한다. |
| `.explain()` | 블루프린트를 텍스트로 설명한다. |
| `.explainLevel(levelNum)` | 특정 레벨의 배치를 설명한다. |
| `.check(bot)` | 실제 월드와 블루프린트를 비교하여 불일치/일치를 반환한다. |
| `.checkLevel(bot, levelNum)` | 특정 레벨을 검증한다. |
| `.explainBlueprintDifference(bot)` | 전체 블루프린트의 차이점을 설명한다. |
| `.explainLevelDifference(bot, levelNum)` | 특정 레벨의 수정 필요 사항을 설명한다. |
| `.autoBuild()` | 블루프린트를 `/setblock` 명령어 목록으로 변환한다. |
| `.autoDelete()` | 블루프린트 영역을 공기로 채우는 명령어를 생성한다. |
| `resetConstructionWorld(bot, blueprint)` | 건축 영역을 초기화한다. |
| `checkLevelBlueprint(agent, levelNum)` | 레벨 완성도를 확인하는 래퍼. |
| `checkBlueprint(agent)` | 전체 블루프린트 확인 래퍼. |
| `proceduralGeneration(...)` | 절차적으로 다층 건물을 생성한다 (방, 문, 창문, 계단, 카펫). |
| `worldToBlueprint(startCoord, y, x, z, bot)` | 월드의 영역을 블루프린트로 변환한다. |
| `blueprintToTask(blueprint_data, num_agents)` | 블루프린트를 태스크 데이터로 변환한다. |

---

### `src/agent/tasks/cooking_tasks.js`
**목적:** 요리 태스크 환경 초기화. 농작물, 동물, 집 등을 자동 배치한다.

| 내보내기 | 설명 |
|---|---|
| `CookingTaskInitiator` (class) | 요리 태스크 환경 설정자. |
| `.init()` | 작물 심기, 동물 소환, 집 건설, 치트 명령어로 환경을 구축한다. |
| `.plantCrops(x, z, crop, till)` | 6×6 농작물을 심는다. |
| `.plantSugarCane(patches)` | 사탕수수를 물 주변에 심는다. |
| `.plantMushrooms(x, z)` | 버섯을 균사토 위에 심는다. |
| `.summonAnimals(animals, amount)` | 동물을 소환한다. |
| `.killEntities(entities)` | 특정 엔티티를 제거한다. |
| `.buildHouse(x, z)` | 돌벽돌 집을 건설한다 (화로, 제작대, 훈연기 포함). |

---

## 9. src/models/ — AI 모델 인터페이스

### `src/models/_model_map.js`
**목적:** 모델 API 매핑 시스템. 프로필에서 적절한 모델 클래스를 자동으로 선택/생성한다.

| 내보내기 | 설명 |
|---|---|
| `selectAPI(profile)` | 모델 이름에서 API를 자동 감지하고 프로필을 정규화한다. |
| `createModel(profile)` | 프로필 기반으로 모델 인스턴스를 생성한다. |

**동적 로딩:** 같은 디렉토리의 모든 `.js` 파일에서 `static prefix` 속성을 가진 클래스를 자동으로 발견한다.

---

### `src/models/prompter.js`
**목적:** LLM 프롬프트 생성/전송의 중앙 관리자. 프로필 설정, 예제 로드, 프롬프트 템플릿 치환.

| 내보내기 | 설명 |
|---|---|
| `Prompter` (class) | 프롬프트 관리자. |
| `.getName()` | 에이전트 이름을 반환한다. |
| `.getInitModes()` | 초기 모드 설정을 반환한다. |
| `.initExamples()` | 대화/코딩 예제를 임베딩으로 로드한다. |
| `.replaceStrings(prompt, messages, ...)` | 프롬프트 템플릿의 변수(`$NAME`, `$STATS`, `$INVENTORY` 등)를 치환한다. |
| `.promptConvo(messages)` | 대화 응답을 생성한다 (최대 3회 재시도). |
| `.promptCoding(messages)` | 코드 생성 프롬프트를 전송한다. |
| `.promptMemSaving(to_summarize)` | 메모리 요약을 생성한다. |
| `.promptShouldRespondToBot(new_message)` | 다른 봇 메시지에 응답할지 결정한다. |
| `.promptVision(messages, imageBuffer)` | 비전 분석을 요청한다. |
| `.promptGoalSetting(messages, last_goals)` | LLM에 다음 목표를 물어본다 (deprecated). |

**지원 변수:** `$NAME`, `$STATS`, `$INVENTORY`, `$ACTION`, `$COMMAND_DOCS`, `$CODE_DOCS`, `$EXAMPLES`, `$MEMORY`, `$TO_SUMMARIZE`, `$CONVO`, `$SELF_PROMPT`, `$LAST_GOALS`, `$BLUEPRINTS`

---

### `src/models/gpt.js`
**목적:** OpenAI GPT API 인터페이스. 채팅, 임베딩, 비전, TTS를 지원한다.

| 내보내기 | 설명 |
|---|---|
| `GPT` (class, prefix: `openai`) | OpenAI 모델 클라이언트. |
| `.sendRequest(turns, systemMessage, stop_seq)` | 채팅 완성 요청을 전송한다. 추론 모델(o1/o3) 대응. |
| `.sendVisionRequest(turns, systemMessage, imageBuffer)` | 이미지를 포함한 비전 요청을 전송한다. |
| `.embed(text)` | 텍스트 임베딩 벡터를 반환한다. |
| `TTSConfig` (객체) | TTS 설정 및 `sendAudioRequest()` 함수. |

---

### `src/models/claude.js`
**목적:** Anthropic Claude API 인터페이스.

| 내보내기 | 설명 |
|---|---|
| `Claude` (class, prefix: `anthropic`) | Anthropic 모델 클라이언트. |
| `.sendRequest(turns, systemMessage)` | 채팅 요청 전송. thinking 모드 지원. |
| `.sendVisionRequest(turns, systemMessage, imageBuffer)` | 비전 요청 전송. |
| `.embed(text)` | 임베딩 (비지원, 빈 배열 반환). |

---

### `src/models/gemini.js`
**목적:** Google Gemini API 인터페이스. 안전 설정, 비전, TTS를 지원한다.

| 내보내기 | 설명 |
|---|---|
| `Gemini` (class, prefix: `google`) | Google Gemini 모델 클라이언트. |
| `.sendRequest(turns, systemMessage)` | 채팅 요청 전송. |
| `.sendVisionRequest(turns, systemMessage, imageBuffer)` | 단일/다중 이미지 비전 분석. |
| `.embed(text)` | 텍스트 임베딩 벡터를 반환한다. |
| `TTSConfig` (객체) | Gemini TTS 설정 및 `sendAudioRequest()` 함수. |

---

### `src/models/ollama.js`
**목적:** Ollama 로컬 모델 API 인터페이스.

| 내보내기 | 설명 |
|---|---|
| `Ollama` (class, prefix: `ollama`) | Ollama 로컬 모델 클라이언트. |
| `.sendRequest(turns, systemMessage)` | 채팅 요청 전송 (최대 5회 재시도). |
| `.embed(text)` | 로컬 임베딩을 반환한다. |

---

### `src/models/groq.js`
**목적:** Groq Cloud API 인터페이스.

| 내보내기 | 설명 |
|---|---|
| `GroqCloudAPI` (class, prefix: `groq`) | Groq 모델 클라이언트. |
| `.sendRequest(turns, systemMessage, stop_seq)` | 채팅 요청 전송. |
| `.embed(text)` | 임베딩 (비지원, 빈 배열 반환). |

---

### `src/models/replicate.js`
**목적:** Replicate API 인터페이스 (스트리밍).

| 내보내기 | 설명 |
|---|---|
| `ReplicateAPI` (class, prefix: `replicate`) | Replicate 모델 클라이언트. |
| `.sendRequest(turns, systemMessage)` | 스트리밍으로 응답을 수신한다. |
| `.embed(text)` | 임베딩 (비지원, 빈 배열 반환). |

---

### `src/models/huggingface.js`
**목적:** Hugging Face Inference API 인터페이스.

| 내보내기 | 설명 |
|---|---|
| `HuggingFace` (class, prefix: `huggingface`) | HuggingFace 모델 클라이언트. |
| `.sendRequest(turns, systemMessage)` | 스트리밍 채팅 요청 전송. |
| `.embed(text)` | 임베딩 (비지원, 빈 배열 반환). |

---

### `src/models/mistral.js`
**목적:** Mistral AI API 인터페이스.

| 내보내기 | 설명 |
|---|---|
| `Mistral` (class, prefix: `mistral`) | Mistral 모델 클라이언트. |
| `.sendRequest(turns, systemMessage)` | 채팅 요청 전송. |
| `.embed(text)` | 임베딩 (비지원, 빈 배열 반환). |

---

### `src/models/deepseek.js`
**목적:** DeepSeek API 인터페이스 (OpenAI 호환).

| 내보내기 | 설명 |
|---|---|
| `DeepSeek` (class, prefix: `deepseek`) | DeepSeek 모델 클라이언트. |
| `.sendRequest(turns, systemMessage, stop_seq)` | 채팅 요청 전송. |
| `.embed(text)` | 임베딩 (비지원, 빈 배열 반환). |

---

### `src/models/qwen.js`
**목적:** Qwen (통의천문) API 인터페이스 (DashScope 호환).

| 내보내기 | 설명 |
|---|---|
| `Qwen` (class, prefix: `qwen`) | Qwen 모델 클라이언트. |
| `.sendRequest(turns, systemMessage, stop_seq)` | 채팅 요청 전송. |
| `.embed(text)` | 임베딩 (비지원, 빈 배열 반환). |

---

### `src/models/grok.js`
**목적:** xAI Grok API 인터페이스 (OpenAI 호환).

| 내보내기 | 설명 |
|---|---|
| `Grok` (class, prefix: `xai`) | Grok 모델 클라이언트. |
| `.sendRequest(turns, systemMessage)` | 채팅 요청 전송. |
| `.embed(text)` | 임베딩 (비지원, 빈 배열 반환). |

---

### `src/models/openrouter.js`
**목적:** OpenRouter API 인터페이스. 다양한 모델을 단일 API로 접근.

| 내보내기 | 설명 |
|---|---|
| `OpenRouter` (class, prefix: `openrouter`) | OpenRouter 모델 클라이언트. |
| `.sendRequest(turns, systemMessage, stop_seq)` | 채팅 요청 전송. |
| `.embed(text)` | 임베딩 (비지원, 빈 배열 반환). |

---

### `src/models/azure.js`
**목적:** Azure OpenAI API 인터페이스. GPT 클래스를 상속한다.

| 내보내기 | 설명 |
|---|---|
| `AzureGPT` (class, extends GPT, prefix: `azure`) | Azure OpenAI 클라이언트. apiVersion 필수. |

---

### `src/models/glhf.js`
**목적:** GLHF.chat API 인터페이스.

| 내보내기 | 설명 |
|---|---|
| `GLHF` (class, prefix: `glhf`) | GLHF 모델 클라이언트. |
| `.sendRequest(turns, systemMessage, stop_seq)` | 채팅 요청 전송. |
| `.embed(text)` | 임베딩 (비지원, 빈 배열 반환). |

---

### `src/models/hyperbolic.js`
**목적:** Hyperbolic API 인터페이스 (HTTP 직접 호출).

| 내보내기 | 설명 |
|---|---|
| `Hyperbolic` (class, prefix: `hyperbolic`) | Hyperbolic 모델 클라이언트. |
| `.sendRequest(turns, systemMessage, stopSeq)` | 채팅 요청 전송 (fetch 기반). |
| `.embed(text)` | 임베딩 (비지원, 빈 배열 반환). |

---

### `src/models/novita.js`
**목적:** Novita AI API 인터페이스 (OpenAI 호환).

| 내보내기 | 설명 |
|---|---|
| `Novita` (class, prefix: `novita`) | Novita 모델 클라이언트. |
| `.sendRequest(turns, systemMessage, stop_seq)` | 채팅 요청 전송. |
| `.embed(text)` | 임베딩 (비지원, 빈 배열 반환). |

---

### `src/models/vllm.js`
**목적:** vLLM/SGLang 셀프호스팅 API 인터페이스.

| 내보내기 | 설명 |
|---|---|
| `VLLM` (class, prefix: `vllm`) | vLLM 모델 클라이언트. |
| `.sendRequest(turns, systemMessage, stop_seq)` | 채팅 요청 전송. |
| `.embed(text)` | OpenAI 임베딩 API를 사용한다. |

---

### `src/models/cerebras.js`
**목적:** Cerebras Cloud API 인터페이스.

| 내보내기 | 설명 |
|---|---|
| `Cerebras` (class, prefix: `cerebras`) | Cerebras 모델 클라이언트. |
| `.sendRequest(turns, systemMessage, stop_seq)` | 채팅 요청 전송. |
| `.embed(text)` | 임베딩 (비지원, 빈 배열 반환). |

---

### `src/models/mercury.js`
**목적:** Mercury (Inception Labs) API 인터페이스.

| 내보내기 | 설명 |
|---|---|
| `Mercury` (class, prefix: `mercury`) | Mercury 모델 클라이언트. |
| `.sendRequest(turns, systemMessage, stop_seq)` | 채팅 요청 전송. |
| `.embed(text)` | 임베딩 (비지원, 빈 배열 반환). |

---

## 10. src/utils/ — 유틸리티

### `src/utils/translator.js`
**목적:** Google Translate API를 이용한 메시지 번역.

| 내보내기 | 설명 |
|---|---|
| `handleTranslation(message)` | 메시지를 설정된 언어로 번역한다. |
| `handleEnglishTranslation(message)` | 메시지를 영어로 번역한다. |

---

### `src/utils/examples.js`
**목적:** 임베딩 기반 유사 예제 검색 시스템. 프롬프트에 관련 예제를 삽입한다.

| 내보내기 | 설명 |
|---|---|
| `Examples` (class) | 예제 관리자. |
| `.load(examples)` | 예제를 로드하고 임베딩한다. |
| `.getRelevant(turns)` | 현재 대화와 가장 유사한 예제를 반환한다. |
| `.createExampleMessage(turns)` | 선택된 예제를 프롬프트 메시지로 포맷한다. |

---

### `src/utils/math.js`
**목적:** 수학 유틸리티.

| 내보내기 | 설명 |
|---|---|
| `cosineSimilarity(a, b)` | 두 벡터의 코사인 유사도를 계산한다. |

---

### `src/utils/text.js`
**목적:** 텍스트 처리 유틸리티. 턴 직렬화, 프롬프트 포맷팅, 메시지 정규화.

| 내보내기 | 설명 |
|---|---|
| `stringifyTurns(turns)` | 턴 배열을 읽기 쉬운 텍스트로 변환한다. |
| `toSinglePrompt(turns, system, stop_seq, model_nickname)` | 턴을 단일 프롬프트 문자열로 합친다 (비채팅 모델용). |
| `wordOverlapScore(text1, text2)` | 두 텍스트의 단어 중복 점수를 계산한다 (Jaccard 유사). |
| `strictFormat(turns)` | 턴 순서/역할을 엄격하게 정규화한다. system→user 변환, 반복 역할 병합. |

---

### `src/utils/keys.js`
**목적:** API 키를 `keys.json` 또는 환경 변수에서 로드한다.

| 내보내기 | 설명 |
|---|---|
| `getKey(name)` | API 키를 반환한다. 없으면 에러를 던진다. |
| `hasKey(name)` | API 키 존재 여부를 확인한다. |

---

### `src/utils/mcdata.js`
**목적:** 마인크래프트 데이터 접근 및 봇 초기화. 아이템/블록/레시피/엔티티 정보 조회.

| 내보내기 | 설명 |
|---|---|
| `WOOD_TYPES` | 나무 종류 목록 (`oak`, `spruce`, `birch` 등). |
| `MATCHING_WOOD_BLOCKS` | 나무 종류별 대응 블록 (`log`, `planks`, `door` 등). |
| `WOOL_COLORS` | 양털 색상 목록. |
| `initBot(username)` | mineflayer 봇을 생성하고 플러그인을 로드한다. |
| `isHuntable(mob)` | 사냥 가능한 동물인지 확인한다 (성체만). |
| `isHostile(mob)` | 적대적 몹인지 확인한다. |
| `mustCollectManually(blockName)` | collectBlock으로 수집 불가한 블록인지 확인한다. |
| `getItemId(itemName)` | 아이템 이름으로 ID를 반환한다. |
| `getItemName(itemId)` | 아이템 ID로 이름을 반환한다. |
| `getBlockId(blockName)` | 블록 이름으로 ID를 반환한다. |
| `getBlockName(blockId)` | 블록 ID로 이름을 반환한다. |
| `getEntityId(entityName)` | 엔티티 이름으로 ID를 반환한다. |
| `getAllItems(ignore)` | 모든 아이템 목록을 반환한다. |
| `getAllItemIds(ignore)` | 모든 아이템 ID 목록을 반환한다. |
| `getAllBlocks(ignore)` | 모든 블록 목록을 반환한다. |
| `getAllBlockIds(ignore)` | 모든 블록 ID 목록을 반환한다. |
| `getAllBiomes()` | 모든 바이옴을 반환한다. |
| `getItemCraftingRecipes(itemName)` | 아이템의 제작 레시피를 반환한다. |
| `isSmeltable(itemName)` | 제련 가능한 아이템인지 확인한다. |
| `getSmeltingFuel(bot)` | 인벤토리에서 연료를 찾는다. |
| `getFuelSmeltOutput(fuelName)` | 연료 1개당 제련 가능 횟수를 반환한다. |
| `getItemSmeltingIngredient(itemName)` | 제련 결과물의 원료를 반환한다. |
| `getItemBlockSources(itemName)` | 아이템을 드롭하는 블록 목록을 반환한다. |
| `getItemAnimalSource(itemName)` | 아이템을 드롭하는 동물을 반환한다. |
| `getBlockTool(blockName)` | 블록 채굴에 필요한 도구를 반환한다. |
| `makeItem(name, amount)` | prismarine Item 인스턴스를 생성한다. |
| `ingredientsFromPrismarineRecipe(recipe)` | prismarine 레시피에서 재료를 추출한다. |
| `calculateLimitingResource(available, required, discrete)` | 제한 자원과 가능 횟수를 계산한다. |
| `getDetailedCraftingPlan(targetItem, count, inventory)` | 상세 제작 계획을 텍스트로 반환한다. |

---

> 이 문서는 mindcraft 코드베이스의 전체 구조를 이해하기 위한 참조 문서입니다.
> 각 함수의 구현 세부사항보다는 **역할과 인터페이스**에 초점을 맞추었습니다.
