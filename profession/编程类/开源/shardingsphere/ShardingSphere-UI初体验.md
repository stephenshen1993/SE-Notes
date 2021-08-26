# ShardingSphere-UI 初体验

------

## 简介

ShardingSphere-UI 是 ShardingSphere 的一个简单而有用的web管理控制台。它用于帮助用户更简单的使用 ShardingSphere 的相关功能，目前提供注册中心管理、动态配置管理、数据库编排等功能。

本篇将对 ShardingSphere-UI 进行简单探索，运行 UI 控制台，体验最基础的功能。



## 准备工作

探索之前，先来看看官方文档：

- [ShardingSphere-UI](https://shardingsphere.apache.org/document/current/cn/user-manual/shardingsphere-ui/)

ShardingSphere-UI 是 ShardingSphere 的 web 管理控制台，通过注册中心实现对 ShardingSphere 配置的管理和运行状态监控。

所以在使用 ShardingSphere-UI 前，需要先启动 ShardingSphere 和 注册中心：

- ShardingSphere：此处使用 ShardingSphere-Proxy，启动过程参考前一篇博文，此处不再赘述；

- 注册中心：此处使用 Zookeeper，网上使用文档较多，启动过程此处不再赘述；



## 部署运行

1、获取安装包：https://mirrors.tuna.tsinghua.edu.cn/apache/shardingsphere/shardingsphere-ui-5.0.0-alpha/apache-shardingsphere-5.0.0-alpha-shardingsphere-ui-bin.tar.gz

2、解压后运行 bin/start.sh



## 功能体验

访问 http://localhost:8088/，进去默认就填好了用户名和密码，点击登录即可。

左侧面板显示，目前 ShardingSphere-UI 功能树如下：

- 分布式治理
  - 注册中心
  - 配置管理
  - 运行状态
- 数据扩容



选择注册中心，在页面上添加一个注册中心，完成后点击激活，页面提示已经连接了注册中心。