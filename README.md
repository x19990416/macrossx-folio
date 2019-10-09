# **FOLIO 知识图谱**
## **图书馆服务系统历史**

## **FOLIO 基本介绍**

- 概念原理

- 微服务

- FOLIO能带来什么

- FOLIO的核心组件及其功能
- - okapi
- - mod

## FOLIO 快速入门
- [环境介绍](./FOLIO快速入门/环境介绍.md)
- 快速部署FOLIO  
  FOLIO推荐采用Docker部署，通过Docker可以按需部署模块。同时也提供Vagrant的完整镜像。
  - [Vagrant部署](./FOLIO快速入门/vagrant部署.md)
  - [Docker部署](./FOLIO快速入门/docker部署.md)

## **一系列的例子——案例**

## **架构**
- Okapi

- - Okapi 与 微服务的区别
  1. OKAPI涉及两个不同模块之间的每个事务。当请求从一个模块传递到另一个模块时Okapi会成为一个GateKeeper，并在该位置确定租户是否允许当前请求，是否还可以执行其他功能。
  2. 与微服务中服务拥有自己各自的存储不同，Okapi是共享存储服务。
- - proxy  
      `/_/proxy` 配置代理服务
      
- - discovery  
`/_/discovery` 服务发现

- - deployment  
`/_/deployment` 模块部署

- - env  
`/_/env` 环境变量

- Security


### **设计目标**



## **最佳实践**

### **部署**
- [vagrant](https://app.vagrantup.com/folio)


### **示例**
  
## **相关论文**

