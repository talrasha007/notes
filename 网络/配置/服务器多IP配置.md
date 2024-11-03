# 服务器多IP配置
一个服务器在加了多网卡，多IP之后，会遇到一些IP不能被连接或者连接慢的问题，这是在多IP环境下，需要对路由做比较好的管理，最常见的情况就是客户端否则连IP2，但是IP2的回包会从默认路由的网关给他回回去，就非常不符合预期。

## 向外访问用默认路由，但客户端连接其它IP时，需要用此IP的路由把包回回去
我这边服务器的需求还算简单，只需要编辑一下/etc/network/if-up.d/custom_routes。
```sh
#!/bin/bash

ip route add [IP2]/[mask] dev eth1 src [IP2] table 101
ip route add default via [IP2 Gateway] dev eth1 table 101

ip rule add from [IP2] table 101
```
再
```sh
chmod +x /etc/network/if-up.d/custom_routes
/etc/network/if-up.d/custom_routes
```
这样就可以用正确的方式回包了，重启也有效。