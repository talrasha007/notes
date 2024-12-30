# 绑定ipv6前缀下的任意地址
参考这这个[回答](https://serverfault.com/questions/590038/adding-a-whole-ipv6-64-block-to-an-network-interface-on-debian)

把前缀加到lo上，然后把lo上前缀的路由给加一下，开启ip_nonlocal_bind，万事大吉。
```sh
sudo ip addr add 2001:41d0:2:ad64::/64 dev lo
sudo ip route add local 2001:41d0:2:ad64::/64 dev eth0
sudo sysctl -w net.ipv6.ip_nonlocal_bind=1
```