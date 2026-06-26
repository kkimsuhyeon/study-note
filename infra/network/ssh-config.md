# SSH config (~/.ssh/config) - 긴 ssh 명령을 별칭 한 줄로

> 한 줄 요약: `~/.ssh/config`에 Host 별칭을 등록하면 `ssh -i 키 -L ... user@IP` 대신 `ssh 별칭`으로 끝난다. **함정은 `LocalForward`를 박아두면 그 별칭으로 접속하는 모든 경우에 터널이 따라온다는 것** — 그래서 용도별로 별칭을 나누는 게 좋다.

## 언제 쓰나

- 같은 서버에 자주 접속하는데 매번 키 경로·IP·유저명·`-L` 옵션을 치기 귀찮을 때
- 포트 포워딩 설정을 명령어 대신 파일로 고정해두고 싶을 때
- 여러 서버를 별칭으로 관리하고 싶을 때

## 사용 예시 (문법)

`~/.ssh/config` (Windows는 `C:\Users\<유저>\.ssh\config`):

```
Host mac_mini
    HostName 39.115.161.51
    User test
    IdentityFile ~/.ssh/mac_mini
```

이러면 아래 둘이 동일:

```bash
ssh -i ~/.ssh/mac_mini test@39.115.161.51   # 길게
ssh mac_mini                                  # config 덕분에 짧게
```

내용 확인:

```bash
cat ~/.ssh/config           # macOS/Linux
type .ssh\config            # Windows PowerShell
```

## 주요 키워드

| 키워드 | 의미 |
| --- | --- |
| `Host` | 별칭 (이걸로 `ssh 별칭` 호출). 공백으로 여러 개 묶기 가능 |
| `HostName` | 실제 IP/도메인 |
| `User` | 로그인 유저명 |
| `IdentityFile` | 개인키 경로 (`-i`에 해당) |
| `LocalForward` | `-L`에 해당. `LocalForward 10531 localhost:10531` |
| `ServerAliveInterval` | N초마다 신호 보내 연결 유지 (터널 끊김 방지) |

## 함정 / 메커니즘 ⚠️

- **`LocalForward`를 넣으면 그 별칭의 *모든* 접속에 터널이 붙는다.** 그냥 파일 좀 보려고 `ssh mac_mini` 했을 뿐인데 10531 터널까지 열린다. 이미 다른 세션이 그 포트를 쓰고 있으면 `bind: Address already in use` 경고가 뜨고 터널이 안 열린다.
- **그래서 용도별로 별칭을 분리하는 게 깔끔하다.** 작업용/터널용을 나누면 안 엉킨다:

```
Host mac_mini
    HostName 39.115.161.51
    User test
    IdentityFile ~/.ssh/mac_mini

Host mac_mini-tunnel
    HostName 39.115.161.51
    User test
    IdentityFile ~/.ssh/mac_mini
    LocalForward 10531 localhost:10531
```

→ 평소엔 `ssh mac_mini` (터널 X), 도구 쓸 땐 `ssh -fN mac_mini-tunnel` (터널 O).

- **권한 주의(특히 키 파일).** 키가 너무 개방돼 있으면 `Permissions ... too open`으로 거부. macOS/Linux는 `chmod 600 ~/.ssh/mac_mini`.
- **이미 config에 `LocalForward`가 있으면** `ssh 별칭`만 해도 터널이 자동으로 뜬다 → `-L`을 또 칠 필요 없음. (접속 전 `cat ~/.ssh/config`로 확인하는 습관)

## 💡 판단 기준 (관점)

**접속 빈도와 용도로 `LocalForward`를 박을지 정한다.**
- 그 서버를 **거의 도구 연동(터널) 용도로만** 접속 → 그냥 메인 Host에 `LocalForward` 박아 `ssh 별칭` 한 줄로.
- 그 서버에 **일반 작업으로도 자주** 들어감 → 메인은 깨끗이 두고 `-tunnel` 별칭을 따로. 포트 충돌·불필요한 터널을 피한다.
- 터널을 백그라운드로 띄울 거면 `-fN` + (불안정하면) `ServerAliveInterval 60` 또는 `autossh`.

> 실제 케이스: `ssh mac_mini`가 키·IP 없이 그냥 됐다 = 이미 config에 별칭이 등록돼 있던 것. 다만 `LocalForward`는 없어서 로그인만 되고 터널은 안 열렸다. 그래서 `-tunnel` 별칭을 추가하거나 `ssh -L ...`을 얹는 쪽으로 정리.

## 참고

- 선행: [ssh-port-forwarding.md](./ssh-port-forwarding.md) — `-L`의 정체
- 학습 날짜: 2026-06-26
- 계기: `ssh mac_mini`가 짧게 되는 걸 보고 config 별칭의 존재를 인지, `LocalForward`를 박을지 말지 트레이드오프 정리.
