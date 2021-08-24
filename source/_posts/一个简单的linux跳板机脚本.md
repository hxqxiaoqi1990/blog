---
title: 一个简单的linux跳板机脚本 
tags: [shell脚本]
categories: 脚本
date: 2019-06-03 17:27:43
---

# 实现效果
前提需要设置ssh免密登录，这个跳板机脚本可以快速的切换登录服务器，通过脚本登录服务器之后，退出服务器，依然会返回跳板机菜单，因为只是个简单的脚本没有安全防护，推荐内网使用，或在跳板机脚本的服务器上设置安全策略。

# 服务器设置

 1. 上传脚本到服务器/home/test/
 2. 修改/etc/profile.d/jump.sh，这个文件是连接服务器时执行的脚本，匹配你要登录的用户UID
``` bash
#!/bin/bash
[ $UID -eq 1000 ] && /bin/bash /home/test/tbj.sh
```

# tbj.sh脚本配置
``` bash
#!/bin/bash

#kill信号，屏蔽的是HUP INT QUIT TSTP几个信号,用于防止退出操作
function traper(){
        trap '' INT QUIT TSTP TERM HUP
}

#主菜单函数
function menu(){
        echo "
================Host List==============
    1)项目1
	2)项目2
	3)有跳板机脚本服务器
	q)退出
================Host End===============
"
}
#项目1,二级目录函数
function Project1(){
        echo "
================Host List==============
    1)服务1
	2)服务2
    q)返回上一级
================Host End===============
"
}

#项目2,二级目录函数
function Project2(){
        echo "
================Host List==============
    1)服务1
	2)服务2
    q)返回上一级
================Host End===============
"
}

#登录服务器连接函数，需要设置ssh免密登录
function host(){
USER="test"
port=22
	case "$1" in
	1)
		Project1
		read -p 'Pls input your choice:' num1
		if [ $num1 == "1" ];then
			ssh $USER@172.16.136.2 -p $port
		elif [ $num1 == "2" ];then
			ssh $USER@172.16.136.3 -p $port
		else
			echo "input number"
		fi
	;;
	2)
		Project2
		read -p 'Pls input your choice:' num2
		if [ $num2 == "1" ];then
			ssh $USER@172.16.136.4 -p $port
		elif [ $num2 == "2" ];then
			ssh $USER@172.16.136.5 -p $port
		else
			echo "input number"
		fi
	;;
	3)
		exit
	;;
	q)
		export	$PPID
		kill -9 $PPID
		exit
	esac
}

#主函数
function main(){
  while true
    do
        traper
        clear
        menu
        read -p 'Pls input your choice:' num
        host $num
  done
}

#程序主入口
main
```