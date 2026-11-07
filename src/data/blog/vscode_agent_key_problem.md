---
title: "VS Code Remote-SSH 및 터미널 SSH 접속 충돌 해결 기록"
tags: 
   - "Software"
description: "Windows 환경에서 VS Code Remote-SSH를 사용하여 Linux 서버에 접속할 때 발생하는 복합적인 접속 충돌 문제와 그 해결 과정을 정리한 가이드입니다. "
author: Louk
pubDatetime: 2026-11-07T22:20:17
featured: false
draft: false
---

## 🛑 주요 증상

1. **단방향 접속 성공:** 터미널(PowerShell)이나 VS Code 편집기로는 SSH 접속이 되지만, VS Code Agent 원격 접속만 실패함.
2. **known_hosts 초기화 시 임시 해결:** `known_hosts` 파일을 완전히 삭제하면 일시적으로 접속이 되지만, 재접속 시 다시 실패함.
3. **무한 로딩 및 에러 메시지:** VS Code 출력(Output) 로그에 다음과 같은 에러 발생:
   * `Failed to resolve SSH known-hosts files: Cannot determine SSH known-hosts file paths from ambiguous ssh -G output...`
   * `ssh: Could not resolve hostname...`

---

## 🔍 원인 분석

이번 접속 장애는 단일 문제가 아닌 **3가지 요인이 복합적으로 얽혀 발생**했습니다.

### 1. 호스트 키 충돌 (Host Key Verification Failed)

서버 재부팅, 네트워크 변경, 혹은 설정 변경으로 인해 서버의 최신 호스트 키(Host Key)와 로컬 PC의 `known_hosts`에 저장된 과거 키가 불일치했습니다.
이때 `known_hosts` 파일에 동일한 IP에 대해 여러 암호화 방식(`ssh-ed25519`, `ssh-rsa`, `ecdsa-sha2-nistp25`)의 키가 혼재되어 있어 검증 과정에서 충돌이 발생했습니다.

### 2. VS Code 확장의 경로 파싱 버그 (핵심 원인)

VS Code Remote-SSH 확장은 키 충돌 에러가 발생하면 사용자가 직접 `known_hosts` 파일을 수정할 수 있도록 경로를 찾아주는 복구 로직을 실행합니다.
이때 백그라운드에서 `ssh -G` 명령어를 사용하는데, OpenSSH는 2개의 파일 경로(`known_hosts`, `known_hosts2`)를 공백(스페이스)으로 구분하여 출력합니다.
문제는 **Windows 사용자 계정명에 공백(예: `My Username`)이 포함된 경우**, VS Code 내부 파서(Parser)가 이를 제대로 해석하지 못하고 경로 탐색에 실패(`ambiguous ssh -G output`)하며 프로세스를 중단시켜 버리는 버그가 있었습니다.

* `known_hosts`를 삭제했을 때 접속이 된 이유는, 첫 접속으로 인식되어 키 검증 에러 자체가 발생하지 않았고, 따라서 VS Code의 문제 많은 복구 로직도 실행되지 않았기 때문입니다.

### 3. 대소문자 엄격 구분 (Case-sensitivity)

SSH `config` 파일에 등록된 Host 별칭(`My_Server`)과 VS Code가 내부적으로 호출하는 이름(`my_server`)의 대소문자가 달라, OpenSSH가 이를 DNS 도메인으로 오인하고 조회를 시도하다 연결이 끊어졌습니다.

---

## 🛠️ 해결 과정

문제를 근본적으로 차단하기 위해 SSH 설정 파일(`~/.ssh/config`)을 다음과 같이 최적화했습니다.

### 1. 8.3 단축 경로명 적용 (공백 파싱 버그 차단)

경로에 포함된 공백 문제를 우회하기 위해, Windows 파일시스템이 자동으로 부여하는 띄어쓰기 없는 8.3 규격 단축 경로(예: `MYUSER~1`)를 사용하도록 경로를 명시했습니다.

* cmd에서 `dir /x C:\Users` 명령어로 자신의 8.3 단축 경로를 확인할 수 있습니다.

### 2. known_hosts 단일 파일 지정

`UserKnownHostsFile` 옵션을 명시하여 기본값인 2개 대신 단 1개의 파일만 참조하도록 강제했습니다.

### 3. 대소문자 호환 별칭 등록

Host 이름에 대문자(`My_Server`)와 소문자(`my_server`) 별칭을 모두 등록하여 대소문자 구분 문제를 해결했습니다.

---

## ✅ 최종 SSH Config 설정 템플릿

로컬 PC의 `C:\Users\<사용자명>\.ssh\config` 파일을 아래와 같이 작성합니다. (경로의 `MYUSER~1` 부분은 확인한 자신의 단축 경로명으로 변경)

```ssh-config
Host My_Server my_server 192.168.x.x
    HostName 192.168.x.x
    User <서버접속계정>
    Port 22
    IdentityFile C:/Users/MYUSER~1/.ssh/<개인키이름>
    UserKnownHostsFile C:/Users/MYUSER~1/.ssh/known_hosts
    StrictHostKeyChecking accept-new
