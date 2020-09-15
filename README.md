

- Reference

    [An Open Source Load Balancer for OpenShift](https://www.openshift.com/blog/an-open-source-load-balancer-for-openshift)

    [Loadbalancer for Openshift 4.3 on vSphere](https://www.reddit.com/r/openshift/comments/f69bpk/loadbalancer_for_openshift_43_on_vsphere/)

    [HAProxy 를 사용한 레이어 4 로드밸런싱 part 3](https://soulsearcher.github.io/2018/01/29/haproxy-part-3/)

Test 환경이며, HTTP Server 2대가 Round Robin 방식으로 Load Balancing이 되는지 간단히 테스트해보았습니다. 그 외, 간단한 Proxy Server Stats를 구현해보았습니다.

<< Contents >>

# Test 환경

- HAProxy Server

    192.168.100.10/24 - NAT

    192.168.101.10/24 - Host-Only

- Web #1

    192.168.101.11/24 - NAT (HAProxy Server 구축 후 해당 Interface Down)

    192.168.101.11/24 - Host-Only

- Web #2

    192.168.101.12/24 - NAT (HAProxy Server 구축 후 해당 Interface Down)

    192.168.101.12/24 - Host-Only

Web Server 구축은 생략하였습니다.














# HAProxy Server 구축

- Package 설치

```bash
yum -y install haproxy.x86_64

systemctl start haproxy
```

- haproxy.cfg original file 백업

```bash
cp /etc/haproxy/haproxy.cfg ~/haproxy.cfg.bak
```

- haproxy.cfg 파일 수정

    global, defaults, frontend, backend 총 4개의 Section이 있다. 알맞게 수정해준다.

    - global : 전체 영역에 걸쳐 적용되는 기본 설정을 담당
    - defaults : 이후 오는 영역(frontend, backend, listen)에 적용되는 설정
    - frontend : 클라이언트 연결을 받아들이는 소켓에 대한 설정
    - backend : 앞에서 들어온 연결에 할당될 프록시 서버들에 대한 설정
    - listen : frontend와 backend로 사용되는 포트를 한번에 설정하는 영역으로 TCP 프록시에서만 이용

    haproxy.x86_64 0:1.5.18-9.el7

```bash
vi /etc/haproxy/haproxy.cfg

global
    maxconn     20000
    log         /dev/log local0 info
    chroot      /var/lib/haproxy
    pidfile     /var/run/haproxy.pid
    user        haproxy
    group       haproxy
    daemon

    # turn on stats unix socket
    stats socket /var/lib/haproxy/stats

#---------------------------------------------------------------------
# common defaults that all the 'listen' and 'backend' sections will
# use if not designated in their block
#---------------------------------------------------------------------
defaults
    log                     global
    mode                    http
    option                  httplog
    option                  dontlognull
    option http-server-close
    option redispatch
    option forwardfor       except 127.0.0.0/8
    retries                 3
    maxconn                 20000
    timeout http-request    10000ms
    timeout http-keep-alive 10000ms
    timeout check           10000ms
    timeout connect         40000ms
    timeout client          300000ms
    timeout server          300000ms
    timeout queue           50000ms

# Enable HAProxy stats
listen stats
    bind :9000
    stats uri /stats
    stats refresh 10000ms

# Frontend
# 해당 서버의 80번 포트로 오는 요청을 Backend Server로 Proxy해준다.
frontend http_ingress_frontend
    bind :80
    default_backend http_ingress_backend
    mode tcp

# Backend
# Roundrobin
backend http_ingress_backend
    balance roundrobin
    mode tcp
    server      http1      192.168.101.11:80 check
    server      http2      192.168.101.12:80 check
```

- Balance Options

1. roundrobin : 순차적으로 분배

2. static-rr : 서버에 부여된 가중치에 따라서 분배

3. leastconn : 접속수가 가장 적은 서버로 분배

4. source : 운영중인 서버의 가중치를 나눠서 접속자 IP 해싱(hashing)해서 분배

5. uri : 접속하는 URI를 해싱해서 운영중인 서버의 가중치를 나눠서 분배 (URI의 길이 또는 depth로 해싱)

6. url_param : HTTP GET 요청에 대해서 특정 패턴이 있는지 여부 확인 후 조건에 맞는 서버로 분배 (조건 없는 경우 round_robin으로 처리)

7. hdr : HTTP 헤더에서 hdr(<name>)으로 지정된 조건이 있는 경우에 대해서만 분배 (조건없는 경우 round robin 으로 처리)

8. rdp-cookie : TCP 요청에 대한 RDP 쿠키에 따른

- 방화벽 Open & 서비스 시작

```bash
# httpd port open
firewall-cmd --add-service=http --permanent 

# stats port open
firewall-cmd --add-port=9000/tcp --permanent 

# reload
firewall-cmd --reload

# 서비스 시작
systemctl start haproxy.service

systemctl enable haproxy.service 

# SELinux name_bind access
setsebool -P haproxy_connect_any 1
```

# Test

- HAProxy Statistics

    모니터링 및 트러블슈팅 할 때 유용하게 볼 수 있다.

```bash
http://{HAProxy_IP_Address}"9000/stats

192.168.101.10:9000/stats--
```

![Untitled](https://user-images.githubusercontent.com/67780144/93227762-b23e8280-f7af-11ea-8747-f6f87c020129.png)

- Check

    외부 Network에서, 내부 Network에서 192.168.100/101.10 으로 접근 시 모두 Load Balancing이 되는 것을 확인할 수 있음

```bash
# Master에서
[root@master haproxy]# curl 192.168.101.10
client1
[root@master haproxy]# curl 192.168.101.10
client2

[root@master haproxy]# curl 192.168.100.10
client1
[root@master haproxy]# curl 192.168.100.10
client2

# Client1 에서
[root@client1 html]# curl 192.168.100.10
curl: (7) Failed to connect to 192.168.100.10: Network is unreachable

[root@client1 html]# curl 192.168.101.10
client1
[root@client1 html]# curl 192.168.101.10
client2

# Client2 에서
[root@client2 ~]# curl 192.168.100.10
curl: (7) Failed to connect to 192.168.100.10: Network is unreachable
[root@client2 ~]# curl 192.168.101.10
client1
[root@client2 ~]# curl 192.168.101.10
client2
```

Host에서 외부 Network에서도 접근 가능 (web1, web2 외부 네트워크 Interface Down 시킨 상태)

![Untitled 1](https://user-images.githubusercontent.com/67780144/93227774-b4084600-f7af-11ea-9d9a-77bbb4e8f2c6.png)
