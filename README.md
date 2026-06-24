# 카톡 단톡방 자율 응답 봇 (테스트용)

[kakaocli](https://github.com/silver-flight-group/kakaocli)로 단톡방을 폴링하면서,
톡방별로 학습한 **내 말투**(`STYLE.md`)로 적절한 타이밍에만 자율 응답하는 봇.
응답 여부 판단과 초안 생성은 Anthropic API(Claude)로 처리한다.

> ⚠️ 기술 검증/실험용. 실제 사람들이 있는 단톡방에 내 이름으로 메시지가 나가므로,
> 처음엔 `DRY_RUN=true` 또는 `--use-self`(자기채팅으로만 전송)로 충분히 검증하고 쓰세요.

## 요구사항
- macOS + [kakaocli](https://github.com/silver-flight-group/kakaocli) (Full Disk Access + Accessibility 권한)
- Python 3.9+, `pip install anthropic`
- Anthropic API 키 (Claude Pro 구독과 별개. https://console.anthropic.com)

## 설치
```bash
pip install anthropic
cp .env.example .env      # 그리고 .env 에 ANTHROPIC_API_KEY 등 입력
```

## 폴더 구조
```
kakao_bot.py          봇 본체
update_style.py       말투(STYLE.md) 갱신 스크립트
.env                  전역 설정(키 등) — git 제외
rooms/
  <톡방이름>/
    STYLE.md          이 톡방용 말투 프로파일
    examples.txt      이 톡방용 대화 예시
    state.json        처리 상태(중복 방지·발화 이력) — git 제외
    bot_log.jsonl     로그 — git 제외
    config.env        이 톡방 전용 설정(선택) — git 제외
    STOP              있으면 이 톡방만 정지
```
`rooms/<톡방>/` 폴더는 없으면 자동 생성된다. (개인정보라 git에는 안 올라감)

## 사용
```bash
# 한 사이클(테스트)
python3 kakao_bot.py --target "롤로롤리린이"

# 지속 가동
python3 kakao_bot.py --target "롤로롤리린이" --loop

# 실제 전송 + 설정 조정
python3 kakao_bot.py --target "롤로롤리린이" --no-dry-run --loop \
    --context-limit 50 --poll-seconds 8 --delay-min 3 --delay-max 9
```
정지: `touch STOP`(전체) 또는 `touch "rooms/<톡방>/STOP"`(해당 톡방). 재개 시 파일 삭제.

## 설정 (우선순위: 명령줄 > 환경변수/.env > 톡방 config.env > 기본값)

| 키 / 인수 | 기본값 | 설명 |
|---|---|---|
| `TARGET` / `--target` | 롤로롤리린이 | 톡방 이름(부분일치) 또는 chat-id |
| `DRY_RUN` / `--dry-run`,`--no-dry-run` | true | 초안만 vs 실제 전송 |
| `USE_SELF` / `--use-self` | false | 읽기는 TARGET, 전송은 자기채팅으로 |
| `MODEL` / `--model` | claude-opus-4-8 | 사용 모델 |
| `CONTEXT_LIMIT` / `--context-limit` | 40 | LLM에 넘기는 최근 맥락 메시지 수 |
| `POLL_SECONDS` / `--poll-seconds` | 10 | 루프 폴링 주기(초) |
| `DELAY_MIN` / `--delay-min` | 3 | 응답 전 랜덤 대기 하한(초) |
| `DELAY_MAX` / `--delay-max` | 9 | 상한 |
| `MIN_GAP` / `--min-gap` | 5 | 내 마지막 발화 후 최소 간격(초) |
| `MAX_PER_HOUR` / `--max-per-hour` | 0 | 시간당 최대 발화(0=무제한) |
| `FETCH_LIMIT` / `--fetch-limit` | 80 | 한 번에 읽는 메시지 수 |

## 말투 갱신 (`update_style.py`)
봇이 보낸 메시지는 빼고, **내가 직접 친 메시지만**으로 그 톡방의 `STYLE.md`·`examples.txt`를 다시 생성한다.
(봇 발화는 `state.json`의 `sent_ids`로 구분.) 자주 할 필요는 없고 가끔(예: 주 1회) 돌리면 된다.
```bash
python3 update_style.py --target "롤로롤리린이"
python3 update_style.py --target "롤로롤리린이" --my-messages 200 --pairs 30
```
처음 쓰는 톡방은 이 스크립트로 `STYLE.md`를 먼저 만들어 두면 된다.

## 안전장치
- 봇이 방금 보낸 메시지에는 절대 반응하지 않음(루프 방지, `sent_ids`)
- 같은 메시지에 두 번 응답 금지(`responded_ids`)
- 내 마지막 발화 후 최소 간격 / 시간당 상한
- 응답 전 랜덤 대기(사람처럼)
- 전송 전 `[DRAFT]`, 전송 후 `[SENT]` 로그(JSON Lines) → 사후 검증
- `STOP` 파일 킬 스위치 / `DRY_RUN` 모드
