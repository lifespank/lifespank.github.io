---
title: "IBM World Community Grid에 대해"
date: 2021-05-16 14:44:00 +0900
categories: 여러가지
tags: 라즈베리파이 우한폐렴 코로나
---
[World Community Grid](https://join.worldcommunitygrid.org/?recruiterId=1144103)

IBM World Community Grid는 컴퓨터 또는 스마트 기기를 가진 누구나 본인의 남는 컴퓨팅 파워로 환경과 기아 문제를 해결하는 최신 연구의 데이터를 처리해 지원하는 프로젝트다. 이것에 참여하는 법을 소개해본다.

![picture 2](https://i.imgur.com/8fmzuOP.png)  

![picture 3](https://i.imgur.com/jk1LCok.png)  

먼저 회원가입을 하고 프로젝트를 선택한다. COVID-19, 결핵, 암 등등 여러 프로젝트가 있다.

![picture 4](https://i.imgur.com/Nxv9PzX.png)  

본인의 운영체제에 맞춰 클라이언트를 설치한다. 나는 라즈베리파이 4에 우분투를 갖고 있으니 우분투용을 받는다.

1. 필요한 패키지 설치

    `sudo apt install boinc-client boinc-manager`
2. BOINC 클라이언트가 부팅 시 자동으로 동작하게 설정

    `sudo systemctl enable boinc-client`
3. BOINC 클라이언트 시작

    `sudo systemctl start boinc-client`
4. 클라이언트 파일에 그룹 접근 권한 추가

    `sudo chmod g+r /var/lib/boinc-client/gui_rpc_auth.cfg`
5. 본인 리눅스 계정을 BOINIC 그룹에 추가

    `sudo usermod -a -G boinc $USER(본인 리눅스 계정)`
6. 터미널이 새로운 그룹의 previlege를 가져오도록 설정

    `exec su $USER(본인 리눅스 계정)`
7. 같은 터미널 내에서, BOINC 클라이언트 실행

    `boincmgr -d /var/lib/boinc-client`

여기서 헤들리스 서버라면 에러가 떠서 boinctui라는 것을 설치해야 한다. CLI 환경에서 boinc 클라이언트를 관리할 수 있다.

```
sudo apt install boinctui screen
screen boinctui
```
boinctui 설치 후 실행하기 전 config 파일을 편집해준다.

/etc/boinc-client/cc_config.xml을 백업한 후, 내용에 \<options\>를 추가해주고, 재부팅한다. 이것을 수정하지 않고 실행하면 aarch64는 World Community Grid에서 지원하지 않는 플랫폼이라고 뜨면서 동작하지 않는다.

```xml
<!--
This is a minimal configuration file cc_config.xml of the BOINC core client.
For a complete list of all available options and logging flags and their
meaning see: https://boinc.berkeley.edu/wiki/client_configuration
-->
<cc_config>
  <log_flags>
    <task>1</task>
    <file_xfer>1</file_xfer>
    <sched_ops>1</sched_ops>
  </log_flags>
  <options>
          <alt_platform>arm-unknown-linux-gnueabihf</alt_platform>
  </options>
</cc_config>
```

![picture 5](https://i.imgur.com/Q24mckm.png)  

Add Project에서 World Community Grid를 선택하고 아까 가입했던 이메일과 비밀번호를 입력하면 태스크가 할당된다.

![picture 6](https://i.imgur.com/ryB12Yc.png)

COVID-19 관련 태스크가 실행되고 있다.

세상이 어지러운데 남는 컴퓨터가 있다면 이런 것을 통해 좀 더 착하게 살아보는 것은 어떨까 싶다.