---
layout:     post                    # 使用的布局（不需要改）
title:      linux服务器javaWeb项目转pdf后汉字不显示      # 标题 
subtitle:   word转pdf后汉字不显示 					#副标题
date:       2020-06-14              # 时间
author:     BY tang                     # 作者
header-img: img/post-bg-2015.jpg    #这篇文章标题背景图片
catalog: true                       # 是否归档
tags:                               #标签
    - linux
---

# linux服务器javaWeb项目转pdf后汉字不显示 #
> 环境：centOS7，java8，tomcat8
> 
> 转换工具：aspose.word 
> 
> 症状：系统转换出的pdf里汉字不显示，数字、字母正常 
> 
> 解决办法：centOS7安装汉字字体


1，在windows上的c:/windows/font目录中拷贝 宋体、黑体（文件名分别为simsun.ttc、simhei.ttf）到linux服务器； 

2，检查linux服务器/usr/share目录下存在“fonts”目录，如果不存在执行升级： 
    
	yum -y install fontconfig

3，如果无法执行则需要更新linux 

	yum update

4，将2个字体文件放入/usr/share/fonts目录中 

5，对字体文件进行授权

	chmod -R 755 /usr/share/fonts/sim* 

6，安装字体检索工具ttmkfdir并执行命令 
	
	yum -y install ttmkfdir ttmkfdir -e /usr/share/X11/fonts/encodings/encodings.dir

7，重载linux字体库 fc-cache

8，查看是否已载入宋体和黑体 fc-list

9，重启tomcat，测试，成功！

