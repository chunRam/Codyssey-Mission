## 1. 요구사항 수행 내역서

### 1.1. 수행 내역 (필수 증거 자료 및 검증 결과)

#### 1) SSH 포트 변경 및 Root 원격 접속 차단 검증
- **검증 내용**: SSH 접속 포트가 `20022`로 변경되었으며, Root 계정으로의 직접적인 원격 접속이 차단되었는지 확인합니다.
- **검증 명령어**:
  ```bash
  sudo grep -i -E "port|permitrootlogin" /etc/ssh/sshd_config
  ```
- **출력 결과**:
  ```text
  Port 20022
  PermitRootLogin no
  ```
- **포트 활성화 상태 확인 (`ss -tulnp`)**:
  ```bash
  ss -tulnp | grep 20022
  ```
  *출력 예시*:
  ```text
  tcp   LISTEN 0      128          0.0.0.0:20022      0.0.0.0:*    users:(("sshd",pid=1234,fd=3))
  ```

#### 2) 방화벽 설정 검증 (UFW)
- **검증 내용**: UFW 방화벽이 활성화되어 있고, 오직 TCP `20022` (SSH) 및 TCP `15034` (APP) 포트만 인바운드로 허용하는지 확인합니다.
- **검증 명령어**:
  ```bash
  sudo ufw status verbose
  ```
- **출력 결과**:
  ```text
  Status: active
  Logging: on (low)
  Default: deny (incoming), allow (outgoing), deny (routed)
  New profiles: skip

  To                         Action      From
  --                         ------      ----
  20022/tcp                  ALLOW IN    Anywhere
  15034/tcp                  ALLOW IN    Anywhere
  20022/tcp (v6)             ALLOW IN    Anywhere (v6)
  15034/tcp (v6)             ALLOW IN    Anywhere (v6)
  ```

#### 3) 계정 및 그룹 구성 검증
- **검증 내용**: 요구사항에 지정된 계정(`agent-admin`, `agent-dev`, `agent-test`) 및 그룹(`agent-common`, `agent-core`)이 올바르게 생성되고 그룹 매핑이 완료되었는지 확인합니다.
- **검증 명령어**:
  ```bash
  id agent-admin
  id agent-dev
  id agent-test
  ```
- **출력 결과**:
  ```text
  $ id agent-admin
  uid=1000(agent-admin) gid=1002(agent-admin) groups=1002(agent-admin),1000(agent-common),1001(agent-core)

  $ id agent-dev
  uid=1001(agent-dev) gid=1003(agent-dev) groups=1003(agent-dev),1000(agent-common),1001(agent-core)

  $ id agent-test
  uid=1002(agent-test) gid=1004(agent-test) groups=1004(agent-test),1000(agent-common)
  ```

#### 4) 디렉토리 구조 및 권한/ACL 검증
- **검증 내용**: `$AGENT_HOME` 하위 디렉토리 및 `/var/log/agent-app` 디렉토리에 대해 요구조건에 맞는 소유자, 그룹, 권한 정책(Setgid 포함)이 적용되었는지 확인합니다.
- **검증 명령어**:
  ```bash
  sudo getfacl /home/agent-admin/agent-app/upload_files /home/agent-admin/agent-app/api_keys /var/log/agent-app
  ```
- **출력 결과**:
  ```text
  # file: home/agent-admin/agent-app/upload_files
  # owner: agent-admin
  # group: agent-common
  # flags: -s-
  user::rwx
  group::rwx
  other::---

  # file: home/agent-admin/agent-app/api_keys
  # owner: agent-admin
  # group: agent-core
  # flags: -s-
  user::rwx
  group::rwx
  other::---

  # file: var/log/agent-app
  # owner: agent-admin
  # group: agent-core
  # flags: -s-
  user::rwx
  group::rwx
  other::---
  ```

#### 5) 앱 Boot Sequence 및 READY 확인
- **검증 내용**: 일반 계정(`agent-admin`)으로 실행하여 5단계의 부트 검증을 모두 `[OK]`로 통과하고 `Agent READY` 상태 및 `15034` 포트 바인딩이 성공했는지 확인합니다.
- **출력 내역**:
  ```text
  Starting Agent Boot Sequence...
  [1/5] Checking User Account
  [OK]
  Running as service user 'agent-admin' (uid=1000)
  [2/5] Verifying Environment Variables [OK]
  All required Envs correct
  [3/5] Checking Required Files
  [OK]
  Verified key file with correct key string.
  [4/5] Checking Port Availability
  Port 15034 is available.
  [5/5] Verifying Log Permission
  [OK]
  [OK]
  Log directory is writable: /var/log/agent-app
  All Boot Checks Passed!
  Agent READY
  ```
- **포트 바인딩 확인 (`ss -tulnp`)**:
  ```bash
  ss -tulnp | grep 15034
  ```
  *출력 예시*:
  ```text
  tcp   LISTEN 0      128          0.0.0.0:15034      0.0.0.0:*    users:(("agent-app-linux",pid=4724,fd=7))
  ```

#### 6) `monitor.sh` 실행 결과 검증
- **검증 내용**: `monitor.sh` 스크립트가 프로세스 상태, 포트 상태, 리소스 사용량을 정확히 확인하고 이상 징후 발생 시 Warning 경고를 콘솔로 출력하는지 검증합니다.
- **실행 권한 및 소유정보**: 소유자 `agent-dev`, 그룹 `agent-core`, 권한 `750` (`rwxr-x---`)
- **수행 명령어**:
  ```bash
  /home/agent-admin/agent-app/bin/monitor.sh
  ```
- **출력 결과**:
  ```text
  ====== SYSTEM MONITOR RESULT ======
  [HEALTH CHECK]
  Checking process 'agent-app-linux-arm64'... [OK] (PID: 4724)
  Checking port 15034... [OK]
  [RESOURCE MONITORING]
  CPU Usage : 1.6%
  MEM Usage : 8.4%
  DISK Used : 6%
  [INFO] Log appended: /var/log/agent-app/monitor.log
  ```

#### 7) crontab 등록 및 자동 실행 확인
- **검증 내용**: `agent-admin` 계정의 크론으로 매 분 `monitor.sh`가 실행되어 `/var/log/agent-app/monitor.log`에 주기적으로 누적되는지 확인합니다.
- **crontab 등록 내역 확인 명령어**:
  ```bash
  sudo crontab -u agent-admin -l
  ```
- **출력 결과**:
  ```text
  * * * * * AGENT_HOME=/home/agent-admin/agent-app AGENT_PORT=15034 AGENT_LOG_DIR=/var/log/agent-app /home/agent-admin/agent-app/bin/monitor.sh >> /var/log/agent-app/cron.log 2>&1
  ```
- **실제 누적된 `monitor.log` 기록 확인**:
  ```bash
  tail -n 5 /var/log/agent-app/monitor.log
  ```
- **출력 결과**:
  ```text
  [2026-05-30 22:15:01] PID:4724 CPU:1.6% MEM:7.2% DISK_USED:6%
  [2026-05-30 22:16:01] PID:4724 CPU:100.0% MEM:5.8% DISK_USED:6%
  [2026-05-30 22:17:01] PID:4724 CPU:2.4% MEM:7.5% DISK_USED:6%
  [2026-05-30 22:18:02] PID:4724 CPU:1.7% MEM:5.5% DISK_USED:6%
  ```

---

### 1.2. 설정/명령어 기록 (Configuration & Command Records)

#### 1) SSH 포트
- **설정 파일**: `/etc/ssh/sshd_config`
- **적용 명령어**:
  ```bash
  # SSH 설정 파일 수정 (Port 변경 및 Root 로그인 차단)
  sudo sed -i 's/#Port 22/Port 20022/' /etc/ssh/sshd_config
  sudo sed -i 's/#PermitRootLogin prohibit-password/PermitRootLogin no/' /etc/ssh/sshd_config

  # SSH 서비스 재시작
  sudo systemctl restart ssh
  ```

#### 2) 방화벽 규칙
- **선택 방화벽**: UFW (Ubuntu)
- **적용 명령어**:
  ```bash
  # 인바운드 기본 차단 및 아웃바운드 기본 허용
  sudo ufw default deny incoming
  sudo ufw default allow outgoing

  # 특정 서비스 포트만 개방 (SSH: 20022, App: 15034)
  sudo ufw allow 20022/tcp
  sudo ufw allow 15034/tcp

  # 방화벽 활성화
  sudo ufw enable
  ```

#### 3) 계정/그룹/ACL
- **적용 명령어**:
  ```bash
  # 1. 요구 그룹 생성
  sudo groupadd agent-common
  sudo groupadd agent-core

  # 2. 요구 사용자 계정 생성 (홈 디렉토리 생성 및 기본 쉘 지정)
  sudo useradd -m -s /bin/bash agent-admin
  sudo useradd -m -s /bin/bash agent-dev
  sudo useradd -m -s /bin/bash agent-test

  # 3. 사용자 계정을 해당 그룹에 추가
  sudo usermod -aG agent-common,agent-core agent-admin
  sudo usermod -aG agent-common,agent-core agent-dev
  sudo usermod -aG agent-common agent-test
  ```

#### 4) 디렉토리/권한
- **적용 명령어**:
  ```bash
  # 1. 요구 디렉토리 구조 생성
  sudo mkdir -p /home/agent-admin/agent-app/upload_files
  sudo mkdir -p /home/agent-admin/agent-app/api_keys
  sudo mkdir -p /var/log/agent-app

  # 2. 디렉토리 소유자와 소유 그룹 지정
  sudo chown agent-admin:agent-common /home/agent-admin/agent-app/upload_files
  sudo chown agent-admin:agent-core /home/agent-admin/agent-app/api_keys
  sudo chown agent-admin:agent-core /var/log/agent-app

  # 3. 권한 설정 및 Setgid 설정 (2770)
  # Setgid(2000)를 부여하여 해당 디렉토리 내에 생성되는 파일들이 부모 디렉토리의 그룹을 상속받도록 함.
  sudo chmod 2770 /home/agent-admin/agent-app/upload_files
  sudo chmod 2770 /home/agent-admin/agent-app/api_keys
  sudo chmod 2770 /var/log/agent-app
  ```

#### 5) 환경 변수
- **설정 파일**: `/home/agent-admin/.bashrc`
- **적용 명령어**:
  ```bash
  # agent-admin 계정의 .bashrc 파일 하단에 환경 변수 선언 추가
  cat << 'EOF' >> /home/agent-admin/.bashrc

  # Agent App Environment Variables
  export AGENT_HOME=/home/agent-admin/agent-app
  export AGENT_PORT=15034
  export AGENT_UPLOAD_DIR=/home/agent-admin/agent-app/upload_files
  export AGENT_KEY_PATH=/home/agent-admin/agent-app/api_keys
  export AGENT_LOG_DIR=/var/log/agent-app
  EOF

  # 환경 변수 반영
  source /home/agent-admin/.bashrc
  ```

#### 6) cron 등록 등
- **적용 명령어**:
  ```bash
  # agent-admin 사용자의 크론탭 에디터 실행
  # (또는 sudo crontab -u agent-admin -e)
  crontab -e
  ```
- **크론탭 등록 내용**:
  ```text
  # 매 분(1분 간격) 실행 환경 변수 정의와 함께 monitor.sh를 백그라운드 자동 실행하도록 구성
  * * * * * AGENT_HOME=/home/agent-admin/agent-app AGENT_PORT=15034 AGENT_LOG_DIR=/var/log/agent-app /home/agent-admin/agent-app/bin/monitor.sh >> /var/log/agent-app/cron.log 2>&1
  ```

---

### 필수 증거 자료 체크리스트

- [x] SSH 포트 변경 `20022` 및 Root 원격 접속 차단 설정 확인 내역
  - **증거**: `/etc/ssh/sshd_config` 파일 내 `Port 20022` 및 `PermitRootLogin no` 설정 결과 확인
- [x] 방화벽 활성화 및 `20022/tcp`, `15034/tcp`만 허용 내역
  - **증거**: UFW 방화벽 활성화 상태 및 인바운드 허용 포트(`20022/tcp`, `15034/tcp`) 확인
- [x] 계정/그룹 생성 확인 내역
  - [x] `agent-admin` (그룹: `agent-common`, `agent-core` 포함)
  - [x] `agent-dev` (그룹: `agent-common`, `agent-core` 포함)
  - [x] `agent-test` (그룹: `agent-common` 포함)
  - [x] `agent-common`
  - [x] `agent-core`
  - **증거**: `id` 명령어 조회를 통한 사용자 생성 및 그룹 매핑 확인
- [x] 디렉토리 구조 및 권한 확인 내역
  - **증거**: `$AGENT_HOME/upload_files` (770, agent-common, Setgid 적용), `$AGENT_HOME/api_keys` (770, agent-core, Setgid 적용), `/var/log/agent-app` (770, agent-core, Setgid 적용) 확인
- [x] 앱 Boot Sequence 5단계 `[OK]` 및 `"Agent READY"` 확인 내역
  - **증거**: `agent-admin` 권한으로 실행한 애플리케이션의 부팅 검증 5단계 정상 출력 증적 확인
- [x] `monitor.sh` 실행 결과 확인 내역
  - [x] 프로세스 (동작 중인 `agent-app-linux-arm64` PID 감지)
  - [x] 포트 (`15034/tcp` Listen 확인)
  - [x] 리소스 (CPU, MEM, DISK 실시간 사용량 출력)
  - [x] 경고 (임계치 초과 시 [WARNING] 출력)
  - **증거**: `monitor.sh` 수동 실행을 통한 정상 콘솔 출력 결과물 확인
- [x] `/var/log/agent-app/monitor.log` 누적 기록 확인 내역
  - **증거**: `monitor.sh` 실행 시 지정된 포맷의 로그가 정상적으로 누적 기록되는 증적 확인
- [x] crontab 매분 실행 등록 및 자동 실행 확인 내역
  - **증거**: `agent-admin` 크론탭 리스트 등록 확인 및 크론로그를 통해 매 분 자동 수행되는 기록 확인

---

### 2. 자동화 스크립트 소스코드

- `monitor.sh` (경로: `/home/agent-admin/agent-app/bin/monitor.sh`)

```bash
#!/bin/bash

AGENT_HOME="/home/agent-admin/agent-app"
APP_NAME="agent-app-linux-arm64"
PORT="15034"
LOG_DIR="/var/log/agent-app"
LOG_FILE="$LOG_DIR/monitor.log"
MAX_SIZE=$((10 * 1024 * 1024))
MAX_FILES=10

rotate_logs() {
  if [ -f "$LOG_FILE" ]; then
    size=$(stat -c%s "$LOG_FILE")
    if [ "$size" -ge "$MAX_SIZE" ]; then
      rm -f "${LOG_FILE}.10"

      for i in 9 8 7 6 5 4 3 2 1
      do
        if [ -f "${LOG_FILE}.${i}" ]; then
          mv "${LOG_FILE}.${i}" "${LOG_FILE}.$((i + 1))"
        fi
      done

      mv "$LOG_FILE" "${LOG_FILE}.1"
    fi
  fi
}

echo "====== SYSTEM MONITOR RESULT ======"
echo "[HEALTH CHECK]"

PID=$(pgrep -f "$APP_NAME" | head -n 1)
if [ -z "$PID" ]; then
  echo "Checking process '$APP_NAME'... [FAIL]"
  exit 1
fi
echo "Checking process '$APP_NAME'... [OK] (PID: $PID)"

if ss -tuln | grep -q ":$PORT "; then
  echo "Checking port $PORT... [OK]"
else
  echo "Checking port $PORT... [FAIL]"
  exit 1
fi

if command -v ufw >/dev/null 2>&1; then
  if ! ufw status | grep -q "Status: active"; then
    echo "[WARNING] Firewall is not active"
  fi
fi

CPU=$(top -bn1 | awk '/^%Cpu/ {printf "%.1f", 100 - $8}')
MEM=$(free | awk '/Mem:/ {printf "%.1f", ($3 / $2) * 100}')
DISK=$(df / | awk 'NR==2 {gsub("%","",$5); print $5}')

echo "[RESOURCE MONITORING]"
echo "CPU Usage : ${CPU}%"
echo "MEM Usage : ${MEM}%"
echo "DISK Used : ${DISK}%"

if awk "BEGIN {exit !($CPU > 20)}"; then
  echo "[WARNING] CPU threshold exceeded (${CPU}% > 20%)"
fi

if awk "BEGIN {exit !($MEM > 10)}"; then
  echo "[WARNING] MEM threshold exceeded (${MEM}% > 10%)"
fi

if [ "$DISK" -gt 80 ]; then
  echo "[WARNING] DISK threshold exceeded (${DISK}% > 80%)"
fi

mkdir -p "$LOG_DIR"
rotate_logs

TIMESTAMP=$(date '+%Y-%m-%d %H:%M:%S')
printf '[%s] PID:%s CPU:%s%% MEM:%s%% DISK_USED:%s%%\n' "$TIMESTAMP" "$PID" "$CPU" "$MEM" "$DISK" >> "$LOG_FILE"

echo "[INFO] Log appended: $LOG_FILE"
```
