# haproxy-test
Thử nghiệm và benchmark cơ chế forward TCP connection của HAProxy

## Cài đặt môi trường
1. Cài đặt vagrant theo [hướng dẫn](https://developer.hashicorp.com/vagrant/docs/installation)
2. Chạy lệnh để tạo 3 máy ảo trên vagrant:
   ```bash
   vagrant up
   ```
Sau khi chạy xong sẽ tạo ra 3 VMs cấu hình 1 vCPU, 512 MB RAM với topo như sau:

![topo](docs/images/topo.png)

- server: listen trên port 5000 do chạy lệnh này `nohup ncat -lk 5000 --keep-open --exec "/bin/cat" &`
- proxy: cài HAProxy, listen trên port 5000 và forward về server
```
frontend ft_tcp
    bind *:5000
    default_backend be_tcp

backend be_tcp
    server srv1 192.168.56.10:5000
EOF
```

## Test 1
HAProxy viết trong [document](https://docs.haproxy.org/3.1/intro.html#3.2:~:text=HAProxy%20is%20%3A%0A%0A%20%20%2D-,a%20TCP%20proxy,-%3A%20it%20can%20accept)
```
HAProxy is :

  - a TCP proxy : it can accept a TCP connection from a listening socket,
    connect to a server and attach these sockets together allowing traffic to
    flow in both directions; IPv4, IPv6 and even UNIX sockets are supported on
    either side, so this can provide an easy way to translate addresses between
    different families.
```
Kiểm chứng xem thử xem mỗi kết nối tới HAProxy có phải tạo ra 2 sockets hay không?

1. SSH vào máy `client` và kết nối vào server (qua HAProxy):
   ```bash
   vagrant ssh client
   ncat 192.168.56.11 5000
   ```
2. List non-listening sockets (e.g. TCP/UNIX/UDP) that have established connection trên máy `proxy`:
   ```bash
   ss | grep ":5000"

   tcp   ESTAB  0      0                     192.168.56.11:5000      192.168.56.12:34512
   tcp   ESTAB  0      0                     192.168.56.11:32898     192.168.56.10:5000
   ```

HAProxy tạo 2 sockets cho mỗi kết nối mà nó nhận được.

## Test 2

1. SSH vào máy `client`:
   ```bash
   vagrant ssh client
   ```
2. Benchmark HAProxy (qua proxy):
   ```bash
   ab -n 1000 -c 50 -k http://192.168.56.11:5000/
   ```

---

## Mở rộng

Muốn benchmark iptables thay cho HAProxy?
- Vào máy `proxy`:
  ```bash
  vagrant ssh proxy
  ```
- Tắt HAProxy:
  ```bash
  systemctl stop haproxy
  ```
- Cấu hình iptables:
  ```bash
  iptables -t nat -A PREROUTING -p tcp --dport 5000 -j DNAT --to-destination 192.168.56.10:5000
  iptables -t nat -A POSTROUTING -j MASQUERADE
  ```

Rồi benchmark lại từ `client` như cũ.

---