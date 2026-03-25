# 에이전트 시스템 심층 분석

## 에이전트 초기화 및 라이프사이클

### `Agent` 클래스 (`src/agent/agent.js`)

`start(load_mem, init_message, count_id)` 메서드로 시작:

1. **컴포넌트 초기화**: ActionManager, Prompter, History, Coder, NPCController, MemoryBank, SelfPrompter, ConversationManager
2. **이름 검증**: 3~16자 영숫자/언더스코어 (validateNameFormat)
3. **예제 로드**: `prompter.initExamples()` — 대화/코딩 예제 + 스킬 라이브러리 임베딩
4. **메모리 로드**: `history.load()` (load_mem=true 시)
5. **태스크 생성**: `new Task(agent, settings.task, taskStartTime)`
6. **mineflayer 봇 생성**: `initBot(name)` → Minecraft 서버 접속
7. **이벤트 바인딩**: login, spawn, kicked, end, error
8. **스폰 후 초기화**: 브라우저 뷰어, 비전 인터프리터, 이벤트 핸들러, 태스크 초기화

### 업데이트 루프

```javascript
// 300ms 간격, 이전 update() 완료를 대기 (블로킹 방지)
while (true) {
    await agent.update(delta);       // modes.update() + self_prompter.update() + checkTaskDone()
    await sleep(remaining_time);     // 300ms - 실행시간
}
```

### 핵심 메서드

| 메서드 | 설명 |
|--------|------|
| `handleMessage(source, message, max_responses)` | 메시지 처리의 핵심. LLM 호출 → 명령 실행 루프 |
| `routeResponse(to_player, message)` | 응답을 적절한 대상에게 라우팅 (봇간 대화 or 공개 채팅) |
| `openChat(message)` | 번역 → 채팅 전송 + TTS + 서버 출력 |
| `requestInterrupt()` | 현재 실행 중인 코드/동작을 인터럽트 |
| `shutUp()` | 모든 채팅 중단, 셀프 프롬프팅 중단, 대화 종료 |
| `cleanKill(msg, code)` | 히스토리 저장 후 프로세스 종료 |
| `killAll()` | MindServer 전체 종료 요청 |

---

## 명령 시스템

### 구조 (`src/agent/commands/`)

- **`index.js`**: 명령어 파서, 실행기, 문서 생성
- **`actions.js`**: 월드에 영향을 미치는 동작 명령어
- **`queries.js`**: 정보만 반환하는 쿼리 명령어

### 명령어 파싱

문법: `!commandName` 또는 `!commandName("arg1", 1.2, true)`

```
정규식: /!(\w+)(?:\(((?:-?\d+(?:\.\d+)?|true|false|"[^"]*")(?:\s*,\s*(?:-?\d+(?:\.\d+)?|true|false|"[^"]*"))*)\))?/
```

- 타입 검증: `int`, `float`, `boolean`, `string`, `BlockName`, `ItemName`, `BlockOrItemName`
- 도메인 검증: `[min, max, endpointType]` (예: `[0, Infinity]`)
- 블랙리스트: `blacklistCommands()`로 특정 명령어 비활성화

### 액션 명령어 (총 35개)

`runAsAction()` 래퍼로 `ActionManager`를 통해 실행:

| 명령어 | 설명 |
|--------|------|
| `!newAction(prompt)` | LLM이 코드를 생성하여 실행 (insecure_coding 필요) |
| `!stop` | 모든 동작 강제 중지 |
| `!stfu` | 채팅/셀프 프롬프팅 중단, 현재 동작은 계속 |
| `!restart` | 에이전트 프로세스 재시작 |
| `!clearChat` | 대화 이력 초기화 |
| `!goToPlayer(name, dist)` | 플레이어에게 이동 |
| `!followPlayer(name, dist)` | 플레이어 무한 추적 |
| `!goToCoordinates(x, y, z, dist)` | 좌표로 이동 |
| `!searchForBlock(type, range)` | 블록 탐색 및 이동 |
| `!searchForEntity(type, range)` | 엔티티 탐색 및 이동 |
| `!moveAway(distance)` | 현재 위치에서 이탈 |
| `!rememberHere(name)` | 현재 위치를 이름으로 저장 |
| `!goToRememberedPlace(name)` | 저장된 위치로 이동 |
| `!givePlayer(name, item, num)` | 아이템 전달 |
| `!consume(item)` | 아이템 섭취 |
| `!equip(item)` | 아이템 장착 |
| `!putInChest(item, num)` | 상자에 넣기 |
| `!takeFromChest(item, num)` | 상자에서 꺼내기 |
| `!viewChest` | 상자 내용 확인 |
| `!discard(item, num)` | 아이템 버리기 |
| `!collectBlocks(type, num)` | 블록 채굴 (10분 타임아웃) |
| `!craftRecipe(name, num)` | 아이템 제작 |
| `!smeltItem(name, num)` | 아이템 제련 |
| `!clearFurnace` | 화로 비우기 |
| `!placeHere(type)` | 현재 위치에 블록 설치 |
| `!attack(type)` | 가장 가까운 엔티티 공격 |
| `!attackPlayer(name)` | 특정 플레이어 공격 |
| `!goToBed` | 침대에서 수면 |
| `!stay(seconds)` | 현재 위치에서 대기 (모든 모드 일시정지) |
| `!setMode(name, on)` | 모드 활성화/비활성화 |
| `!goal(prompt)` | 셀프 프롬프팅 목표 설정 |
| `!endGoal` | 셀프 프롬프팅 중단 |
| `!startConversation(name, msg)` | 다른 봇과 대화 시작 |
| `!endConversation(name)` | 대화 종료 |
| `!lookAtPlayer(name, dir)` | 플레이어 보기/같은 방향 보기 |
| `!lookAtPosition(x, y, z)` | 좌표 방향 보기 |
| `!digDown(distance)` | 아래로 파기 |
| `!goToSurface` | 지표면으로 이동 |
| `!useOn(tool, target)` | 도구 사용 |
| `!panoramicScan(prompt)` | 360도 파노라마 스캔 (비전) |
| `!showVillagerTrades(id)` | 주민 거래 목록 |
| `!tradeWithVillager(id, idx, count)` | 주민과 거래 |

### 쿼리 명령어 (11개)

동작 없이 정보만 반환:

| 명령어 | 반환 정보 |
|--------|----------|
| `!stats` | 위치, 체력, 배고픔, 시간대, 게임모드, 날씨, 바이옴, 현재 동작, 근처 플레이어 |
| `!inventory` | 인벤토리 아이템 수량 + 장비 |
| `!nearbyBlocks` | 근처 블록 유형 + 발밑/머리위 블록 |
| `!craftable` | 현재 인벤토리로 제작 가능한 아이템 |
| `!entities` | 근처 엔티티 (플레이어, 봇, 주민 직업 포함) |
| `!modes` | 모든 모드와 상태 |
| `!savedPlaces` | 저장된 위치 목록 |
| `!checkBlueprintLevel(n)` | 건축 블루프린트 레벨 검증 |
| `!checkBlueprint` | 전체 블루프린트 검증 |
| `!getBlueprint` | 블루프린트 설명 |
| `!getCraftingPlan(item, qty)` | 상세 제작 계획 |
| `!searchWiki(query)` | Minecraft 위키 검색 |
| `!help` | 전체 명령어 도움말 |

---

## History (대화 이력 & 메모리)

### `History` 클래스 (`src/agent/history.js`)

- **turns**: 현재 대화 턴 배열 `{role, content}[]`
- **memory**: 자연어 요약 메모리 (LLM이 생성)
- **max_messages**: 컨텍스트에 유지할 최대 메시지 수 (settings.max_messages)
- **summary_chunk_size**: 한 번에 요약할 메시지 수 (5개)

#### 메모리 관리 흐름

```
turns 배열이 max_messages 도달
    ↓
처음 5개 턴을 chunk로 분리
    ↓
promptMemSaving(chunk) → LLM이 요약 생성
    ↓
memory 문자열 업데이트 (500자 제한)
    ↓
chunk를 full_history 파일에 append
```

#### 저장 데이터

`./bots/{name}/memory.json`:
```json
{
    "memory": "요약 문자열",
    "turns": [...],
    "self_prompting_state": 0|1|2,
    "self_prompt": "현재 목표",
    "taskStart": 타임스탬프,
    "last_sender": "마지막 대화 상대"
}
```

---

## MemoryBank (위치 기억)

### `MemoryBank` 클래스 (`src/agent/memory_bank.js`)

단순한 키-값 저장소로 장소 좌표를 기억:

- `rememberPlace(name, x, y, z)`: 위치 저장
- `recallPlace(name)`: 위치 조회 → `[x, y, z]`
- `getKeys()`: 저장된 이름 목록
- 사망 시 `last_death_position` 자동 저장

---

## ConversationManager (봇간 대화)

### `ConversationManager` 클래스 (`src/agent/conversation.js`)

봇 간 대화를 관리하는 싱글톤:

- **대화 흐름**: `startConversation()` → `sendToBot()` → MindServer 중계 → `receiveFromBot()` → 대화 큐 → 스케줄링 → `handleMessage()`
- **스케줄링 로직**: 양쪽 바쁨 상태에 따라 응답 타이밍 결정
  - 둘 다 여유: 200ms (즉시)
  - 상대만 바쁨: 5000ms (짧은 동작 대기)
  - 내가 바쁨: LLM에게 응답 여부 질문
  - 둘 다 바쁨: talkOver 액션(stay, followPlayer, mode:*)이면 200ms, 아니면 보류
- **대화 종료**: `!endConversation` 메시지 감지 시 자동 종료
- **연결 모니터링**: 상대 봇 접속 해제 10초 후 대화 종료
- **타임아웃**: 응답 대기 30초 시작, 미응답 시 2배씩 증가

---

## SelfPrompter (자율 프롬프팅)

### `SelfPrompter` 클래스 (`src/agent/self_prompter.js`)

봇이 스스로 목표를 향해 반복적으로 LLM에게 질문하는 시스템:

#### 상태
- `STOPPED (0)`: 비활성
- `ACTIVE (1)`: 활성 루프 실행 중
- `PAUSED (2)`: 일시 중지 (대화 중)

#### 루프 동작

```
while (!interrupt) {
    msg = "You are self-prompting with goal: '{prompt}'. Your next response MUST contain a command."
    used_command = await agent.handleMessage('system', msg)
    if (!used_command) {
        no_command_count++
        if (no_command_count >= 3) → 자동 중지
    } else {
        await sleep(cooldown=2000ms)
    }
}
```

#### update(delta)에서의 자동 재시작

- 상태가 ACTIVE이고 루프가 멈춰있으면
- 에이전트가 idle 상태로 cooldown 이상 있으면
- 루프를 자동 재시작

---

## NPC 시스템

### 구조

```
npc/
├── controller.js  — NPC 목표 관리 및 실행
├── data.js        — NPC 데이터 구조
├── build_goal.js  — 건축 목표 실행
├── item_goal.js   — 아이템 획득 목표 실행
└── utils.js       — 유틸리티 (블록/아이템 매칭, 회전)
```

### `NPCController` 클래스

봇이 자율적으로 목표를 추구하는 NPC 행동 관리:

- **goals**: 달성할 목표 리스트 (아이템 또는 건축물)
- **constructions**: JSON 파일에서 로드한 건축물 설계
- **루틴**: `do_routine=true`면 밤에는 귀가하여 수면, 낮에는 목표 추구
- **idle 이벤트**: 봇이 idle이고 resume 함수가 없으면 `executeNext()` 호출

#### 목표 실행 흐름

```
executeNext()
    ↓
do_routine && 밤? → 귀가 + 수면
    ↓
executeGoal()
    ↓
건축물 목표? → BuildGoal.executeNext()
    → 누락 블록이 있으면 temp_goals에 추가
아이템 목표? → ItemGoal.executeNext()
    → 재료 트리 탐색 → 채굴/제작/제련/사냥
```

### `ItemGoal` 클래스

재귀적 아이템 획득 트리:

- `ItemWrapper`: 아이템의 모든 획득 방법을 래핑 (제작, 채굴, 제련, 사냥)
- `ItemNode`: 개별 획득 방법 (recipe, collectable, smeltable, huntable)
- 최적 방법 선택: `getDepth()` + `getFails()` 비용 기반
- 순환 의존성 감지
- 블랙리스트: 고가 블록(다이아몬드 블록 등) 제외

### `BuildGoal` 클래스

3D 블록 배열 기반 건축:

- 랜덤 방향 배치
- 누락 블록 목록 반환 → 임시 목표로 추가
- 기존 블록이 다르면 파괴 후 재배치

---

## 비전 시스템

### 구조

```
vision/
├── vision_interpreter.js  — 비전 분석 인터페이스
├── camera.js              — 오프스크린 렌더링 카메라
└── browser_viewer.js      — 브라우저 기반 뷰어
```

### `Camera` 클래스 (`camera.js`)

- **3D 렌더링**: Three.js + node-canvas-webgl로 오프스크린 렌더링
- **해상도**: 800×512
- **뷰 거리**: 12 청크
- `capture()`: 봇 시점 스크린샷 캡처 → JPEG 저장

### `VisionInterpreter` 클래스 (`vision_interpreter.js`)

- `lookAtPlayer(name, direction)`: 플레이어를 보거나 같은 방향을 본 후 분석
- `lookAtPosition(x, y, z)`: 좌표를 본 후 분석
- `analyzeImage(filename)`: 이미지 → LLM 비전 분석 + 중앙 블록 정보
- `getCenterBlockInfo()`: 시선 중앙의 블록 정보 (최대 128블록)

### `addBrowserViewer` (`browser_viewer.js`)

prismarine-viewer의 mineflayer 뷰어를 `localhost:3000+count_id`에서 호스팅

---

## 태스크 시스템

### `Task` 클래스 (`src/agent/tasks/tasks.js`)

구조화된 작업을 관리:

- **타입**: `construction`, `cooking`, `techtree`
- **검증기**: 작업 완료 여부를 주기적으로 확인
  - `ConstructionTaskValidator`: 블루프린트와 월드 블록 비교
  - `CookingCraftingTaskValidator`: 인벤토리 아이템 확인
- **Hell's Kitchen**: 파일 기반 다중 에이전트 진행 추적 (`hells_kitchen_progress.json`)
- **타임아웃**: `data.timeout` 초과 시 태스크 실패
- **초기 인벤토리**: 에이전트별 초기 아이템 지급

#### `initBotTask()` 흐름

1. 인벤토리 초기화 (`/clear`)
2. 요리 태스크면 `CookingTaskInitiator` 실행 (농장, 집, 동물 소환)
3. 봇 텔레포트
4. 대화 시작 (다중 에이전트)
5. 목표 설정

### `Blueprint` 클래스 (`construction_tasks.js`)

- `check(bot)`: 블루프린트와 실제 월드 비교 → matches/mismatches
- `explain()`: 블루프린트 텍스트 설명
- `autoBuild()`: `/setblock` 명령어 생성
- `autoDelete()`: 영역 초기화 명령어 생성
- `proceduralGeneration()`: 랜덤 방/창문/카펫/계단 생성

### `CookingTaskInitiator` (`cooking_tasks.js`)

요리 태스크 월드 셋업:
- 50×50 영역 평탄화
- 밀, 비트루트, 감자, 당근, 호박, 사탕수수, 버섯 농장 배치
- 제작대, 화로, 훈연기가 있는 집 건축
- 동물 소환 (닭, 소, 라마, 무시룸, 돼지, 토끼, 양)

---

## Coder (코드 생성)

### `Coder` 클래스 (`src/agent/coder.js`)

`!newAction` 명령어 시 LLM이 JavaScript 코드를 생성하여 실행:

#### 실행 흐름

```
generateCode(history)
    ↓ (최대 5회 시도)
LLM에게 코드 생성 요청
    ↓
코드 블록 추출 (```)
    ↓
_stageCode(code)
    ↓ 코드 전처리
    - console.log → log(bot, ...) 변환
    - interrupt 체크 코드 삽입
    - 템플릿에 삽입
    ↓
_lintCode(code) → ESLint 검사 + 스킬 함수 존재 확인
    ↓
SES Compartment에서 평가
    ↓ (노출 범위: skills, world, Vec3, log)
executionModule.main(bot) 실행
    ↓
결과 요약 반환
```

#### 보안

- **SES (Secure EcmaScript)**: `lockdown()`으로 글로벌 환경 잠금
- **Compartment**: 코드가 접근할 수 있는 객체를 명시적으로 제한
  - `skills`, `world`, `Vec3`, `log`만 노출
  - `agent`, `bot` 등 민감한 참조는 차단
- **ESLint**: 문법 오류 사전 감지

---

## ActionManager (액션 관리)

### `ActionManager` 클래스 (`src/agent/action_manager.js`)

모든 동작의 실행을 중앙 관리:

- **단일 실행**: 한 번에 하나의 액션만 실행 (`executing` 플래그)
- **인터럽트**: 새 액션이 현재 액션을 중단 가능 (10초 타임아웃)
- **이력 행동 감지**: 20ms 이내 반복 실행 감지 → 3회 시 resume 취소, 5회 시 프로세스 종료
- **타임아웃**: 분 단위 실행 제한
- **resume**: 특정 액션을 idle 시 반복 실행 (예: `followPlayer`)

#### 출력 요약

`getBotOutputSummary()`: `bot.output` 문자열을 500자 이내로 요약. 길면 처음/끝 각 250자만 표시.

---

## 모드 시스템

### `ModeController` 클래스 (`src/agent/modes.js`)

틱마다 환경에 자동 반응하는 행동 모드:

| 모드 | 인터럽트 | 기본 | 설명 |
|------|---------|------|------|
| `self_preservation` | all | ON | 익사, 화상, 낮은 체력 시 긴급 대응 |
| `unstuck` | all | ON | 20초 이상 같은 위치 → 이동 시도 |
| `cowardice` | all | ON | 16블록 내 적 → 도주 |
| `self_defense` | all | ON | 8블록 내 적 → 전투 |
| `hunting` | followPlayer | ON | 8블록 내 사냥 가능 동물 → 사냥 |
| `item_collecting` | followPlayer | ON | 8블록 내 아이템 → 2초 대기 후 수집 |
| `torch_placing` | followPlayer | ON | 어두운 곳 → 횃불 설치 (5초 쿨다운) |
| `elbow_room` | followPlayer | ON | 0.5블록 이내 플레이어 → 회피 |
| `idle_staring` | 없음 | ON | 근처 엔티티를 바라보는 애니메이션 |
| `cheat` | 없음 | OFF | `/setblock`, `/tp` 치트 사용 |

#### 우선순위

리스트 순서대로 처리. 하나의 모드가 active 상태면 나머지는 스킵.

#### 인터럽트 시스템

- `interrupts: ['all']`: idle이 아니어도 실행 가능
- `interrupts: ['action:followPlayer']`: 특정 액션만 인터럽트 가능
- 인터럽트 후 자동으로 이전 동작에 대해 재프롬프트

#### 보안 경고

> ModeController는 LLM 생성 코드에서 접근 가능 (`bot.modes`). 따라서 `agent` 같은 민감 참조를 저장하면 안 된다. 모듈 스코프의 `_agent` 변수로 분리.

---

## Speak (TTS)

### `speak()` 함수 (`src/agent/speak.js`)

큐 기반 TTS 시스템:

- **system**: OS 내장 TTS (`say`/`espeak`/`PowerShell`)
- **openai**: OpenAI Audio Speech API
- **google**: Gemini TTS (PCM → WAV 변환)
- 큐에 넣고 순차 처리 (프리패칭으로 병렬 다운로드)
- `ffplay`로 오디오 재생

---

## Connection Handler

### `connection_handler.js`

연결 해제 사유 분석:

- `name_conflict`: 중복 로그인
- `access_denied`: 화이트리스트/밴
- `server_full`: 서버 만원
- `version_mismatch`: 버전 불일치
- `maintenance`: 서버 점검
- `network_error`: 타임아웃/연결 끊김
- `behavior`: 비행/스팸 감지

`[LoginGuard]` 접두사로 일관된 로깅.

---

## MindServer Proxy

### `MindServerProxy` 클래스 (`src/agent/mindserver_proxy.js`)

각 에이전트 프로세스가 메인 프로세스의 MindServer에 연결하는 싱글톤 프록시:

- **connect(name, port)**: Socket.IO로 연결 → 설정 수신 → 프로세스 등록
- **이벤트 처리**:
  - `chat-message`: 봇간 메시지 중계 → ConversationManager
  - `agents-status`: 에이전트 목록 업데이트
  - `restart-agent`: 에이전트 재시작
  - `send-message`: 외부에서 직접 메시지 전달
  - `get-full-state`: 에이전트 전체 상태 반환

---

## 스킬 라이브러리

### `SkillLibrary` 클래스 (`src/agent/library/skill_library.js`)

코드 생성 시 관련 스킬 문서를 선택:

- **임베딩 기반**: 메시지와 스킬 문서의 코사인 유사도로 상위 N개 선택
- **fallback**: 임베딩 모델 없으면 단어 겹침 점수(word overlap) 사용
- **always_show**: `placeBlock`, `wait`, `breakBlockAt`은 항상 포함
- `getRelevantSkillDocs(message, count)`: 가장 관련 있는 스킬 문서 반환

### Settings (에이전트 로컬)

`src/agent/settings.js`: 극도로 가벼운 객체. 모든 파일에서 import 가능. `setSettings()`로 MindServer에서 받은 설정으로 교체.
