---
title: Shell脚本：kill应用示例
date: 2017-07-12 10:07:41
categories: [Shell]
tags: [Shell]
---
<Excerpt in index | 首页摘要>
Shell脚本：kill应用示例<!-- more -->
<The rest of contents | 余下全文>

# 根据进程名杀死某个进程

```bash
kill -9 `ps -ef|grep "processname" | grep -v "grep"|awk '{print   $2}'`
```

# 根据进程名批量杀死进程

```bash
# !/bin/sh
for pid in $(ps -ef | grep "curl" | grep -v grep | cut -c 15-20); do    #（获取进程名含有curl进程id数组，并循环杀死所有进程）
    echo $pid
    kill -9 $pid
done
```

# 根据进程名数组批量杀死进程

```bash
# !/bin/sh
KILL_PREFIX="/opt/xxx/basApp/"
KILL_APPS=("app1.jar" "app2.jar" "app3.jar")
for pname in ${KILL_APPS[@]}; do
    pid=`ps -ef|grep "$KILL_PREFIX$pname" | grep -v "grep"|awk '{print   $2}'`
    echo $pname":"$pid
    if [ $pid ]; then
    	kill -9 $pid
    fi
done
```

# 附：后台启动

```bash
nohup java -jar /xxx/xxx.jar >/dev/null 2>&1 &
```





# 引用

[Linux kill 杀死指定进程](http://blog.csdn.net/ithomer/article/details/6817318)