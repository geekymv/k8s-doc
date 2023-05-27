```md
tcpdump -nn -i enp0s8 port 8080

Flags [S]
Flags [S.]
Flags [.]

[S] SYN 开始连接
[P] PSH 推送数据
[F] FIN 结束连接
[R] RST 重置连接
[.] 没有 Flag, 代表除上面4种类型外的其他情况，可能是 ACK(确认标志) 也有可能是 URG(紧急标志)

man tcpdump
tcpdump -h 查看帮助

-i 指定监听网络接口，默认监听在第一块网卡上
tcpdump -i any 监听所有的网卡接口

tcpdump -i eth0 -w pack.pcap
-w 将捕获到的信息保存到文件中，且不分析和打印在屏幕
导出的文件可以设置为 cap 或者 pcap 的格式，可以用 wireshark 工具打开

-n 不把IP转换成域名，直接显示IP，避免执行 DNS lookups 的过程，速度会快很多
-nn 不把协议和端口转换成名字，速度也会快很多

-t 在每行的输出中不输出时间
-tttt 在每行打印的时间戳之前添加日期，这种输出的时间最直观

-v、-vv、-vvv 产生详细的输出

-c 指定收取数据报的次数，在收到指定数量的数据包之后退出tcpdump，停止抓包
tcpdump -c 20 -w pack.pcap

-A 以ASCII格式打印出所有的分组，并且读取此文件，可以方便使用 grep 等工具解析出内容
tcpdump -A | grep baidu

逻辑运算符
and or not
tcpdump src 192.168.56.102 and port 8080 抓取来自主机 192.168.56.101 端口 8080 的包

多个过滤器进行组合，需要用到()，括号在shell 中是特殊符号，需要使用引号进行包含
tcpdump "src 192.168.56.10 and (dst port 3389 or 22)"

route -n 查看路由转发表
判断我们要抓取的网卡

```