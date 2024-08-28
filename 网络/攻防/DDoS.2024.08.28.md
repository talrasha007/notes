# ksoftirqd占用100%，服务几乎不可用
是把在运行的网络服务都停掉，也没任何好转的那种。

## 几个没用的尝试
```sh
# 本想简单让多核来处理的
ethtool -L eth0 combined 4

# 但是
ethtool -l eth0
# 由于这个虚拟机的网卡驱动限制，只能单核处理
# Pre-set maximums:
# RX:             n/a
# TX:             n/a
# Other:          n/a
# Combined:       1

sudo apt install irqbalance
sudo systemctl enable irqbalance
sudo systemctl start irqbalance
# irqbalance也不能改变这个结果
```

## 追查问题
```sh
# 看下是谁在狂中断
cat /proc/interrupts
# 隔几秒看看，对比一下，发现是44号技师，virtio3-input.0，嗯，确实是网络引起的，一秒大几万次

# 那，接下来，看下到底是怎么回事
tcpdump -i eth0
# 07:59:43.626881 IP 119.188.70.211.https > myip.ssh: Flags [S.], seq 367914457, ack 1078716331, win 65535, options [mss 1416,nop,nop,sackOK,nop,wscale 10], length 0
# 07:59:43.626882 IP myip.ssh > 119.188.70.211.https: Flags [R], seq 1078716331, win 0, length 0
# 07:59:43.626885 IP 112.30.128.121.https > myip.ssh: Flags [.], ack 1490502707, win 55, length 0
# 充斥了这样的从远端的443端口到我机器的22端口的非正常数据包，通过伪造SYN_ACK，空数据包，引起系统中断来处理，回个RST，占用系统中断资源。
# 攻击者把源端口用443，目标端口22，两边都是well-known的服务端口，这样绕开了这边的防火墙，将流量打到了机器上。
# 这种攻击，修改sshd的端口并不能解决任何问题，它根本就没走到服务，完全没有TCP连接，都是占用了内核的中断资源。

# 防火墙设置
vim /etc/nftables.conf
# 在chain input里加上：tcp sport 443 tcp dport 22 drop，这样这种包就直接被抛弃了，到此，服务就恢复了。
top
# 这下子ksoftirqd占用就不再是100%了，但是仍然动不动就是70-80%的，还是让人很不放心。

# 以下是搜来的几个，应该对这个场景用处不大
sysctl -w net.ipv4.tcp_syn_retries=1
sysctl -w net.ipv4.tcp_synack_retries=1
sysctl -w net.ipv4.conf.all.forwarding=0
sysctl -w net.ipv4.conf.all.send_redirects=0
sysctl -w net.ipv4.conf.all.accept_redirects=0

# 然后是GPT的几个建议，启用RPS和RFS
echo f > /sys/class/net/eth0/queues/rx-0/rps_cpus
echo 32768 > /proc/sys/net/core/rps_sock_flow_entries
echo 4096 > /sys/class/net/eth0/queues/rx-0/rps_flow_cnt
```
这一通操作之后，ksoftirqd占用就很低了，偶尔到50%左右，大部分时间都在30%以下，至此，基本就不会有挂掉的担忧了。当然了，完美解决还是跟服务商联系，让他们更新防火墙配置。