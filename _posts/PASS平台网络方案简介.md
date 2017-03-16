title: PAAS平台网络方案简介
date: 2016-05-24 12:44:10
tags:
 - ovs
categories: devops

---
![Alt text](https://s3.jaywaychou.com/images/vlan.png)

> 1.交换机预设vlan给租户使用
> 2.交换机接宿主机的port 都设置成 ` trunk ` 模式
> 3.把可用的`vlan `提前注册到`etcd`服务
> 4.当前租户创建`pod`时，从`etcd`里请求获取未分使用的`IP`和`vlan`
> 5.通过`pipework`把网络配置给`pod`的所属的容器
> 6.删除`pod`时，向`etcd` 发送POST请求，释放占用的`IP`和`vlan`


**注意**
    > *1. 如果`pod`的容器被分配到同一主机上，容器之间共享网络栈，只需要分配一个`IP`*
    > *2. 整个平台使用一个大网段，`IP`和`vlan`通过标志位来实别使用状态*
    

## 后记
近日对之前的笔记做了整理，希望对以后有所帮助 --- 2017.3.16 执笔



