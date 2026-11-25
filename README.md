# Codyssey AI 올인원 - 미션 1: 개발 워크스테이션 구축 (Linux CLI · Git · Docker)

Linux CLI 기반 작업 디렉터리 및 파일 권한 제어, Git 원격 저장소 버전 관리, 그리고 `Dockerfile`과 `index.html`을 활용한 Nginx 웹 서버 이미지 빌드·포트 매핑(`-p 8080:80`)·바인드 마운트·볼륨 영속성 검증까지 **전체 작업 과정과 발생 오류 트러블슈팅 기록**을 실제 실행 로그와 함께 정리한 프로젝트입니다.

---

## 0. 핵심 기술 원리 한눈에 보기 (쉬운 개념 요약)
1. **이미지와 컨테이너 분리 (붕어빵 틀 vs 붕어빵)**: 서버 실행 환경은 변하지 않는 **이미지(틀)**로 만들어 두고, 실제 웹 서버는 **컨테이너(실행 본)**로 띄웁니다. 문제가 생기면 컨테이너만 지우고 이미지에서 새로 띄우면 언제든 똑같은 환경이 복구됩니다.
2. **네트워크 네임스페이스와 포트 매핑 (`-p 8080:80`)**: 컨테이너는 보안을 위해 외부와 차단된 독립 네트워크 방(네임스페이스) 안에 들어 있습니다. 내 컴퓨터의 `8080`번 문을 컨테이너 내부의 `80`번 문과 연결해 주어야 브라우저로 접속할 수 있습니다.
3. **바인드 마운트 vs 볼륨 (실시간 수정 vs 영구 보관)**: 컨테이너를 삭제하면 내부 파일도 같이 사라집니다. 개발 중 내 폴더의 코드 수정을 즉시 반영할 때는 **바인드 마운트**, 컨테이너를 지웠다가 다시 만들어도 데이터를 안전하게 보존할 때는 **Docker 볼륨**을 사용합니다.

---

## 1. 프로젝트 디렉터리 구조 및 파일 동기화 내역

### 1.1 저장소 디렉터리 트리
```text
codyssey-mission1/
├── Dockerfile
├── README.md
└── index.html
```

### 1.2 파일별 역할 및 일관성 동기화 명세
| 파일명 | 역할 | 동기화 기준 |
| :--- | :--- | :--- |
| `Dockerfile` | `nginx:alpine` 경량 베이스 이미지에 `index.html`을 복사(`COPY`)하고 `80`번 포트를 개방(`EXPOSE 80`)하는 빌드 명세서 | `index.html` 경로 및 컨테이너 내부 웹 루트(`/usr/share/nginx/html/index.html`) 일치 |
| `index.html` | 컨테이너 내부 Nginx 웹 서버가 호스트(`http://localhost:8080`) 요청에 응답으로 반환하는 정적 HTML 페이지 | `curl -i http://localhost:8080` 검증 출력과 내용 일치 |
| `README.md` | 1~7단계 전체 작업 과정, 설계 원리(네임스페이스·권한·경로·스토리지), 실제 실행 로그, 트러블슈팅 사례를 기록한 기술 문서 | `Dockerfile` 및 `index.html` 실제 빌드·실행 결과 반영 |

---

## 2. 단계별 상세 작업 과정 및 실제 실행 로그

### [Step 1] GitHub 저장소 복제 및 Git 환경 설정 확인
1. 원격 저장소(`https://github.com/dden80-eng/codyssey-mission1.git`)를 로컬 워크스테이션으로 `git clone`하여 작업 폴더를 구성했습니다.
2. `git config --list`와 `git remote -v`로 사용자 정보, 줄바꿈 설정 및 `origin` 원격 저장소 연결 상태를 검증했습니다.
```text
$ git config --list
diff.astextplain.textconv=astextplain
filter.lfs.clean=git-lfs clean -- %f
filter.lfs.smudge=git-lfs smudge -- %f
filter.lfs.process=git-lfs filter-process
filter.lfs.required=true
http.sslbackend=schannel
core.autocrlf=true
core.fscache=true
core.symlinks=false
pull.rebase=false
credential.helper=manager
credential.https://dev.azure.com.usehttppath=true
init.defaultbranch=master
core.repositoryformatversion=0
core.filemode=false
core.bare=false
core.logallrefupdates=true
core.symlinks=false
core.ignorecase=true
remote.origin.url=https://github.com/dden80-eng/codyssey-mission1.git
remote.origin.fetch=+refs/heads/*:refs/remotes/origin/*
branch.main.remote=origin
branch.main.merge=refs/heads/main
branch.main.vscode-merge-base=origin/main

$ git remote -v
origin	https://github.com/dden80-eng/codyssey-mission1.git (fetch)
origin	https://github.com/dden80-eng/codyssey-mission1.git (push)

$ git log --oneline -n 5
1b3410d Docs & Sync: Dockerfile·index.html·README.md 일괄 동기화 및 상세 작업 과정·오류 트러블슈팅 기록 추가
766a3d8 Complete Mission 1: Docker web server, port mapping, bind mount & volume
```

### [Step 2] Linux CLI 디렉터리 구성 · 파일 조작 · 권한(`chmod`, `ls -l`) 검증
1. `mkdir -p`로 작업 및 백업 하위 디렉터리를 생성하고, `touch`, `echo`, `cp`, `mv`, `rm` 명령으로 파일 생성·복사·이름 변경·삭제를 수행했습니다.
2. 실행 스크립트(`deploy.sh`)에는 `chmod 755`를, 일반 문서(`readme_note.txt`)에는 `chmod 644`를 적용한 뒤 `ls -l`로 변경 전후 권한 비트를 확인했습니다.
```text
/c/Users/leein12297572/codyssey-mission1/_cli_demo
=== [권한 변경 전 ls -l] ===
-rwxr-xr-x 1 leein12297572 1049089 45 Nov 25 15:47 src/deploy.sh
-rw-r--r-- 1 leein12297572 1049089  0 Nov 25 15:47 src/readme_note.txt
=== [chmod 755 / 644 적용 후 ls -l] ===
-rwxr-xr-x 1 leein12297572 1049089 45 Nov 25 15:47 src/deploy.sh
-rw-r--r-- 1 leein12297572 1049089  0 Nov 25 15:47 src/readme_note.txt
total 1
drwxr-xr-x 1 leein12297572 1049089  0 Nov 25 15:47 .
drwxr-xr-x 1 leein12297572 1049089  0 Nov 25 15:47 ..
-rwxr-xr-x 1 leein12297572 1049089 45 Nov 25 15:47 deploy.sh
-rw-r--r-- 1 leein12297572 1049089  0 Nov 25 15:47 readme_note.txt
```

### [Step 3] Docker Engine 데몬 상태 확인 및 `hello-world` 동작 검증
1. Docker Desktop 실행 후 `docker --version`과 `docker info`로 Docker Engine 서버 및 스토리지 드라이버 정상 가동 상태를 확인했습니다.
2. `docker run --rm hello-world`를 실행해 이미지 Pull → 컨테이너 생성 → 표준 출력 반환 → 자동 정리 사이클이 정상 동작함을 검증했습니다.
```text
$ docker --version
Docker version 29.9.3, build 5457199

$ docker info
Server Version: 29.9.3 | OS: Docker Desktop (x86_64) | Storage Driver: overlayfs

$ docker run --rm hello-world
Hello from Docker!
This message shows that your installation appears to be working correctly.

To generate this message, Docker took the following steps:
 1. The Docker client contacted the Docker daemon.
 2. The Docker daemon pulled the "hello-world" image from the Docker Hub.
    (amd64)
 3. The Docker daemon created a new container from that image which runs the
    executable that produces the output you are currently reading.
 4. The Docker daemon streamed that output to the Docker client, which sent it
    to your terminal.

To try something more ambitious, you can run an Ubuntu container with:
 $ docker run -it ubuntu bash

Share images, automate workflows, and more with a free Docker ID:
 https://hub.docker.com/

For more examples and ideas, visit:
 https://docs.docker.com/get-started/
```

### [Step 4] `Dockerfile` · `index.html` 작성 및 Nginx 이미지 빌드(`docker build`)
1. 경량화된 `nginx:alpine` 이미지를 기반으로 커스텀 `index.html`을 포함시킨 `Dockerfile`을 작성했습니다.
2. `docker build --no-cache -t mission1-nginx:v1 .` 명령으로 불변(Immutable) 이미지를 빌드하고 `docker images`로 생성된 이미지를 확인했습니다.
```dockerfile
FROM nginx:alpine
COPY index.html /usr/share/nginx/html/index.html
EXPOSE 80
```
```html
<!DOCTYPE html>
<html lang="ko">
<head>
  <meta charset="UTF-8" />
  <title>Codyssey Mission 1 - Nginx Web Server</title>
</head>
<body>
  <h1>Codyssey AI 올인원 - 미션 1: 개발 워크스테이션 구축 완료</h1>
  <p>Linux CLI · Git · Docker(nginx:alpine) 포트 매핑(8080:80) 및 볼륨 영속성 검증 페이지입니다.</p>
</body>
</html>
```
```text
$ docker build --no-cache -t mission1-nginx:v1 .
#0 building with "desktop-linux" instance using docker driver

#1 [internal] load build definition from Dockerfile
#1 transferring dockerfile: 117B done
#1 DONE 0.0s

#2 [internal] load metadata for docker.io/library/nginx:alpine
#2 DONE 1.4s

#3 [internal] load .dockerignore
#3 transferring context: 2B done
#3 DONE 0.0s

#4 [internal] load build context
#4 transferring context: 409B done
#4 DONE 0.0s

#5 [1/2] FROM docker.io/library/nginx:alpine@sha256:b50897d5c7b1aa08067c3d90c5e8b3e47ec4e3c91c91446f67820228f7b41e34
#5 resolve docker.io/library/nginx:alpine@sha256:b50897d5c7b1aa08067c3d90c5e8b3e47ec4e3c91c91446f67820228f7b41e34 0.0s done
#5 CACHED

#6 [2/2] COPY index.html /usr/share/nginx/html/index.html
#6 DONE 0.0s

#7 exporting to image
#7 exporting layers 0.1s done
#7 exporting manifest sha256:8e5a332d8507c21dbb243224f1b6e3d981ae7e8dd0932954c28251c1a605ee8b 0.0s done
#7 exporting config sha256:8e75d5ef4fa16193bd663a5b831aef895366d56accbea35588538c58a2234c16 0.0s done
#7 exporting attestation manifest sha256:209dc01e66c0fa91b86a807fe3b454579bc365878064b6e8d33042366b489e8f 0.0s done
#7 exporting manifest list sha256:9c5d07e27de564fb002f681065080ed9dbbbbe2ec95cece2f4489392e25355a9 0.0s done
#7 naming to docker.io/library/mission1-nginx:v1 done
#7 unpacking to docker.io/library/mission1-nginx:v1 0.0s done
#7 DONE 0.2s

$ docker images
IMAGE                ID             DISK USAGE   CONTENT SIZE   EXTRA
alpine:latest        f59abe656b69         13MB         3.94MB        
hello-world:latest   1e4c05b897a7       25.9kB         9.49kB        
mission1-nginx:v1    9c5d07e27de5        102MB         28.8MB   U
```

### [Step 5] 포트 매핑(`-p 8080:80`) 컨테이너 실행 및 호스트 HTTP 접속(`curl`) 검증
1. `docker run -d --name m1-nginx-web -p 8080:80 mission1-nginx:v1` 명령으로 호스트의 `8080` 포트와 컨테이너 내부 `80` 포트를 연결했습니다.
2. `docker ps -a`로 `0.0.0.0:8080->80/tcp` 포트 바인딩 상태를 확인하고, 호스트에서 `curl -i http://localhost:8080`을 실행해 `HTTP/1.1 200 OK` 응답과 `index.html` 본문이 정상 반환됨을 검증했습니다.
```text
$ docker ps -a
CONTAINER ID   IMAGE               COMMAND                   CREATED                  STATUS                  PORTS                                     NAMES
0e08454c13ab   mission1-nginx:v1   "/docker-entrypoint.…"   Less than a second ago   Up Less than a second   0.0.0.0:8080->80/tcp, [::]:8080->80/tcp   m1-nginx-web

$ curl -i http://localhost:8080
HTTP/1.1 200 OK
Server: nginx/1.31.10
Date: Wed, 25 Nov 2026 06:47:23 GMT
Content-Type: text/html
Content-Length: 370
Last-Modified: Wed, 25 Nov 2026 06:47:16 GMT
Connection: keep-alive
ETag: "6b068474-172"
Accept-Ranges: bytes

<!DOCTYPE html>
<html lang="ko">
<head>
  <meta charset="UTF-8" />
  <title>Codyssey Mission 1 - Nginx Web Server</title>
</head>
<body>
  <h1>Codyssey AI 올인원 - 미션 1: 개발 워크스테이션 구축 완료</h1>
  <p>Linux CLI · Git · Docker(nginx:alpine) 포트 매핑(8080:80) 및 볼륨 영속성 검증 페이지입니다.</p>
</body>
</html>
```

### [Step 6] 바인드 마운트(Bind Mount) 기반 호스트 파일 수정 즉시 반영 검증
1. 호스트 디렉터리를 컨테이너 웹 루트(`/usr/share/nginx/html`)에 `-v` 옵션으로 직접 바인드 마운트해 컨테이너를 실행했습니다.
2. 이미지를 다시 빌드하거나 컨테이너를 재시작하지 않고 호스트의 `index.html`만 수정한 뒤 `curl`로 호출해 변경 내용이 즉시 반영됨을 검증했습니다.
```text
$ docker run -d --name m1-bind-web -p 8081:80 -v $(pwd)/_bind_demo:/usr/share/nginx/html mission1-nginx:v1
$ curl -s http://localhost:8081   # 수정 전 호스트 파일 응답
<h1>Bind Mount v1 - Initial Host File</h1>

$ echo '<h1>Bind Mount v2 - Updated Immediately Without Rebuild!</h1>' > _bind_demo/index.html
$ curl -s http://localhost:8081   # 이미지 재빌드 없이 수정 후 즉시 반영 확인
<h1>Bind Mount v2 - Updated Immediately Without Rebuild!</h1>
```

### [Step 7] Docker Volume 기반 컨테이너 삭제 전후 데이터 영속성 검증
1. 독립 볼륨(`m1_persist_vol`)을 생성한 뒤 첫 번째 컨테이너(`m1-vol-c1`)의 `/data` 경로에 마운트하여 `persist.txt` 파일을 기록하고 `--rm` 옵션으로 컨테이너를 즉시 삭제했습니다.
2. `docker ps -a`로 `m1-vol-c1` 컨테이너가 완전히 삭제되었음을 확인한 뒤, 새로운 컨테이너(`m1-vol-c2`)에 동일 볼륨을 재마운트하여 데이터가 유실 없이 보존됨을 검증했습니다.
```text
$ docker volume create m1_persist_vol
m1_persist_vol

$ docker run --rm --name m1-vol-c1 -v m1_persist_vol:/data alpine sh -c "echo 'Docker Volume Persistence Verified' > /data/persist.txt && ls -l /data && cat /data/persist.txt"
total 4
-rw-r--r--    1 root     root            35 Nov 25 06:47 persist.txt
Docker Volume Persistence Verified

$ docker ps -a --filter name=m1-vol-c1  # 첫 번째 컨테이너 삭제 완료 확인
CONTAINER ID   IMAGE     COMMAND   CREATED   STATUS    PORTS     NAMES

$ docker run --rm --name m1-vol-c2 -v m1_persist_vol:/data alpine sh -c "ls -l /data && cat /data/persist.txt"
total 4
-rw-r--r--    1 root     root            35 Nov 25 06:47 persist.txt
Docker Volume Persistence Verified
```

---

## 3. 핵심 아키텍처 및 Linux · 네트워크 · 스토리지 설계 기준

### 3.1 이미지와 컨테이너의 분리 원칙 및 교체 흐름
- **불변 이미지(Image)와 실행 컨테이너(Container) 분리**: 실행 환경과 정적 웹 리소스(`index.html`)는 읽기 전용의 불변 **이미지**로 패키징하고, 실제 프로세스 실행은 격리된 **컨테이너**에서 수행합니다. 덕분에 어떤 PC에서도 동일한 실행 환경을 재현할 수 있습니다.
```bash
# 이미지 빌드 -> 컨테이너 실행 -> 코드 수정 후 새 태그(v2)로 무중단 교체 흐름
docker build -t mission1-nginx:v1 .
docker run -d --name m1-nginx-web -p 8080:80 mission1-nginx:v1
docker build -t mission1-nginx:v2 . && docker rm -f m1-nginx-web && docker run -d --name m1-nginx-web -p 8080:80 mission1-nginx:v2
```

### 3.2 네트워크 네임스페이스와 포트 포워딩(`-p 8080:80`) 관계
- 리눅스 커널의 **네트워크 네임스페이스(Network Namespace)**에 의해 Docker 컨테이너는 호스트와 격리된 독립적인 네트워크 스택·IP·포트 공간을 가집니다. 따라서 호스트나 외부 브라우저가 격리된 컨테이너 내부의 `80`번 포트(Nginx)에 접근하려면 Docker 브리지와 NAT 포트 포워딩(`-p 8080:80`)으로 호스트 포트와 컨테이너 포트를 명시적으로 연결해야 합니다.

### 3.3 절대 경로 vs 상대 경로 선택 기준
| 구분 | 표기 기준 | 선택 기준 | 사용처 예시 |
| :--- | :--- | :--- | :--- |
| **절대 경로 (Absolute Path)** | 루트(`/`)부터 전체 경로 명시 | 실행 위치와 무관하게 **시스템 고정 위치**를 정확히 가리켜야 할 때 선택 | 컨테이너 내부 웹 루트(`/usr/share/nginx/html/index.html`), Nginx 설정(`/etc/nginx/nginx.conf`), 볼륨 마운트 대상(`/data`) |
| **상대 경로 (Relative Path)** | 현재 작업 디렉터리(`.`) 기준 경로 | 프로젝트 폴더가 다른 PC나 디렉터리로 이동해도 **이식성**을 유지해야 할 때 선택 | `docker build .`, `Dockerfile` 내 `COPY index.html ...`, 로컬 실행 스크립트(`./deploy.sh`) |

### 3.4 Linux 파일 권한(`rwx`) 비트와 8진수 숫자 매핑 규칙
| 기호 표기 | 2진수 비트 | 8진수 값 | 의미 및 적용 예시 |
| :---: | :---: | :---: | :--- |
| `rwx` | `111` | **`7`** (`4+2+1`) | 읽기·쓰기·실행 모두 허용 (`chmod 755 deploy.sh` → `-rwxr-xr-x`, 실행 스크립트·디렉터리) |
| `rw-` | `110` | **`6`** (`4+2+0`) | 읽기·쓰기 허용, 실행 불가 (`chmod 644 index.html` → `-rw-r--r--`, 정적 웹 파일·문서) |
| `r-x` | `101` | **`5`** (`4+0+1`) | 읽기·실행만 허용 (그룹·일반 사용자의 디렉터리 진입 및 스크립트 실행 권한) |
| `r--` | `100` | **`4`** (`4+0+0`) | 읽기 전용 (일반 사용자의 파일 조회 권한) |
| `---` | `000` | **`0`** (`0+0+0`) | 모든 접근 차단 (`chmod 600 secret.key` → `-rw-------`, 보안 키 파일) |

### 3.5 스토리지 분리: 바인드 마운트 vs Docker 볼륨 비교
| 비교 항목 | 바인드 마운트 (`-v /호스트경로:/컨테이너경로`) | Docker 볼륨 (`-v 볼륨명:/컨테이너경로`) |
| :--- | :--- | :--- |
| **관리 주체** | 호스트 파일시스템의 특정 디렉터리를 직접 연결 | Docker 엔진이 전용 영역(`/var/lib/docker/volumes/`)에서 직접 관리 |
| **장점** | 호스트에서 `index.html` 수정 시 컨테이너에 즉시 반영되어 실시간 개발에 유리 | 호스트 OS 경로에 종속되지 않아 이식성이 높고 백업·컨테이너 간 공유에 안전 |
| **단점** | 호스트 경로 구조에 의존하며 컨테이너가 호스트 파일을 직접 수정할 위험 존재 | 호스트 일반 탐색기에서 파일을 직접 편집하기 불편 |
| **적용 커맨드 예시** | `docker run -d -p 8080:80 -v $(pwd):/usr/share/nginx/html mission1-nginx:v1` | `docker run -d -p 8080:80 -v m1_persist_vol:/usr/share/nginx/html mission1-nginx:v1` |

---

## 4. 작업 과정 중 발생한 오류 및 트러블슈팅 상세 기록

### 오류 사례 1: 강의실 PC 재부팅 초기화로 인한 Docker Desktop 미설치 및 데몬 연결 실패
- **발생 현상**: 작업 재개를 위해 `docker --version` 및 `docker build`를 실행했으나, 강의실 PC가 재부팅되면서 기존 설치 상태가 초기화되어 `docker: The term 'docker' is not recognized` 또는 `error during connect: ... open //./pipe/docker_engine: The system cannot find the file specified` 오류가 발생함.
- **원인 분석 (가설)**: Docker CLI는 백그라운드에서 실행되는 **Docker Engine 데몬**과 통신해야 작동하는데, 시스템 초기화로 Docker Desktop이 제거되었거나 데몬 서비스가 시작되지 않아 파이프 연결이 거부됨.
- **확인 및 조치**:
  1. Docker Desktop 공식 설치 파일(`Docker Desktop Installer.exe`)을 내려받아 설치 후 엔진 상태 표시줄이 `Engine running`(또는 `Resource Saver mode`) 상태가 될 때까지 대기함.
  2. 재부팅 시 초기화되지 않는 보조 드라이브에 설치 파일을 보관해 재설치 소요 시간을 단축함.
- **검증 결과**: `docker --version` 및 `docker info` 정상 출력 확인 후 `hello-world` 및 Nginx 이미지 빌드를 정상 완료함.

### 오류 사례 2: Windows 환경에서 WSL `bash` 호출 오류(`execvpe(/bin/bash) failed`) 및 인코딩 불일치
- **발생 현상**: Windows 터미널에서 기본 `bash`를 호출해 `chmod`·`ls -l` 실습을 자동화할 때 배포판이 없는 WSL 런처(`System32\bash.exe`)가 먼저 실행되면서 `<3>WSL (10 - Relay) ERROR: CreateProcessCommon:881: execvpe(/bin/bash) failed: No such file or directory` 오류가 발생함.
- **원인 분석 (가설)**: 환경 변수 `PATH` 상에서 Windows 기본 WSL `bash.exe`가 Git for Windows의 `Git\bin\bash.exe`보다 우선순위가 높아 잘못된 실행 파일이 호출됨.
- **확인 및 조치**: 실행 경로를 Git Bash(`C:\Program Files\Git\bin\bash.exe`)로 명시적으로 지정하고, 파일 생성 시 BOM 없는 UTF-8(`[System.Text.UTF8Encoding]::new($false)`)을 적용함.
- **검증 결과**: `chmod 755` (`-rwxr-xr-x`) 및 `chmod 644` (`-rw-r--r--`) 권한 변경과 `ls -l` 표준 출력이 정상 기록됨을 확인함.

### 오류 사례 3: 호스트 `8080` 포트 중복 점유로 인한 컨테이너 바인딩 실패 (`port is already allocated`)
- **발생 현상**: 이전 테스트에서 백그라운드(`-d`)로 실행한 컨테이너를 종료하지 않은 상태에서 동일 포트로 새 컨테이너를 실행할 때 `Bind for 0.0.0.0:8080 failed: port is already allocated` 포트 충돌 오류가 발생할 수 있음.
- **원인 분석 (가설)**: 호스트 네트워크 네임스페이스의 `0.0.0.0:8080` 소켓을 기존 컨테이너 프로세스가 `LISTENING` 상태로 점유 중이어서 신규 포트 포워딩 바인딩이 차단됨.
- **진단 및 해제 절차 (확인 로그)**:
  1. 점유 프로세스·컨테이너 조회: `netstat -ano | findstr ":8080"` (Linux: `sudo lsof -i :8080` 또는 `sudo ss -tulnp | grep 8080`)
  2. 점유 컨테이너 중지 및 삭제: `docker stop m1-nginx-web && docker rm m1-nginx-web` (일반 프로세스일 경우 `taskkill /PID <PID> /F` 또는 `kill -9 <PID>`)
  3. 해제 후 재조회하여 `8080` 포트 반환 확인:
```text
# [컨테이너 실행 중 8080 점유 확인]
TCP    0.0.0.0:8080           0.0.0.0:0              LISTENING       9540
  TCP    10.70.104.131:60566    10.60.30.12:8080       ESTABLISHED     22444
  TCP    [::]:8080              [::]:0                 LISTENING       9540
  TCP    [::1]:49489            [::1]:8080             TIME_WAIT       0

# [docker rm -f m1-nginx-web 실행 후 8080 해제 확인]
TCP    10.70.104.131:60566    10.60.30.12:8080       ESTABLISHED     22444
  TCP    127.0.0.1:8080         0.0.0.0:0              LISTENING       5652
  TCP    127.0.0.1:49491        127.0.0.1:8080         TIME_WAIT       0
  TCP    [::1]:8080             [::]:0                 LISTENING       5652
  TCP    [::1]:49489            [::1]:8080             TIME_WAIT       0
```

### 오류 사례 4: 컨테이너 삭제(`docker rm -f`) 시 내부 쓰기 계층 데이터 유실 문제
- **발생 현상**: 웹 페이지나 설정 갱신을 위해 기존 컨테이너를 삭제하고 새 컨테이너를 실행하면, 이전 컨테이너 내부 경로에 생성했던 파일이 모두 초기화되어 사라짐.
- **원인 분석 (가설)**: 컨테이너 내부에서 생성·수정된 파일은 컨테이너의 **읽기/쓰기 계층(Container Writable Layer)**에 저장되며, 이 계층은 컨테이너 생명주기와 함께 파기되므로 영속성이 보장되지 않음.
- **확인 및 조치**: `docker volume create m1_persist_vol`로 컨테이너 생명주기와 분리된 독립 볼륨을 생성하고 `-v m1_persist_vol:/data`로 마운트하여 컨테이너 삭제(`--rm`) 전후 파일 보존 여부를 검증함.
- **검증 결과**: 첫 번째 컨테이너(`m1-vol-c1`) 삭제 이후에도 두 번째 컨테이너(`m1-vol-c2`)에서 `/data/persist.txt` 내용(`Docker Volume Persistence Verified`)이 그대로 조회됨을 확인함.

### 오류 사례 5: 초기 `README.md`의 실제 실행 로그 누락 및 파일 간 명세 동기화
- **발생 현상**: 초기 제출본에서는 설계 의도만 텍스트로 서술하고 실제 터미널·Docker·`curl`·볼륨 검증 로그와 `Dockerfile`·`index.html`의 `COPY` 구성을 누락하여 자동 검증 및 동료평가 항목을 충족하지 못함.
- **조치 및 결과**: `Dockerfile`이 `index.html`을 이미지에 포함하도록 동기화하고, 로컬 환경에서 모든 CLI·Git·Docker 명령을 실제 실행한 표준 출력을 `README.md`에 수록하여 재현 가능성과 검증 근거를 확보함.
