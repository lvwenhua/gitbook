# flume笔记

### 1.Flume架构

![1572233145494](C:\Users\lv\AppData\Roaming\Typora\typora-user-images\1572233145494.png)

### 2.使用

官方文档地址：http://flume.apache.org/releases/content/1.7.0/FlumeUserGuide.html

source - channel - sink

* source 是负责接收数据到Flume Agent的组件

* sink不断轮询Channel中的事件并批量移除，并将这些事件批量写入到存储或索引系统、或被发送到另一个Flume Agent

* Channel：位于source和sink间的缓冲区。Flume自带两种Channel：Memory Channel和File Channel以及Kafka Channel

  