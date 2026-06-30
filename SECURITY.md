- 이 저장소의 보안 정책. **사람이든 AI 에이전트(Claude Code, Hermes 등)든 여기서 작업할 때 반드시 따른다.**

## 시크릿 위치 (저장소엔 없음)
모든 토큰/시크릿은 `~/.hermes/`(Hermes 프로파일) 안에만 있고, **이 저장소에는 없다.**
- Discord 봇 토큰, Notion 통합 토큰, Figma PAT, 모델 인증 등 →
  `~/.hermes/profiles/qa-bot/.env`, `~/.hermes/config.yaml`(`env:` 필드), `~/.hermes/auth.json`, `~/.hermes/mcp-tokens/`
- 저장소의 `.env`는 **gitignore**되고 식별자만 담는다(값 없음). `.env.example`는 빈 템플릿.

## AI 에이전트 작업 규칙 (필수)
**에이전트는 토큰 값을 절대 확인·출력·전송하지 않는다.** 이 저장소에서 작업하는 모든 에이전트는 다음을 지킨다:

1. **시크릿 파일을 읽지 않는다.** `~/.hermes/**/.env`, `config.yaml`의 `env:`/`token`/`auth` 필드, `auth.json`, `mcp-tokens/` 를 `cat`/`grep`/`Read`로 **값이 보이게** 열지 않는다.
2. **토큰을 명령 인자·출력에 노출하지 않는다.** `grep TOKEN ...`, `echo $X_TOKEN`, `--figma-api-key figd_xxx` 처럼 값이 찍히는 명령 금지.
   - 존재 여부만 확인할 땐 **마스킹**한다: 예) `grep -E '^X_TOKEN=' file | sed 's/=.*/=✓/'`
3. **이름으로만 참조한다.** "Figma 토큰", "`DISCORD_BOT_TOKEN`"처럼 키 이름만 쓰고 값은 다루지 않는다.
4. **시크릿 입력은 사용자가 직접 한다.** 토큰 설정이 필요하면, 에이전트가 값을 받지 말고 **사용자가 진짜 터미널에서 직접 입력**하도록 안내한다.
5. **MCP/서비스 설정 시** 토큰은 `.env` 파일(`--env <file>`)로 주입하고, CLI 인자로 평문 노출하지 않는다.

> 요약: **에이전트는 "토큰이 있다/없다"까지만 알면 되고, 그 값은 절대 보지 않는다.**

## 이미 적용된 보호
- `.gitignore`로 `.env`·`*.bak`·`.omc/`·`knowledge/`·`eval/questions.md` 제외 → 시크릿·PRD 비공개
- Hermes 게이트웨이 **secret redaction** 활성 (출력·로그·응답에서 시크릿 자동 마스킹)
- 봇 툴 제한: `file/terminal/web` 비활성 → 봇이 로컬 파일시스템을 못 뒤짐

## 최소 권한
- **Notion**: 필요한 PRD 페이지/DB만 통합에 공유 (워크스페이스 전체 X)
- **Figma**: 읽기(`File content`)만, PAT에 만료일 설정
- **Discord**: 최소 채널·권한, allowlist로 사용자 제한

## 토큰 회전
- 토큰이 한 번이라도 노출되면(로그/화면/커밋 등) **즉시 재발급**한다.
- 정기 회전 권장: Figma PAT·Notion 통합 토큰 분기별.

##  알려진 노출 (회전 필요)
- 설정 과정에서 **Figma PAT가 출력/CLI 인자에 노출**된 적 있음 → **재발급 권장.**
