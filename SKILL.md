---
name: gmail-triage
description: Gmail 받은편지함을 자동 분류해 미회신 메일 우선순위와 회신 초안을 Slack DM으로 공유. 두레이 승인 적체 현황 포함. 평일 09:00 자동 실행 또는 /gmail-triage 수동 실행.
triggers:
  - /gmail-triage
---

# Gmail Triage Skill

Gmail 받은편지함을 AI가 자동으로 분석해 미회신 메일의 우선순위를 분류하고, 회신 초안을 생성하며, Slack DM으로 결과를 공유합니다.

---

## 실행 전 준비

1. `~/.claude/skills/gmail-triage/config.json` 파일을 Read 도구로 읽는다.
2. 오늘 날짜 확인 (YYYY-MM-DD, Asia/Seoul 기준).
3. CronList로 `gmail-triage` 레이블 Cron이 등록되어 있는지 확인해둔다.

---

## Step 1: Gmail에서 미회신 스레드 수집

Gmail MCP `search` 도구로 검색:

```
in:inbox after:YYYY/MM/DD
```

- `YYYY/MM/DD` = 오늘로부터 `fetch_days`일 전 날짜
- 본인 이메일과 표시 이름(display name)은 `getProfile`로 가져온다. 실패 시 사용자에게 직접 묻는다.
- 추가로, 최근 발송 메일 1~2건의 서명란을 파싱해 한국어 이름 등 별칭을 추출한다:
  - `search` 도구로 `from:MY_EMAIL in:sent` 검색 후 최신 1건 본문 하단 서명 영역에서 이름 패턴 추출
  - 추출 실패 시 getProfile의 display name만 사용 (전체 중단 없음)
- 수집된 이름 목록 전체를 `MY_NAMES` 배열로 관리 (예: `["Yujin Jung", "정유진", "Diana"]`)
- 이메일/이름은 이후 모든 단계에서 `MY_EMAIL`, `MY_NAMES`로 참조한다 (하드코딩 금지).
- **미회신 기준:** 스레드의 마지막 메시지 발신자가 본인이 아닌 경우
- MCP 호출 실패 시 해당 스레드 건너뛰고 계속 진행 (전체 중단 없음)

---

## Step 2: 스레드 분류

### 두레이 스레드 먼저 분리

발신자에 `dooray` 또는 `두레이`가 포함된 스레드는 **두레이 섹션**으로 별도 처리:
- 회신 불필요, Draft 저장 없음
- **unread인 경우만** 포함 (이미 읽은 것은 제외)
- 지연 일수 계산 (아래 공식 동일)

config.json에 `high_priority_notify.sender_keywords`가 있으면 해당 키워드도 동일하게 처리.

### 나머지 스레드 자동 분류: High / Medium / Low

#### 수신 위치 확인 (분류 전 선행 판단)

각 스레드에서 본인 이메일이 `To:` 필드에 있는지 `Cc:` 필드에만 있는지 확인한다:

- **직접 수신 (To):** 정상 분류 기준 적용
- **참조 수신 (Cc):** 우선순위를 한 단계 낮춰 판단
  - High 해당 → Medium으로 강등
  - Medium 해당 → Low로 강등
  - 단, Cc라도 아래 조건 중 하나라도 해당하면 강등 없이 원래 등급 유지:
    - 메일 본문/제목에 `MY_NAMES 중 하나`(또는 이름 일부)이 직접 언급된 경우
    - `MY_EMAIL`이 To/Cc 중 유일한 내부 수신자인 경우
    - 직접 답변을 요청하는 표현이 있는 경우

#### 👀 Cc 중요 의사결정 마킹

Cc로 수신된 메일이라도 아래 조건에 해당하면 분류 등급과 무관하게 `👀 확인 필요` 태그를 추가한다:

- 본인이 의사결정 체인에 포함된 경우 (예: 조직 내 상위 결정권자로서 승인/방향 결정 필요)
- 계약, 파트너십, 예산, 인사 관련 내용이 포함된 경우
- `MY_EMAIL` 도메인 조직의 중요한 방향 전환이나 전략 결정이 논의되는 경우
- `MY_NAMES 중 하나` 또는 `MY_EMAIL`을 간접적으로 지칭하며 결정을 요청하는 표현이 있는 경우 (예: "팀장님께 확인", "담당자 의견" 등)

`👀 확인 필요` 태그가 붙은 메일은 해당 등급 섹션 내에서 별도 표시하며, 회신 초안은 생성하지 않는다.

config.json에 커스텀 규칙(`custom_rules`)이 있으면 수신 위치 판단보다 먼저 적용한다 (custom_rules는 Cc 강등 규칙보다 우선).
커스텀 규칙에 해당하지 않는 스레드는 Claude AI가 발신자 + 제목 + snippet + 수신 위치를 보고 판단:

**High** — 직접 수신(To)이며, 명확한 요청/결정/답변이 필요한 메일
- 예: 계약 검토 요청, 데이터 확인 요청, 미팅 확정 필요, 긴급 문의
- → 회신 초안 생성 + Gmail Draft 자동 저장

**Medium** — 회신이 필요하지만 당장 긴급하지 않은 메일 (또는 Cc로 온 High급 내용)
- 예: 일반 업무 문의, 자료 요청, 일정 조율, Cc로 수신된 결정 필요 메일
- → 회신 초안 제안 (Draft 저장 없음)

**Low** — 회신 불필요한 메일 (또는 Cc로 온 Medium급 내용)
- 예: 뉴스레터, 공지사항, 참조 메일, 자동 발송, Cc로 수신된 일반 업무 공유
- → 제목 목록만 표시

**지연 일수:** `delay_days = floor((오늘 - 첫 수신일) / 1일)` — 정수값

---

## Step 3: 회신 초안 생성

**High 메일:**
1. `getThread` 또는 `getMessage`로 전체 본문 가져오기 (실패 시 snippet으로 폴백)
2. 발신자 + 제목 + 본문 기반 한국어 회신 초안 작성:
   - 정중하고 간결한 업무 이메일 톤
   - 핵심 요청에 직접 응답, 3~6문장, 서명 제외

**Medium 메일:**
- snippet + 제목으로 짧은 초안 작성
- 불확실한 내용은 `[내용 확인 필요]` 플레이스홀더 사용

**Low / 두레이:** 초안 생성 없음

---

## Step 4: Gmail Draft 저장 (High만)

High 메일 각각에 대해 `createDraft` 호출:
- `to`: 원본 발신자, `subject`: `Re: [원본 제목]`, `body`: Step 3 초안, `threadId`: 원본 스레드 ID
- 성공: `✓` 기록, 실패: `⚠️` 기록 (절대 조용히 넘기지 않음)

---

## Step 5: Slack DM으로 결과 공유

**Draft 저장 완료 후 실행.**

Slack MCP로 본인 user_id 조회: `auth_test` → `users_identity` → `users_list` 순서로 시도.
`conversations_open` → `chat_postMessage`로 발송.
실패 시 Claude 채팅에 직접 출력: `⚠️ Slack DM 발송 실패. 분석 결과를 여기에 표시합니다:`

**메시지 형식:**

```
📬 Gmail Triage — YYYY-MM-DD HH:MM

🔴 High (N건)
• [발신자] 제목 — N일 지연
  → 회신 초안: "..."
  → Gmail Draft 저장 완료 ✓  (또는 ⚠️)

🟡 Medium (N건)
• [발신자] 제목 — N일 지연
  → 회신 초안: "..."
• 👀 [발신자] 제목 — N일 지연 (Cc 수신, 의사결정 관여 가능성)

⚪ Low (N건) — 회신 불필요
• 제목1, 제목2... (최대 5건, 초과 시 "외 N건")
• 👀 [발신자] 제목 — N일 지연 (Cc 수신, 의사결정 관여 가능성)

📋 두레이 확인 필요 (N건)  ← unread만
• [기안 제목] — N일 지연 (미열람)
• [기안 제목] — N일 지연 (미열람)
ℹ️ 승인 권한이 있다면 적체된 기안을 확인하세요.
```

`👀 확인 필요` 메일이 있으면 섹션 하단에 추가 안내:
```
💡 👀 표시 메일: Cc로 수신됐지만 의사결정 관여가 필요할 수 있습니다. 내용을 확인해보세요.
```

빈 카테고리는 섹션 전체 생략. 전체 0건: `✅ 미회신 메일 없음 — 받은 편지함 깨끗해요!`

---

## Step 6: 결과 후 규칙 추가 안내

Slack DM 발송 후 Claude 채팅에 아래 메시지를 출력한다:

```
분석 완료 ✅

특정 발신자나 라벨을 항상 특정 우선순위로 분류하고 싶으시면 말씀해주세요.
예: "관광청 라벨은 항상 High로 해줘", "noreply@는 항상 Low로 해줘"
→ config.json을 바로 업데이트해드릴게요.
```

사용자가 규칙을 요청하면 config.json의 `custom_rules`에 추가하고 저장한다:

```json
"custom_rules": [
  { "type": "label", "value": "관광청", "priority": "high" },
  { "type": "sender", "value": "noreply@", "priority": "low" }
]
```

---

## Step 7: Cron 자동 등록

CronList에 `gmail-triage` 레이블이 없으면 CronCreate로 등록:
- label: `gmail-triage`, schedule: `0 9 * * 1-5`, prompt: `/gmail-triage --scheduled`
- 등록 후: `✅ Cron 등록 완료: 평일 오전 9시에 자동으로 실행됩니다.`
- 이미 있으면: 건너뜀 (메시지 없음)

---

## 우선순위 규칙 변경

`~/.claude/skills/gmail-triage/config.json` 직접 편집 또는 대화로 요청:
- `fetch_days`: 검색 기간 (기본 7일)
- `custom_rules`: 고정 우선순위 규칙 배열
- `high_priority_notify.sender_keywords`: 두레이 외 추가 알림 전용 발신자

변경사항은 다음 실행 시 즉시 반영됩니다.
