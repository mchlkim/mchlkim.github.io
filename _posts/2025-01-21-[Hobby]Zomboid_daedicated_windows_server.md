---
title: Zomboid Dedicated Windows 서버 구축
date: 2025-01-21 12:00:00 +0900
categories: [Hobby, Game, Zomboid]
tags: [Zomboid, Dedicated Server, Steam] # TAG names should always be lowercase
---

## 개요

좀보이드를 멀티플레이 서버를 인게임에서 구성하여 사용할 수 있지만 여유있는 컴퓨터 리소스가 없거나 반드시 게임을 구동 중인 상태에서만 접속을 해야하는 단점이 있다.

이러한 문제를 해결하기 위해서 별도 서버를 구축하여 접속이 가능하다.

### SteamCMD 다운로드

SteamCMD는 Steam 서버를 관리하는 명령줄 도구이다.

[SteamCMD 다운로드](https://steamcdn-a.akamaihd.net/client/installer/steamcmd.zip)[^1]

## 좀보이드 서버 요구 사항

##### 포트

| 프로토콜 | 포트  |
| -------- | ----- |
| UDP      | 16261 |
| UDP      | 16262 |

### 좀보이드 서버 파일 다운로드

C드라이브에 새 폴더 생성 후 이름 `SteamCMD`로 수정

SteamCMD 폴더 내에 `install.bat` 파일 생성

텍스트 파일 생성 후 확장자를 `.bat`로 변경하면 된다.

`우클릭 - 편집`을 눌러 파일 내용을 다음과 같이 작성한다.

```install.bat
steamcmd.exe +login anonymous +app_update 380870 +quit
```

install.bat을 클릭하여 실행하면 좀보이드 서버 파일이 다운로드 된다.
위의 파라미터에서 380870은 Steam App ID이다.

작업이 완료되면 창이 자동으로 닫히고 `C:\Steam\steamapps\common\Project Zomboid Dedicated Server` 폴더에 좀보이드 서버 파일이 다운로드 된다.

### 좀보이드 서버 구동

위의 경로에서 `StartServer64.bat` 파일을 `우클릭 - 편집`을 누른 다음 아래내용 중 -Xms, -Xmx 파라미터를 서버의 메모리 값의 70% 정도로 설정한다.

메모리가 16GB일 때 12GB 정도로 설정하면 된다.

메모리가 부족한 경우 Out of Memory 오류가 발생할 수 있으므로 추가적으로 메모리를 할당하거나 메모리 사용량을 줄여야 한다.

```bat
...
-Xms12g -Xmx12g
...
```

{: file='install.bat'}

`StartServer64.bat`을 클릭하여 실행하면 좀보이드 서버가 구동된다.

최초 구동 시 패스워드를 입력하라고 나오는데 자신이 사용할 패스워드를 입력하면 된다.

아래 이미지처럼 로그가 나오면 문제없이 실행 중이다.
![image](/assets/img/posts/2025-01-21-[Hobby]Zomboid_daedicated_windows_server/zomboid3.png)

다만, 위와 같이 설정한 경우 바로 서버에 접속 할 수 없기 때문에 서버 설정이 필요하다.

### 좀보이드 서버 설정

서버가 정상적으로 구동되었다면 사용자 경로에 `Zomboid`라는 폴더가 생성된다.

파일탐색기 주소창에 `%UserProfile%\Zomboid`로 이동

![image](/assets/img/posts/2025-01-21-[Hobby]Zomboid_daedicated_windows_server/zomboid1.png)

자세한 서버 설정을 위해 `Server` 폴더로 이동
![image](/assets/img/posts/2025-01-21-[Hobby]Zomboid_daedicated_windows_server/zomboid2.png)

각 파일의 역할은 다음과 같다.

| 파일명                      | 역할                    |
| --------------------------- | ----------------------- |
| server.ini                  | 서버 기본 설정          |
| servertest_SandboxVars.lua  | 게임 옵션 설정          |
| servertest_spawnpoints.lua  | 플레이어 스폰 위치 설정 |
| servertest_spawnregions.lua | 플레이어 스폰 지역 설정 |

기본적인 서버 설정을 위해 `server.ini` 파일을 수정해준다.
주석이 잘 작성되어 있기 때문에 수정하는데 어려움은 없다.

필수적으로 `server_browser_announced_ip`는 수정이 필요하다.

브라우저에서 [공인 아이피 주소 확인](https://www.myip.com/)[^2]로 접속하여 내 아이피 주소를 확인한다.

```ini
# 위에서 확인한 공인 아이피 주소를 입력
server_browser_announced_ip=<public_ip>
```

{: file='server.ini'}

위와 같이 설정하게 된다면 게임 내에서 접속이 가능하다.

## 서버 접속

프로젝트 좀보이드 실행
여러명이서 하기로 접속
![image](/assets/img/posts/2025-01-21-[Hobby]Zomboid_daedicated_windows_server/zomboid4.png)

우측 정보에서 아래 표와 같이 값을 입력 후 추가해준다.

| 항목      | 값                                 |
| --------- | ---------------------------------- |
| 서버이름  | 위에서 확인한 공인 아이피 주소     |
| IP        | 서버 공인 아이피 주소              |
| 포트      | 수정한 경우에만 변경 (기본: 16261) |
| 서버 암호 | 미설정 시 입력 필요없음            |
| 서버 설명 | 생략 가능                          |
| 유저이름  | 서버에서 사용할 닉네임             |
| 패스워드  | 서버에서 사용할 패스워드           |

유저이름은 스팀 계정과 무관하게 서버에서 사용되는 닉네임이다.

위와 같이 설정하게 된다면 문제없이 접속이 가능하다.

## Ref.

[^1]: [SteamCMD 다운로드](https://steamcdn-a.akamaihd.net/client/installer/steamcmd.zip)
[^2]: [My Ip.com](https://www.myip.com/)
[^3]: [Zomboid Dedicated server](https://pzwiki.net/wiki/Dedicated_server)
