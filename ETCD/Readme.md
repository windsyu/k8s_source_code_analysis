# Abstract

kubernetes的所有资源都是存储在etcd cluster中的，它是一个典型的kv数据库，提供存储查询，更新，监控对象变化的watch等操作。

此处重点介绍K8s中与etcd进行交互的数据访问层DAO接口以及实现的介绍。