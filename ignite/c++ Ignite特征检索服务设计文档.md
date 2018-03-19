# C++ Ignite特征检索服务设计文档

标签（空格分隔）： Ignite

*@王振 @李坤*

[TOC]

## 原特征检索服务业务逻辑

![](https://raw.githubusercontent.com/Nukil/Bimg/master/images/netposa/ignite/TIM%E6%88%AA%E5%9B%BE20171221180214.png)

图中所有操作均由`Java`完成。

## C++版本服务业务逻辑

![](https://raw.githubusercontent.com/Nukil/Bimg/master/images/netposa/ignite/TIM%E6%88%AA%E5%9B%BE20171221180249.png)

其中 **红色** 标注部分需要由 `C++` 完成，**蓝色** 部分依旧由 `Java` 完成。

### 风险项

- 数据依旧由`Java`接入，`C++`如果要使用这些数据，需要将`Java`接入并存储在`Ignite`里的数据结构改成一种跨平台的数据结构，`Ignite`提供了一种数据序列化方式来达到该目的，验证后有效，但是针对大规模数据使用时会存在什么问题暂时无从得知。
- 调研到`C++ Ignite`需要通过广播的方式分发（Map）计算任务，但是无法直接拿到并汇聚（Reduce）结果，目前的解决方案是所有计算节点将计算完的结果存入一块`cache`，客户端读取这一`cache`里的数据进行结果汇聚。但是`Ignite`各节点存数据时存在网络传输，所以该解决方案可能存在性能问题。
- `Ignite`在进行计算时，可能会牵扯到`HBase`里的冷数据，这时可能需要使用`C++`从`HBase`里读取数据，如图中最下方的 **红色箭头** 所示，目前使用`C++`操作`HBase`的方法还未开始调研，该条列为严重风险，建议第一期服务内部对冷数据不作处理，只关注热数据。