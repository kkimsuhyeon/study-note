# JetBrains 원격 개발

> 원격 컴퓨터에 코드와 IDE 처리 엔진을 두고, 접속 컴퓨터의 JetBrains Client에서 편집하는 방식이다.

## 언제 쓰나

집 데스크톱의 개발 환경을 유지하면서 외부 노트북에서 편집·실행·디버깅할 때 쓴다. 단순히 저장소에 올라간 코드를 읽으려는 목적이라면 저장소 웹 화면이 더 간단하다.

## 사용 예시: 외부 노트북에서 집 맥 접속

1. 집 맥의 `시스템 설정 → 일반 → 공유 → 원격 로그인`에서 SSH 접속을 설정한다. 허용 사용자는 필요한 계정으로 제한한다.
2. 외부 노트북에서 집 맥으로 접근할 네트워크 경로를 준비한다. 예를 들어 두 기기를 Tailscale 사설망에 연결하고 macOS 기본 SSH를 이용한다.
3. 외부 노트북에 JetBrains Toolbox App을 설치한다.
4. Toolbox의 환경 선택 메뉴에서 `SSH → New SSH Connection`으로 접속 주소와 인증을 설정한다.
5. 원격 컴퓨터의 IDE와 프로젝트를 선택한다. 예시 프로젝트 경로는 `/Users/developer/projects/sample-app`이다.
6. 연결된 편집 화면에서 파일을 수정하면 원격 컴퓨터의 프로젝트가 변경된다. 빌드·실행도 원격 컴퓨터에서 처리된다.

SSH는 암호화된 원격 접속 프로토콜이다. 네트워크 주소에 도달하는 문제와 SSH 인증에 성공하는 문제는 별개다. [SSH config](ssh-config.md), [포트와 listen](ports-and-listen.md), [SSH 포트 포워딩](ssh-port-forwarding.md)을 함께 참고한다.

## 방식 비교

| 방식 | 동작 | 적합한 경우 | 비용·제약 |
|---|---|---|---|
| JetBrains 원격 개발 | IDE 처리 엔진은 원격, 편집 UI는 접속 기기 | 원격 개발 환경을 그대로 사용 | SSH·네트워크 준비와 실행 중인 호스트 필요 |
| 원격 데스크톱 | 원격 컴퓨터의 데스크톱 화면을 조작 | 이미 열린 IntelliJ 창과 다른 앱까지 사용 | 화면 전송 품질과 입력 지연 영향 |
| 저장소 웹 열람 | 서버에 올라간 파일을 웹에서 확인 | 코드 읽기·간단한 검색 | 로컬의 미업로드 변경과 실행 환경은 확인할 수 없음 |

## ⚠️ 함정과 메커니즘

- **로컬 IntelliJ 창에 그대로 들어가는 화면 공유와는 다르다.** 원격 개발용 IDE 세션을 사용한다. 같은 작업 폴더를 로컬과 원격에서 동시에 편집하면 파일 변경이 겹칠 수 있다.
- **`The IDE is not running in remote dev mode`가 뜨면 실행 모드를 확인한다.** 실제 사례에서는 Toolbox의 `serverMode` 실행 요청이 이미 실행 중인 일반 IDE로 전달됐고, 로그에 `Reusing already opened project`와 `"unattendedMode": false`가 함께 나타났다. SSH로 요청은 도착했지만 원격 개발 세션으로 전환되지 않은 것이다. 이런 증거가 있을 때는 호스트의 일반 IntelliJ를 정상 종료하고, 접속 기기의 Toolbox에서 프로젝트를 다시 연다. 창만 닫으면 macOS 앱 프로세스가 남을 수 있으므로 앱 종료를 구분한다. 이 조치는 해당 사례의 재시도 준비이며, 모든 동일 문구의 원인이 같다는 뜻은 아니다. 재접속 성공 또는 원격 모드 상태를 확인해야 해결로 판단한다.
- **집 맥이 잠들거나 꺼지면 접속을 유지할 수 없다.** 전원과 네트워크를 유지하고 실제 외부 네트워크에서 연결을 확인해야 한다.
- **SSH를 켠 것만으로 외부에서 접속되지는 않는다.** 집 공유기의 사설 주소는 외부에서 바로 접근할 수 없다. VPN 등 접근 경로가 추가로 필요하다.
- **Tailscale 사설망에서 일반 SSH를 쓰는 것과 Tailscale SSH는 별개다.** macOS 앱 종류에 따라 Tailscale SSH 서버 지원이 다르다. 사설망 연결 + macOS 원격 로그인 조합에서는 기본 SSH 서버가 인증을 담당한다.
- **휴대폰 웹에서 IntelliJ를 여는 기능으로 생각하면 안 된다.** 여기서 설명한 Toolbox/JetBrains Client 방식은 컴퓨터용 클라이언트를 전제로 한다.
- **Code With Me 종료 일정은 버전 종속 정보다.** 2026-09-19 확인한 공식 발표 기준으로 2026.1이 마지막 공식 지원 IDE 버전이며, 공개 중계 서비스 종료는 2027년 1분기 예정이다. 원격 개발 기능은 종료 대상이 아니다.

## 참고

- [JetBrains 원격 개발 개요](https://www.jetbrains.com/help/idea/remote-development-overview.html)
- [Toolbox 원격 프로젝트 연결 순서](https://www.jetbrains.com/help/toolbox-app/gettings-started-with-ssh.html)
- [Toolbox 설치·SSH 요구사항](https://www.jetbrains.com/help/toolbox-app/installation.html)
- [Apple: 맥 원격 로그인 설정](https://support.apple.com/guide/mac-help/allow-a-remote-computer-to-access-your-mac-mchlp1066/mac)
- [Tailscale를 통한 SSH](https://tailscale.com/docs/reference/ssh-over-tailscale)
- [Tailscale SSH 지원 범위](https://tailscale.com/docs/features/tailscale-ssh)
- [Code With Me 종료 안내](https://blog.jetbrains.com/platform/2026/03/sunsetting-code-with-me/)

학습 날짜: 2026-09-19. 계기: Code With Me 메뉴가 없는 이유와 외부에서 개인 개발 컴퓨터에 접속하는 방식 비교.

💡 집 컴퓨터의 미커밋 파일과 실행 환경까지 써야 하면 원격 개발을, 이미 열린 앱 화면 전체가 필요하면 원격 데스크톱을, 업로드된 코드만 읽으면 저장소 웹 열람을 고른다.
