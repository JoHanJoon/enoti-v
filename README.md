# Eventlog Notification for VSFTP

작성자: 홍현준 llane@gscdn.com

---

[TOC]

Eventlog Notification for VSFTP(이하: enoti-v)는 Logstash를 이용하여 VSFTP의 이벤트 감지 및 통지를 위한 방법을 가이드합니다.

## Requirements

Logstash를 구동하기 위한 Java 1.8이 필요합니다.

## Installation

Logstash를 인스톨하기 위한 기본 가이드는 [이곳](https://www.elastic.co/kr/downloads/logstash)을 참고하십시오. 기본적인 절차는 다음과 같습니다.

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
# update configuration
...
# run
initctl status logstash
```

자세한 내용은 [공식 문서: installing-logstash](https://www.elastic.co/guide/en/logstash/current/installing-logstash.html)를 참고하시길 바랍니다.



## Configuration

본 가이드는 세가지 설정 파일이 필요합니다.

- /etc/logstash/conf.d/___vsftp-logs.conf___
  - Logstash의 파이프라인을 정의합니다.
  - 파이프라인은 크게 세가지 섹션(_input, output, filter_)으로 구성됩니다.
  - 자세한 내용은 [공식 문서: Structure of a Config File](https://www.elastic.co/guide/en/logstash/current/configuration-file-structure.html)를 참고하시길 바랍니다.
- /etc/logstash/conf.d/patterns/___vsftp-patterns___
  - Logstash의 _filter_ 섹션의 _grok_ 플러그인에서 사용할 사용자 정의 패턴을 생성합니다.
  - 본 가이드에서는 VSFTP의 로그 파일을 파싱하기위한 패턴들을 설정합니다.
  - 자세한 내용은 [공식 문서: plugins-filters-grok#_custom_patterns](https://www.elastic.co/guide/en/logstash/current/plugins-filters-grok.html#_custom_patterns)을 참고하시길 바랍니다.
- /etc/logstash/conf.d/users/___vsftp-users.yml___
  - Logstash의 _filter_ 섹션의 _translate_ 플러그인에서 사용할 딕셔너리 파일을 생성합니다.
  - 본 가이드에서는 YAML 형식으로 VSFPT의 사용자 정보를 입력합니다.
  - 자세한 내용은 [공식 문서: plugins-filters-translate#_dictionary_path](https://www.elastic.co/guide/en/logstash/current/plugins-filters-translate.html#plugins-filters-translate-dictionary_path)을 참고하시길 바랍니다.

### vsftp-logs.conf

Logstash의 환경 설정은 이벤트 프로세싱 파이프라인을 처리하기 위해 추가하는 플러그인의 타입에 따라 크게 세가지 섹션을 정의합니다.

- input: Logstash가 수신할 이벤트 소스를 설정합니다.
- output: Logstash가 이벤트 데이터를 전달할 대상을 설정합니다.
- filter: 필터링, 파싱 등의 전달 받은 이벤트의 변환 절차를 설정합니다.

자세한 내용은 __공식 문서:__ [Input plugins](https://www.elastic.co/guide/en/logstash/current/input-plugins.html), [Output plugins](https://www.elastic.co/guide/en/logstash/current/output-plugins.html), [Filter plugins](https://www.elastic.co/guide/en/logstash/current/filter-plugins.html) 를 참고하시길 바랍니다. 다음은 주요 설정에 대한 예시 입니다.

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

Logstash의 _filter_ 섹션의 _grok_ 플러그인에서 사용할 사용자 정의 패턴을 생성합니다. _grok_ 플러그인은 기 정의된 정규식 패턴을 연계하고 재정의하여 텍스트를 파싱 후 매핑된 텍스트에 이름을 부여할 수 있습니다. 기본 문법은 `%{SYNTAX:SEMANTIC}` 와 같이 정의됩니다. 예를 들어 `3.44`는 `NUMBER`에, `10.0.0.1`는 `IP`로 매핑하여 각각 이름을 부여하려면, `%{NUMBER:duration} %{IP:client}`와 같이 정의 할 수 있습니다. 다음은 VSFTP 로그 파일에서 `CONNECT` 동작에 대한 패턴 예시입니다.

```shell
VSFTPD_DATETIME %{DAY} %{MONTH} %{SPACE}%{NUMBER} %{TIME} %{YEAR}
VSFTPD_CONNECT \A%{VSFTPD_DATETIME:datetime} \[pid %{NUMBER:vsftpd_pid}] %{WORD:vsftpd_action}: Client "%{IP:vsftpd_client_ip}"
```

전체 예시는 [첨부파일](etc/logstash/conf.d/patterns/vsftp-patterns)을 참고하세요.

###  vsftp-users.yml

Logstash의 _filter_ 섹션의 _translate_ 플러그인에서 사용할 딕셔너리 파일을 생성합니다. _translate_ 플러그인은 외부 파일로 로드한 Key/Value 사전으로부터 값을 검색하거나 변환할 수 있습니다. 본 가이드에서는 VSFTP의 사용자 계정 별 메타 정보를 확인하기 위해 활용되며 이벤트를 수신할 Email과 SMS 번호를 저장합니다. `{FTP_ACCOUNT}: {USER_EMAIL}/{USER_PHONE_NUMBER}`와 같은 형식으로 정의하며 다음은 예시입니다.

```yaml
ftpguest: ftpguest@sample.com/01012345678
```

전체 예시는 [첨부파일](etc/logstash/conf.d/users/vsftp-users.yml)을 참고하세요.

## Next

