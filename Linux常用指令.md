# Linux常用指令
## 查看被占用的端口
`sudo lsof -i:8442`
## 按照进程名称终结进程
`sudo kill -9 $(pidof xxxxx)`
## Ubuntu输入突然变大
```
sudo rm -rf ~/.cache/ibus/libpinyin/
ibus-daemon -r -d -x
ibus restart
```

## 配置网络时延、丢包、带宽等
1. 查看网络流量管理
`tc qdisc show`
2. 时延
`sudo tc qdisc add dev 网卡名称 root netem delay 时延数值`
如果想要删除之前设置的时延
`sudo tc qdisc del dev eth0 root netem delay 15ms`
也可以直接更改
`sudo tc qdisc change dev eth0 root netem delay 30ms`
3. 丢包
`sudo tc qdisc add dev 网卡名称 root netem loss 丢包数值 成功率`
`sudo tc qdisc add dev eth0 root netem loss 0.1%`
`sudo tc qdisc add dev eth0 root netem loss 0.1% 30%`
4. 带宽
`sudo tc qdisc add dev 网卡名称 root netem rate 带宽数值`
`sudo tc qdisc add dev eth0 root netem rate 10Gbps`
5. 模拟延迟波动
该命令将 eth0 网卡的传输设置为延迟 100ms ± 10ms (90 ~ 110 ms 之间的任意值)发送。 还可以更进一步加强这种波动的随机性
`tc qdisc add dev eth0 root netem delay 100ms 10ms`
6. 模拟包重复
该命令将 eth0 网卡的传输设置为随机产生 1% 的重复数据包
`tc qdisc add dev eth0 root netem duplicate 1%`
7. 模拟包损坏
该命令将 eth0 网卡的传输设置为随机产生 0.2% 的损坏的数据包
`tc qdisc add dev eth0 root netem corrupt 0.2%`
8. 模拟包乱序
该命令将 eth0 网卡的传输设置为:有 25% 的数据包(50%相关)会被立即发送,其他的延迟10 秒。
`tc qdisc change dev eth0 root netem delay 10ms reorder 25% 50%`
`tc qdisc add dev eth0 root netem delay 100ms 10ms`·
9. 联合使用
`sudo tc qdisc add dev eth0 root netem delay 15ms loss 1% rate 1000Mbps`

