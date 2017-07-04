# Eventlog Notification for VSFTP

작성자: 홍현준 llane@gscdn.com

---

[TOC]

Eventlog Notification for VSFTP(이하: enoti-v)는 Logstash를 이용하여 VSFTP의 이벤트 감지 및 통지를 위한 방법을 가이드합니다.

## Requirements

- Logstash를 구동하기 위한 Java 1.8이 필요합니다.
- 메일 전송을 위해 sendmail 서비스가 로컬에 실행되어 있어야 합니다.

## Installation

Logstash를 인스톨하기 위한 기본 가이드는 [공식 문서: download-logstash](https://www.elastic.co/kr/downloads/logstash)를 참고하십시오. 기본적인 절차는 다음과 같습니다.

- Logstash 다운로드
- RPM 설치 혹은 Unzip
- 플러그인 설치
- 설정 파일 갱신
- 실행

RPM 방식 기준 인스톨은 다음의 예시를 참고하세요

```shell
# download & install
curl -L -O https://artifacts.elastic.co/downloads/logstash/logstash-5.4.2.rpm
rpm -i logstash-5.4.2.rpm

# install plugin
/usr/share/logstash/bin/logstash-plugin install logstash-output-email
/usr/share/logstash/bin/logstash-plugin install logstash-filter-translate
```

자세한 내용은 [공식 문서: installing-logstash](https://www.elastic.co/guide/en/logstash/current/installing-logstash.html)를 참고하시길 바랍니다.

## Quick Start

본 가이드를 통해 서비스를 실행 시키기 위해서는 네 가지 설정 파일이 필요합니다. 그 중 두가지는 Logstash 동작과 패턴을 정의하는 설정이고 기본 설정을 유지한다면 별도로 수정할 필요가 없지만 SMS API 설정 정보와 FTP 사용자 정보는 반드시 초기 입력이 필요합니다. 만약 기본 디렉터리 구조를 유지 한다면 레포지토리로 부터 기본 환경설정 파일을 내려받아 초기 구성을 셋업할 수도 있습니다. 

```shell
# configuration clone
curl -L -O https://github.com/gsneotek-wisen/enoti-v/archive/v0.1.tar.gz
tar xvzf v0.1.tar.gz
cp ./enoti-v-0.1/etc/logstash/conf.d/* /etc/logstash/conf.d/ -rf

# change sms setting
vim /etc/logstash/conf.d/props/vsftp-props.yml

# add user information
vim /etc/logstash/conf.d/props/vsftp-users.yml

# run
initctl start logstash
```

정상적으로 서비스가 실행되었다면 모든 통지 이벤트에 대한 이력은 CSV 형태의 파일에 추가 기록됩니다. 이 파일을 통해 통지 이력을 확인 할 수 있으며 기본 경로는 `/var/log/logstash/vsftp_event.csv` 입니다.

```shell
tail /var/log/logstash/vsftp_event.csv
2017-06-27T09:59:19,ftpguest,10.0.0.1,LOGIN,OK,,,
2017-06-28T05:33:35,ftpguest,10.0.0.1,UPLOAD,OK,/home/ftpguest/sample,6,
2017-06-28T05:33:55,ftpguest,10.0.0.1,DOWNLOAD,OK,/home/ftpguest/sample,6,
2017-06-28T05:34:06,ftpguest,10.0.0.1,RENAME,OK,/home/ftpguest/sample,,/home/ftpguest/samples
2017-06-28T05:34:19,ftpguest,10.0.0.1,DELETE,OK,/home/ftpguest/samples,,
```



## Configuration

본 가이드는 네 가지 설정 파일을 사용합니다.

- __vsftp-logs.conf:__ Logstash의 프로세싱 Logstash의 파이프라인을 정의합니다.
- __vsftp-patterns:__ VSFTP의 로그 패턴을 정의합니다.
- __vsftp-props.yml:__ SMS API를 사용하기 위한 기본 정보를 설정합니다.
- __vsftp-users.yml:__ FTP 상용자 별 Email, SMS 번호를 설정합니다.

### vsftp-logs.conf

> /etc/logstash/conf.d/___vsftp-logs.conf___

Logstash의 파이프라인을 정의합니다. 환경설정은 크게 세가지 섹션(_input, output, filter_)으로 구성되며 이벤트 프로세싱 파이프라인을 처리하기 위해 추가하는 플러그인의 타입에 따라 구분 됩니다.

- input: Logstash가 수신할 이벤트 소스를 설정합니다.
- output: Logstash가 이벤트 데이터를 전달할 대상을 설정합니다.
- filter: 필터링, 파싱 등의 전달 받은 이벤트의 변환 절차를 설정합니다.

자세한 내용은 [공식 문서: Structure of a Config File](https://www.elastic.co/guide/en/logstash/current/configuration-file-structure.html)를 참고하시길 바라며 각 섹션의 자세한 내용은 __공식 문서:__ [Input plugins](https://www.elastic.co/guide/en/logstash/current/input-plugins.html), [Output plugins](https://www.elastic.co/guide/en/logstash/current/output-plugins.html), [Filter plugins](https://www.elastic.co/guide/en/logstash/current/filter-plugins.html) 를 참고하시길 바랍니다. 다음은 주요 설정에 대한 예시 입니다.

```shell
input {
  # 파일로 부터 스트림 이벤트를 수신합니다. 기본적으로 `tail -0F` 동작과 유사합니다.
  file {
    path => "/var/log/vsftpd*"
  }
}

filter {
  # 전달받은 비정형 텍스트를 구조적으로 파싱합니다.
  grok {
    patterns_dir => ["/etc/logstash/conf.d/patterns"]
    match => {
      "message" => [
        ...
      ]
    }
  }
  ...
}

output {
  # HTTP로 파싱된 데이터를 전달합니다.
  http {
    ...
  }
  ...
}
```

전체 예시는 [첨부파일](etc/logstash/conf.d/vsftp-logs.conf)을 참고하세요.

### vsftp-patterns

> /etc/logstash/conf.d/patterns/___vsftp-patterns___

Logstash의 _filter_ 섹션의 _grok_ 플러그인에서 사용할 사용자 정의 패턴을 생성합니다. _grok_ 플러그인은 기 정의된 정규식 패턴을 연계하고 재정의하여 텍스트를 파싱 후 매핑된 텍스트에 이름을 부여할 수 있습니다. 기본 문법은 `%{SYNTAX:SEMANTIC}` 와 같이 정의됩니다. 예를 들어 `3.44`는 `NUMBER`에, `10.0.0.1`는 `IP`로 매핑하여 각각 이름을 부여하려면, `%{NUMBER:duration} %{IP:client}`와 같이 정의 할 수 있습니다. 자세한 내용은 [공식 문서: plugins-filters-grok#_custom_patterns](https://www.elastic.co/guide/en/logstash/current/plugins-filters-grok.html#_custom_patterns)을 참고하시길 바랍니다. 다음은 VSFTP 로그 파일에서 `CONNECT` 동작에 대한 패턴 예시입니다.

```shell
VSFTPD_DATETIME %{DAY} %{MONTH} %{SPACE}%{NUMBER} %{TIME} %{YEAR}
VSFTPD_CONNECT \A%{VSFTPD_DATETIME:datetime} \[pid %{NUMBER:vsftpd_pid}] %{WORD:vsftpd_action}: Client "%{IP:vsftpd_client_ip}"
```

전체 예시는 [첨부파일](etc/logstash/conf.d/patterns/vsftp-patterns)을 참고하세요.

###  vsftp-props.yml

> /etc/logstash/conf.d/props/___vsftp-props.yml___

Logstash의 _filter_ 섹션의 _translate_ 플러그인에서 사용할 딕셔너리 파일을 생성합니다. _translate_ 플러그인은 외부 파일로 로드한 Key/Value 사전으로부터 값을 검색하거나 변환할 수 있습니다. 자세한 내용은 [공식 문서: plugins-filters-translate#_dictionary_path](https://www.elastic.co/guide/en/logstash/current/plugins-filters-translate.html#plugins-filters-translate-dictionary_path)을 참고하시길 바랍니다. 본 가이드에서는 Logstash에서 사용할 환경값을 설정하기 위해 활용되며 SMS API를 사용할 기본 정보가 저장됩니다. `{KEY}: "{VALE}"`와 같은 형식으로 정의하며 다음은 예시입니다.

```yaml
sms_id: "SMS_ID"
sms_passwd: "SMS_PASSWD"
sms_back: "0212345678"
email_from: "ftpguest@sample.com"
```

전체 예시는 [첨부파일](etc/logstash/conf.d/props/vsftp-props.yml)을 참고하세요.

###  vsftp-users.yml

> /etc/logstash/conf.d/props/___vsftp-users.yml___

Logstash의 _filter_ 섹션의 _translate_ 플러그인에서 사용할 딕셔너리 파일을 생성합니다. _translate_ 플러그인은 외부 파일로 로드한 Key/Value 사전으로부터 값을 검색하거나 변환할 수 있습니다. 자세한 내용은 [공식 문서: plugins-filters-translate#_dictionary_path](https://www.elastic.co/guide/en/logstash/current/plugins-filters-translate.html#plugins-filters-translate-dictionary_path)을 참고하시길 바랍니다. 본 가이드에서는 VSFTP의 사용자 계정 별 메타 정보를 확인하기 위해 활용되며 이벤트를 수신할 Email과 SMS 번호를 저장합니다. `{FTP_ACCOUNT}: "{USER_EMAIL}/{USER_PHONE_NUMBER}"`와 같은 형식으로 정의하며 멀티 대상을 지정 시 `콤마 ,{USER_EMAIL}`로 추가 할 수 있습니다. 다음은 예시입니다.

```yaml
ftpguest: "ftpguest@sample.com/01012345678"
ftpother: "ftpother1@sample.com,ftpother2@sample.com/01012345678,01087654321"
```

전체 예시는 [첨부파일](etc/logstash/conf.d/props/vsftp-users.yml)을 참고하세요.

## Problems

### VSFTP 로그 퍼미션 문제

VSFTP 로그의 읽기 권한 문제로 Logstash가 해당 파일을 접근하지 못하는 경우가 있습니다. 보통의 경우 VSFTP에서 로그 파일 생성 시 소유자에 한하여 읽기 권한을 부여하기 때문입니다. 이 경우 해당 파일에 대한 공개 읽기 권한을 열어 줌으로써 문제를 해결 할 수 있습니다.

```shell
# input 파일에 대한 읽기 권한이 없어 발생한 WARN 로그 확인
tail -n 100 /var/log/logstash/logstash-plain.log | grep WARN
[2017-06-27T09:49:00,927][WARN ][logstash.inputs.file     ] failed to open /var/log/vsftpd.log: Permission denied - /var/log/vsftpd.log

# 해당 파일에 읽기 권한을 부여
chmod +r /var/log/vsftpd.log
```

### Email 전송이 실패하는 경우

Logslash의 email 플러그인은 다양한 메일 서버를 지원하지만 본 가이드에서는 기본 설정으로 로컬의 [Sendmail](https://en.wikipedia.org/wiki/Sendmail) 을 사용해 메일을 전송합니다. 별도의 메일 서버(_gmail and etc.._)를 사용하기 위해서는 해당 밴더의 가이드에따라 설정 변경이 필요합니다. 만약 기본 설정을 유지 한다면 하기와 같이 Sendmail이 정상적으로 실행 중인지 확인이 필요합니다.

```shell
# Sendmail 서비스 상태 확인
service sendmail status
sendmail (pid  19772) is running...
sm-client (pid  19782) is running...

# Sendmail이 설치되지 않았거나 재설치가 필요할 경우
rpm -qa sendmail*
yum install sendmail sendmail-cf
/etc/init.d/sendmail start
```

### 한글 파일 문제

VSFTP를 소스 빌드하여 설치하지 않는한 기본적으로 로그에 한글 깨짐 현상이 발생합니다. 정규식을 통해 해당 Path를 매칭하는 것은 가능 하지만 복원은 불가능 합니다. 추가로 Upload/Download 시 파일 크기가 나오지 않습니다.

```shell
Fri Jun 30 12:08:04 2017 [pid 30490] [test] OK UPLOAD: Client "10.0.0.1", "/test/??? Microsoft Word ??????.docx", 0.00Kbyte/sec
```

본 가이드에서는 파일 크기 누락시 0 bytes 로 예외 처리 합니다.

## Next

