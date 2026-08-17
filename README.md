# 1TEAM-Injection-Subagent

팀원3(임희영) — Injection Subagent 담당 산출물.

## 이 저장소에 있는 것

| 산출물 | 위치 | 내용 |
|---|---|---|
| Subagent 정의 | `.claude/agents/injection-agent.md` | 역할(SQLi/Command Injection/SSTI, 인가 이슈는 스코프 밖), tool 범위 제한, 컨텍스트 격리 설계, 출력 계약 |
| 진단 절차 skill | `.claude/skills/injection-diagnostics/SKILL.md` | Check 1~5를 "You are an expert" 식이 아니라 클래스별 구체적 절차(구문 깨기 → 회복으로 인과관계 증명 → 무엇을 자제할지)로 구조화 |
| 컨텍스트 격리 실측 | `docs/context-isolation-test.md` | 4038바이트 응답을 실제로 발생시켜, subagent가 이를 내부에서 전부 읽고 메인 세션엔 약 1,550자 요약만 반환함을 바이트 단위로 검증 |

## Tool 범위 제한

```yaml
tools: Read, Grep, Glob, Bash, WebFetch
```

- **Execution 도구(`Bash`)**: payload 전송 자동화(baseline→attack→control curl 시퀀스)
- **Web 도구(`WebFetch`)**: 응답 비교가 크롤링을 필요로 할 때
- **`Read`/`Grep`/`Glob`**: 소스가 있을 때 파라미터가 실제로 인터프리터에 닿는지 사전 확인
- 인가/객체참조 관련 tool 사용(access-control-agent 영역)은 의도적으로 배제 — 스코프를 넘는 취약점을 "겸사겸사" 보고하지 않도록 agent 정의에 명시

## 진단 절차 구조화 방식

`SKILL.md`는 세 클래스(SQLi/Command Injection/SSTI) 각각에 대해:
1. 구문을 깨는 구체적 페이로드
2. **회복(recovery)으로 인과관계를 증명**하는 대조 페이로드 — 단순 500 응답은 증거가 아니라는 원칙
3. 안전상 자제할 것(추출/파괴 페이로드)과 그것이 confidence를 낮추지 않는 이유

를 각 클래스마다 구체적으로 적어뒀다 — "SQL injection 전문가처럼 행동하라" 같은 역할극 프롬프트가 아니라, 실제로 무엇을 보내고 무엇을 비교할지를 절차로 명시했다.

## 컨텍스트 격리 설계와 실측

`injection-agent.md`의 "Context isolation" 절이 명시하는 원칙:
- 큰 응답/로그는 subagent 자신의 컨텍스트에서 전체 크기로 처리한다 (판단을 위해 스스로 잘라 읽지 않는다)
- 메인 세션에는 계약이 요구하는 만큼만 돌려준다 (`MAX_EXCERPT` 2048자 관례를 기준으로)
- 프로브 시퀀스는 전사(transcribe)하지 않고 요약한다

`docs/context-isolation-test.md`는 이걸 실제로 4KB대 응답을 만들어 검증한
로그다 — subagent가 4038바이트 응답을 실제로 읽었지만 메인 세션에 돌아온
최종 리포트는 약 1,550자였고, 4000자짜리 페이로드 원문은 어디에도 없었다.

## 다른 저장소와의 관계

통합 저장소 [`SECURITY-1TEAM-Orchestrator-chain`](https://github.com/chhhd/SECURITY-1TEAM-Orchestrator-chain)에
같은 파일이 `recon-agent`/`access-control-agent`와 나란히 들어 있다. 이
저장소는 그중 injection 부분만 떼어낸 슬라이스다.

## 알아둘 것

- 이 세션 환경에서는 `.claude/agents/*.md`가 Agent tool의 invokable
  subagent 목록에 자동 등록되지 않아, 위 실측 테스트는 `general-purpose`
  에이전트에 이 정의 전체를 주입해서 진행했다. 파일 자체는 표준 Claude Code
  세션에서 `/agents`로 정상 등록되는 유효한 정의다.
