---
date: 2025-09-26
categories:
  - Computer Science
  - Network
---

# 멀티캐스트 DNS
**멀티캐스트 DNS**는 중앙 DNS 서버 없이 로컬 네트워크 환경에서 호스트 이름을 IP 주소로 확인하는 프로토콜이다.

일반적인 DNS 서버는 FQDN 을 IP 주소로 변환하지만, 멀티캐스트 DNS는 네트워크 안에서 호스트 이름을 IP 주소로 변환한다.

IETF RFC 6762 로 정의된 기술 규격으로, 별도의 설정 없이 네트워크 장치 간에 호스트 이름을 인식할 수 있다.

기본적으로 로컬 Link 전용 도메인인 **.local** 로 끝나는 호스트 이름만 확인된다.

## 동작 원리
### 1. 질의 전송 (Query)

IP 주소를 확인하려는 장치(클라이언트)는 원하는 호스트 이름(예: printer.local)에 대한 mDNS 질의 패킷을 보낸다.

이 질의는 로컬 서브넷 내에서 mDNS 가 활성화된 모든 장치가 수신한다.

### 2. 응답 (Response)

질의를 수신한 장치들은 질의된 호스트 이름을 확인한다.

해당 이름을 소유한 장치는 자신의 IP 주소를 포함하는 mDNS 응답 패킷을 작성하고,

다시 멀티캐스트로 전송되어 서브넷 내의 모든 장치에게 자신의 이름-IP 주소 매핑 정보를 알린다.

### 3. 캐싱 (Caching)

응답 패킷을 수신한 모든 mDNS 지원 장치들은 해당 이름-IP 매핑 정보를 자체 mDNS 캐시에 저장한다.

이후 같은 호스트 이름에 대한 질의가 발생하면 캐시된 정보를 사용하여 네트워크 트래픽을 최소화한다.

## mDNS 설정하기

```toml
# /etc/systemd/resolved.conf
[Resolve]
MulticastDNS=yes
```

리눅스에서 DNS resolution 기능은 `systemd-resolved` 서비스를 통해 제공된다.

설정 파일은 보통 `/etc/systemd/resolved.conf` 경로에 위치하는데, 위와 같이 MulticastDNS 를 활성화해야 한다.

```shell
$ sudo systemctl restart systemd-resolved.service
```

설정 파일을 저장했다면, 서비스를 재시작하여 변경사항을 적용하자.

```shell
$ resolvectl mdns
Global: yes
Link 2 (ens1): no
```

만일 위와 같이 mDNS 가 활성화되지 않은 링크가 존재한다면, `resolvectl mdns` 명령을 사용해서 활성화해야 한다.

```shell
$ sudo resolvectl mdns ens1 yes

$ resolvectl mdns
Global: yes
Link 2 (ens1): yes
```

이제 같은 네트워크에 있는 다른 장치들도 같은 방법으로 mDNS 를 설정하면 된다.

```shell
$ ping test-node.local
PING test-node.local (192.168.1.XX) 56(84) bytes of data.
64 bytes from test-node.local (192.168.1.XX): icmp_seq=1 ttl=64 time=0.763 ms
64 bytes from test-node.local (192.168.1.XX): icmp_seq=2 ttl=64 time=0.765 ms
64 bytes from test-node.local (192.168.1.XX): icmp_seq=3 ttl=64 time=0.687 ms
^C
--- test-node.local ping statistics ---
3 packets transmitted, 3 received, 0% packet loss, time 2016ms
rtt min/avg/max/mdev = 0.687/0.738/0.765/0.036 ms
```

다른 장치(컴퓨터)의 호스트 이름이 인식되는지 확인해보자. 위와 같이 ping 명령으로 호스트가 인식되면 성공이다.
