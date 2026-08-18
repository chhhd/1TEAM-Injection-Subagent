# target-info.md — 허용 대상 범위

**단일 진실 소스는 [`1TEAM-MEMORY`의 target-info.md](https://github.com/chhhd/1TEAM-MEMORY/blob/main/target-info.md)다.**
실제 대회/훈련 대상은 이 레포가 아니라 그쪽에서 관리하며, 팀 전체가 같은
대상을 보고 있어야 Recon → Injection으로 이어지는 핸드오프가 성립한다.
이 파일은 그 사실을 명시하고, 이 레포 자체 검증용 로컬 대상만 별도로 적는다.

## 이 저장소의 로컬 검증용 대상 (실제 게임 대상 아님)

| 대상 | 용도 | 비고 |
| --- | --- | --- |
| `http://127.0.0.1:5000` | `1TEAM-Main-Orchestration-Project-Infrastructure`의 `vulnapp/app.py` — injection-agent 실전 검증용 | `/search`(SQLi), `/lookup`(대조군) |

## 원칙

- loopback(`127.0.0.1`/`localhost`) 또는 `1TEAM-MEMORY/target-info.md`에
  명시적으로 등록된 대상만 테스트한다.
- 실제 Phase 1~4 게임 진행 시에는 이 표가 아니라 `1TEAM-MEMORY/target-info.md`의
  "인가된 대상" 표를 기준으로, recon-agent가 넘긴 엔드포인트에 대해서만 테스트한다.
