# PRD·디자인 Q&A 디스코드 봇

Notion PRD와 Figma 디자인을 근거로 **팀의 질문에 답하는 Discord Q&A 봇**.
런타임은 [Hermes Agent](https://github.com/NousResearch)(Discord 게이트웨이 + 에이전트 루프), 빌더는 Claude Code.

리뷰/평가 봇이 아니라 **근거 기반 Q&A 봇** — "PRD·디자인 지식을 검색해서 출처와 함께 답한다"가 전부다.

## 핵심 교훈 (실측 기반)

> **실시간 MCP 참조는 신뢰성이 부족하다. 정적 지식은 "스냅샷 in 시스템 프롬프트"로 결정화하라.**

PoC에서 실시간 검색이 3겹으로 비결정적이었다:
1. 호스티드 Notion MCP(OAuth): DB 쿼리가 플랜에 막힘 → 일부 답이 빔
2. 통합토큰 Notion MCP: DB는 읽지만 연속 호출 시 404/레이트리밋 붕괴
3. Hermes 스킬: progressive-disclosure라 본문 로드가 들쭉날쭉

→ 스프린트 PRD 분량은 작으므로 **전체를 한 번 추출해 스냅샷으로 고정**하고,
Hermes의 **항상 로드되는 시스템 프롬프트(SOUL.md)** 에 내장. 런타임 검색 0 → 결정적.

| 구성 | 정답률 |
|---|---|
| 실시간 MCP (호스티드) | 75~85% (경계선) |
| 실시간 MCP (통합토큰, 부하) | ~30% (404 붕괴) |
| **스냅샷 in SOUL** | **92% (결정적)** |

## 구조

```
CLAUDE.md            # 빌더(Claude Code)용 아키텍처·작업 가이드
README.md            # 이 파일
.env.example         # 설정 템플릿 (Notion/Figma/Discord 식별자·토큰)
knowledge/           # PRD 스냅샷 (비공개 — gitignore). README 참고
eval/                # 평가셋 (questions.md 비공개). README 참고
```

봇의 런타임 산출물(SOUL 스냅샷, 스킬, 일일 업데이터 cron)은 `~/.hermes/`에 산다 — 이 레포가 아니라.

## 셋업 개요

1. Hermes 설치 + 모델 연결 (`hermes setup`)
2. Notion·Figma MCP 연결 (`hermes mcp add ...`) — DB 쿼리가 필요하면 **통합 토큰 방식**(`@notionhq/notion-mcp-server`)을 쓸 것 (호스티드는 Business 플랜 필요)
3. PRD+FR을 추출해 스냅샷 작성 → Hermes 프로파일 `SOUL.md`에 내장
4. 답변 그라운딩은 **툴셋 제한으로 강제** (`-t memory` 등 — `file/terminal/web` 켜면 봇이 로컬을 뒤져 그라운딩이 깨짐)
5. 평가셋으로 정답률 측정 (목표 80%)
6. `hermes gateway` → Discord 연결, `launchd`로 상시 구동
7. 일일 스냅샷 갱신은 `hermes cron`으로 자동화

자세한 내용·실측 로그는 `CLAUDE.md` 참고.
