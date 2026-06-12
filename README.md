# Omil

디스코드 기반 **AI 공부 도우미 봇**. 주제를 주면 Top-Down 커리큘럼을 만들어 저장하고, 진도를 추적하며, 정해진 시간에 먼저 알림을 보냅니다. 사용자는 `/omil` 한 번으로 시작해, 이후엔 전용 스레드에서 자연스럽게 대화합니다.

- **스택**: Python · FastAPI · discord.py(게이트웨이) · SQLAlchemy · SQLite(개발) · APScheduler · Claude(인터페이스 뒤, 교체 가능)
- **아키텍처**: 클래식 3계층 (Interface → Service → Repository → DB) + 외부 연동 어댑터 격리
- **운영**: 단일 프로세스 (FastAPI + discord.py 게이트웨이 + APScheduler 한 앱에서 구동)

---

## 사용 흐름 (UX)

1. **시작** — 아무 채널에서 `/omil 리액트 공부할래` → 봇이 전용 스레드를 만들고 Top-Down 커리큘럼 생성·저장 후 첫 응답.
2. **대화** — 그 스레드에서 슬래시 커맨드 없이 평범하게 타이핑. 봇이 의도를 파악해 설명하거나 진도를 갱신("오늘 useState 끝냈어" → 완료 처리).
3. **조회** — "지금 어디까지 했지?" 같은 자연어로 진행률 확인.
4. **알림(아웃바운드)** — 스케줄러가 정해진 시간에 봇이 먼저 메시지 전송("공부 시간이에요" / "오늘 공부 어땠어요?").

명령어는 `/omil` 하나뿐이며, 나머지는 모두 자연어 대화로 처리됩니다.

---

## 아키텍처

### 계층 규칙

- `interface` → `services`만 호출 (repository/DB 직접 접근 금지)
- `services` → `repositories` + `adapters`만 호출 (Discord/FastAPI 객체를 직접 받지 않고 DTO·원시값으로 소통)
- `repositories` → `models` + DB 세션
- `adapters` → 외부 SDK(Discord·Claude·스케줄러)를 감싸 인터페이스로 노출 → Service는 구체 구현(Claude인지 OpenAI인지)을 모름

### 디렉터리 구조

```
omil/
  app/
    main.py                # FastAPI 앱 생성, lifespan에서 게이트웨이 봇·스케줄러 시작/종료
    config.py              # 환경설정(.env): DISCORD_PUBLIC_KEY, BOT_TOKEN, AI_API_KEY, DB_URL

    interface/             # ── Interface 계층 (인바운드 진입점)
      gateway_bot.py       #    discord.py 클라이언트: /omil 슬래시 커맨드 + 스레드 on_message (모두 게이트웨이 수신)
      health.py            #    GET /health (FastAPI)

    schemas/               # Pydantic DTO (계층 간 전달 모델)

    services/              # ── Service 계층 (도메인 로직, Discord/FastAPI를 모름)
      chat_service.py      #    대화 1턴 오케스트레이션: 문맥 로드 → AI(tool-use) → 도구 실행 → 기록 → 응답
      curriculum_service.py#    AI로 Top-Down 커리큘럼 생성·파싱·저장
      study_service.py     #    진도 완료, 다음 토픽, 진행률 계산
      reminder_service.py  #    알림 대상 조회 + 메시지 구성
      ai_tools.py          #    AI tool-use 정의 ↔ 도메인 서비스 호출 매핑

    repositories/          # ── Repository 계층 (DB 접근만)
      user_repo.py  curriculum_repo.py  topic_repo.py
      conversation_repo.py  message_repo.py  study_log_repo.py

    models/                # SQLAlchemy ORM 엔티티
    db/                    # 엔진/세션, get_session 의존성

    adapters/              # ── 외부 연동 (인터페이스 뒤로 격리)
      discord_client.py    #    아웃바운드 Discord REST (스레드 생성, 메시지/DM 전송, followup)
      ai/base.py           #    AIClient 인터페이스(Protocol): chat(messages, tools) → 응답/tool 호출
      ai/claude_client.py  #    Claude 구현 (tool-use)
      scheduler.py         #    APScheduler 래퍼
  tests/
  pyproject.toml  .env.example
```

---

## 데이터 모델

| 엔티티 | 주요 필드 | 설명 |
|---|---|---|
| **User** | `discord_user_id`, `timezone`, `reminder_time`, `reminder_enabled` | 봇 사용자 |
| **Curriculum** | `user_id`, `topic`, `status`(active/paused/completed) | 한 주제의 학습 과정 |
| **Topic** | `curriculum_id`, `parent_id`, `title`, `order_index`, `depth`, `status`(pending/in_progress/done), `completed_at` | **자기참조 트리**로 Top-Down 표현 |
| **Conversation** | `discord_thread_id`, `user_id`, `curriculum_id`, `status` | 스레드 ↔ 사용자/커리큘럼 연결 |
| **Message** | `conversation_id`, `role`(user/assistant/tool), `content`, `created_at` | 대화 기록 (AI 문맥 + 회고) |
| **StudyLog** | `user_id`, `date`, `note` | 일일 회고·streak 계산 |

### Top-Down 트리 예시

```
React (depth 0)
├─ 컴포넌트 & JSX (depth 1)
├─ 상태관리 (depth 1)
│   ├─ useState (depth 2)
│   └─ useEffect (depth 2)
└─ 라우팅 (depth 1)
```

각 Topic이 `parent_id`로 부모를 가리키며, `/next`(다음 토픽 추천)는 이 트리를 순회해 다음 미완료 토픽을 찾습니다.

---

## 핵심 흐름

### 1. 대화 시작 — `/omil` (인바운드, 게이트웨이)

```
Discord ──게이트웨이(웹소켓)──▶ interface/gateway_bot (app_commands)
  1) 슬래시 커맨드 즉시 "지연 응답(defer)"으로 3초 규칙 회피
  2) curriculum_service: AI로 커리큘럼 트리 생성·저장
  3) discord_client: 전용 스레드 생성 + 첫 응답 전송, Conversation/Message 기록
```

> 게이트웨이 연결로 커맨드를 받으므로 공개 URL·Ed25519 서명검증이 필요 없음.

### 2. 스레드 대화 (인바운드, 게이트웨이)

```
discord.py on_message (봇 자신·비추적 스레드는 무시)
  → chat_service.handle_turn(thread_id, text)
      1) conversation/message 문맥 로드
      2) AI(tool-use) 호출 — 의도에 따라 tool 호출 반환
      3) ai_tools가 도메인 서비스로 디스패치 (커리큘럼 생성/완료 처리/진행률 등)
      4) user·assistant 메시지 저장
  → discord_client로 스레드에 응답 전송
```

### 3. 능동 알림 (아웃바운드, 스케줄러)

```
APScheduler(분 단위 tick) ──▶ reminder_service.find_due()
  → 대상 User별 메시지 구성 ("공부 시간이에요" / "오늘 공부 어땠어요?")
  → discord_client로 스레드/DM 전송 → Message 기록
```

---

## 에러 처리 & 테스트

- **에러**: 서명 실패 → 401 / AI 타임아웃·실패 → 사용자에 "잠시 후 재시도" 안내 + 로깅(작업 중단 없음) / 스케줄러 잡은 try-except로 감싸 스케줄러 자체는 중단 안 됨 / DB는 세션 의존성에서 rollback / 게이트웨이 끊김은 discord.py 자동 재연결.
- **테스트(TDD)**: Service는 가짜 repo·가짜 AIClient 주입해 단위테스트 / Repository는 in-memory SQLite / interface는 TestClient + Ed25519 서명검증 테스트 / AI tool 디스패치 매핑 테스트.

---

## 보안

- 봇 토큰·AI 키는 환경변수(`.env`)로 관리, 절대 커밋하지 않음.
- discord.py는 **Message Content Intent** 활성화 필요 (봇이 100개 서버 미만이면 토글만, 심사 불필요).
- 게이트웨이 연결은 봇 토큰으로 인증된 단일 채널이라 위조 요청이 불가능 → 별도 서명검증 불필요.

## 배포

- **Railway** 단일 서비스로 상시 구동 (FastAPI + 게이트웨이 봇 + 스케줄러 한 프로세스).
- DB는 초기엔 SQLite + 볼륨으로 충분 (별도 Postgres 서비스 불필요 → 비용 절감). 규모가 커지면 Postgres로 전환.
- 능동 알림 때문에 상시 프로세스가 필요하므로 절전(sleep) 비활성 유지.
