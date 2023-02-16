# udm-skbtv
UDM(Unifi Dream Machine Pro)에 SKB IPTV를 직렬하기 위한 튜토리얼입니다.

### Table of Contents

1. [들어가기](#들어가기)
2. [SKB를 위한 별도의 VLAN 네트워크 추가](#SKB를-위한-별도의-VLAN-네트워크-추가)
3. [UDM 포트 맵핑하기](#UDM-포트-맵핑하기)
4. [방화벽 설정 추가하기](#방화벽-설정-추가하기)
5. [igmpproxy-설치-및-설정](#igmpproxy-설치-및-설정)
6. [igmproxy-자동실행-시키기](#igmproxy-자동실행-시키기)
7. [Credits](#credits)

### 들어가기

UDM은 기본적으로 IGMP를 지원하지 않는것인지,, 모르겠지만 UDM을 경유해서 셋톱박스에 연결을 하면 연결이 되지 않습니다.

본 튜토리얼은 

UniFi OS 2.4.27 에 기반하여 작성했습니다.

#

### SKB를 위한 별도의 VLAN 네트워크 추가

#

UDM에 기본적으로 구성된 네트워크 이외의 별도의 SKB가 접근가능한 네트워크를 추가합니다.

192.168.1.xx 대역이 아닌 192.168.33.xx 가 필요합니다.


1. UDM 네트워크 Settings -> Networks -> Create New Network를 눌러 아래의 내용으로 새로운 네트워크를 생성합니다.
    * Network Name : BTV (편한대로 설정하세요)
    * Auto-Scale Network 비활성화
    * Host Address : 192.168.33.1
      
      이렇게 입력하면 하단에
      * Gateway IP : 192.168.33.1
      * Broadcast IP : 192.168.33.255
      * USABLE IPs : 249
      * IP Range : 192.168.33.6 - 192.168.33.254
      * Subnet Mask : 255.255.255.0
      
      으로 자동으로 설정됩니다.      
    * Advanced  : Manual
    * *VLAN ID* : 33
    * IGMP Snooping : Enable
    * Multicast DNS : Enable
    
    나머지 설정은 기본값으로 두시면 됩니다.
    
2. *Add Network* 를 눌러 저장합니다.

### UDM 포트 맵핑하기

#

IPTV가 연결된 포트에 위에서 생성한 네트워크를 이용하도록 설정합니다.

1. UNIFI DEVICES -> UDM 선택 -> Ports -> Port Manager 선택

2. IPTV가 연결된 다운링크 포트를 선택합니다. (저는 8번에 다운링크가 있고, 10번 SFP 포트에 업링크가 연결돼 있습니다.)
    * Port Profile : BTV
    * Apply Changes 눌러 저장하기
    
![PORT Manager](https://github.com/myriky/udm-skbtv/blob/78d4a554e47ff1b315ed5baf512a6c1b590d96c4/images/port.png)


### 방화벽 설정 추가하기

#

SKB 모뎀에서 들어오는 IPTV 관련된 트래픽을 앞서 설정한 BTV 네트워크로 바이패스 하도록 방화벽 설정을 추가합니다.
    
1. UDM 네트워크 Settings -> Firewall & Security 페이지로 이동합니다.
2. Firewall Rules 섹션에서 Create New Rule 버튼을 누릅니다.
  * Type : Internet In
  * Description : Allow IPTV Multicast
  * Rule Applied : Before Predefined Rules
  * Action : Accept
  * IPv4 Protocol : TCP and UDP
  * Source
    * Source Type : Port/IP Group
    * IPv4 Address Group
    * Port Group : Any
      * Create New Port/IP Group 를 눌러 새로운 IP Group을 생성합니다
        * Profile Name : BTV
        * IPv4 Address/Subnet : **192.168.0.0/16**
        * Create New Port/IP Group 을 눌러 입력내용을 저장합니다.
      * 방금 추가한 IGMP 를 선택합니다.
    * IPv4 Address Group : BTV
  * Destination
    * Source Type : Port/IP Group
    * IPv4 Address Group
      * Create New Port/IP Group 를 눌러 새로운 IP Group을 생성합니다
        * Profile Name : IGMP
        * IPv4 Address/Subnet : **224.0.0.0/4**
        * Create New Port/IP Group 을 눌러 입력내용을 저장합니다.
      * 방금 추가한 IGMP 를 선택합니다.
    * IPv4 Address Group : IGMP
    * Port Group : Any
  * Advanced : Auto
3. Apply Changes 눌러 입력한 방화벽 설정 추가
4. 같은 방식으로 방화벽 규칙 추가
  * Type : Internet Local
  * Description : Allow IGMP Traffic
  * Rule Applied : Before Predefined Rules
  * Action : Accept
  * IPv4 Protocol : **IGMP**
  * Match all protocols except for this : 체크안함
  * Source
    * Source Type : Port/IP Group
    * IPv4 Address Group : Any
    * Port Group : Any
  * Destination
    * Source Type : Port/IP Group
    * IPv4 Address Group : Any
    * Port Group : Any
  * Advanced
    * Manual로 설정
    * States
      * Match State New 활성화
      * Match State Established 활성화
      * Match State Invalid 활성화
      * Match State Related 활성화  
5. Apply Changes 눌러 입력한 방화벽 설정 추가


### igmpproxy 설치 및 설정

#
UDM에 직접 접속해서 igmpproxy를 설치하고, 설정파일을 변경하는 방법을 안내해드립니다.

1. UDM 게이트웨이 SSH 로 접속합니다. 기본적으론 192.168.1.1 을 사용하지만, 각자의 설정대로 바꿔 접속하세요.

    ```sh
    ssh root@192.168.1.1
    ```

* SSH 접속이 불가능하다면 UniFi OS 설정을 확인하세요. 
  1. Unifi OS -> Settings -> System -> SSH 활성화 및 비밀번호 설정 

2. igmpproxy 파일과 설정파일을 다운받습니다.

    ```sh
    mkdir /mnt/data/igmpproxy
    cd /mnt/data/igmpproxy
    curl -Lo igmpproxy https://raw.githubusercontent.com/peacey/udm-telus/main/igmpproxy
    curl -Lo igmpproxy.conf https://raw.githubusercontent.com/peacey/udm-telus/main/igmpproxy.conf
	chmod +x igmpproxy
    ```
    * 참고사항. igmpproxy 바이너리는 아래 패키지 내용을 추출해서 작성됐습니다. 바이너리를 신뢰하지 못하신다면 아래 링크의 소스를 직접 빌드하실 수 있습니다.
    * Note the igmpproxy binary included in this repository was extracted directly from the [debian arm64 package](https://packages.debian.org/sid/igmpproxy). If you do not trust the binary on this github, you can download it and extract it manually from the official debian package instead.

3. 다운받은 igmpproxy.conf 파일을 수정합니다.

    기본적으로 WAN2 SFP+ 를 통해 네트워크-인이 되고,
    VLAN ID 33의 네트워크-아웃 되는 설정입니다.

    본인의 UDM 상황에 맞춰서 설정값을 변경하시면 됩니다.

    네트워크 장비의 eth 번호를 정확히 모르겠다면 
    ```sh
    ifconfig
    ```
    명령어를 통해서 찾으시면 됩니다.

    *vim 사용방법은 설명하지 않겠습니다.*

    ```sh
    vi igmpproxy.conf
    ```
    명령어를 통해 igmpproxy.conf 파일의 내용을 수정하세요

    ```
    quickleave

    # upstream = modem interface
    phyint eth9 upstream ratelimit 0 threshold 1
            altnet 0.0.0.0/0;

    # lan interface of iptv device
    phyint br33 downstream ratelimit 0 threshold 1
            altnet 0.0.0.0/0;

    # disable all unused interfaces
    phyint lo disabled
    phyint eth9 disabled
    ```

    igmppxory.conf의 기본 설정입니다. eth9 / br33 이 돼있는 부분을 수정하시면 됩니다.



4. 수정한 값으로 igmpprxy가 제대로 동작하는지 확인하기


    ```sh
    ./igmpproxy -ndv ./igmpproxy.conf
    ```
  
    eth 설정이 잘못됐다면 아래와 같은 오류가 발생합니다.
    
    ```
    There must be at least 1 Vif as upstream.
    ```
    br 설정이 잘못됐다면 실행해도 아무런 메시지가 출력되지 않습니다.

    ```
    $ ./igmpproxy -ndv ./igmpproxy.conf
    adding VIF, Ix 0 Fl 0x0 IP 0xd4dd8add eth9, Threshold: 1, Ratelimit: 0
    adding VIF, Ix 1 Fl 0x0 IP 0x0121a8c0 br33, Threshold: 1, Ratelimit: 0
    Joining group 224.0.0.2 on interface br33
    Joining group 224.0.0.22 on interface br33
    RECV Membership query   from 192.168.33.1    to 224.0.0.1
    RECV V2 member report   from 192.168.33.1    to 224.0.0.2
    The IGMP message was from myself. Ignoring.
    ```

    제대로 작동한다면 위와 같은 메시지들이 지속적으로 표시 됩니다.

6. TV를 켜봅시다.
7. 만약 정상적으로 IPTV가 송출된다면 제대로 설정이 됐습니다.

    ctrl + c를 눌러 igmpproxy 를 종료합니다.

    ```sh
    ./igmpproxy ./igmpproxy.conf
    ```
    
    위 명령어를 동작하면 UDM이 작동하는 동안 계속 igmpproxy가 동작하게됩니다.
  
9. UDM을 껐다 켤때마다 수동으로 igmpproxy를 구동해줘야 하는데, 다음 세션에서 자동으로 실행하는 방법에 대해 설명드리겠습니다. [read the next section](#igmpproxy-자동실행-시키기).
 

### igmproxy 자동실행 시키기

#

1. UDM Boot Script 설치
  
    1. UDM Boot Script를 통해 igmpproxy를 자동 구동하게 설정합니다. [here](https://github.com/boostchicken/udm-utilities/blob/master/on-boot-script/README.md)
    ```sh
    curl -fsL "https://raw.githubusercontent.com/unifi-utilities/unifios-utilities/HEAD/on-boot-script/remote_install.sh" | /bin/sh
    ```
    2. 위 명령어를 실행하면 자동으로 UDM Boot Script 가 설치되고, /data/on_boot.d 경로에 원하는 스크립트를 작성하면 됩니다.

2. igmpproxy 부트 스크립트 설치

    ```sh
    cd /data/on_boot.d
    curl -Lo 99-run-igmpproxy.sh https://raw.githubusercontent.com/peacey/udm-telus/main/run-igmpproxy.sh
    chmod +x 99-run-igmpproxy.sh
    ```
    
3. 이제 UDM이 재시작 될때마다 igmpproxy가 실행됩니다.
  
  만약 igmpproxy가 제대로 동작하는지 확인하고 싶다면 아래의 명령어를 입력하세요
  
  ```sh
  # ps aux | grep igmp
  root     25779  0.0  0.0   1772    88 pts/0    S    16:51   0:00 /mnt/data/igmpproxy/igmpproxy /mnt/data/igmpproxy/igmpproxy.conf
  ```

  이렇게 나온다면 정상적으로 igmp프록시가 동작하는 것입니다.


### Credits

#

본 UDM-SKB 튜토리얼은 아래의 페이지에서 도움을 받았습니다.

[udm-telus](https://github.com/peacey/udm-telus)

[unifios-utilities](https://github.com/unifi-utilities/unifios-utilities)

[unifi 에서 애플티비 skb btv 보기 @ 클리앙 올드보이님](https://www.clien.net/service/board/cm_nas/16802767?combine=true&q=udm&p=0&sort=recency&boardCd=&isBoard=false)

Copyrights:
* igmpproxy: Copyright (C) 2005 Johnny Egeland
