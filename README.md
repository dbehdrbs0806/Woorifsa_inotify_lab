# 쉘 스크립트 기반 inotify CI/CD 자동화 시스템 구현

## 📌 1. 개요

본 실습은 **로컬 환경에서 코드 변경을 자동**으로 감지하고 빌드한 뒤 원격 VM으로 전송, **배포하여 재실행을 수행**하는 자동화 구조를 구성  
Jenkins나 GitHub Actions 와 같은 툴 없이, `Bash 스크립트와 Linux의 inotify 기능`만을 이용해 **간단한 CI/CD 흐름**을 직접 구현  
<br>
<img width="1000" height="400" alt="스크린샷 2026-03-24 174214" src="https://github.com/user-attachments/assets/92c9cbb3-d241-48c6-93ae-61bd7559cab8" />

---

## 2. 사용 기술

### Git Bash / Bash Script
- 로컬 Windows 환경에서 쉘 스크립트를 실행하기 위해 사용
- 프로젝트 변경 감지, 빌드, 원격 전송 과정을 스크립트로 자동화하는 역할

### Gradle
- Spring Boot 프로젝트를 빌드하여 실행 가능한 JAR 파일을 생성
- 코드 변경 발생 시 `bootJar` 작업을 수행해 최신 배포 파일을 생성

### SSH / scp
- 로컬에서 빌드된 JAR 파일을 원격 VM으로 전송
- scp 명령어를 통해 VM에 파일을 전송

### inotifywait
Linux 에서 특정 디렉토리 또는 파일의 변경 이벤트를 감지하기 위해 사용   
VM에서 새로운 JAR 파일의 배포를 감지하고, 기존 애플리케이션 종료 후 새 JAR를 다시 실행하는 자동 재배포 역할을 수행

---

## 3. 구현 구조

본 자동화 시스템은 다음과 같은 흐름으로 동작한다.

* **로컬 Windows(Git Bash)** 에서 Spring Boot 프로젝트 파일의 변경 사항을 감지한다.
* 변경이 발생하면 `Gradle bootJar`를 실행하여 최신 JAR 파일을 생성한다.
* 생성된 JAR 파일을 **SCP를 통해 원격 Ubuntu VM**으로 전송한다.
* 이때 전송 중인 파일이 잘못 실행되는 문제를 방지하기 위해 `.tmp` 이름으로 먼저 업로드한 뒤, 전송 완료 후 최종 JAR 이름으로 변경한다.
* 원격 VM에서는 `inotifywait`가 `/home/ubuntu/deploy` 경로를 감시하고 있다가, 새 JAR 파일이 반영되면 기존 실행 중인 애플리케이션을 종료하고 새로운 JAR를 실행한다.

즉, **로컬 코드 변경 → 로컬 빌드 → 원격 전송 → 원격 재실행**의 흐름으로 CI/CD가 구성된다.

---

## 4. 변경 사항 감지 및 배포 대상

### 변경 감지 위치

로컬 Windows 환경의 Spring Boot 프로젝트 디렉토리

```bash
C:\ce6\04. SpringBoot\SpringTest_Build-Deploy
```

Git Bash 기준 경로:

```bash
/c/ce6/04. SpringBoot/SpringTest_Build-Deploy
```

### 배포 대상 위치

원격 Ubuntu VM의 배포 디렉토리

```bash
/home/ubuntu/deploy
```

### 감시 및 실행 대상 파일

최종 배포되는 JAR 파일

```bash
SpringTest_Build-Deploy-0.0.1-SNAPSHOT.jar
```

---

## 5. 핵심 스크립트 구성

본 구현은 크게 두 개의 스크립트로 구성된다.

### 5.1 로컬 배포 스크립트

로컬 프로젝트에서 변경 사항을 감지하고, 빌드 후 원격 서버로 JAR를 전송하는 역할을 수행한다.

#### 주요 기능

* 프로젝트 변경 감지
* Gradle 빌드 수행
* 최신 JAR 파일 탐색
* 원격 VM으로 `.tmp` 파일 전송
* 전송 완료 후 최종 JAR로 이름 변경

#### 핵심 코드 예시

```bash
send_jar() {
  local jar_file="$1"

  if [ -z "$jar_file" ]; then
    log "전송할 jar 파일을 찾지 못했습니다."
    return 1
  fi

  local base_name
  base_name="$(basename "$jar_file")"

  log "원격 서버 디렉토리 확인"
  ssh -p "$REMOTE_PORT" "${REMOTE_USER}@${REMOTE_HOST}" "mkdir -p '$REMOTE_TARGET_DIR'"

  log "scp 임시 파일 전송 시작: ${base_name}.tmp"
  scp -P "$REMOTE_PORT" "$jar_file" "${REMOTE_USER}@${REMOTE_HOST}:${REMOTE_TARGET_DIR}/${base_name}.tmp"

  log "원격 파일 교체: ${base_name}.tmp -> ${base_name}"
  ssh -p "$REMOTE_PORT" "${REMOTE_USER}@${REMOTE_HOST}" \
    "mv '${REMOTE_TARGET_DIR}/${base_name}.tmp' '${REMOTE_TARGET_DIR}/${base_name}'"

  log "scp 전송 완료"
}
```

---

### 5.2 원격 VM 감시 스크립트

원격 서버에서 배포 디렉토리를 감시하고, JAR 변경 시 애플리케이션을 재실행하는 역할을 수행한다.

#### 주요 기능

* 배포 디렉토리 감시
* JAR 변경 이벤트 감지
* 기존 Java 프로세스 종료
* 새 JAR 파일 실행
* PID 및 로그 관리

#### 핵심 코드 예시

```bash
inotifywait -m -e moved_to,close_write "$DEPLOY_DIR" |
while read -r directory events filename; do
  CURRENT_TIME=$(date +%s)

  if [ "$filename" = "$JAR_NAME" ]; then
    if (( CURRENT_TIME - LAST_RUN > COOLDOWN )); then
      log "jar 변경 감지: $filename ($events)"
      LAST_RUN=$CURRENT_TIME
      stop_app
      start_app
    else
      log "쿨다운 기간 중. 재시작 생략"
    fi
  fi
done
```

---

## 6. inotify 명령어 설명

본 구현에서 사용한 `inotifywait`는 Linux 파일 시스템 이벤트를 실시간으로 감지하기 위한 명령어이다.

사용한 예시는 다음과 같다.

```bash
inotifywait -m -e moved_to,close_write "$DEPLOY_DIR"
```

### 옵션 설명

* `-m`
  감시를 한 번만 수행하는 것이 아니라 지속적으로 모니터링한다.

* `-e moved_to`
  파일이 특정 디렉토리로 이동되어 들어오는 이벤트를 감지한다.
  본 구현에서는 `.tmp` 파일이 최종 JAR 파일명으로 `mv` 되는 순간을 감지하기 위해 사용하였다.

* `-e close_write`
  파일 쓰기 작업이 완료되고 닫히는 시점을 감지한다.
  파일 저장 완료 시점을 기준으로 이벤트를 처리할 수 있다.

즉, `inotifywait`를 통해 **새로운 JAR가 최종 배포 디렉토리에 반영된 시점**을 정확히 감지하고, 그때 애플리케이션을 재시작하도록 구성하였다.

---

## 7. 결론

본 구현을 통해 별도의 대형 CI/CD 도구 없이도, **쉘 스크립트와 inotify 기반만으로 간단한 자동 빌드 및 자동 배포 환경을 구성할 수 있음**을 확인하였다.
특히 `.tmp` 파일 전송 후 최종 이름으로 교체하는 방식을 적용하여, 전송 중 파일이 잘못 실행되는 문제를 줄일 수 있었다.
또한 원격 서버에서 inotify를 활용해 변경 사항을 즉시 감지하고 애플리케이션을 재실행함으로써, 코드 수정 이후 배포 반영 과정을 자동화할 수 있었다.

---
