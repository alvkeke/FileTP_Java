# File-Transfer-Java

If you need a README in English, please refer to [README_EN](./README_EN.md)

## 概述

这是新版的文件传输工具，此工具仍然分为前后端，此为后端程序。

前端工具： 
* [Linux平台命令行工具](https://github.com/alvkeke/LAN-File-Transfer-Frontend)
* [通用Java图形前端](https://github.com/alvkeke/LAN-File-Transfer-Frontend-UI)

## 使用方法

本程序目前可以通过配置文件设置端口/设备名等属性, 当没有指定配置文件时，程序会寻找默认的配置文件`config`，该文件位于程序目录下。 

如果需要使用另外的配置文件，通过命令行参数`-c <conf>`进行指定。

配置文件通过`key = value`的结构组织, 允许使用注释,注释行以`#`为开头

目前可用的属性字段为:
* `recv port`  : 接收模块监听的端口(TCP)
* `scan port` : 扫描模块使用的端口(UDP)
* `ctrl port` : 控制模块使用的端口(TCP)
* `device name` : 设备名称
* `recv path` : 接受文件的保存路径

## 程序结构

此程序将具体功能模块化分割，分别分割为：

* 发送模块
* 接收模块
* 发现模块
* 控制模块

这几个模块分别完成其功能，组合起来实现文件传输后端的功能。本程序中，这些模块都是Thread类的子类。

### 发送模块

1. 通过一个`BlockingQueue`完成线程调度，确保线程不会空转。
2. `BlockingQueue`中存放发送任务，任务中包含：
   1. 文件路径（本地）
   2. 目标地址与端口
3. 每一个文件建立一个TCP短链接，完成传输后关闭。

### 接收模块

接受模块时刻监听一个端口，该端口可以指定。发送方建立连接时对接到此端口。 结构为正常的TCP多线程服务端。

### 扫描模块

此模块用来扫描局域网内的合法设备，可以得到局域网中响应的所有设备的

* IP地址
* 设备名称
* 接收TCP端口

**1. 此模块运行在UDP协议下，从而完成对局域网中设备的广播发现。**

**2. 端口统一指定，不可修改**

### 控制模块

此模块用来完成程序前、后端交互的，配置与发送控制都通过此模块。

## 协议

### 传输协议

下表为文件传输时使用的数据格式，表中出现的长度字段都使用小端序进行传输。

此应用使用TCP完成数据传输，所以可靠性由TCP协议保证。

| 字段 | 长度 | 说明 |
|:---:|:---:|:---|
| datalen | 8 bytes | 文件长度 |
| namelen | 4 bytes | 文件名长度 |
| name | n1 | 文件名字段，长度由`namelen`字段指定 |
| data | n2 | 文件数据字段，长度由`datalen`字段指定 |


### 控制协议

控制命令采用文本格式，每一个换行为命令的结束。下为目前可用命令：

* `scan`
    * 扫描局域网中可用设备，此命令不返回设备信息
    * 返回值为`"success"`或`"failed"`
* `get <data>`
    * 从服务中获取数据,当前可选`data`项为:
        * `dev-ol`: 获取局域网中在线设备，返回值格式为：
            * `Name1 IP_Addr1 Port1|Name2 IP_Addr2 Port2|...`
            * 每一个设备的条目拥有 设备名称(Name)、IP地址(IP_Addr)、端口(Port) 三项数据
            * 一个条目的的多个数据通过空格进行分割，多个条目由换行符进行分割
        * `save-path`: 文件接收路径
            * 返回值为一个字符串
* `set <conf> <value>`
    * 设置`conf`为`value`，当前可用`conf`项为：
        * `save-path`: 接收文件保存路径
    * 有返回值，`"success"` or `"failed"`
* `send-addr <ip> <port> <file-path>`
    * 通过`IP`与`Port`指定设备，并向其发送文件
    * `ip`: 目标设备IP地址
    * `port`: 目标设备端口
    * `full-path`: 待发送文件绝对路径
* `send <dev-name> <file-path>`
    * 通过设备名发送文件，设备名对应的设备需要已经被扫描到
    * `dev-name`: 设备名称，用于在*在线列表*中查找设备
    * `file-path`: 待发送文件绝对路径
* `exit`
    * 断开控制连接
    * 无返回值

### 扫描协议

此小节描述用于发现局域网中合法设备的协议。

此协议运行在UDP协议下，一个完整的协议通信流程如下：

1. 请求方向局域网中发起广播，携带数据为：`"SCAN"`
2. 局域网中的合法设备接收到此数据包后，向发起设备返回：
   1. 接收模块所使用的端口[4字节],**小端序**
   2. 设备名称[最大36字节]
   3. 不需要特定传输自身地址，UDP协议使得发起方可以得到IP
3. 局域网中的所有设备都完成（2）中的步骤，流程结束