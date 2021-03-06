# GRPC接口服务

[GRPC](https://grpc.io/)正如其名,是目前应用较广的一种RPC(Remote Procedure Call)协议.

## RPC

RPC(Remote Procedure Call),远程过程调用.它的设计目标就是希望调用起来和使用本地的函数一样简单.也就是说RPC只是一种形式,本质上还是一种请求响应模式的服务.

RPC技术由来已久,也很早就被应用,比如jsonrpc,xmlrpc这些.但多数RPC技术也有一些顽疾:

1. 语言绑定

   RPC技术既然是让远程调用像本地调用一样使用的技术,那必然会和”本地调用”的语言绑定.几乎没有一个RPC技术可以说是语言无关的,只是支持的语言多少的问题.

2. 本地调用和远程调用并不相同

   RPC的核心想法是隐藏远程调用的复杂性,但是很多RPC的实现隐藏得有些过头了,而会造成一些问题.

   1. 使用本地调用不会引起性能问题,但是RPC会花大量的时间对负荷进行序列化和反序列化,更别提网络通信所需要的时间.这意味着要使用不同的思路来设计远程和本地的API(需要考虑rpc的服务端是否可用等).简单地把一个本地的API改造成为跨服务的远程API往往会带来问题.
   2. 由于隐藏的太好开发人员会在不知道该调用是远程调用的情况下对其进行使用.这会进一步放大上面的问题
   3. 数据冗余,如果一个函数它的返回值有数据冗余,我们往往还是可以到处复用的,因为除了一点内存外几乎没有成本,但RPC不同它的返回和函数一样也是固定的,所以冗余数据会沿着网络整个传过来,这就造成了序列化反序列化时cpu内存的浪费以及网络带宽的浪费.成本就高了

上面说这么多当然不是说RPC不好,恰恰相反,RPC是非常好用的工具,在拆分服务时改动最小,而且也最好维护.所有大厂都会使用RPC技术,很多开源项目如tensorflow都会有rpc的应用.

比较常用的关于grpc的资料集合维护在[awesome-grpc](https://github.com/grpc-ecosystem/awesome-grpc)项目.

## GRPC

GRPC是谷歌开源出来的一个RPC协议,目前官方有3大类实现

1. C实现
2. golang实现
3. java实现

而C实现又绑定了大量其他语言的接口,比如python,node,C#,C++等.从使用的技术上来说它使用HTTP2协议作为传输协议,protobuf作为序列化协议.加上社区一直以来的优化,总体而言GRPC在各种RPC协议中性能属于上游水平.但与其说GRPC是一个RPC技术,不如说它是一套解决方案.GRPC官方实现了一整套包括反射,健康检测,负载均衡和服务发现,debug性能调优等在内的工具和模块,无比庞大无比复杂.老实说想都整明白还是有点费劲的.本文的目的就是把GRPC和与其配套的技术整体理一理,并给出一个相对通用的使用模板来.

不过由于我不会java,所以本文只会介绍C实现(以python为例)和Go实现.C实现绑定的语言过多,就不一一介绍了,我用到了会进行补充.

## 基本使用

GRPC的基本使用流程是:

1. 服务端与客户端开发者协商创建一个protobuf文件用于定义rpc的形式和方法名以及不同方法传输数据的schema
2. 客户端和服务端分别编译这个protobuf文件为自己需要的目标语言的模块
3. 服务端:
   1. 导入protobuf文件编译成的模块
   2. 继承定义service服务类(go语言是自己创建类)
   3. 实现service服务类其中定义的方法
   4. 这个类的一个实例注册到grpc服务器
   5. 启动grpc服务器提供服务
4. 客户端:
   1. 导入protobuf文件编译成的模块
   2. 创建通讯连接(C实现中叫`Channel`,go实现中叫`ClientConn`)
   3. 使用这个通讯连接实例化protobuf文件编译成的模块中的service客户端类
   4. 用这个service客户端类的实例发送请求获得响应

一些C实现绑定的编程语言比如js或者python可以直接加载protobuf文件构造对象用于使用,这可以很大程度上为原型设计提供帮助.

一个典型的rpc定义proto文件如下:

```go
syntax = "proto3";
package test.foo;
option go_package = "./foo";

service Bar {
    rpc Square (Message) returns (Message){}
    rpc RangeSquare (Message) returns (stream Message){}
    rpc SumSquare (stream Message) returns (Message){}
    rpc StreamrangeSquare (stream Message) returns (stream Message){}
}
message Message {
    double Message = 1;
}
```

可以看出grpc的声明实际上时protobuf语法的一个扩展,新增了如下关键字来描述一个grpc服务

| 关键字    | 说明                                       |
| :-------- | :----------------------------------------- |
| `service` | 申明定义的是一个grpc的Service              |
| `rpc`     | 申明这一行定义的是服务下的一个远程调用方法 |
| `returns` | 声明本行定义的`rpc`的返回值形式            |
| `stream`  | 声明这个数据是个流数据                     |

