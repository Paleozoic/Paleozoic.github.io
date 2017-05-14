---
title: Storm之启动脚本解析
date: 2017-04-27 23:24:16
categories: [Hadoop,Storm]
tags: [Bash,Python,Storm]
---
<Excerpt in index | 首页摘要>
好奇了一下storm的命令启动过程，结果发现是通过bash调用python脚本。<!-- more -->
<The rest of contents | 余下全文>
# [storm](https://github.com/apache/storm/blob/master/bin/storm)
```bash
#!/usr/bin/env bash
#
# Licensed to the Apache Software Foundation (ASF) under one
# or more contributor license agreements.  See the NOTICE file
# distributed with this work for additional information
# regarding copyright ownership.  The ASF licenses this file
# to you under the Apache License, Version 2.0 (the
# "License"); you may not use this file except in compliance
# with the License.  You may obtain a copy of the License at
#
#     http://www.apache.org/licenses/LICENSE-2.0
#
# Unless required by applicable law or agreed to in writing, software
# distributed under the License is distributed on an "AS IS" BASIS,
# WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
# See the License for the specific language governing permissions and
# limitations under the License.
#

# Resolve links - $0 may be a softlink
PRG="${0}" #源程序名

# 这个while循环最终回到一个结果 相对路径的PRG=./storm 或者绝对路径的 PRG=/opt/storm
while [ -h "${PRG}" ]; do  # -h表示判断${PRG}文件是否存在并且是一个symbolic link (also symlink or soft link)。可以理解为快捷方式。为什么使用while？因为防止嵌套的快捷方式
  ls=`ls -ld "${PRG}"`  #   返回${PRG}的详细参数。结果类似于：lrwxrwxrwx. 1 holmes holmes 12 Apr 26 11:06 storm -> /opt/storm/bin/storm    # ls -ld  -l 参数 以详细格式列表 -d 参数 仅列指定文件。
  link=`expr "$ls" : '.*-> \(.*\)$'` #通过expr命令处理$ls变量。 # expr value : expression  通过正则提取symlink -> 后面的hard link
  if expr "$link" : '/.*' > /dev/null; then  # 如果是绝对路径，表示是 / 开头 。 表达式的结果输出到/dev/null（垃圾回收站）
    PRG="$link"
  else
    PRG=`dirname "${PRG}"`/"$link"  # dirname：获取父路径，如果是/和.则直接返回。 如果是相对路径，在前面加上 ./ 或者绝对路径（取决于真正运行bash的时候使用的是./storm 或者是/opt/storm/...）
  fi
done

# check for version
PYTHON="/usr/bin/env python"  #python环境变量

# （1）$PYTHON -V 输出python版本：Python 2.6.6
# （2）$PYTHON -V 2>&1 1指的是标准输出 2指的是错误输出 2>&1指的是将错误输出流重定向到标准输出流。（这里个人觉得可以不加上，如果命令错误了不如直接报错）
# （3）echo 'Python 2.6.6' | awk '{print $2}' 对输出的结果Python 2.6.6以空格为分隔符，打印第二个参数。这里会打印2.6.6
# （4）echo '2.6.6'| cut -d'.' -f1` 对输出结果2.6.6以.为分隔符。-d'.' 定义分隔符号为. -f1 表示取第一个分割到的字符串
majversion=`$PYTHON -V 2>&1 | awk '{print $2}' | cut -d'.' -f1` #输出python大版本，比如2  
minversion=`$PYTHON -V 2>&1 | awk '{print $2}' | cut -d'.' -f2` #输出python小版本，比如6
numversion=$(( 10 * $majversion + $minversion)) # python 2.6版本会输出：2*10+6 = 26
if (( $numversion < 26 )); then
  echo "Need python version > 2.6"
  exit 1
fi

STORM_BIN_DIR=`dirname ${PRG}` #获取父路径 即/opt/storm/bin/
export STORM_BASE_DIR=`cd ${STORM_BIN_DIR}/..;pwd` # 返回上一级，即/opt/storm/，pwd输出当前目录路径

#check to see if the conf dir or file is given as an optional argument
if [ $# -gt 1 ]; then # 如果参数个数大于1。 $#表示此脚本传入的参数个数
  if [ "--config" = "$1" ]; then # 如果第一个参数是--config则取第二个参数（--config接着的参数）作为conf_file
    conf_file=$2
    if [ -d "$conf_file" ]; then # 如果conf_file目录存在，conf_file设置为$conf_file/storm.yaml
      conf_file=$conf_file/storm.yaml
    fi
    if [ ! -f "$conf_file" ]; then # 如果storm.yaml不存在，则抛出异常
      echo "Error: Cannot find configuration file: $conf_file"
      exit 1
    fi
    STORM_CONF_FILE=$conf_file #storm.yaml目录 即/opt/storm/conf/storm.yaml
    STORM_CONF_DIR=`dirname $conf_file` #获取配置文件目录 即/opt/storm/conf/
  fi
fi

# export 的变量被所有的子shell共享（父子shell的变量传递）
# :-  例子：x=${1:-y}表示 如果${1}存在且不空则x=${1}，否则x=y
export STORM_CONF_DIR="${STORM_CONF_DIR:-$STORM_BASE_DIR/conf}"
export STORM_CONF_FILE="${STORM_CONF_FILE:-$STORM_BASE_DIR/conf/storm.yaml}"

if [ -f "${STORM_CONF_DIR}/storm-env.sh" ]; then # 如果storm-env.sh存在，则行改文件。
  . "${STORM_CONF_DIR}/storm-env.sh"
fi

exec "${STORM_BIN_DIR}/storm.py" "$@" #调用storm.py的Python脚本  exec执行脚本  $@表示：storm脚本传入的所有参数。例子：storm a b c  转为storm.py "a" "b" "c"
```
# [storm.py](https://github.com/apache/storm/blob/master/bin/storm.py)
```python
# 待解析
```
