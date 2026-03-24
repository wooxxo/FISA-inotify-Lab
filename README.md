# 🚀 Spring Boot 자동 배포 자동화 프로젝트

> Windows 개발 환경에서 빌드한 Spring Boot jar 파일을 Linux 서버에 자동 배포하고,
> 파일 변경을 감지하여 무중단에 가깝게 재배포하는 자동화 파이프라인 구현

![test](/Images/test1.gif)


## 👥 팀원 소개
| <img width="150" height="150" alt="image" src="https://github.com/user-attachments/assets/bff28997-d960-484e-bc47-7ed9c99f0e6a" /> | <img width="150" height="150" alt="image" src="https://github.com/user-attachments/assets/9a580c23-6e4b-4be7-a000-63345ba3657f" /> |
| :---: | :---: |
| 우승연<br>[@wooxxo](https://github.com/wooxxo) | 이채유<br>[@chaeyuuu](https://github.com/chaeyuuu) |


## 📌 기술 스택

| 항목 | 내용 |
|---|---|
| 개발 환경 | Windows (Git Bash) |
| 서버 환경 | Ubuntu 22.04 (VMware) |
| 빌드 도구 | Gradle |
| 배포 방식 | SCP + inotify-tools |
| 실행 포트 | 2026 |


## 🏗️ 아키텍처
```
[ Windows - 개발 환경 ]
        │
        │  ./deploy.sh (Git Bash)
        │  1. Gradle 빌드 → jar 생성
        │  2. SCP로 Linux 서버 전송
        ▼
[ Ubuntu - Linux 서버 ]
        │
        │  deploy.sh (inotify-tools)
        │  3. jar 파일 변경 감지
        │  4. 기존 프로세스 kill (포트 충돌 방지)
        │  5. 새 jar 자동 실행
        ▼
[ Spring Boot App 서비스 중 :2026 ]
```


## 📂 파일 구조
```
프로젝트 루트/
├── deploy.sh                         # Windows(Git Bash) 빌드 & 전송 스크립트
├── src/
└── build/libs/
    └── step00_Jar-0.0.1-SNAPSHOT.jar

Ubuntu 서버/
├── ~/step00_Jar-0.0.1-SNAPSHOT.jar   # 전송된 jar
├── ~/deploy.sh                       # 변경 감지 & 재배포 스크립트
└── ~/app.log                         # 실행 로그
```


## ⚙️ 실행 방법

### 1. Linux 서버 - 변경 감지 & 재배포 스크립트

inotify-tools를 활용해 jar 파일 변경을 감지하고 자동으로 재배포
```bash
# inotify-tools 설치
sudo apt-get install -y inotify-tools

# 실행 권한 부여 및 실행
chmod +x deploy.sh
./deploy.sh
```

**deploy.sh**
```bash
#!/bin/bash

JAR_PATH="/home/ubuntu/step00_Jar-0.0.1-SNAPSHOT.jar"
PORT=2026
COOLDOWN=10
LAST_RUN=0

# 실행 중인 포트 있으면 종료 (포트 충돌 방지)
kill_existing() {
    EXISTING_PID=$(lsof -ti tcp:$PORT)
    if [ -n "$EXISTING_PID" ]; then
        echo "$(date): 포트 $PORT 종료 PID=$EXISTING_PID"
        kill -9 $EXISTING_PID
        sleep 1
    fi
}

# jar 파일 실행
start_app() {
    kill_existing
    echo "$(date): 앱 실행 시작"
    nohup java -jar $JAR_PATH --server.port=$PORT > /home/ubuntu/app.log 2>&1 &
    echo "$(date): PID=$! 로 실행됨"
}

# 최초 실행
start_app

# 변경 감지 루프
inotifywait -m -e close_write "$JAR_PATH" |
while read -r directory events filename; do
    CURRENT_TIME=$(date +%s)
    if (( CURRENT_TIME - LAST_RUN > COOLDOWN )); then
        echo "$(date): 변경 감지 → 재배포"
        LAST_RUN=$CURRENT_TIME
        start_app
    else
        echo "$(date): 쿨다운 중"
    fi
done
```

</details>


### 2. Windows - 빌드 & 배포 스크립트

Git Bash에서 Gradle 빌드 후 SCP로 Linux 서버에 jar를 전송
```bash
# Git Bash에서
chmod +x deploy.sh
./deploy.sh
```

**deploy.sh**
```bash
#!/bin/bash

SERVER_IP="172.21.31.21"
SERVER_PORT=2020
SERVER_USER="ubuntu"
JAR_NAME="step00_Jar-0.0.1-SNAPSHOT.jar"

echo "=== 1. 빌드 시작 ==="
./gradlew build -x test

echo "=== 2. jar 전송 ==="
scp -P $SERVER_PORT \
    build/libs/$JAR_NAME \
    $SERVER_USER@$SERVER_IP:/home/ubuntu/$JAR_NAME

echo "=== 완료! 서버에서 자동 재배포 중 ==="
```

</details>


## 🔄 배포 흐름
```
코드 수정
    ↓
./deploy.sh 실행 (Git Bash)
    ↓
Gradle 빌드 (-x test)
    ↓
SCP 전송 → Ubuntu ~/app.jar
    ↓
inotifywait close_write 감지
    ↓
기존 PID kill → 새 jar 실행
    ↓
서비스 갱신 완료
```


## 🔧 Troubleshooting

### 1️⃣ 포트 바인딩 권한 오류 (Permission denied)

**문제**
```
Caused by: java.net.BindException: Permission denied
Unable to start embedded Tomcat server
```

: 리눅스는 **1024 이하의 포트 (Well-known Port)** 에 대해 root 권한만 바인딩을 허용

| 포트 범위 | 명칭 | 권한 |
|---|---|---|
| 0 ~ 1023 | Well-known Port | root 전용 |
| 1024 ~ 49151 | Registered Port | 일반 사용자 가능 |
| 49152 ~ 65535 | Dynamic Port | 일반 사용자 가능 |

**해결**
```bash
application.properties 포트 설정을 80 포트 대신 2026 포트로 변경
```


### 2️⃣ 포트 충돌 오류

**문제**

: 재배포 시 이미 포트가 사용 중이라 앱 실행 실패

**해결** 

: `deploy.sh` 내 `kill_existing()` 함수로 자동 처리
```bash
EXISTING_PID=$(lsof -ti tcp:$PORT)
kill -9 $EXISTING_PID
```


### 3️⃣ 유니코드 공백 오류

**문제**
```
scp: stat local "\302\240\302\240build/libs/...": No such file or directory
```

: 코드 복붙 시 유니코드 공백문자(`\xc2\xa0`)가 섞여 들어옴  
눈으로 보면 일반 공백처럼 보이지만 실제로는 다른 문자라 경로 인식 실패

**해결**
```bash
# 유니코드 공백을 일반 공백으로 교체
sed -i 's/\xc2\xa0/ /g' deploy.sh

# 확인
cat -A deploy.sh | grep scp
```

### 4️⃣ sh / bash 문법 오류

**문제**
```
deploy.sh: 37: CURRENT_TIME: not found
```

: `#!/bin/bash` 파일인데 `sh`로 실행하면 `(( ))` 산술 문법 미지원

**해결**
```bash
# sh 말고 bash로 실행
bash deploy.sh
```
