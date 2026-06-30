# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

# PRD·디자인 Q&A 디스코드 봇

## 무엇을 만드나
Notion PRD와 Figma 디자인을 근거로 **팀의 질문에 답해주는 Discord 봇**.
**Q&A 봇**이다. 즉 "PRD·디자인 지식을 검색해서 근거 있게 답한다"가 전부다.
(예: "로그인 화면에 뭐가 있어?", "결제 플로우 어떻게 돼?", "이 기능 요구사항이 뭐였지?")

런타임은 **Hermes Agent**(Nous Research) — Discord 게이트웨이와 에이전트 루프를 기본 제공.
설계·빌드는 **Claude Code + oh-my-claudecode(OMC)** 로 진행한다. OMC는 Claude Code 위에 얹는 멀티에이전트 오케스트레이션 플러그인(빌더 전용)이다.
Hermes는 만들어지는 결과물(런타임)이다. **셋의 역할을 혼동하지 말 것** — OMC = 빌드 시점 오케스트레이션, Hermes = 런타임 에이전트. OMC의 에이전트/팀은 산출물에 들어가지 않는다.

## 아키텍처 — 2단계로 진행 (MCP PoC → RAG)

### ① PoC (지금 목표): 런타임에 MCP로 실시간 참조
```
Discord → Hermes Agent → MCP 커넥터(Notion · Figma) → 실시간 데이터
```
- 벡터 DB 없음. Hermes가 질문마다 Notion/Figma MCP를 호출해서 답한다.
- 목적: 빠르게 동작 증명 + 어떤 질문이 들어오는지, 토큰을 얼마나 쓰는지, 어디서 검색 품질이 떨어지는지 로그 수집.

### ② RAG (병목 보이면 전환): 인덱싱 후 검색
```
[오프라인] MCP·API 수집 → 인덱싱 파이프라인(청킹·임베딩) → 벡터 DB(Qdrant)
[런타임]   Discord → Hermes Agent(RAG 검색 스킬) → 벡터 DB
```
- **MCP는 버리는 게 아니라 역할 전환**: 런타임 호출 경로 → 데이터 수집 경로로 재사용.
- Discord와 Hermes Agent는 두 단계 내내 그대로. 바뀌는 건 데이터 접근 레이어뿐.

## PoC 결과 & 현재 아키텍처 (2026-06-29, 실측 기반)

**핵심 교훈: 실시간 MCP 참조는 신뢰성 부족 → 정적 지식은 "스냅샷 in 시스템 프롬프트"로 결정화.**

- 실시간 검색이 3겹으로 비결정적이었음(실측):
  1. 호스티드 Notion MCP(OAuth): **DB 쿼리 불가** (Business 플랜 필요). 요구사항(FR)이 DB에 있어 일부 기능 답이 비었음.
  2. `notion-api`(통합토큰, npx `@notionhq/notion-mcp-server`): DB는 읽지만 **연속 호출 시 404/레이트리밋 붕괴** (27문항 배치에서 ~30%로 폭락).
  3. Hermes 스킬: progressive-disclosure라 **본문이 들쭉날쭉 로드**됨.
- 해결: **현재 스프린트 PRD+FR 전체를 한 번 추출해 스냅샷으로 고정** → Hermes **SOUL.md(항상 로드되는 시스템 프롬프트)** 에 내장. 런타임 검색 0.
  - 평가셋 재측정: 실시간 75~85%/30% → **스냅샷 92% (25/27), 결정적.**
- **격리**: 봇은 `qa-bot` 프로파일(`~/.hermes/profiles/qa-bot`, 래퍼 명령 `qa-bot`)로 분리. 전역 hermes SOUL은 기본값 유지(오염 없음).
  - 프로파일 SOUL: `~/.hermes/profiles/qa-bot/SOUL.md` (스냅샷 내장)
  - 원본 스냅샷: 레포 `knowledge/<스프린트>.md` (gitignore — 비공개 PRD 데이터). **PRD 변경 시 `notion-api`로 재추출(레이트리밋 피해 간격 두고) → SOUL 동기화.**
  - 평가셋: 레포 `eval/questions.md` (27문항, 목표 80%).
- 남은 갭: **Figma 깊이 조회**(파일 6~23MB, nodeId로 좁혀도 불안정) — 디자인 화면 세부 질문은 아직 약함. 필요 시 Figma 메타데이터도 오프라인 스냅샷에 포함.
- ②단계(벡터DB) 전환은 **스프린트가 누적돼 스냅샷이 시스템 프롬프트에 안 들어갈 만큼 커질 때**. 단일 스프린트는 스냅샷으로 충분.

## 설계 원칙 (PoC 단계부터 지킬 것)
1. **검색 인터페이스를 추상화한다.** Hermes가 호출하는 함수를 `search_context(query) -> chunks` 형태로 통일.
   1단계에선 내부 구현이 MCP 호출, 2단계에선 벡터 검색. Hermes 쪽 변경은 스킬 하나로 끝나게.
2. **근거 기반 답변 프롬프트 + 툴셋 제한(둘 다 필수).** "주어진 컨텍스트 안에서만 답하고, 컨텍스트에 없으면 모른다고 말해라." (환각 방지가 Q&A 봇 품질의 핵심)
   - ⚠️ **프롬프트만으론 안 된다(실측됨).** `file`·`terminal`·`web` 툴이 켜져 있으면 에이전트가 PRD/Figma 대신 로컬 파일시스템·다른 레포를 뒤져 엉뚱하게 답한다.
   - **봇은 반드시 제한된 툴셋으로 실행한다: `-t notion,figma,skills,memory`** (file/terminal/code_execution/browser/web/computer_use 제외). 그래야 물리적으로 Notion/Figma만 보고, 없으면 "모른다"가 강제된다.
3. **답변에 출처 표기.** 메타데이터에 출처(어느 PRD / 어느 화면)를 심어 답변 근거로 노출. (교차검증용 아님, 단순 출처용)
4. **Figma는 노드 메타데이터만.** 텍스트 레이어·프레임 이름·컴포넌트 이름을 텍스트로 추출해 사용. VLM 캡셔닝은 일단 제외(필요해지면 나중에).
5. **전환 트리거를 숫자로.** "질문당 평균 토큰 X 초과" 또는 "검색 정확도 체감 저하" 같은 기준을 PoC 로그로 정한 뒤 ②로 넘어간다.

## 환경
- **PoC는 개발·운영 모두 Mac (Apple Silicon) 로컬**: 빌드·스킬 작성은 물론 Hermes 게이트웨이도 Mac에서 24/7 상시 구동한다. (Docker 홈랩·Oracle 계획은 후순위 옵션으로 보류)
  - Discord 게이트웨이는 **아웃바운드 websocket만** 쓴다 → 인바운드 포트 개방·공유기 포트포워딩 불필요.
  - 상시 구동: `launchd`(또는 tmux/pm2)로 `hermes gateway`를 데몬화 + `caffeinate`로 Mac 절전 방지.
  - **전제: Mac을 항상 켜둔다.** 항상 켜둘 수 없게 되면 아래 "배포 옵션"으로 이전.
- **② 단계 Qdrant**: 같은 Mac에 네이티브 바이너리로(Docker 안 씀). `localhost` 전용.
- 시크릿(봇 토큰, API 키)은 `~/.hermes/.env` 또는 Hermes config에 둔다. **레포에 커밋 금지.** `.gitignore` 확인.

## PoC 성공 기준 (이 숫자를 넘으면 "동작 증명" 완료)
- 실제 팀이 던질 만한 **대표 질문 20개**로 평가셋 구성.
- 그중 **16개 이상(80%)** 에 **출처가 달린 정답**. PoC 판정은 수동(눈으로).
- 컨텍스트에 근거 없으면 단정 대신 "모른다" — **틀린 단정은 오답 처리(환각 0이 최우선)**.
- 각 답변에 출처(PRD 페이지 / Figma 화면) 노출.
- 단발 멘션 Q&A 기준. 멀티턴 대화는 Non-Goal.
- 상세 스펙: `.omc/specs/deep-interview-prd-design-qa-bot.md` (deep-interview 산출물)

## PoC 단계 할 일 (순서대로)
- [ ] Hermes Agent 설치 및 `hermes doctor` 통과
- [ ] `hermes setup`으로 LLM 프로바이더·모델 연결 (Nous Portal / OpenRouter / OpenAI / 로컬 중 택1)
- [ ] `hermes` CLI에서 모델이 응답하는지 스모크 테스트 (게이트웨이 붙이기 전에 먼저)
- [ ] `hermes mcp`로 Notion MCP 연결 + 인증
- [ ] `hermes mcp`로 Figma MCP(또는 Figma REST API 경로) 연결
- [ ] `hermes tools`에서 위 MCP 툴셋 활성화
- [ ] `search_context(query)` 추상화 + 근거 기반 답변 프롬프트를 스킬로 작성
- [ ] **대표 질문 20개 평가셋** 작성 (성공 기준 측정용)
- [ ] CLI에서 평가셋 답변 검증 → 출처 있는 정답 16개(80%) 이상 확인
- [ ] `hermes gateway setup` → Discord 봇 토큰 연결 → 채널 멘션 테스트 (Mac 로컬 상시 구동 상태)
- [ ] 질문/토큰/오답 로그 수집 시작

## 배포 옵션 (Mac 상시 구동이 어려워지면)
PoC는 Mac 로컬이 기본. Mac을 24/7 켜둘 수 없거나 외부 접근이 필요해지면 아래로 이전한다(우선순위 순):
1. **집 홈랩에 직접** — 이미 보유. `systemd`로 상시 구동, ②단계 Qdrant도 같은 박스. 비용 0.
2. **Hetzner VPS** — 월 €4~, ARM(CAX11) 가능. 클라우드 중 가성비 최고.
3. **Oracle Cloud Always Free (Ampere A1 / ARM)** — 무료지만 가용성 변동 가능 → 백업 카드.
- 공통: 시크릿은 서버에서 직접 입력(전송·커밋 금지), `hermes gateway`를 systemd로 등록해 재부팅 자동복구, 인바운드는 SSH(22)만.
- **같은 봇 토큰으로 두 곳에서 게이트웨이 동시 구동 금지.** (Mac ↔ 이전한 서버 중 한 곳만)

## Hermes 주요 명령 참고
```
hermes setup            # 전체 설정 마법사
hermes model            # LLM 프로바이더/모델 선택 (코드 변경 없이 교체)
hermes tools            # 활성화할 툴셋(=MCP 포함) 구성
hermes mcp ...          # MCP 서버 설치/설정/인증 (원격은 OAuth 2.1 PKCE)
hermes gateway setup    # 메시징 게이트웨이(Discord 등) 설정
hermes gateway          # 게이트웨이 실행
hermes doctor           # 진단
```
- Hermes는 모델에 종속되지 않음(OpenAI 호환 엔드포인트면 무엇이든). 처음엔 가벼운 API로 시작, 나중에 홈랩 로컬 모델로 교체 가능.
- 스킬은 agentskills.io 표준 호환. `~/.hermes/skills/`에 두면 슬래시 커맨드로 자동 등록. 커스텀 RAG 검색 스킬도 여기로.

## 작업 방식
- **설계 도구 = Claude Code + oh-my-claudecode(OMC)**. 빌더 전용. 산출물(스킬·프롬프트·`search_context` 추상화)만 Hermes 호환(agentskills.io 표준)이면 OMC를 어떻게 쓰든 무관.
  - 설치: Claude Code에서 `/plugin marketplace add https://github.com/Yeachan-Heo/oh-my-claudecode` → `/plugin install oh-my-claudecode` → `/setup`. (CLI 대안: `npm i -g oh-my-claude-sisyphus@latest`)
  - `/team` `/autopilot` `/deep-interview` 등은 **빌드 시점에만** 동작. Hermes(런타임)와 혼동 금지.
- 한 번에 한 레이어씩. CLI → 모델 → 툴(MCP) → Discord 순으로, 각 단계를 눈에 보이는 결과로 검증한 뒤 다음으로.
- 막히면 `hermes doctor` 먼저.
