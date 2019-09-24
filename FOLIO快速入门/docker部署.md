> - https://hub.docker.com/u/folioorg/  
> - https://hub.docker.com/r/folioci/  
> - https://github.com/folio-org/okapi/tree/master/okapi-core


folio的docker镜像可以从 https://hub.docker.com/u/folioorg/  此处获取。

在Ubuntu上以Docker方式部署需要参照以下步骤： 

1. 安装Docker  

安装docker必要工具
  ```
  sudo apt-get update
  sudo apt-get -y install apt-transport-https ca-certificates curl software-properties-common
  ```
   安装GPG证书
  ```
  curl -fsSL http://mirrors.aliyun.com/docker-ce/linux/ubuntu/gpg | sudo apt-key add -  
   ```
  写入软件源信息
  ```
  sudo add-apt-repository "deb [arch=amd64] http://mirrors.aliyun.com/docker-ce/linux/ubuntu $(lsb_release -cs) stable"
  ```
   更新并安装Docker-CE
  ```
  sudo apt-get -y update
  sudo apt-get -y install docker-ce
  ```
2. 安装Okapi

  获取Okapi 镜像
  ```
  sudo docker pull folioorg/okapi
  ```
  查看获取的镜像
  ```
  sudo docker images
  ```
  ![github](./images/docker_1.png)  
  运行镜像，此处以集群方式运行(cluster)可以使用dev模式
  ```
  sudo docker run -p 9130:9130 -e JAVA_OPTIONS="-Dloglevel=DEBUG" folioorg/okapi cluster
  ```
  查看镜像运行状态
  ```
  sudo docker ps -a
  ```
   ![github](./images/docker_2.png)
   通过了浏览器访问 
   ```
   http://127.0.0.1:9130
    
    会提示如下信息：
    
    No suitable module found for path / for tenant supertenant
    
    表明Okapi 已经运行。
   
   ```
3. 安装其他模块
  
  
