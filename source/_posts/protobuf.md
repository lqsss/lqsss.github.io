---
title: ProtoBuf生成
tag: ProtoBuf
categories: tools
---

ProtoBuf如何生成相关的java类
<!--more-->


## 环境准备
1. 下载Protocol Buffers
[Protocol Buffers v3.5.1](https://github.com/google/protobuf/releases/tag/v3.5.1)

2. 下载protoc.exe
[protoc.exe](http://central.maven.org/maven2/com/google/protobuf/protoc/3.5.1/)

3. 将protoc.exe放在刚刚下载Protocol Buffers的src文件夹下
D:\ChromeDownload\protobuf-java-3.5.1\protobuf-3.5.1\src

4. 配置环境变量
Path： ...;D:\ChromeDownload\protobuf-java-3.5.1\protobuf-3.5.1\src

## 测试
1. BrokerMessage.proto文件
```
syntax = "proto3";

option java_outer_classname = "BrokerMessage";  //对外输出的Class名

option java_package = "com.cug.liqiushi.client.vo";  //指定包名
```

2. cmd生成对应.java文件

```
C:\Users\lqs>protoc -I E:\IdeaProjects\BRMQ\client\src\main\resources --java_out
=E:\IdeaProjects\BRMQ\client\src\main\java E:\IdeaProjects\BRMQ\client\src\main\resources\BrokerMessage.proto
```

rule: 
1. protoc -I 输入文件(包含.proto的文件夹) 
2. --java out= 目标文件夹
3. .proto文件的路径

![demo](http://op7scj9he.bkt.clouddn.com/_%7BJY%5D_4%7BR0$6LTXFC74580K.png)