# Test keepalived
Lý thuyết đủ rồi, kiểm chứng thực tế bằng tcpdump phát, thử các kịch bản và capture network packets
## Lab
Tạo máy chủ client (192.168.56.100), lb1 (192.168.56.11), lb2 (192.168.56.12)
```
cd keepalived
vagrant up
```
Cấu hình keepalived trên lb1 với VIP 192.168.56.15
Thêm dòng `net.ipv4.ip_nonlocal_bind=1` vào file `/etc/sysctl.conf`. Chạy lệnh
```
sudo sysctl -p
```
Tạo file `/etc/keepalived/keepalived.conf` với nội dung
```
global_defs {
  router_id test1                            # khai báo route_id của keepalived
}

vrrp_script chk_haproxy {
  script "killall -0 haproxy"
  interval 2
  weight 2
}

vrrp_instance VI_1 {
  virtual_router_id 51
  advert_int 1
  priority 100
  state MASTER
  interface enp0s8                            #thông tin tên interface của server, bạn dùng lệnh `ifconfig` để xem và điền cho đúng       
  virtual_ipaddress {
    192.168.56.15 dev enp0s8           #Khai báo Virtual IP cho interface tương ứng
  }

  authentication {
     auth_type PASS
     auth_pass 123456                    #Password này phải khai báo giống nhau giữa các server keepalived
  }

  track_script {
    chk_haproxy
  }
}
```
Start keepalived
```
sudo service keepalived start
```

Capture VRRP packets trên lb2 sẽ thấy các packets broadcast bởi lb1 khẳng định nó là master
```
vagrant@lb2:~$ sudo tcpdump -i enp0s8 vrrp
tcpdump: verbose output suppressed, use -v[v]... for full protocol decode
listening on enp0s8, link-type EN10MB (Ethernet), snapshot length 262144 bytes
07:34:10.658299 IP 192.168.56.11 > vrrp.mcast.net: VRRPv2, Advertisement, vrid 51, prio 102, authtype simple, intvl 1s, length 20
07:34:11.687031 IP 192.168.56.11 > vrrp.mcast.net: VRRPv2, Advertisement, vrid 51, prio 102, authtype simple, intvl 1s, length 20
07:34:12.707566 IP 192.168.56.11 > vrrp.mcast.net: VRRPv2, Advertisement, vrid 51, prio 102, authtype simple, intvl 1s, length 20
07:34:13.721442 IP 192.168.56.11 > vrrp.mcast.net: VRRPv2, Advertisement, vrid 51, prio 102, authtype simple, intvl 1s, length 20
07:34:14.722397 IP 192.168.56.11 > vrrp.mcast.net: VRRPv2, Advertisement, vrid 51, prio 102, authtype simple, intvl 1s, length 20
07:34:15.745256 IP 192.168.56.11 > vrrp.mcast.net: VRRPv2, Advertisement, vrid 51, prio 102, authtype simple, intvl 1s, length 20
07:34:16.807662 IP 192.168.56.11 > vrrp.mcast.net: VRRPv2, Advertisement, vrid 51, prio 102, authtype simple, intvl 1s, length 20
07:34:17.807606 IP 192.168.56.11 > vrrp.mcast.net: VRRPv2, Advertisement, vrid 51, prio 102, authtype simple, intvl 1s, length 20
07:34:18.816160 IP 192.168.56.11 > vrrp.mcast.net: VRRPv2, Advertisement, vrid 51, prio 102, authtype simple, intvl 1s, length 20
07:34:19.830151 IP 192.168.56.11 > vrrp.mcast.net: VRRPv2, Advertisement, vrid 51, prio 102, authtype simple, intvl 1s, length 20
07:34:20.830995 IP 192.168.56.11 > vrrp.mcast.net: VRRPv2, Advertisement, vrid 51, prio 102, authtype simple, intvl 1s, length 20
^C
11 packets captured
11 packets received by filter
0 packets dropped by kernel
vagrant@lb2:~$ sudo tcpdump -ni enp0s8 vrrp
tcpdump: verbose output suppressed, use -v[v]... for full protocol decode
listening on enp0s8, link-type EN10MB (Ethernet), snapshot length 262144 bytes
07:34:24.913385 IP 192.168.56.11 > 224.0.0.18: VRRPv2, Advertisement, vrid 51, prio 102, authtype simple, intvl 1s, length 20
07:34:25.915389 IP 192.168.56.11 > 224.0.0.18: VRRPv2, Advertisement, vrid 51, prio 102, authtype simple, intvl 1s, length 20
07:34:26.917311 IP 192.168.56.11 > 224.0.0.18: VRRPv2, Advertisement, vrid 51, prio 102, authtype simple, intvl 1s, length 20
07:34:27.918365 IP 192.168.56.11 > 224.0.0.18: VRRPv2, Advertisement, vrid 51, prio 102, authtype simple, intvl 1s, length 20
```
Giải thích
| Thành phần                          | Ý nghĩa                                                                                    |
| ----------------------------------- | ------------------------------------------------------------------------------------------ |
| `07:34:10.658299`                   | Thời điểm gói tin được nhận                                                                |
| `IP 192.168.56.11 > vrrp.mcast.net` | Gói tin đi từ IP `192.168.56.11` (lb1) đến địa chỉ multicast `224.0.0.18` (vrrp.mcast.net) |
| `VRRPv2`                            | Phiên bản VRRP đang dùng (v2)                                                              |
| `Advertisement`                     | Đây là gói tin quảng bá vai trò master                                                     |
| `vrid 51`                           | Virtual Router ID: nhóm VRRP bạn đang cấu hình                                             |
| `prio 102`                          | Ưu tiên của máy gửi (lb1). Cao hơn thì sẽ trở thành master                                 |
| `authtype simple`                   | Kiểu xác thực (mặc định: simple, có thể cấu hình khác như AH)                              |
| `intvl 1s`                          | Chu kỳ gửi gói advertisement: 1 giây/lần                                                   |
| `length 20`                         | Độ dài của gói VRRP (20 byte)                                                              |

Cấu hình keepalived trên lb2 nhưng ở state BACKUP
Thêm dòng `net.ipv4.ip_nonlocal_bind=1` vào file `/etc/sysctl.conf`. Chạy lệnh
```
sudo sysctl -p
```
Tạo file `/etc/keepalived/keepalived.conf` với nội dung
```
global_defs {
  router_id test2                            # khai báo route_id của keepalived
}

vrrp_script chk_haproxy {
  script "killall -0 haproxy"
  interval 2
  weight 2
}

vrrp_instance VI_1 {
  virtual_router_id 51
  advert_int 1
  priority 99
  state BACKUP
  interface enp0s8                            #thông tin tên interface của server, bạn dùng lệnh `ifconfig` để xem và điền cho đúng       
  virtual_ipaddress {
    192.168.56.15 dev enp0s8           #Khai báo Virtual IP cho interface tương ứng
  }

  authentication {
     auth_type PASS
     auth_pass 123456                    #Password này phải khai báo giống nhau giữa các server keepalived
  }

  track_script {
    chk_haproxy
  }
}
```
Start keepalived
```
sudo service keepalived start
```
SSH vào máy client và thử capture packets bằng tcpdump
```
vagrant@client:~$ sudo tcpdump -i enp0s8 '(vrrp or arp)'
tcpdump: verbose output suppressed, use -v[v]... for full protocol decode
listening on enp0s8, link-type EN10MB (Ethernet), snapshot length 262144 bytes
07:53:29.454255 IP 192.168.56.11 > vrrp.mcast.net: VRRPv2, Advertisement, vrid 51, prio 102, authtype simple, intvl 1s, length 20
07:53:30.459533 IP 192.168.56.11 > vrrp.mcast.net: VRRPv2, Advertisement, vrid 51, prio 102, authtype simple, intvl 1s, length 20
07:53:31.464049 IP 192.168.56.11 > vrrp.mcast.net: VRRPv2, Advertisement, vrid 51, prio 102, authtype simple, intvl 1s, length 20
07:53:32.465263 IP 192.168.56.11 > vrrp.mcast.net: VRRPv2, Advertisement, vrid 51, prio 102, authtype simple, intvl 1s, length 20
```
Server lb1 đang liên tục broadcast mình là master. Giờ stop haproxy trên lb1. Xem packets capture trên client
```
vagrant@client:~$ sudo tcpdump -i enp0s8 '(vrrp or arp)'
tcpdump: verbose output suppressed, use -v[v]... for full protocol decode
listening on enp0s8, link-type EN10MB (Ethernet), snapshot length 262144 bytes
07:54:38.577692 IP 192.168.56.11 > vrrp.mcast.net: VRRPv2, Advertisement, vrid 51, prio 102, authtype simple, intvl 1s, length 20
07:54:39.580745 IP 192.168.56.11 > vrrp.mcast.net: VRRPv2, Advertisement, vrid 51, prio 102, authtype simple, intvl 1s, length 20
07:54:40.581152 IP 192.168.56.11 > vrrp.mcast.net: VRRPv2, Advertisement, vrid 51, prio 100, authtype simple, intvl 1s, length 20
07:54:41.581123 IP 192.168.56.11 > vrrp.mcast.net: VRRPv2, Advertisement, vrid 51, prio 100, authtype simple, intvl 1s, length 20
07:54:42.582720 IP 192.168.56.11 > vrrp.mcast.net: VRRPv2, Advertisement, vrid 51, prio 100, authtype simple, intvl 1s, length 20
07:54:43.188304 IP 192.168.56.12 > vrrp.mcast.net: VRRPv2, Advertisement, vrid 51, prio 101, authtype simple, intvl 1s, length 20
07:54:43.189484 ARP, Request who-has 192.168.56.15 (Broadcast) tell 192.168.56.15, length 46
07:54:43.189484 ARP, Request who-has 192.168.56.15 (Broadcast) tell 192.168.56.15, length 46
07:54:43.189819 ARP, Request who-has 192.168.56.15 (Broadcast) tell 192.168.56.15, length 46
07:54:43.189820 ARP, Request who-has 192.168.56.15 (Broadcast) tell 192.168.56.15, length 46
07:54:43.189820 ARP, Request who-has 192.168.56.15 (Broadcast) tell 192.168.56.15, length 46
07:54:44.190181 IP 192.168.56.12 > vrrp.mcast.net: VRRPv2, Advertisement, vrid 51, prio 101, authtype simple, intvl 1s, length 20
07:54:45.192427 IP 192.168.56.12 > vrrp.mcast.net: VRRPv2, Advertisement, vrid 51, prio 101, authtype simple, intvl 1s, length 20
07:54:46.192823 IP 192.168.56.12 > vrrp.mcast.net: VRRPv2, Advertisement, vrid 51, prio 101, authtype simple, intvl 1s, length 20
07:54:47.193082 IP 192.168.56.12 > vrrp.mcast.net: VRRPv2, Advertisement, vrid 51, prio 101, authtype simple, intvl 1s, length 20
07:54:48.189682 ARP, Request who-has 192.168.56.15 (Broadcast) tell 192.168.56.15, length 46
07:54:48.189682 ARP, Request who-has 192.168.56.15 (Broadcast) tell 192.168.56.15, length 46
07:54:48.189682 ARP, Request who-has 192.168.56.15 (Broadcast) tell 192.168.56.15, length 46
07:54:48.189682 ARP, Request who-has 192.168.56.15 (Broadcast) tell 192.168.56.15, length 46
07:54:48.189682 ARP, Request who-has 192.168.56.15 (Broadcast) tell 192.168.56.15, length 46
07:54:48.194369 IP 192.168.56.12 > vrrp.mcast.net: VRRPv2, Advertisement, vrid 51, prio 101, authtype simple, intvl 1s, length 20
07:54:49.195545 IP 192.168.56.12 > vrrp.mcast.net: VRRPv2, Advertisement, vrid 51, prio 101, authtype simple, intvl 1s, length 20
07:54:50.195921 IP 192.168.56.12 > vrrp.mcast.net: VRRPv2, Advertisement, vrid 51, prio 101, authtype simple, intvl 1s, length 20
```
Priority của lb1 đang là 102 bị giảm về 100 theo cấu hình trong track_script, priority của lb2 đang là 101 (=99+2) nên sẽ lấy trạng thái MASTER, lấy VIP và broadcast địa chỉ MAC của nó ra sử dụng gói ARP.

Từ đây lb2 sẽ broadcast gói VRRP liên tục thể hiện nó là MASTER và cầm VIP

Tắt keepalived trên server lb2. Capture packets trên client.
```
vagrant@client:~$ sudo tcpdump -i enp0s8 '(vrrp or arp)'
tcpdump: verbose output suppressed, use -v[v]... for full protocol decode
listening on enp0s8, link-type EN10MB (Ethernet), snapshot length 262144 bytes
08:01:20.657591 IP 192.168.56.12 > vrrp.mcast.net: VRRPv2, Advertisement, vrid 51, prio 101, authtype simple, intvl 1s, length 20
08:01:21.657464 IP 192.168.56.12 > vrrp.mcast.net: VRRPv2, Advertisement, vrid 51, prio 101, authtype simple, intvl 1s, length 20
08:01:22.657937 IP 192.168.56.12 > vrrp.mcast.net: VRRPv2, Advertisement, vrid 51, prio 101, authtype simple, intvl 1s, length 20
08:01:23.658205 IP 192.168.56.12 > vrrp.mcast.net: VRRPv2, Advertisement, vrid 51, prio 101, authtype simple, intvl 1s, length 20
08:01:24.660248 IP 192.168.56.12 > vrrp.mcast.net: VRRPv2, Advertisement, vrid 51, prio 101, authtype simple, intvl 1s, length 20
08:01:25.660975 IP 192.168.56.12 > vrrp.mcast.net: VRRPv2, Advertisement, vrid 51, prio 101, authtype simple, intvl 1s, length 20
08:01:26.662907 IP 192.168.56.12 > vrrp.mcast.net: VRRPv2, Advertisement, vrid 51, prio 101, authtype simple, intvl 1s, length 20
08:01:27.663407 IP 192.168.56.12 > vrrp.mcast.net: VRRPv2, Advertisement, vrid 51, prio 101, authtype simple, intvl 1s, length 20
08:01:28.664292 IP 192.168.56.12 > vrrp.mcast.net: VRRPv2, Advertisement, vrid 51, prio 101, authtype simple, intvl 1s, length 20
08:01:29.664403 IP 192.168.56.12 > vrrp.mcast.net: VRRPv2, Advertisement, vrid 51, prio 101, authtype simple, intvl 1s, length 20
08:01:30.664301 IP 192.168.56.12 > vrrp.mcast.net: VRRPv2, Advertisement, vrid 51, prio 101, authtype simple, intvl 1s, length 20
08:01:31.665719 IP 192.168.56.12 > vrrp.mcast.net: VRRPv2, Advertisement, vrid 51, prio 101, authtype simple, intvl 1s, length 20
08:01:32.533099 IP 192.168.56.12 > vrrp.mcast.net: VRRPv2, Advertisement, vrid 51, prio 0, authtype simple, intvl 1s, length 20
08:01:33.179462 IP 192.168.56.11 > vrrp.mcast.net: VRRPv2, Advertisement, vrid 51, prio 100, authtype simple, intvl 1s, length 20
08:01:33.181218 ARP, Request who-has 192.168.56.15 (Broadcast) tell 192.168.56.15, length 46
08:01:33.181219 ARP, Request who-has 192.168.56.15 (Broadcast) tell 192.168.56.15, length 46
08:01:33.181219 ARP, Request who-has 192.168.56.15 (Broadcast) tell 192.168.56.15, length 46
08:01:33.181219 ARP, Request who-has 192.168.56.15 (Broadcast) tell 192.168.56.15, length 46
08:01:33.181219 ARP, Request who-has 192.168.56.15 (Broadcast) tell 192.168.56.15, length 46
08:01:34.216829 IP 192.168.56.11 > vrrp.mcast.net: VRRPv2, Advertisement, vrid 51, prio 100, authtype simple, intvl 1s, length 20
08:01:35.316983 IP 192.168.56.11 > vrrp.mcast.net: VRRPv2, Advertisement, vrid 51, prio 100, authtype simple, intvl 1s, length 20
08:01:36.426458 IP 192.168.56.11 > vrrp.mcast.net: VRRPv2, Advertisement, vrid 51, prio 100, authtype simple, intvl 1s, length 20
08:01:37.449609 IP 192.168.56.11 > vrrp.mcast.net: VRRPv2, Advertisement, vrid 51, prio 100, authtype simple, intvl 1s, length 20
08:01:38.185709 ARP, Request who-has 192.168.56.15 (Broadcast) tell 192.168.56.15, length 46
08:01:38.185710 ARP, Request who-has 192.168.56.15 (Broadcast) tell 192.168.56.15, length 46
08:01:38.185710 ARP, Request who-has 192.168.56.15 (Broadcast) tell 192.168.56.15, length 46
08:01:38.185710 ARP, Request who-has 192.168.56.15 (Broadcast) tell 192.168.56.15, length 46
08:01:38.185710 ARP, Request who-has 192.168.56.15 (Broadcast) tell 192.168.56.15, length 46
08:01:38.468402 IP 192.168.56.11 > vrrp.mcast.net: VRRPv2, Advertisement, vrid 51, prio 100, authtype simple, intvl 1s, length 20
08:01:39.519780 IP 192.168.56.11 > vrrp.mcast.net: VRRPv2, Advertisement, vrid 51, prio 100, authtype simple, intvl 1s, length 20
08:01:40.575778 IP 192.168.56.11 > vrrp.mcast.net: VRRPv2, Advertisement, vrid 51, prio 100, authtype simple, intvl 1s, length 20
```
Bật lại keepalived trên server lb2
```
08:02:58.548058 IP 192.168.56.11 > vrrp.mcast.net: VRRPv2, Advertisement, vrid 51, prio 100, authtype simple, intvl 1s, length 20
08:02:59.548652 IP 192.168.56.11 > vrrp.mcast.net: VRRPv2, Advertisement, vrid 51, prio 100, authtype simple, intvl 1s, length 20
08:03:00.551552 IP 192.168.56.11 > vrrp.mcast.net: VRRPv2, Advertisement, vrid 51, prio 100, authtype simple, intvl 1s, length 20
08:03:01.493684 IP 192.168.56.12 > vrrp.mcast.net: VRRPv2, Advertisement, vrid 51, prio 101, authtype simple, intvl 1s, length 20
08:03:01.495262 ARP, Request who-has 192.168.56.15 (Broadcast) tell 192.168.56.15, length 46
08:03:01.495262 ARP, Request who-has 192.168.56.15 (Broadcast) tell 192.168.56.15, length 46
08:03:01.495262 ARP, Request who-has 192.168.56.15 (Broadcast) tell 192.168.56.15, length 46
08:03:01.495262 ARP, Request who-has 192.168.56.15 (Broadcast) tell 192.168.56.15, length 46
08:03:01.495262 ARP, Request who-has 192.168.56.15 (Broadcast) tell 192.168.56.15, length 46
08:03:02.496733 IP 192.168.56.12 > vrrp.mcast.net: VRRPv2, Advertisement, vrid 51, prio 101, authtype simple, intvl 1s, length 20
08:03:03.497340 IP 192.168.56.12 > vrrp.mcast.net: VRRPv2, Advertisement, vrid 51, prio 101, authtype simple, intvl 1s, length 20
08:03:04.497240 IP 192.168.56.12 > vrrp.mcast.net: VRRPv2, Advertisement, vrid 51, prio 101, authtype simple, intvl 1s, length 20
08:03:05.498145 IP 192.168.56.12 > vrrp.mcast.net: VRRPv2, Advertisement, vrid 51, prio 101, authtype simple, intvl 1s, length 20
08:03:06.496477 ARP, Request who-has 192.168.56.15 (Broadcast) tell 192.168.56.15, length 46
08:03:06.496478 ARP, Request who-has 192.168.56.15 (Broadcast) tell 192.168.56.15, length 46
08:03:06.496478 ARP, Request who-has 192.168.56.15 (Broadcast) tell 192.168.56.15, length 46
08:03:06.496478 ARP, Request who-has 192.168.56.15 (Broadcast) tell 192.168.56.15, length 46
08:03:06.496478 ARP, Request who-has 192.168.56.15 (Broadcast) tell 192.168.56.15, length 46
08:03:06.498590 IP 192.168.56.12 > vrrp.mcast.net: VRRPv2, Advertisement, vrid 51, prio 101, authtype simple, intvl 1s, length 20
08:03:07.501253 IP 192.168.56.12 > vrrp.mcast.net: VRRPv2, Advertisement, vrid 51, prio 101, authtype simple, intvl 1s, length 20
08:03:08.500892 IP 192.168.56.12 > vrrp.mcast.net: VRRPv2, Advertisement, vrid 51, prio 101, authtype simple, intvl 1s, length 20
```

## Lưu ý về multicast và unicast
VRRP chỉ hoạt động trong cùng một subnet, không qua được router mặc định do nó gửi vào IP link local 224.0.0.0/24. Khi cấu hình keepalived trên nhiều dải mạng, dùng mode unicast.

## Tham khảo
https://viblo.asia/p/trien-khai-dich-vu-high-available-voi-keepalived-haproxy-tren-server-ubuntu-jOxKdqWlzdm