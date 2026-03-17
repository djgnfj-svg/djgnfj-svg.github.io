---
title: Dropbox + systemd로 자동 배포 만들기
date: 2026-03-17 00:00:00 +0900
categories: [DevOps]
tags: [CI/CD, Dropbox, systemd, Cloudflare Tunnel]
---

# 배경

로봇 관제 시스템을 웹으로 전환하는 프로젝트를 하고 있었다. 개발은 재택으로 하는데, 서버는 연구실 PC에 보안 정책 때문에 원격 접속이 안 된다는 것이었다.

배포를 하려면 직접 연구실에 가서 배포 명령어를 쳐야 했다. 재택근무인데 배포할 때마다 출근해야 하는 상황이었다.

프로젝트 자체는 개인적으로 Git으로 관리하고 있었지만, CI/CD 파이프라인에 Git을 쓸 수는 없었다. 팀장님이 최대한 가볍게 가고 싶어하셨다(같은 이유로 DB도 못쓰고 csv랑 json만 사용했다..). GitHub 계정 만들고 레포 세팅하고 webhook 연결하고 하는 과정 없이, PC만 있으면 바로 배포할 수 있는 환경을 원하셨다.

도메인도 없었다. 그래서 도메인 없이 PC만으로 외부 접속이 가능한 Cloudflare Tunnel을 선택했다.

정리하면 이런 제약 조건이었다:
- 서버 PC에 원격 접속 불가
- CI/CD에 Git 기반 도구 사용 불가 (GitHub Actions, webhook 등)
- 도메인 없음
- 가능한 한 가볍게


# 구조 설계

## Dropbox를 Git 대신

코드 전달은 Dropbox로 해결했다. 개발 PC와 서버 PC 모두 Dropbox를 설치하고 같은 폴더를 동기화했다. 로컬에서 코드를 수정하면 Dropbox가 알아서 서버 PC로 동기화해준다.

여기서 테스트 서버와 본 서버를 분리했다.

| 구분 | 포트 | 배포 트리거 |
|------|------|------------|
| 테스트 서버 | 81 | Dropbox에 코드 올리면 자동 반영 |
| 본 서버 | 80 | config 폴더에 특정 파일을 touch하여 생성하면 반영 |

테스트 서버는 코드가 동기화되면 바로 반영된다.

본 서버는 의도적으로 한 단계를 더 뒀다. 테스트에서 확인이 끝나면 config 폴더에 특정 파일을 `touch`로 생성하면 그때 본 서버가 업데이트된다. 연구중에 본 서버가 바뀌는 걸 막기 위한 안전장치였다.

여기에 하나 더 안전장치를 넣었다. 본 서버 배포가 트리거되면 바로 실행하는 게 아니라, `/api/status`에서 현재 연결된 로봇이 있는지 확인하고 nginx의 active connections도 체크한다. **아무도 사용하고 있지 않을 때만** 배포가 진행된다. 로봇 관제 시스템이라 누군가 로봇을 제어하는 도중에 서버가 내려가면 위험하기 때문이다.

그리고 가능하면 새벽에 업데이트 했다.

## Cloudflare Tunnel로 외부 접속

도메인도 없고 포트포워딩으로 포트를 열기도 싫었다. Cloudflare Tunnel을 쓰면 PC에 에이전트만 설치하면 외부에서 접속할 수 있고, 서버 포트가 외부에 직접 노출되지 않는다.

서브도메인을 다르게 설정해서 각각 연결했다.

```
test.example.com  →  localhost:81  (테스트 서버)
app.example.com   →  localhost:80  (본 서버)
```

외부에서 보면 별개의 사이트처럼 보이지만, 실제로는 같은 PC에서 포트만 다르게 돌아가고 있는 구조다.


## systemd + 폴링으로 자동 배포

Dropbox가 코드를 동기화해주는 건 좋은데, 코드가 바뀌었다고 서비스가 알아서 재시작되진 않는다. 이 부분을 systemd로 해결했다.

처음에는 systemd의 Path Unit(`PathModified`)을 고려했지만, Dropbox 동기화는 파일이 하나씩 내려오기 때문에 변경이 여러 번 트리거될 수 있었다. 대신 **10초 간격 폴링 + md5sum 해시 비교** 방식을 선택했다.

```bash
# 감지 루프 (간략화)
while true; do
    NEW_HASH=$(find /path/to/dropbox/frontend /path/to/dropbox/backend -type f | xargs md5sum | md5sum)
    if [ "$NEW_HASH" != "$OLD_HASH" ]; then
        sleep 10  # debounce — Dropbox 동기화가 끝날 때까지 대기
        /path/to/auto_deploy.sh
        OLD_HASH=$NEW_HASH
    fi
    sleep 10
done
```

이 스크립트를 systemd service로 등록해서 서버가 켜지면 자동으로 돌아가게 했다.

```ini
# dropbox-watch.service
[Unit]
Description=Watch Dropbox folder and auto deploy

[Service]
Type=simple
ExecStart=/path/to/watch_and_deploy.sh
Restart=always

[Install]
WantedBy=multi-user.target
```

변경이 감지되면 10초 대기(debounce)를 거친 뒤 `auto_deploy.sh`가 실행된다. 10초를 두는 이유는 Dropbox가 파일을 하나씩 동기화하기 때문에, 절반만 내려온 상태에서 배포가 트리거되는 걸 막기 위해서다. 완벽하진 않지만 실무에서 문제가 된 적은 없었다.

config 파일은 Docker 볼륨으로 마운트되어 있어서 따로 감지할 필요가 없었다. 볼륨 마운트라 컨테이너 재시작 없이도 즉시 반영되기 때문에 감시 대상에서 제외했다.

배포는 `docker compose down` → `docker compose up -d --build`로 전체 재배포했다.


# 전체 흐름

```
[개발 PC]
    │
    │  코드 수정
    ▼
[Dropbox 동기화]
    │
    ▼
[서버 PC - Dropbox 폴더]
    │
    │  systemd service가 10초 폴링 + md5sum 비교
    ▼
[변경 감지 → 10초 debounce]
    │
    ▼
[auto_deploy.sh 실행]
    │
    ├─ 테스트 서버 → 즉시 docker compose down/up
    └─ 본 서버 → API/nginx 접속자 확인 후 배포
    │
    ▼
[Cloudflare Tunnel]
    │
    ├─ test.app.xxx.com → localhost:81
    └─ app.xxx.com  → localhost:80
```


# 한계점

솔직히 말하면 이 구조에는 한계가 많다.

**배포 중 서비스가 중단된다.** 컨테이너를 내리고 다시 올리는 방식이라 그 사이에 접속이 끊긴다. 사용자가 소수이고 배포 시간이 짧아서 괜찮았지만, 새벽에 아무도 안사용해야 적용되게 해서 문제는 없었다.

다만 어짜피 docker를 사용하기로 했으면 블루/그린은 가능하지 않을까.. 라는 생각이 든다. 결국 nginx가 앞에 있으니까..

**자동 롤백이 없다.** Git 자체는 쓰고 있었기 때문에 코드 추적이나 `git revert`로 되돌리는 건 가능했다. 하지만 배포 파이프라인에 Git이 연결되어 있지 않으니, 롤백도 결국 수동이다. 코드를 되돌리고 Dropbox에 다시 올리고 배포가 트리거되기를 기다려야 한다.

서버 PC가 과부화 걸려서 dropbox는 작동을 안해서 문제가 있었다. 이건 report 되었고 아직 수정되지 않았다. 사실 log는 구지 드롭박스로 공유 안되게 하면 그만이다.


# 배운 것
AI가 있으니 모든 조건들을 입력하면서 최고의 경우의 수를 찾기가 쉬워졌다. 오히려 불안하고 무섭네...
