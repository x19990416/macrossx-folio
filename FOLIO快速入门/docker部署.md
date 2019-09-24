> - https://hub.docker.com/u/folioorg/ 发布版
> - https://hub.docker.com/r/folioci/ 快照版


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
  
