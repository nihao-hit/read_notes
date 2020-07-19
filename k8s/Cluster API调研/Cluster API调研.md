# Cluster API调研

[TOC]

## 1 架构图

![Cluster API架构图](./images/Cluster API架构图.png)

## 2 可借鉴设计

通过Ref与Owner属性将通用Cluster API对象与特定Infrastructure资源对象解耦。不同Controller各自监听自己关注的API对象上的更新事件，并更新自己创建的API对象，来完成Controller之间的交互。

