# 绑定ipv6前缀下的任意地址
参考这这个[回答](https://serverfault.com/questions/590038/adding-a-whole-ipv6-64-block-to-an-network-interface-on-debian)，以及[这个](https://linux.do/t/topic/202151)

分为两类情况
### 如果有单独分配给服务器的路由前缀，则：
```sh
sudo ip route add local 2001:41d0:2::/48 dev lo
sudo sysctl -w net.ipv6.ip_nonlocal_bind=1
```

### 如果没有分配路由前缀，但服务器可以使用前缀里的任意地址，则
```sh
sudo sysctl -w net.ipv6.ip_nonlocal_bind=1
ip route add local 2406:8dc0:6008:dddd::/64 dev eth0
```
此外，再安装ndppd,编辑/etc/ndppd.conf

```conf
route-ttl 30000

proxy eth0 {
    router no
    timeout 500
    ttl 30000
    rule 2406:8dc0:6008:dddd::/64 {
        static
    }
}
```
在面临大量地址压力的情况下，也可以使用ndpresponder来处理nd proxy，使用docker-compose启动如下：

```yaml
services:
  ndpresponder:
    image: yoursunny/ndpresponder
    container_name: ndpresponder
    network_mode: host
    cap_add:
      - NET_RAW
      - NET_ADMIN
    environment:
      - NDPRESPONDER_LOG=INFO
    command:
      - "-i"
      - "eth0"
      - "-n"
      - "2406:8dc0:6008:dddd::/64"
    restart: always
    logging:
      driver: "json-file"
      options:
        max-size: "10m"
        max-file: 8
```