# Eventlog Notification for VSFTP

작성자: 홍현준 llane@gscdn.com

개정이력: 

* __2017-05-29:__ 최초작성

---

[TOC]

## TL;DR

* FTP 로그 분석을 통해 접근 계정과 이벤트를 감시하고 이를 통지하기 위한 솔루션을 개발한다.


## 배경

* 본 프로젝트는 고객사의 FTP 서비스의 보안 강화 요청에 의해 제안되었다.
* 서버의 접근 감시 및 이벤트 관제에 필요한 최소한의 정보를 제공하기 위한 소프트웨어를 개발하는 것을 목적으로 한다.

## 개요

### 주요 기능

1. **이벤트 감지:** 로그 파일을 분석하여 이벤트를 감지한다.
2. **이벤트 통보:** 지정된 이벤트 발생 시 지정된 사용자에게 통지한다.
3. **설정:** 웹 인터페이스를 통해 통지 메시지를 전달 받을 수신자를 설정한다.
4. (옵션)이벤트 열람: 이벤트 히스토리 확인을 위한 UI를 제공한다.

[___기능 명세서 참조___](docs/(2017-05-18) ftp-mornitoring_requirement.md)

### 주요 특징

- Logstash와 같은 범용 로그 수집기 활용
- 사용자 편의를 위한 웹인터페이스 제공
- 도커를 이용한 패키징

### 참여 인력 역할과 권한

- 개발담당: 홍현준
- 영업담당: 백영규
- 인력담당: 김종건
- 운영담당: 노규현

### UX

TBD

## Next

* __ASAP:__ 기능명세 확정
* __2017-06-23:__ 프로토 버전 완료
* __2017-06-30:__ 개발 완료 및 배포

## 참조

* [{참조 정보 이름}]({리소스 or url})

